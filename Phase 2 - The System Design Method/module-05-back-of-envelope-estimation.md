# Module 5 — Back-of-Envelope Estimation: The Numbers You Need Cold, and How to Use Them Under Pressure
*Phase 2: The System Design Method · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Module 3 placed estimation as step 2 of the framework and gave it a strict rule: *every number must end with "so…"*. Module 4 showed where the inputs come from — the users, actions, peaks, sizes and retention periods you elicited or assumed in the first five minutes. This module is about the arithmetic in between, and about the reference numbers that make the arithmetic mean something.

The central claim of this module, which we will derive rather than assert:

> **An estimate is a comparison. A number on its own says nothing; a number set against a known capacity — one machine, one partition, one network link, one budget — tells you which design you need.**

"We need 900,000 writes per second" is trivia. "We need 900,000 writes per second, and one Cosmos DB physical partition gives 10,000 RU/s, so we're looking at hundreds of partitions and a quarter of a million dollars a month for throughput alone" is a design decision. The difference is the second half of the sentence — the reference number — and that is why this module spends as much time on *what one thing can do* as on *how to multiply*.

Seniority shows up here in three ways. First, **fluency**: you convert DAU to QPS, bytes to petabytes and milliseconds to round trips without visible effort, so the interviewer's attention stays on the design. Second, **calibration**: your reference numbers are current. One of the easiest ways to look like you learned system design from a 2015 blog post is to treat a 100 GB database as "big" or an SSD read as 1 ms. Third, **judgment**: you know which estimates matter, you do only those, and you're comfortable concluding "this is not a scale problem."

We build the material one concept at a time:

- **Part A** — what estimation is for, why order of magnitude is enough, and how to decompose an unknown.
- **Part B** — the arithmetic toolkit: powers, units, time constants, mental tricks, sanity checks, averages versus peaks.
- **Part C** — the numbers to know cold: the latency ladder, the physics of distance, protocol costs, bandwidths, what one box and one managed service can do, data sizes, failure rates.
- **Part D** — the recipes: traffic, storage, bandwidth, memory, concurrency, fleet size, partition count, latency budgets, cost.
- **Part E** — using numbers under pressure: the board, the narration, decision thresholds, traps, sensitivity analysis, calibration by level.
- **Part F** — the architect variant, measuring your own numbers in .NET and Azure, and .NET-specific memory arithmetic.
- **Part G** — seven worked examples, from a URL shortener to an insurer whose "scale problem" turns out not to be one.

| # | Concept | The one-line takeaway |
|---|---|---|
| **Part A — What estimation is for** | | |
| 1 | Estimates are comparisons | A number earns its place only by being compared with a capacity and changing a decision. |
| 2 | Order of magnitude is enough | Design thresholds are ~10× apart, so ~2× error rarely changes the answer. |
| 3 | Fermi decomposition | Break the unknown into factors you can bound; errors partially cancel. |
| 4 | When and how much to estimate | 2–4 minutes, 3–5 numbers, upfront or inline — never decoration. |
| **Part B — The arithmetic toolkit** | | |
| 5 | Powers of two and ten, bits and bytes | 2¹⁰ ≈ 10³; network is in bits; KiB vs KB drifts 10% by TiB. |
| 6 | Time constants and rate conversion | A day ≈ 10⁵ s; 1M/day ≈ 12/s; a year ≈ π × 10⁷ s. |
| 7 | Mental arithmetic | One significant figure, mantissas × mantissas, exponents + exponents. |
| 8 | Units and sanity checks | Carry units; cross-check top-down against bottom-up and per-user. |
| 9 | Averages, peaks and skew | Provision for peak; size for the hottest key; seconds hide bursts. |
| **Part C — The numbers to know cold** | | |
| 10 | The latency ladder (2026) | ns for CPU and memory, µs for NVMe and the network card, ms for networks and disks. |
| 11 | The physics of distance | ~1 ms of round trip per 100 km of fibre; measured Azure region pairs. |
| 12 | Protocol costs | Cold HTTPS costs 2–3 round trips before your first byte; windows cap throughput. |
| 13 | Bandwidth and throughput | Memory hundreds of GB/s, NVMe ~10 GB/s, NIC tens of Gbps, HDD ~200 MB/s. |
| 14 | What one thing can do | Per-core request rates, a relational primary, a cache shard, Azure service limits. |
| 15 | Data sizes | Primitives, .NET object overheads, records, media bitrates, compression ratios. |
| 16 | Failure rates at scale | 1–2% disk AFR means daily failures in a fleet. |
| **Part D — The recipes** | | |
| 17 | Traffic | DAU → actions → average → peak → read/write → fan-out → amplification. |
| 18 | Storage | Size × count × retention × replication × overhead × headroom. |
| 19 | Bandwidth | Payload × rate, in bits, split ingress/egress/east-west. |
| 20 | Memory and cache sizing | Working set, skew, hit rate → backend load. |
| 21 | Concurrency (Little's Law) | L = λ × W sizes threads, pools, connections and ports. |
| 22 | Fleet size and headroom | Measured capacity ÷ target utilization, plus zone loss and deploys. |
| 23 | Partition and shard count | The maximum of throughput- and storage-driven counts; then check the hottest key. |
| 24 | Latency budgets | Serial sums, parallel maxima, tails and retries. |
| 25 | Cost | Unit cost; compute, storage, throughput and egress as order-of-magnitude rates. |
| **Part E — Under pressure** | | |
| 26 | The estimation block on the board | Assumptions labelled, results rounded, each with its "so". |
| 27 | Narration | Say the formula, round out loud, invite correction. |
| 28 | Thresholds that turn numbers into decisions | A table of 2026 heuristics. |
| 29 | Traps | Bits/bytes, averages, replication, amplification, stale hardware numbers. |
| 30 | Sensitivity and calibration by level | Find the assumption that flips the decision; know what each level sounds like. |
| **Part F — Architect variant and .NET** | | |
| 31 | Capacity planning and cost models | Ranges, runways, scenarios — estimates that survive a budget meeting. |
| 32 | Measuring your own numbers | BenchmarkDotNet, load tests, RU charges, VM latency tests. |
| 33 | .NET memory arithmetic | Object headers, strings, dictionaries, the LOH — what 10 million objects cost. |
| **Part G — Worked examples** | | |
| 34 | URL shortener | Storage is trivial; the read path and the ID space are not. |
| 35 | Chat at WhatsApp scale | Heartbeats outnumber messages; history storage is the cost problem. |
| 36 | News feed | Fan-out is the real write volume; celebrities break push. |
| 37 | Video platform | Egress economics decide the architecture. |
| 38 | Metrics platform | Cardinality, not hosts, is the scale. |
| 39 | Flash-sale ticketing | The write is small; admission is the system. |
| 40 | Architect: an insurer's storm surge | The estimate proves it's an integration problem, not a scale problem. |

---

# Part A — What estimation is for

## Concept 1 — Estimates are comparisons

Start from what you're trying to do in a design interview: choose a structure. Every structural choice has a **capacity** — how much one instance of it can handle — and a **cost**. A single relational primary can absorb some number of durable writes per second. One cache node holds some number of gigabytes. One physical partition of a managed database gives some throughput. One region is some number of milliseconds from your users.

An estimate is the other side of that comparison: *how much of the thing will we need?* The decision falls out of the ratio:

> **needed ÷ capacity-per-unit = how many units, or whether one unit is enough**

When the ratio is well below 1, the simple design is right and the estimate has *saved* you from over-engineering. When it's in the tens or hundreds, you need partitioning, replication or a different technology. When it's in the millions, you're probably looking at the wrong unit — you need a different architecture entirely, not more of the same thing.

That's why "so it's a lot" is the weakest possible conclusion. Interviewers don't learn anything from it, because "a lot" doesn't distinguish between one big server, ten shards and a dedicated edge tier. Compare these two statements:

| Weak | Strong |
|---|---|
| "Peak is about 5,000 writes per second, which is a lot." | "Peak is about 5,000 writes a second. A single Azure SQL primary on a decent tier handles that with room to spare, so I won't shard — I'll put the depth into contention on hot rows instead." |
| "We'll store about 2 PB a year." | "About 2 PB a year. Keeping that in Cosmos DB costs roughly half a million dollars a month in storage alone, against tens of thousands in Blob Storage — so recent history in the database, older history tiered out." |

The strong versions have three parts: **the number, the reference capacity, the decision.** Module 3's rule — every number ends with "so…" — is that third part. This module adds the second.

**The most valuable estimate often proves that something doesn't matter.** If storage growth is 300 GB a year, saying so — and then *not* spending deep-dive time on storage — is a judgment signal. The interviewer sees you choosing where to spend the session.

---

## Concept 2 — Order of magnitude is enough

Candidates sometimes freeze because they don't know whether a user sends 30 or 50 messages a day. Here's why it almost never matters.

**Design thresholds are roughly an order of magnitude apart.** The boundaries that change your architecture — "fits on one machine," "needs a handful of shards," "needs hundreds," "needs a dedicated tier" — sit at factors of ten or more from each other. An estimate that's off by 1.5× or even 2× almost never moves you across one of those boundaries. An estimate that's off by 10× might.

**So the target precision is "within a factor of 2–3, and the right power of ten."** That's what *back-of-envelope* means. More precision costs time and buys nothing, because your inputs (users, actions per user, sizes) are themselves guesses with that much uncertainty.

**Think in logarithms.** It helps to treat each estimate as an exponent with a little noise: "about 10⁵ requests per second, give or take a factor of 2." Errors multiply, so in log space they add. And here's the useful part: **independent errors partially cancel.** If you multiply four factors, each uncertain by a factor of 2, the result is not usually off by 2⁴ = 16×; in log space the errors add like random steps, so the typical combined error grows like the square root of the number of factors — roughly 2^√4 = 4× in this case. That's why Fermi estimates are often surprisingly accurate: some factors are high, some low.

The caveat: cancellation only works if the errors are **independent and unbiased**. If you systematically overestimate every factor (a common habit — candidates like big numbers), errors compound rather than cancel. Prefer honest central estimates over "safe" high ones, and add headroom *once*, explicitly, at the end.

**When precision does matter.** Two situations deserve more care:

1. **You're near a threshold.** If the estimate lands at 8,000 writes a second and you think one primary handles about 10,000, a 2× error decides the design. That's the moment to state the uncertainty and either ask, or design an evolution path ("single primary now; partition key chosen so we can split later").
2. **Money at scale.** A 2× error on a ten-thousand-dollar bill doesn't matter; a 2× error on a ten-million-dollar one does. Architects presenting cost models (Concept 31) give ranges and sensitivities, not single numbers.

---

## Concept 3 — Fermi decomposition

Enrico Fermi famously estimated the number of piano tuners in Chicago by chaining together quantities he could guess: population, households, pianos per household, tunings per year, tunings one tuner can do. The method — now called a **Fermi estimate** — is exactly what system design estimation is.

**The method:**

1. **Write the target as a product (or sum) of factors** you can reason about independently.
2. **Bound each factor**: what's the lowest plausible value, and the highest?
3. **Take a central value** for each — for multiplicative uncertainty, the *geometric* mean of the bounds is the right centre (if a user sends between 10 and 100 messages a day, use ~30, not 55).
4. **Multiply, round, and sanity-check** against something you know.

**Example: how many chat messages per second at peak?**

```
messages/s (peak) = DAU × messages per user per day ÷ seconds per day × peak factor
                  = 5×10⁸ × 50 ÷ 10⁵ × 3
                  ≈ 7.5×10⁵  (≈ 0.9M with the exact 86,400 s/day)
```

Each factor is bounded and defensible: DAU from the interviewer or a stated assumption; 50 messages a day sits between "a few" and "a few hundred"; peak factor 3 is a typical daily cycle for a global consumer app.

**Two directions of decomposition, and why you should sometimes use both.**

- **Top-down** starts from population and behaviour: users × actions × size. It's what interviews usually need.
- **Bottom-up** starts from physical capacity: "a server holds about 100,000 connections; how many servers would we need?" or "the payment provider allows 50 requests a second; what does that cap?"

When the two disagree by more than a small factor, one of your assumptions is wrong — and finding which one is usually more valuable than the estimate itself. In real capacity planning this triangulation is standard practice.

**Bounding as a substitute for knowing.** When you have no idea, bound instead of guessing. "A user can't meaningfully send more than one message every few seconds while active, and isn't active more than a few hours a day — so the ceiling is a few thousand messages a day, and typical is far below that." Bounds are defensible; a bare guess isn't.

---

## Concept 4 — When and how much to estimate

Module 3 set the budget: **2–4 minutes in a 45-minute round (3–5 in a 60-minute round), producing 3–5 numbers, each with its "so…"** — or deferred and done inline when a decision needs it.

**Upfront or inline?** Both are legitimate, and some well-known interview guides now recommend skipping the upfront block unless it influences the design. A practical rule:

- **Estimate upfront** when scale *is* the problem and the numbers will shape the high-level design immediately — chat, feeds, video, metrics, ad clicks, anything where "how many partitions / how many connection servers" decides the boxes.
- **Estimate inline** when the design is mostly about correctness or workflow (payments, booking, document collaboration) and scale only matters at a couple of specific decision points — "do we need to shard the bookings table?" — then estimate *at that point*, where the "so" is obvious.
- **Say which you're doing**: *"I'll do a quick traffic and storage estimate now because it decides the storage layer; I'll estimate connection counts later when we get to the real-time path."*

**What to estimate — the usual menu, in rough order of how often it decides something:**

| Estimate | Decides |
|---|---|
| Peak write rate (per entity/operation) | Single primary vs partitioning; queue vs synchronous write |
| Peak read rate and read/write ratio | Caching, replicas, precomputation |
| Fan-out / amplification | The *real* write volume; push vs pull |
| Hot working set | Cache tier size; whether it fits on one node |
| Storage volume and growth | Single node vs partitioned; tiering; cost |
| Concurrent connections | Connection tier, gateway fleet, port/SNAT exhaustion |
| Bandwidth and egress | CDN, edge, cost |
| Latency budget vs physics | Regional deployment, edge, async |

You don't need all eight. Pick the ones attached to the hard part you named in step 1 (Module 4, Concept 23).

**What interviewers are listening for.** Across public rubrics, the positive signals are: the numbers are derived from stated assumptions, rounded sensibly, compared against capacity, and used later; the candidate notices when an estimate means a component is *not* a concern; and the candidate is fast. The negative signals are: asking for QPS instead of deriving it; long silent arithmetic; false precision ("289,351 messages per second"); outdated capacity numbers; and estimates that never reappear in the design.

---

# Part B — The arithmetic toolkit

## Concept 5 — Powers of two and ten, bits and bytes

### 5a. The bridge: 2¹⁰ ≈ 10³

Computers count in powers of two; humans and invoices count in powers of ten. The bridge is that 2¹⁰ = 1,024 ≈ 10³. That lets you move freely between them:

| Power of two | Exact | ≈ Power of ten | Name (binary / decimal) |
|---|---|---|---|
| 2¹⁰ | 1,024 | 10³ | KiB / KB (thousand) |
| 2²⁰ | 1,048,576 | 10⁶ | MiB / MB (million) |
| 2³⁰ | 1,073,741,824 | 10⁹ | GiB / GB (billion) |
| 2⁴⁰ | ~1.10 × 10¹² | 10¹² | TiB / TB (trillion) |
| 2⁵⁰ | ~1.13 × 10¹⁵ | 10¹⁵ | PiB / PB (quadrillion) |
| 2⁶⁰ | ~1.15 × 10¹⁸ | 10¹⁸ | EiB / EB |

Useful landmarks: 2¹⁶ = 65,536 (port numbers, `ushort`); 2³² ≈ 4.3 × 10⁹ (IPv4 space, `uint`, the limit that bites auto-increment `int` IDs at ~2.1 billion when signed); 2⁶⁴ ≈ 1.8 × 10¹⁹ (`ulong`, effectively inexhaustible for counters); 62⁷ ≈ 3.5 × 10¹² (seven base-62 characters — URL shortener codes).

**How much the binary/decimal gap matters.** At kilo it's 2.4%; at giga 7.4%; at tera 10%; at peta 12.6%. For estimation it's noise. For billing and limits it isn't: Azure documents some limits in binary units (Blob Storage's default account capacity is 5 **PiB**; Hyperscale's log rate is 100 **MiB**/s) and prices in decimal GB. Know that the gap exists; don't let it slow you down.

### 5b. Bits versus bytes

**Networks are specified in bits per second; storage and memory in bytes.** This is the most common factor-of-eight error in interviews.

- 1 Gbps ≈ 125 MB/s. 10 Gbps ≈ 1.25 GB/s. 100 Gbps ≈ 12.5 GB/s.
- A 1 GB file over a 1 Gbps link takes ~8 s at line rate — more in practice (protocol overhead, TCP behaviour, Concept 12).
- Video bitrates are in bits: "5 Mbps 1080p" is 625 KB/s, ~2.25 GB per hour.

Habit: write the unit next to every bandwidth number, and convert to one unit before comparing.

### 5c. Scientific notation is the working format

Write everything as *mantissa × 10^exponent* with a one- or two-digit mantissa. You'll multiply mantissas and add exponents (Concept 7). "5 × 10⁸ users × 50 messages" becomes 250 × 10⁸ = 2.5 × 10¹⁰ messages a day — a mental operation, not a long multiplication.

---

## Concept 6 — Time constants and rate conversion

The constants worth memorizing:

| Period | Seconds | Working approximation |
|---|---|---|
| Minute | 60 | 60 |
| Hour | 3,600 | 3.6 × 10³ |
| Day | 86,400 | **≈ 10⁵** (overestimates by 16%) — or 8.64 × 10⁴ when it matters |
| 30-day month | 2,592,000 | **≈ 2.6 × 10⁶** (≈ 2.5 × 10⁶ for mental math) |
| Year | 31,536,000 | **≈ 3 × 10⁷** (≈ π × 10⁷ is a famous mnemonic) |

And the conversions that fall out of them:

| Per day | ≈ Per second (average) |
|---|---|
| 1 million | ~12 |
| 10 million | ~120 |
| 100 million | ~1,200 |
| 1 billion | ~12,000 |
| 10 billion | ~120,000 |

| Per second (sustained) | Per day | Per month | Per year |
|---|---|---|---|
| 1 | ~86,000 | ~2.6 million | ~32 million |
| 1,000 | ~86 million | ~2.6 billion | ~32 billion |
| 100,000 | ~8.6 billion | ~260 billion | ~3 trillion |

Two things to notice:

- **Using 10⁵ for a day makes your per-second numbers ~14% low.** That's within estimation error, and you're about to multiply by a peak factor of 2–10 anyway. But if you're near a threshold, use 86,400.
- **Sustained small rates become large stored volumes.** One write per second is 32 million rows a year. 1,000 events a second is 32 billion rows a year — at 100 bytes each, 3 TB. Storage estimates often come from this direction.

**The per-month conversion is where cost lives.** Cloud services bill per hour (~730 hours a month) and per GB-month. "100 RU/s for a month" is 100 RU/s × 730 hours, and rates are typically quoted per hour. Keep 730 in your head alongside 2.6 × 10⁶.

---

## Concept 7 — Mental arithmetic for estimates

The goal is to never do long arithmetic in front of an interviewer. Five techniques cover almost everything.

**1. Round to one significant figure, preferably 1, 2, 3 or 5.** 86,400 → 10⁵ or 9 × 10⁴. 365 → 400 (or 3 × 10²). 1,440 minutes a day → 1.5 × 10³. Choose roundings that make the next step easy.

**2. Multiply mantissas, add exponents.** (3 × 10⁸) × (4 × 10¹) = 12 × 10⁹ = 1.2 × 10¹⁰.

**3. Divide by subtracting exponents.** 2.5 × 10¹⁰ messages/day ÷ 10⁵ s/day = 2.5 × 10⁵ messages/s.

**4. Round in opposite directions to cancel bias.** If you round one factor up, round another down. 47 × 23 ≈ 50 × 20 = 1,000 (actual 1,081) beats 50 × 25 = 1,250.

**5. Growth by doubling.** For compound growth, think in doublings: the *rule of 72* says something growing at r% a year doubles in about 72 / r years. 40% annual growth doubles in under two years; 100% growth (doubling yearly) is 32× in five years — which is why "design for 10×, plan for 100×" (Module 4) is a 3–5 year horizon for a fast-growing product.

**A worked mental chain** (no paper):

> *"300 million DAU, 10 feed loads each — that's 3 billion a day. Divide by 10⁵ — 30,000 a second on average. Peak around 3× — call it 100,000 reads a second."*

Each step is one operation and is spoken aloud. That's the target.

**Percentages and ratios.** Remember that a 99% cache hit rate means the backend sees 1% — a factor of 100 — and that going from 90% to 99% hit rate reduces backend load by **10×**, not by 9%. Ratios hide inside percentages; convert them (Concept 20).

---

## Concept 8 — Units and sanity checks

### 8a. Carry the units

Dimensional analysis — writing the units and cancelling them — catches most arithmetic errors before they leave your mouth:

```
5×10⁸ users × 50 msg/(user·day) × 200 B/msg = 5×10¹² B/day = 5 TB/day
```

If the units don't cancel to what you expected (TB/day), the formula is wrong. This takes no extra time once it's a habit, and it's visibly rigorous on a whiteboard.

### 8b. Three sanity checks

1. **Per-user check.** Divide the total back by the user count. "5 TB a day for 500 million users is 10 KB per user per day — about fifty short messages. Plausible." If you get "40 MB per user per day" for a text chat, something's off.
2. **Comparison with a known system.** If your estimate implies a chat system stores more data than YouTube, or a startup's MVP needs more servers than Stack Overflow ran on (it famously served one of the largest sites on the internet from a small number of servers), revisit the assumptions.
3. **Physical plausibility.** Does the implied rate fit what one human can do? A metrics agent can emit thousands of samples a second; a person can't send thousands of chat messages a second. Does the implied bandwidth fit a network link? Does the implied storage fit the budget?

### 8c. Cross-check top-down against bottom-up

Concept 3's two directions are also a sanity check. If top-down says 900,000 messages per second and you know WhatsApp-scale systems publicly reported on the order of 100 billion messages a day at their peak era, your per-second number is in the right range — 10¹¹ / 10⁵ = 10⁶. Agreement within a small factor is confidence; disagreement is a question to resolve.

---

## Concept 9 — Averages, peaks and skew

Averages are the input to estimates and almost never the right output. Three corrections turn an average into something you can design for.

### 9a. Peak factor

Load varies over the day, the week and with events. You provision for **peak**, so you multiply the average by a **peak-to-average ratio**:

| Workload | Typical peak ÷ average | Notes |
|---|---|---|
| Global consumer app (users in many time zones) | ~2–3× | Time zones flatten the daily curve |
| Regional consumer app (one country) | ~3–5× | Evening peak, overnight trough |
| Business/B2B (working hours) | ~3–10× | Near-zero at night and weekends; Monday morning spikes |
| Event-driven (sports, ticket on-sales, Black Friday, a push notification to everyone) | 10–100×+ | Spikes are the design problem, not a multiplier |
| Batch/scheduled (midnight jobs, cron) | Can be effectively infinite | Everything starts at 00:00:00 unless you add jitter |

State the factor you're assuming — *"I'll assume 3× for the daily peak"* — because it's one of the assumptions most likely to be challenged (Module 4, Concept 9).

### 9b. Peaks within the second

Per-second rates hide sub-second bursts. A push notification sent to 10 million devices at 12:00:00 doesn't produce 10 million requests spread over the next hour; a large fraction arrive in the first few seconds. A cron job on 5,000 instances that fires "every minute" arrives as a 5,000-request burst at :00. Two consequences:

- **Queues and admission control are sized for bursts, autoscaling for trends.** Autoscalers react in minutes; bursts last seconds.
- **Add jitter** to anything scheduled or retried, and count it as part of the design (Module 13).

### 9c. Skew: size for the hottest key

Totals tell you how many partitions you need; **the hottest key tells you whether partitioning works at all.** Access in social, content and commerce systems is heavy-tailed: a small fraction of keys gets a large share of the traffic. A celebrity account, a viral link, a huge tenant or a flash-sale product can each carry more load than a partition can serve. Cosmos DB, for example, caps each physical partition — and therefore each logical partition key — at 10,000 RU/s. Your total provisioned throughput is irrelevant if one key needs 30,000.

So for any partitioned design, do two estimates: **total ÷ per-partition capacity** (how many partitions) and **hottest key ÷ per-partition capacity** (whether one key needs special handling — splitting, caching, a different read path). Part G's feed and URL-shortener examples show both.

### 9d. Distributions, not just means, for sizes too

Object sizes are skewed as well. The *average* attachment might be 300 KB while the 99th percentile is 50 MB. The average sizes your storage; the tail sizes your timeouts, request limits (Cosmos DB's 2 MB item limit, for example) and memory buffers. When a size has a long tail, say so: *"average 200 bytes per message, but I'll cap message text at 64 KB and send attachments to blob storage."*

---

# Part C — The numbers to know cold

This is the reference half of the module. The aim isn't to memorize every row; it's to know the **order of magnitude** of each and the **ratios** between them, because ratios are what change designs. Where a number is a platform limit, it was checked against current documentation in autumn 2026 — limits move, so treat them as starting points and confirm before you rely on them.

## Concept 10 — The latency ladder (2026)

### 10a. The table

Jeff Dean's "Latency numbers every programmer should know" (circa 2009–2012, popularized by Peter Norvig and a famous GitHub gist) is still the right *shape*, but several rows are badly out of date — notably SSDs, memory bandwidth and datacenter networking. Recent community revisions in 2025–2026 converge on values close to these:

| Operation | Typical latency (2026) | Human scale (if 1 ns = 1 s) | Notes |
|---|---|---|---|
| CPU cycle (~3–4 GHz) | ~0.3 ns | 0.3 s | |
| L1 cache hit | ~1 ns | 1 s | Tens of KB per core |
| Branch mispredict | ~3–5 ns | 3–5 s | Pipeline flush |
| L2 cache hit | ~3–5 ns | 3–5 s | ~1–2 MB per core |
| L3 cache hit | ~10–20 ns | 10–20 s | Tens of MB, shared |
| Uncontended lock acquire/release | ~15–25 ns | ~20 s | `lock` with no contention; contended locks cost µs (context switches) |
| Main memory (DRAM) access, random | ~80–100 ns | ~1.5 min | Serial pointer chasing exposes it; prefetching hides it for sequential access |
| Fast compression of 1 KB (LZ4-class) | ~1–3 µs | ~30 min | |
| Context switch | ~1–5 µs | ~1 h | Thread wake-ups, blocking calls |
| Send 2 KB over a 10 Gbps link (serialization only) | ~2 µs | ~30 min | Wire time, not propagation |
| Read 1 MB sequentially from memory (one core) | ~10–50 µs | ~3–14 h | The 2009 table said 250 µs |
| NVMe SSD random 4 KB read (local) | ~15–100 µs | 4–28 h | Device ~20 µs; queueing and OS add more; the 2009 table said 150 µs for "SSD" |
| Read 1 MB sequentially from NVMe | ~100–200 µs | 1–2 days | PCIe 4/5 drives deliver ~7–14 GB/s |
| Round trip within one datacenter / one availability zone | ~0.1–0.5 ms | 1–6 days | Cloud VM to VM |
| Redis GET or cached DB point read, in-zone | ~0.2–1 ms | 2–12 days | Mostly network |
| Round trip between availability zones, same region | ~0.5–2 ms | 6–23 days | Azure designs zones to stay within about 2 ms round trip |
| Durable write to cloud block storage | ~0.5–2 ms | | Network-attached disks; local NVMe with power-loss protection is tens of µs |
| HDD seek | ~5–10 ms | 2–4 months | |
| Read 1 MB sequentially from HDD | ~5–10 ms | 2–4 months | ~150–250 MB/s |
| Round trip between regions on one continent | ~10–30 ms | 4 months–1 year | e.g., West Europe ↔ North Europe ~17 ms |
| Round trip across the Atlantic | ~70–90 ms | 2–3 years | West Europe → East US ~85 ms |
| Round trip Europe ↔ Southeast Asia / Japan / Australia | ~170 / ~230 / ~260 ms | 5–8 years | Measured Azure P50 values (Concept 11) |

### 10b. What changed since the classic table — and why it matters for design

- **NVMe collapsed the memory-to-disk gap.** A random read from local flash is now ~20 µs — about 200× DRAM, but ~500× faster than an HDD seek. Designs that assume "disk = milliseconds" (and therefore cache everything) are over-engineered for workloads on local NVMe. But beware: **cloud block storage is network-attached**, so a managed disk's latency is closer to a network round trip (sub-ms to a few ms) than to local NVMe, and each disk and VM size has its own IOPS and MB/s caps.
- **Datacenter networks got much faster** — a round trip inside a zone is now often faster than reading 1 MB from a spinning disk. Remote memory (a cache in the same zone) is "cheap" compared to disk, expensive compared to local memory.
- **Wide-area latency didn't change and won't.** It's set by the speed of light in fibre (Concept 11). Every other row improves with hardware; the bottom rows don't.

### 10c. The ratios to carry around

| Ratio | Approximately | Design implication |
|---|---|---|
| DRAM ÷ L1 | ~100× | Data layout and allocation matter for hot loops (Module 17) |
| In-zone network round trip ÷ DRAM | ~1,000–5,000× | In-process caches beat remote caches by three orders of magnitude for hot data (HybridCache's L1, Module 10) |
| NVMe random read ÷ DRAM | ~200× | A local SSD index lookup is cheap; a remote call is not |
| Cross-zone ÷ in-zone | ~3–5× | Synchronous replication across zones costs ~1–2 ms per write |
| Cross-region ÷ cross-zone | ~10–100× | Synchronous cross-region coordination is a deliberate, expensive choice (PACELC, Module 7) |
| Transatlantic RTT ÷ typical service handler time | ~10–100× | A single chatty cross-region dependency dominates a latency budget |

**The practical rule:** a request can afford thousands of memory accesses, a handful of in-zone network round trips, perhaps one cross-zone round trip, and essentially *zero* cross-region round trips if it has an interactive latency target.

---

## Concept 11 — The physics of distance

### 11a. Derive it

Light travels at ~300,000 km/s in a vacuum. In optical fibre the refractive index (~1.47) slows it to roughly **200,000 km/s**, i.e. 200 km per millisecond. A round trip covers the distance twice, so:

> **Every 100 km of fibre path costs about 1 ms of round-trip time. 1,000 km ≈ 10 ms. 10,000 km ≈ 100 ms.**

That's the floor. Real paths are longer than the great-circle distance (cables follow coasts and terrain, routes go through interconnection points) and routers and amplifiers add a little, so measured round trips are typically **1.3–2.3× the great-circle floor**.

### 11b. Check it against measured Azure numbers

Microsoft publishes P50 round-trip latencies between Azure regions, measured continuously across its backbone. The dataset current at the time of writing covers the 30 days ending July 30, 2026, and values are directional (A→B can differ from B→A). Comparing a few pairs with the great-circle floor:

| Source → destination (Azure regions) | Approx. great-circle distance | Physics floor (RTT) | Measured P50 RTT | Ratio |
|---|---|---|---|---|
| West Europe (Netherlands) → UK South | ~360 km | ~4 ms | 11 ms | ~3× (short paths carry fixed overheads) |
| West Europe → North Europe (Ireland) | ~750 km | ~7.5 ms | 17 ms | ~2.3× |
| North Europe → East US | ~5,400 km | ~54 ms | 73 ms | ~1.35× |
| West Europe → East US (Virginia) | ~6,200 km | ~62 ms | 85 ms | ~1.4× |
| East US → West US (California) | ~3,700 km | ~37 ms | 69 ms | ~1.9× |
| West Europe → Southeast Asia (Singapore) | ~10,500 km | ~105 ms | 169 ms | ~1.6× |
| West Europe → Japan East | ~9,300 km | ~93 ms | 233 ms | ~2.5× (routing goes the long way) |
| East US → Australia East | ~15,700 km | ~157 ms | 201 ms | ~1.3× |
| East US → East US 2 | ~200 km | ~2 ms | 8 ms | short-path overhead |

Lessons the table teaches:

- **The physics estimate gets you within 2×** — good enough for every interview decision.
- **Short distances carry a fixed overhead** of several milliseconds; don't assume two nearby regions are "1–2 ms apart."
- **Routes, not maps, decide** — Europe to Japan is much slower than distance alone suggests. If an answer depends on a specific pair, cite the measured value (and mention that Microsoft publishes it).
- **Within a region**, availability zones are separate datacenters a few to tens of kilometres apart, designed to stay within roughly a 2 ms round trip — that's why synchronous zone-redundant replication is affordable and synchronous cross-region replication usually isn't.

### 11c. What physics means for requirements

From Module 4's latency budget: if a user in Singapore must get a p99 under 200 ms and your only region is West Europe at ~170 ms *median* round trip, the target is infeasible before your code runs a single instruction. The design options are fixed by physics: **move compute or data closer** (regional deployment, edge caching, CDN), **reduce round trips** (connection reuse, edge termination, batching), or **change the requirement** (asynchronous UX, optimistic UI). Saying this in requirements time is a strong signal.

---

## Concept 12 — Protocol costs: round trips you didn't write

Propagation latency is per round trip; protocols decide **how many round trips** you pay.

### 12a. Connection setup

| Step | Round trips before the first request byte can be sent | Notes |
|---|---|---|
| DNS lookup (uncached) | ~1 (to a resolver, possibly more upstream) | Usually cached; first-visit cost |
| TCP handshake | 1 | SYN, SYN-ACK, ACK |
| TLS 1.3 handshake | 1 | TLS 1.2 needed 2; TLS 1.3 session resumption can send early data (0-RTT) with replay caveats |
| QUIC / HTTP/3 | 1 for transport + crypto combined | 0-RTT resumption possible |
| The request itself | 1 | |

So a **cold HTTPS request over TCP + TLS 1.3 costs ~3 round trips** (TCP, TLS, request) — plus DNS on a first visit — while a **warm, reused connection costs 1**. For a user 170 ms away, that's ~510 ms versus ~170 ms. This is the arithmetic behind:

- **Connection reuse** — in .NET, a long-lived `HttpClient` / `IHttpClientFactory`-managed `SocketsHttpHandler` with pooled connections, rather than a new client per request (which also exhausts ports, Concept 21).
- **Edge termination** — Azure Front Door or a CDN terminates TCP and TLS a few milliseconds from the user, then forwards over pre-warmed backbone connections. The user pays three *short* round trips plus one long one, not three long ones.
- **HTTP/2 and HTTP/3 multiplexing** — many requests over one connection, avoiding per-request handshakes.

### 12b. TCP slow start and the first response

A new TCP connection doesn't start at full speed. With the standard initial congestion window of 10 segments (~14 KB), the first round trip can carry only about that much data; each subsequent round trip roughly doubles it until loss or the receiver window limits it. Consequence: **a 100 KB response on a cold connection to a distant user takes several round trips** — that's why keeping critical first responses small, and reusing warm connections, matters more than raw bandwidth for interactive latency.

### 12c. The bandwidth-delay product

A single TCP stream can't have more unacknowledged data in flight than its window. So:

> **max throughput of one stream ≈ window size ÷ RTT**

- With a 64 KB window and a 150 ms round trip: 64 KB / 0.15 s ≈ 430 KB/s ≈ **3.5 Mbps**, no matter how fat the pipe. (Modern stacks use window scaling, so windows grow much larger — but the principle stands.)
- To fill a 1 Gbps link at 150 ms you need ~1 Gbps × 0.15 s ≈ **19 MB** in flight.

This explains why cross-continent bulk transfers are slower than the link speed suggests and why tools for large transfers use many parallel streams (AzCopy does this for Blob Storage). In a design, if you're replicating large volumes across regions, estimate whether a single stream can carry it.

### 12d. Serialization (wire) time

Separately from propagation, putting bytes on a link takes time: **size ÷ bandwidth**.

- 1 KB on 10 Gbps: ~0.8 µs — negligible.
- 1 MB on 1 Gbps: ~8 ms — noticeable.
- 1 MB on a 10 Mbps mobile uplink: ~0.8 s — dominant.

For small API payloads in a datacenter, propagation and processing dominate; for large payloads or slow client links, serialization time does. Mobile upload paths in particular (Part G, Concept 40) are often limited by the user's uplink, not your servers.

---

## Concept 13 — Bandwidth and throughput

Latency tells you how long one operation takes; bandwidth tells you how much can flow. The two are related by concurrency (Little's Law, Concept 21), and both matter.

| Resource | Throughput (order of magnitude, 2026) | Notes |
|---|---|---|
| DRAM bandwidth, one modern server socket | Hundreds of GB/s | Many DDR5 channels; one core alone gets ~10–30 GB/s |
| Local NVMe SSD (PCIe 4.0 / 5.0) | ~7 / ~12–14 GB/s sequential; hundreds of thousands to ~1M+ random IOPS | Per drive; servers hold several |
| SATA SSD | ~0.5 GB/s | Interface-limited |
| HDD | ~150–250 MB/s sequential; ~100–200 random IOPS | Random I/O is the killer |
| Cloud managed disks | Per-disk and per-VM caps on IOPS and MB/s | Often the real ceiling — check the disk type and VM size pages |
| Cloud VM NIC | ~10–100+ Gbps depending on size | Larger VMs get more; per-VM caps apply |
| Fast compression (LZ4/Snappy class) | ~GB/s per core | zstd at moderate levels: hundreds of MB/s; gzip: tens to ~100 MB/s per core |
| AES-GCM encryption with hardware support | Several GB/s per core | TLS bulk cost is rarely the bottleneck; handshakes are the expensive part |
| JSON serialization (System.Text.Json, source-generated) | Hundreds of MB/s to ~1 GB/s per core, payload-dependent | Binary formats (Protobuf, MessagePack) are faster and smaller; measure with BenchmarkDotNet |

**Two consequences that come up in interviews:**

1. **Sequential versus random I/O differ by orders of magnitude on HDDs and by a smaller but real factor on SSDs.** That's why log-structured designs (append-only logs, LSM trees, Kafka-style partitions) exist: they turn random writes into sequential ones (Module 12).
2. **CPU work per byte matters at high volume.** At 1 GB/s of ingest, JSON parsing at ~500 MB/s per core needs two cores just for parsing. Compression at 100 MB/s per core needs ten. Estimates of *CPU per byte* or *CPU per request* (Concept 14) convert throughput into fleet size.

---

## Concept 14 — What one thing can do

This is the capacity side of every comparison in Concept 1. Two kinds of number appear here: **first-principles rules of thumb** (hardware-dependent; measure before trusting them) and **documented platform limits** (verified against current Azure documentation; they change over time).

### 14a. One application server: derive it from CPU per request

Don't memorize "a server does N requests per second." Derive it:

> **requests/s per core ≈ 1,000 ÷ (CPU milliseconds per request)**

| CPU time per request | Per core | 8-vCPU instance at 60% target utilization |
|---|---|---|
| 0.1 ms (cached lookup, tiny JSON) | ~10,000 rps | ~48,000 rps |
| 0.5 ms (typical read endpoint, small payload, one cache call) | ~2,000 rps | ~9,600 rps |
| 2 ms (business logic, larger JSON, several dependencies) | ~500 rps | ~2,400 rps |
| 20 ms (heavy computation, PDF/image work) | ~50 rps | ~240 rps |

Note what's *not* in the formula: **time spent waiting on I/O doesn't consume CPU** — as long as the code is asynchronous. With `async`/`await` end to end, an instance can have thousands of requests in flight while waiting on databases. With blocking calls, every waiting request holds a thread, and ThreadPool starvation (Module 15) caps throughput far below the CPU limit. That's a .NET-specific reason the estimate can be wildly wrong if the code isn't async.

Framework overhead itself is small: ASP.NET Core sits near the top of public benchmarks such as TechEmpower, with plaintext and JSON tests reaching very high request rates on large machines. Your handler's work — serialization, allocations, logging, dependency calls — dominates. So in an interview: *"Assume ~1 ms of CPU per request, so ~1,000 per core; at 60% utilization an 8-vCPU instance handles ~5,000 a second. I'd confirm with a load test."*

### 14b. One relational primary

Rules of thumb for a single well-provisioned primary (SQL Server/Azure SQL or PostgreSQL on a large modern machine, working set mostly in memory, sensible indexes):

| Operation | Order of magnitude | What limits it |
|---|---|---|
| Indexed point reads (buffer-pool hits) | 10⁴–10⁵+ per second | CPU, latching, connection handling |
| Simple durable write transactions | ~10³–10⁴ (low tens of thousands with group commit and small transactions) | Transaction log write latency and bandwidth |
| Complex queries / scans | Anything — depends entirely on the plan | Indexes, I/O, memory grants |
| Data size on one node | Single-digit TB is routine; tens of TB is feasible | Backup/restore times, maintenance, IOPS |

**A derived limit worth knowing: log throughput caps write rate.** Azure SQL Database Hyperscale documents a maximum transaction log rate of **100 MiB/s** on standard-series hardware (150 MiB/s on premium-series), with databases up to **128 TB**. If each transaction generates ~2 KB of log (a row or two plus index updates and overhead), then:

> 100 MiB/s ÷ 2 KB ≈ **50,000 transactions/s** — a hard ceiling no matter how many vCores you add.

That one division tells you when a single relational database stops being enough for a write-heavy workload — and it's the kind of derived limit that impresses interviewers far more than a memorized "databases do 10k TPS."

### 14c. One cache shard

Redis executes commands on a single thread per shard, so a shard does on the order of **10⁵ simple operations per second** with sub-millisecond in-zone latency (more with pipelining, less with large values or expensive commands like big `ZRANGE`s). Memory per node ranges from a few GB to hundreds of GB; very large memory-optimized VMs exist with terabytes. Two derived checks:

- **Throughput:** 1M cache reads/s ÷ ~100k per shard ≈ 10+ shards before replicas — even if the data fits on one.
- **Hot key:** one key read 300,000 times a second won't fit on one shard however many shards you have — that's what in-process L1 caching (HybridCache) or key replication is for (Module 10).

Azure Managed Redis is the current Azure offering (Module 10 covered the transition from Azure Cache for Redis); check its SKU tables for memory and throughput per size.

### 14d. Azure managed-service limits (verified autumn 2026)

| Service | Limit or unit | Value | Estimation use |
|---|---|---|---|
| **Cosmos DB** | Cost of a point read of a 1 KB item | 1 RU | Reads: RU/s ≈ reads/s × KB |
| | Cost of a 1 KB write (default indexing) | ~5 RU (rises with item size and indexed properties) | Writes: RU/s ≈ writes/s × ~5 |
| | Throughput per physical partition (and so per logical partition key) | 10,000 RU/s | Hot-key ceiling |
| | Storage per physical partition / per logical partition | 50 GB / 20 GB | Partition counts; hierarchical keys to exceed 20 GB per top-level key |
| | Max item size | 2 MB | Blobs go elsewhere |
| | Default max RU/s per container or shared database | 1,000,000 (raisable by support request) | Very large workloads need a ticket |
| | Minimum manual throughput | max(400 RU/s, 1 RU/s per GB stored, highest-ever RU/s ÷ 100) | **Storage forces a throughput floor** — 1 PB implies ≥1M RU/s |
| | Price (provisioned, single-region writes, list) | ~$0.008 per 100 RU/s per hour (≈ $5.84/month); storage ~$0.25/GB-month | Cost estimates; autoscale costs 1.5×; multi-region writes 2× |
| **Event Hubs (Standard)** | One throughput unit | Ingress 1 MB/s **or** 1,000 events/s (whichever first); egress 2 MB/s or 4,096 events/s | TUs needed ≈ max(MB/s, events/s ÷ 1,000) |
| | Max TUs per namespace | 40 | Beyond ~40 MB/s, Premium or Dedicated |
| | Per-partition guidance | ~1 MB/s ingress, ~2 MB/s egress | Partition count = max(ingress ÷ 1, egress ÷ 2) |
| **Event Hubs (Premium)** | One processing unit | ~5–10 MB/s ingress, ~10–20 MB/s egress (guidance) | |
| **Blob Storage (GPv2 account)** | Default max request rate | 40,000 req/s in major regions (20,000 elsewhere) | Many small objects at high rate → several accounts or a CDN |
| | Default max ingress / egress | 60 / 200 Gbps in major regions | Rarely the limit; raisable |
| | Default max account capacity | 5 PiB (raisable) | |
| **Azure SQL Hyperscale** | Max data size / max log rate | 128 TB / 100 MiB/s (150 premium-series) | Write ceiling (14b) |
| **Outbound connections (SNAT)** | Load Balancer default allocation | Pool-size dependent, e.g. 1,024 ports per instance by default for small pools; ~64,000 per frontend IP | Outbound connection budget (Concept 21) |
| | NAT Gateway | 64,512 ports per public IP, up to 16 IPs (~1M) | The recommended fix for SNAT exhaustion |

Two lessons hide in this table:

1. **Some limits turn one dimension into another.** Cosmos DB's "1 RU/s per GB" minimum means a huge, rarely-read dataset still forces large provisioned throughput — storage volume becomes a throughput bill. Event Hubs' "1,000 events/s per TU" means many tiny events are limited by count, not bytes.
2. **Per-partition limits matter more than totals.** A container can scale to millions of RU/s; a single partition key can't exceed 10,000. Estimate both (Concept 23).

### 14e. One server's connections

A server-side socket is identified by the four-tuple (client IP, client port, server IP, server port), so a server is **not** limited to 65,535 inbound connections. The limits are memory (kernel buffers and application state per connection — a few KB to tens of KB per idle WebSocket in a well-tuned Kestrel service), file descriptors (raise the ulimit), CPU for heartbeats and TLS, and — most important — **blast radius**: if one node holds a million connections, its failure triggers a million reconnects. WhatsApp's engineering blog famously described pushing a single server past two million connections years ago; practical designs often cap at 100,000–500,000 per node for operational reasons.

The **client side** (outbound) is different: each outbound connection to the same destination needs a distinct source port, and in Azure, outbound connections to the internet go through SNAT with the limits above. That's where port exhaustion actually happens.

---

## Concept 15 — Data sizes

### 15a. Primitives and identifiers

| Thing | Size |
|---|---|
| ASCII character / UTF-8 for Latin text | 1 byte (UTF-8: 1–4 bytes per code point; ~2–3 for many non-Latin scripts) |
| .NET `char` (UTF-16) | 2 bytes |
| `int` / `long` / `double` | 4 / 8 / 8 bytes |
| `decimal` | 16 bytes |
| `DateTime` / `DateTimeOffset` (in memory) | 8 / 16 bytes |
| SQL Server `datetime2(7)` / `datetimeoffset(7)` | 8 / 10 bytes |
| `Guid` / SQL `uniqueidentifier` | 16 bytes binary; 36 characters as a hyphenated string |
| Snowflake-style 64-bit ID | 8 bytes |
| IPv4 / IPv6 address | 4 / 16 bytes |
| SHA-256 hash | 32 bytes (64 hex characters) |
| Base62 7-character short code | 7 bytes as ASCII; 62⁷ ≈ 3.5 × 10¹² possible values |

### 15b. Typical record and payload sizes

| Thing | Order of magnitude |
|---|---|
| Relational row in an OLTP table | 100 B – 1 KB (plus index entries: often 1–3× the row in total) |
| JSON document in a document store | 1–10 KB (field names repeat in every document) |
| Short text post or chat message with metadata | 200 B – 1 KB |
| Log line (structured) | 200 B – 1 KB |
| Trace span (OpenTelemetry) | ~0.5–2 KB |
| Metric sample (timestamp + value), raw / compressed | 16 B / ~1.4 B (Facebook's Gorilla paper: 1.37 bytes per point on average) |
| Web page HTML, compressed | Tens of KB; a whole page with assets ~2–3 MB median |
| Typical API response | 1–50 KB |

### 15c. Media

| Thing | Size or rate |
|---|---|
| Phone photo (12–50 MP JPEG/HEIC) | ~2–5 MB |
| Web-optimized image / thumbnail | ~100–300 KB / ~10–50 KB |
| Music stream | 128–320 kbps (~1–2.4 MB per minute) |
| Voice call (Opus) | ~16–64 kbps |
| Video 480p / 720p / 1080p / 4K (streaming bitrates) | ~1 / ~2.5–4 / ~5–8 / ~15–25 Mbps |
| One hour of 1080p at 5 Mbps | ~2.25 GB |

### 15d. Compression ratios

| Data | Typical ratio | Note |
|---|---|---|
| Text, JSON, logs, CSV | 4–10× (gzip/zstd) | Repetitive keys compress well |
| Time series (delta-of-delta + XOR) | ~10× vs raw 16 B/point | Gorilla-style encoding |
| Columnar analytics (Parquet) | 5–20× vs row JSON | Encoding plus compression |
| JPEG, PNG, MP4, ZIP | ~1× | Already compressed — don't count on savings |

### 15e. Overhead you must not forget

Every stored byte carries companions:

- **Indexes**: secondary indexes often double or triple the raw table size.
- **Replication**: 3 copies is typical for durable distributed storage (Azure Storage LRS keeps three copies within a datacenter); erasure coding at large scale gets the overhead down to ~1.3–1.5× for cold data.
- **Per-record metadata**: system properties in document stores, row headers, per-key overhead in caches (Redis: tens of bytes per key before your value; more for small items in rich data structures).
- **Headroom**: plan to run storage at ≤70–80% full.

Part D's storage recipe multiplies these in explicitly.

---

## Concept 16 — Failure rates at scale

Failure arithmetic belongs in estimation because **at scale, failure is a rate, not an event**.

**Disks.** Backblaze, which publishes statistics for hundreds of thousands of drives, reported an annualized failure rate of **1.36% for 2025** (down from 1.55% in 2024). Take ~1–2% as the planning number for HDDs; SSDs fail differently (wear, firmware) but at broadly similar orders.

> 10,000 disks × 1.36% ≈ **136 failures a year ≈ one every 2.7 days.**
> 100,000 disks ≈ **one every 6–7 hours.**

**Servers.** Whole-machine failures (power supplies, memory, motherboards, kernels) run at a few percent a year per server; at 10,000 servers that's several failures every day — before counting deployments, which in practice cause more incidents than hardware.

**What the arithmetic forces:**

1. **Recovery must be automatic** at fleet scale — nobody hand-replaces a disk every few hours (Module 13).
2. **Rebuild time matters.** If re-replicating a failed 20 TB node takes ten hours and failures arrive every few hours, several rebuilds are always in flight — and the window during which a second failure in the same replica set loses data is real. This is why durable stores spread replicas across fault domains and keep rebuilds fast.
3. **Correlated failures dominate tail risk** — a bad batch of drives, a firmware bug, a zone power event, one bad deployment everywhere. Independent-failure math (Module 4, Concept 11d) is a lower bound on risk.

**Nines as minutes (recap from Module 4).** 99.9% ≈ 43 min/month; 99.99% ≈ 4.3 min/month; 99.999% ≈ 26 s/month. Combined with failure frequency: three incidents a year with 20-minute detection and recovery already consume a 99.99% annual budget.

---

# Part D — The recipes

Each recipe is a formula, the factors people forget, and the comparison that turns the result into a decision.

## Concept 17 — Traffic

### 17a. The chain

```
average rate  = active users × actions per user per period ÷ seconds per period
peak rate     = average × peak factor
per operation = split by operation (reads vs writes, by endpoint)
real load     = per-operation rate × fan-out × amplification
```

Use **daily active users** (DAU) for daily behaviour, **peak concurrent users** for connection counts, **monthly active users** (MAU) only for monthly behaviour, and **registered users** only for storage of per-user data. Mixing them is a classic 3–30× error: a product with 100M registered users might have 30M MAU, 10M DAU and 1M peak concurrent.

### 17b. Fan-out

Fan-out is how many downstream operations one action causes.

- **Write fan-out:** a post to 200 followers' timelines is 200 writes. A message to a 50-person group delivered to ~1.5 devices each is ~75 deliveries.
- **Read fan-out:** a search query scatter-gathered across 40 shards is 40 reads; a dashboard that calls 12 services is 12 requests.

Fan-out is often the *real* volume. A feed system with 1,700 posts a second may be doing 350,000 timeline inserts a second (Part G, Concept 36).

### 17c. Amplification

Real systems multiply external requests by internal work. Common amplifiers:

| Amplifier | Typical multiplier | Note |
|---|---|---|
| Internal service calls per external request | 3–30× | Microservice call graphs; each hop is also a latency and availability tax |
| Retries | 1.0–1.1× normally; **up to the retry count** during incidents | Retries amplify load exactly when the system is weakest — retry budgets (Module 13, 25) |
| Client polling | Users × polls per period, independent of activity | Polling every 5 s by 10M clients = 2M rps of mostly empty answers |
| Health checks and heartbeats | Instances × probes per period, or connections ÷ heartbeat interval | 150M connections with a 30 s heartbeat = 5M messages/s (Part G) |
| Database work per API call | Several queries, plus index maintenance per write | One logical write can touch 3–5 index structures |
| Replication | × replica count for writes | Writes to 3 replicas are 3 disk writes and 2 network sends |
| Telemetry | Logs, spans and metrics per request | Can rival the payload in bytes (Module 4, Concept 18) |

**Say amplification out loud when it matters.** *"10,000 external requests a second become about 60,000 internal calls with this call graph, so the internal network and the service mesh are sized for 60k."*

### 17d. Read/write split

Estimate reads and writes separately, because they scale with different mechanisms: reads with caches, replicas and precomputation; writes with partitioning, batching and asynchronous processing. A **read:write ratio** of 100:1 points toward caching and fan-out-on-write; a ratio near 1:1 (or write-heavy, like telemetry) points toward log-structured storage and partitioned ingestion.

---

## Concept 18 — Storage

### 18a. The formula

```
raw        = objects per period × bytes per object × retention period
stored     = raw × index/metadata overhead × replication factor
provisioned= stored ÷ target fill (≈ 0.7–0.8)
```

Plus growth: if the product is growing, the period's object count grows too. For a quick estimate, use the volume at the end of the planning horizon (say, year three) rather than today's.

### 18b. Split by temperature and by store

Most systems store different things in different places, and one big total hides that:

| Data | Typical store | Sizing driver |
|---|---|---|
| Hot, mutable, queried (users, orders, recent messages) | Relational or document database | Rows × row size × index factor |
| Large immutable objects (images, video, attachments, exports) | Blob/object storage | Count × size; usually dominates bytes |
| Derived/denormalized (timelines, search index, caches) | Cache, search engine, read models | Working set, not total |
| Event logs and streams | Kafka/Event Hubs with retention, then archive | Rate × size × retention |
| Telemetry | Log analytics, metrics TSDB | Often the largest and most forgotten |
| Backups and point-in-time restore | Separate storage | Retention × change rate |

**The usual finding**: blobs dominate bytes, the database dominates cost per byte, and telemetry surprises everyone.

### 18c. The per-user and per-entity sanity checks

Divide your totals back: bytes per user per day, bytes per order, bytes per device per hour. If 500M users each generate 10 KB a day of chat text, that's believable; if a smart thermostat emits 10 MB an hour, it isn't (Concept 8).

### 18d. What storage estimates usually decide

- **Single node or partitioned?** Single-digit TB on a relational node is routine; Hyperscale goes to 128 TB; beyond that, or with high write rates, partition (Concept 28).
- **Tiering.** If 95% of data is rarely read after 30 days, the design needs a hot store and a cheap cold store — and the cost difference is often 10× or more (Concept 25).
- **Partition count in a managed store.** Storage-driven partition counts can exceed throughput-driven ones (Cosmos DB: 50 GB per physical partition; Concept 23).
- **Retention policy as a requirement.** "How long do we keep it?" can be worth a 10× swing; ask it (Module 4, Concept 12e).

---

## Concept 19 — Bandwidth

### 19a. The formula

```
bandwidth (bits/s) = requests/s × bytes per request or response × 8
```

Split it three ways, because they cost and scale differently:

| Direction | What it is | Why it matters |
|---|---|---|
| **Ingress** | Data into your system (uploads, writes) | Usually free to receive in cloud billing; sizes upload paths and storage write rates |
| **Egress** | Data out to users or the internet | Billed per GB; drives CDN decisions; can dominate cost for media |
| **East-west** | Data between your own components (replication, cache fills, service calls, cross-zone/cross-region traffic) | Often several times the user-facing volume; cross-region transfer is billed |

### 19b. CDN offload

For cacheable content, the origin sees only the **miss rate** of the CDN:

```
origin egress = total egress × (1 − CDN hit ratio)
```

At a 95% hit ratio, the origin serves 5% — a 20× reduction. Whether content is cacheable at the edge (static assets, video segments, public pages) or not (personalized feeds, API responses) is therefore one of the most valuable questions in any bandwidth-heavy prompt.

### 19c. What bandwidth estimates usually decide

- **Whether a CDN or edge tier is mandatory** — anything in the Tbps range is CDN territory; origin-only designs don't survive.
- **Whether payload size is a design concern** — 100k rps × 50 KB is 40 Gbps of egress; shrinking responses (compression, pagination, field selection) becomes architecture, not polish.
- **Client-side limits** — mobile uplinks (a few to tens of Mbps) bound upload-heavy flows; resumable chunked uploads directly to blob storage (with short-lived SAS tokens) keep app servers out of the data path.
- **Cost** — egress pricing turns bandwidth directly into money (Concept 25).

---

## Concept 20 — Memory and cache sizing

### 20a. Working set, not total

A cache holds the **working set**: the data accessed often enough that caching it is worth the memory. For skewed access (Concept 9c), the working set is a small fraction of the total and captures most of the traffic. A common planning assumption: **~20% of items receive ~80% of reads**, but real distributions are often more extreme (the top 1% receives most).

```
cache memory ≈ hot item count × (value size + per-entry overhead) × (1 + replica count)
```

Per-entry overhead is not negligible: tens of bytes per key in Redis, and in-process caches pay .NET object overhead (Concept 33). For small values, overhead can exceed the payload.

### 20b. Hit rate is a lever with a curve

The backend sees `(1 − hit rate) × read rate`:

| Hit rate | Backend share | Backend load for 100k reads/s |
|---|---|---|
| 80% | 20% | 20,000/s |
| 90% | 10% | 10,000/s |
| 99% | 1% | 1,000/s |
| 99.9% | 0.1% | 100/s |

Each extra nine of hit rate cuts backend load **10×**. That has two consequences: (1) a small improvement in hit rate can remove the need to scale the database; (2) **a cold cache or a cache flush multiplies backend load by 10–100× instantly** — the "thundering herd" after a restart or a mass expiry (Module 10). Estimate the backend's capacity *at a cold cache*, or design for gradual warm-up and request coalescing.

### 20c. What memory estimates usually decide

- **One cache node or a cluster?** Working sets of tens to a few hundred GB fit on one large node; beyond that (or beyond one shard's ops/s), shard.
- **In-process or remote?** A few hundred MB to a few GB of hot reference data per instance fits in-process (HybridCache L1); much more puts pressure on the GC (Concept 33).
- **Can we hold everything in memory?** Sometimes the honest answer is yes — memory-optimized VMs hold terabytes. "The whole dataset is 200 GB; I'd keep it in memory" is a legitimate design, if durability is handled elsewhere.

---

## Concept 21 — Concurrency: Little's Law

### 21a. The law

For any stable system:

> **L = λ × W**
> concurrent items in the system = arrival rate × time each item spends in the system

It holds regardless of distribution, which is what makes it so useful (Module 6 covered it in depth). In estimation it converts **rates and latencies into counts** — threads, connections, sessions, ports, memory.

### 21b. Uses you'll need

| Question | λ | W | L = in-flight |
|---|---|---|---|
| How many requests are in flight on the API tier? | 20,000 rps | 50 ms | **1,000** concurrent requests |
| How many DB connections do we need? | 20,000 queries/s | 5 ms per query (connection held) | **100** connections busy at once — size the pool with headroom; check the database tier's session/worker limits |
| How many threads if the code blocks on I/O? | 20,000 rps | 50 ms | **1,000 threads** — ThreadPool starvation territory; with async the same work needs a handful of threads |
| How many outbound connections to a dependency? | 5,000 calls/s per instance | 40 ms | **200** concurrent connections per instance — compare with the SNAT allocation (1,024 by default on a small Load Balancer pool) |
| How many WebSocket sessions? | 2M connects/hour (≈ 560/s) | 4 h average session | **~8M** concurrent connections |
| How many users in a purchase flow? | 300 admitted/s | 180 s | **54,000** concurrent shoppers |

### 21c. The .NET and Azure traps it reveals

- **Blocking I/O multiplies threads.** The 1,000-thread line above is why "sync over async" in ASP.NET Core collapses under load: the ThreadPool injects threads slowly, requests queue, latency rises, and Little's Law says in-flight count rises further — a feedback loop (Module 15).
- **New connection per request burns ports.** If an instance makes 2,000 outbound calls a second and opens a new connection for each, and each closed connection's port stays reserved for a while (TIME_WAIT and SNAT port reuse delays), the ports in use are roughly 2,000 × (reuse delay). Even a few tens of seconds of reuse delay puts you at tens of thousands of ports — far beyond a 1,024-port allocation. That's the arithmetic behind `IHttpClientFactory`, pooled `SocketsHttpHandler` connections, and NAT Gateway (64,512 ports per IP).
- **Connection pools must match database limits.** 50 instances × a pool of 100 = 5,000 potential connections. Many database tiers cap concurrent sessions or workers well below that; estimate the product, not the per-instance number.

### 21d. Little's Law and memory

In-flight items occupy memory. 1,000 concurrent requests × 200 KB of buffers and objects each = 200 MB live at any moment — fine. 1M concurrent WebSockets × 50 KB = 50 GB — not fine on one node. Multiply concurrency by per-item memory whenever the concurrency estimate is large.

---

## Concept 22 — Fleet size and headroom

### 22a. The formula

```
instances = peak load ÷ (measured capacity per instance × target utilization)
          then × redundancy factor, then + deployment surge
```

**Measured capacity** comes from a load test (Concept 32) or the CPU-per-request derivation (Concept 14a) until you have one.

### 22b. Target utilization: the queueing knee

Why not run at 95%? Because waiting time grows non-linearly with utilization. In the simplest queueing model (M/M/1), mean time in system = service time ÷ (1 − ρ):

| Utilization ρ | Mean time in system (× service time) |
|---|---|
| 50% | 2× |
| 70% | 3.3× |
| 80% | 5× |
| 90% | 10× |
| 95% | 20× |

Real systems aren't M/M/1 — multi-server pools behave better at moderate load — but the knee is universal, and tails degrade earlier than means. **Common targets: 50–70% CPU at peak for latency-sensitive services; higher for batch.** Marc Brooker's essay on utilization, and Module 6's treatment of queueing and the Universal Scalability Law, go deeper.

### 22c. Redundancy

- **N+1 / N+2** for instance failure.
- **Zone loss:** with three availability zones, losing one leaves two-thirds of capacity. To survive it *without* degradation, provision so that two zones can carry peak: **× 1.5** relative to the no-redundancy count.
- **Region loss (active-active):** two regions each able to carry all traffic means **× 2**.

### 22d. Deployments and autoscaling

- **Rolling deployments** take a fraction of capacity out of service (or add surge instances). Account for it, or deploy at off-peak.
- **Autoscaling has lag.** If scale-out takes 3–5 minutes (detect, provision, warm up, pass health checks) and load can grow 20% in that time, keep at least that much buffer. For container platforms with fast start and pre-provisioned capacity the lag is shorter; for VMs and cold JIT it's longer.

### 22e. Worked example

> Peak 50,000 rps. Load test: one 8-vCPU instance sustains 3,300 rps at 60% CPU with p99 inside the SLO.
> Base: 50,000 ÷ 3,300 ≈ **16** instances.
> Survive a zone loss across 3 zones: 16 × 1.5 = **24** (8 per zone).
> Plus surge for rolling deployment and autoscale lag (~15%): **~28**.

Say the final number *and* its composition: the composition is what the interviewer probes.

---

## Concept 23 — Partition and shard count

### 23a. The formula

A partition (shard) has two ceilings — throughput and storage. You need enough partitions for both:

```
partitions = max( peak throughput ÷ throughput per partition,
                  total storage ÷ storage per partition )
             × headroom (1.3–2×)
```

Then check the **hottest key** against a single partition's throughput (Concept 9c). If one key exceeds it, more partitions won't help.

### 23b. Worked examples with real limits

**Event Hubs, telemetry ingest.** 30 MB/s ingress, consumers read it twice (two consumer groups) → 60 MB/s egress.

- Partitions for ingress: 30 ÷ ~1 MB/s ≈ 30. For egress: 60 ÷ ~2 MB/s ≈ 30. Plus headroom → **32** partitions.
- Throughput units (Standard): ingress needs 30 TUs; egress 60 MB/s ÷ 2 MB/s per TU = 30 TUs → **~30 TUs**, under the 40-TU namespace limit but close — Premium is worth considering, or a second namespace.
- Consumer parallelism: at most one owning processor instance per partition per consumer group → up to 32 `EventProcessorClient` instances per group.

**Cosmos DB, order store.** 20,000 writes/s × ~5 RU + 50,000 point reads/s × 1 RU = ~150,000 RU/s; 6 TB of data.

- Throughput-driven: 150,000 ÷ 10,000 = **15** physical partitions.
- Storage-driven: 6,000 GB ÷ 50 GB = **120** physical partitions.
- Storage dominates — the provisioned 150,000 RU/s spreads to ~1,250 RU/s per partition. Now check the hottest key: if one large merchant generates 3,000 writes/s × 5 RU = 15,000 RU/s on its own key, it exceeds even a dedicated partition's 10,000 RU/s → synthetic or hierarchical partition key (merchant + order-id hash, or merchant/day) — Module 8.

Notice how often **storage drives partition count** in managed stores, and how that quietly lowers per-partition throughput. Candidates who only divide throughput by throughput-per-partition miss it.

---

## Concept 24 — Latency budgets

Module 4 introduced latency budgets as a requirement tool. Here's the arithmetic for estimating whether a design fits one.

### 24a. Composition rules

| Structure | Combined latency (rough) |
|---|---|
| Sequential calls A → B → C | Sum of the parts (means add; tails add worse than linearly) |
| Parallel calls, wait for all | Max of the parts — governed by the slowest |
| Parallel calls to *n* backends, each slow with probability p | P(request is slow) = 1 − (1 − p)ⁿ (1% each, n = 100 → 63%) |
| A retry after a timeout | Timeout + second attempt — retries trade latency for availability |
| Hedged request after delay d | Usually ~ min(first, d + second) — cuts tails at a small load cost |

### 24b. A worked budget, with physics included

> **Target:** p99 < 300 ms user-perceived for a personalized home screen. Users in Southeast Asia; app in West Europe only (~170 ms RTT).
>
> - Warm connection: 1 RTT ≈ 170 ms of pure propagation → 130 ms left for everything else at p99. Tight but possible for a single call.
> - Cold connection (TCP + TLS 1.3 + request): 3 RTT ≈ 510 ms → **infeasible**.
> - With edge termination 10 ms from the user and a warm backbone connection: 3 × 10 ms + 1 × ~170 ms ≈ 200 ms → feasible on cold starts; warm requests ≈ 180 ms.
> - The home screen calls 6 services in parallel, each p99 50 ms. The probability at least one exceeds its p99 is 1 − 0.99⁶ ≈ 6% — so the *request's* p99 is set by those services' tails beyond their own p99 (their p99.9). Budget accordingly, or return partial results with timeouts.
>
> **Conclusion:** edge termination is required, not optional; a regional deployment in Southeast Asia is the real fix if the target tightens; optional sections must be time-boxed.

### 24c. The shortcuts

- **Count round trips first, milliseconds second.** Most latency surprises are extra round trips.
- **Anything cross-region in an interactive path needs a reason.**
- **A synchronous chain of more than ~3 services rarely fits a sub-200 ms p99 budget** once tails are counted.

---

## Concept 25 — Cost

### 25a. Unit cost is the useful output

Express cost per unit of business volume — per 1,000 requests, per active user per month, per GB stored, per hour of video served. Unit cost reveals which dimension cost scales with, and therefore which design choices matter (Module 4, Concept 18).

### 25b. Rates to carry (order of magnitude, list prices, 2026 — confirm with the Azure pricing calculator)

| Resource | Order of magnitude | Notes |
|---|---|---|
| General-purpose compute | ~$30–40 per vCPU-month pay-as-you-go (Linux) | Reservations and savings plans cut 30–60%; Windows and SQL licensing add more |
| Blob Storage hot / cool / archive | ~$0.02 / ~$0.01 / ~$0.002 per GB-month | Plus per-operation charges, which matter for billions of small objects |
| Cosmos DB provisioned throughput | ~$0.008 per 100 RU/s per hour (~$5.84 per 100 RU/s per month) | Autoscale 1.5×; multi-region writes 2×; reservations discount |
| Cosmos DB storage | ~$0.25 per GB-month | **~10× hot blob** — tiering matters |
| Internet egress from Europe/North America | First 100 GB/month free, then ~$0.087/GB (first 10 TB), falling to ~$0.05/GB at high volume | CDN and negotiated pricing at very large scale |
| Inter-region transfer | ~$0.02/GB within Europe or North America; ~$0.05/GB from Europe/North America to other continents | Geo-replication and cross-region calls are billed |
| A senior engineer | More per month than many entire small cloud bills | Managed services trade money for people; count both |

### 25c. Three quick cost estimates worth practicing

1. **Throughput cost of a write-heavy store.** 900,000 writes/s × 5 RU = 4.5M RU/s → 45,000 × $5.84 ≈ **$260,000/month** for one region (Part G, Concept 35).
2. **Storage cost of history.** 1.8 PB in Cosmos DB at $0.25 ≈ **$450,000/month**; in hot blob at ~$0.018 ≈ **$33,000/month**; in cool or archive, less again.
3. **Egress cost of media.** 1 PB/month of internet egress at ~$0.05/GB ≈ **$50,000/month** — and a video platform at 100 Tbps of peak demand moves tens of petabytes *per hour* (Concept 37), which is why such platforms build or buy edge delivery rather than pay per-GB cloud egress.

**When to stop.** Cost estimates in interviews need only one or two numbers and one decision ("tier history to blob," "CDN is mandatory," "reserved capacity for the baseline"). Architect rounds go further (Concept 31).

---

# Part E — Using the numbers under pressure

## Concept 26 — The estimation block on the board

Module 3's board layout keeps requirements in a permanent left column. Estimates go directly beneath them, in a compact block that the rest of the session can point at. A template:

```
ASSUME   500M DAU · 50 msg/user/day · peak 3× · 200 B/msg · 30% online at peak
TRAFFIC  25B msg/day → ~300k/s avg → ~0.9M/s peak        so: partitioned write path
FANOUT   ×~3 deliveries (recipients × devices) → ~2.6M/s  so: delivery tier ≠ storage tier
CONN     ~150M concurrent → ~1,500 gateways @100k each     so: dedicated connection tier
HEARTBT  150M ÷ 30 s → 5M/s                                so: heartbeats > messages; keep them off the core path
STORAGE  5 TB/day → ~1.8 PB/yr (×3 replicas)               so: tier history; DB holds recent days only
```

Rules for the block:

- **Assumptions on their own line, labelled.** They're what the interviewer will correct; make them easy to find.
- **One line per estimate, three parts:** the formula's result, rounded; the comparison or decision (the "so"); nothing else.
- **No intermediate arithmetic on the board** — say it, don't write it. The board holds conclusions.
- **Round to one or two significant figures.** "~0.9M/s," never "868,056/s."
- **Leave room to revise.** When an assumption changes, strike and rewrite the affected lines visibly (Module 3, Concept 14).

---

## Concept 27 — Narration

How you *say* the estimate is graded as much as what it says.

**The four-beat pattern, for each number:**

1. **The formula in words** — *"Messages per second is users times messages per user, divided by seconds in a day."*
2. **The rounded inputs** — *"500 million times 50 is 25 billion a day; divide by about 10⁵…"*
3. **The result with its precision** — *"…roughly 250 to 300 thousand a second on average, call it a million at peak with a 3× factor and some headroom."*
4. **The "so"** — *"So writes have to be partitioned — no single primary takes a million durable writes a second."*

**Habits that make it sound senior:**

- **Invite correction at the assumption, not at the end.** *"I'll assume 50 messages a day — tell me if you have a number in mind."* Interviewers often have a reference figure; let them give it to you early.
- **Round out loud.** *"86,400 — I'll use 10⁵, which makes this about 15% low; it doesn't change the conclusion."* That one sentence shows you know the precision you're working at.
- **Skip what doesn't matter, explicitly.** *"Bandwidth for text messages is small — a few hundred MB a second — so I won't spend time on it."*
- **Don't go silent for long divisions.** If a step is genuinely hard, simplify the numbers until it isn't.
- **Use the numbers later.** Refer back to them in the high-level design and deep dives: *"Remember the 5 million heartbeats a second — that's why presence goes on its own tier."* An estimate that reappears is an estimate that mattered.

---

## Concept 28 — Thresholds that turn numbers into decisions

These are **2026 heuristics** for a well-engineered system on current cloud hardware. They're deliberately rough, and they move as hardware improves — but having *some* thresholds is what lets an estimate decide something. When you're within a factor of ~2–3 of a threshold, say so and plan an evolution path rather than pretending the line is sharp.

| Question | Below roughly… | …means | Above roughly… | …means |
|---|---|---|---|---|
| Durable writes/s to one relational primary | ~5–10k (small transactions; derive the log-rate ceiling for your tier) | One primary; no sharding | ~20–50k | Partition, batch, or move high-volume writes to a log/stream |
| Point reads/s from one relational node (cached) | ~10–50k | One primary (+ a replica for HA) | ~100k+ | Read replicas, caching, precomputation |
| Total dataset on one relational node | ~1–5 TB | Routine | Tens of TB (Hyperscale to 128 TB) | Feasible on scale-out managed tiers; beyond, partition or tier to object storage |
| Hot working set for a cache | Tens to low hundreds of GB | One cache node (+ replica) | Hundreds of GB to TB | Sharded cache cluster |
| Cache ops/s | ~100k per shard | One shard | Millions | Many shards; check hot keys |
| One key's request rate | Below one partition's or shard's ceiling | Plain partitioning works | Above it | Split the key, cache in-process, replicate the hot item |
| Event ingest | ~1–10 MB/s | A small partitioned stream | 100 MB/s–GB/s | Many partitions; Premium/Dedicated tiers or self-managed Kafka |
| Concurrent persistent connections | ~10⁴–10⁵ | A few nodes behind the normal load balancer | 10⁶–10⁸ | Dedicated connection tier, connection routing, reconnect storm design |
| Internet egress | A few TB/month | Noise in the bill | PB/month and up | CDN, edge, negotiated pricing — economics drive design |
| Latency target vs RTT to the region | Target ≥ ~4× RTT | One region is fine | Target ≤ ~2× RTT | Edge termination, regional deployment, or async UX |
| Team-operated component count | A handful of managed services | Fine for a small team | Dozens of self-hosted systems | The operability constraint binds before scale does (Module 4, Concept 17) |

**The most useful threshold of all:** *does it fit on one machine?* Modern servers have hundreds of cores, terabytes of memory and tens of terabytes of fast local flash. Many interview prompts — especially at "startup" scale — turn out to fit on one well-provisioned primary plus a replica. Saying so, with numbers, is a mark of experience; it's exactly the over-engineering trap that up-to-date "numbers to know" guides warn about.

---

## Concept 29 — Traps

| Trap | What it looks like | The fix |
|---|---|---|
| **Bits vs bytes** | "1 Gbps link moves 1 GB/s" | Divide bits by 8; write units on every number |
| **Average as design point** | Sizing the fleet for average QPS | Multiply by a stated peak factor; size queues for bursts |
| **Forgetting replication and indexes** | Raw bytes reported as storage | × index factor × replicas ÷ fill target |
| **Forgetting fan-out** | Posts/s used as the write rate | Multiply by followers / members / devices |
| **Forgetting amplification** | External rps used for internal capacity | Count internal calls, retries, polling, heartbeats |
| **DAU, MAU and registered mixed up** | Registered users × daily actions | Use the population that matches the behaviour's period |
| **Wrong period** | Per-day rate treated as per-second | Carry units; check per-user plausibility |
| **False precision** | "868,056 writes per second" | One or two significant figures |
| **Stale hardware numbers** | "SSD reads take a millisecond," "100 GB is a big database" | Use 2026 numbers (Part C); NVMe is ~20 µs; one node holds TBs |
| **Compressing the incompressible** | "We'll gzip the videos" | Media is already compressed |
| **Totals without hot keys** | 200 partitions "so it scales" | Check the hottest key against one partition |
| **Storage-driven limits ignored** | Partition count from throughput only | Also divide storage by per-partition storage |
| **Ignoring the client** | Server sized; mobile uplink not | Check client bandwidth and connection setup |
| **Estimate never used** | Numbers on the board, design unchanged | Every number gets a "so" and reappears later |
| **Overestimating every factor** | "Let's be safe" at every step | Central estimates, then headroom once |
| **Unit-price myopia** | Optimizing RU/s while storage is 2× the bill | Estimate each cost component before optimizing one |

---

## Concept 30 — Sensitivity, and calibration by level

### 30a. Find the assumption that flips the decision

After an estimate, ask: **which single assumption, if it were 10× different, would change the design?** That's the assumption to state loudest, ask about, or design to tolerate.

> *"The conclusion that one primary is enough depends mostly on the write rate. If writes were 10× higher — say, because every view is logged as an event — I'd move event logging to Event Hubs and keep the relational store for orders. Everything else is insensitive."*

A small scenario table makes this explicit when the stakes are high:

| Assumption | Low | Base | High | Decision at High |
|---|---|---|---|---|
| Messages/user/day | 20 | 50 | 150 | Same design, 3× partitions |
| Peak factor | 2× | 3× | 10× (global event) | Add admission control and queue-based leveling |
| History retention | 30 days | 1 year | Forever | Archive tier becomes the main store; cost dominates |

This is cheap in an interview — one or two sentences — and it's a defining habit at staff and architect level.

### 30b. Calibration by level

| Aspect | Mid-level | Senior | Staff | Architect |
|---|---|---|---|---|
| **Inputs** | Asks for QPS and storage | Derives from stated assumptions | Derives, and names which assumption is fragile | Derives from business drivers; ties to growth scenarios |
| **Reference numbers** | Sometimes outdated | Current latencies and capacities | Platform-specific limits (per-partition ceilings, log rates) | Limits *and* prices; operational cost |
| **Comparison** | "It's a lot" | Number vs one capacity → decision | Multiple ceilings (throughput, storage, hot key, connections) | Capacity, cost and team capacity together |
| **Scope** | Traffic and storage for everything | 3–5 numbers that matter | Picks the one estimate that reframes the problem | Runway, ranges and sensitivities for stakeholders |
| **Delivery** | Long arithmetic, false precision | Fast, rounded, narrated | Fast, and proves non-problems | Presents estimates as decisions with confidence levels |
| **Follow-through** | Numbers forgotten | Numbers reused in the design | Numbers drive the deep-dive choice | Numbers become capacity plans, budgets and fitness functions |

**The staff move: the estimate that reframes.** *"At these numbers, the messages are not the scaling problem — heartbeats are, at 5 million a second. I'd like to treat the connection tier as the core of the design."* Like requirement reframing (Module 4, Concept 27), it's powerful when grounded and damaging when it dodges the interviewer's intended problem. Offer it as a proposal.

---

# Part F — The architect variant and the .NET angle

## Concept 31 — Capacity planning and cost models

In architect rounds — and in real architecture work — estimation stops being a two-minute warm-up and becomes a deliverable: a capacity plan or a cost model that a budget holder will act on. The arithmetic is the same; the presentation and the rigor change.

### 31a. Ranges, not points

Executives and finance teams need to know how wrong you might be. Present low/base/high, with what drives the spread:

> *"Year-one infrastructure: €18–35k a month, base case €24k. The spread is mainly document storage volume — we don't yet know how many photos per claim customers will upload. We'll know within two months of launch, and the design tiers storage so the high case doesn't require rework."*

### 31b. Runway

The question capacity planning answers is usually **"when do we hit the wall?"**, not "how big is it now?":

```
runway (months) = (capacity limit − current usage) ÷ monthly growth
```

With compound growth, use doublings: at 8% monthly growth, usage doubles about every 9 months (72 ÷ 8). If the database is at 30% of its log-rate ceiling, you have a bit more than one doubling before 70% — **~12–15 months** to deliver the next step (partitioning, a write-path change). That turns an architecture decision into a dated roadmap item, which is the language organizations plan in.

### 31c. Total cost, not cloud cost

A cost model that ignores people is wrong. Self-hosting Kafka on AKS may save on the Event Hubs bill and cost a fraction of an engineer to operate; at small scale the engineer dominates, at very large scale the bill might. Put both on the same page (Module 33 develops build-vs-buy and TCO).

### 31d. Estimate → measure → refine

Architects are expected to close the loop: the estimate sets the initial design and budget; load tests and early production telemetry replace assumptions with measurements; the plan is revised on a schedule. Saying *"we'll replace these estimates with measured unit costs from the first month's billing exports and re-forecast quarterly"* is the professional version of "every number ends with so."

---

## Concept 32 — Measuring your own numbers in .NET and Azure

Interview estimates use rules of thumb; real systems use measurements. Knowing how you'd measure — and mentioning it briefly — is a credibility signal, especially in .NET-focused interviews.

| What you want to know | How to measure it |
|---|---|
| CPU time, allocations and throughput of a code path | **BenchmarkDotNet** with `[MemoryDiagnoser]` (Module 17) |
| Requests per second per instance at an SLO | **Azure Load Testing** or **k6** with a constant arrival rate (open model, to avoid coordinated omission — Module 4, Concept 10a), stepping load until p99 breaches |
| Live runtime behaviour (ThreadPool queue length, GC, exceptions, request rate) | **dotnet-counters**, **dotnet-trace**, OpenTelemetry metrics |
| RU cost of a Cosmos DB operation | `RequestCharge` on every SDK response; Cosmos DB's capacity planner for whole-workload estimates |
| Query cost in Azure SQL | Query Store, actual execution plans, `sys.dm_db_resource_stats` for log-rate and CPU percentages |
| VM-to-VM network latency | Microsoft's documented VM latency test (with latte on Windows or SockPerf on Linux) |
| Real region-to-region latency | Microsoft's published inter-region round-trip statistics |
| Unit cost | Azure Cost Management exports joined with request volume |

Two small examples of what this looks like in code.

**Capturing RU charge per operation** — the cheapest way to replace "~5 RU per write" with your real number:

```csharp
using Microsoft.Azure.Cosmos;

public sealed class RuMeasuringRepository(Container container)
{
    public async Task<double> UpsertAndMeasureAsync<T>(T item, string partitionKey, CancellationToken ct)
    {
        ItemResponse<T> response = await container.UpsertItemAsync(
            item, new PartitionKey(partitionKey), cancellationToken: ct);

        // Record into a histogram tagged by operation, not by key (keep cardinality low).
        return response.RequestCharge;
    }
}
```

Run it over a representative sample of items with your real indexing policy and you have the per-operation RU cost for the throughput estimate in Concept 23.

**Measuring payload size and serialization cost** — an estimate of bytes per message and CPU per byte in one benchmark:

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using BenchmarkDotNet.Attributes;

[MemoryDiagnoser]
public class MessageSerialization
{
    private readonly ChatMessage _message = new(
        Id: 1_234_567_890_123, ConversationId: 42, SenderId: 7,
        SentAt: DateTimeOffset.UtcNow, Text: new string('x', 120));

    [Benchmark]
    public byte[] SourceGeneratedJson() =>
        JsonSerializer.SerializeToUtf8Bytes(_message, ChatJsonContext.Default.ChatMessage);

    [GlobalSetup]
    public void ReportSize() =>
        Console.WriteLine($"Payload: {SourceGeneratedJson().Length} bytes");
}

public sealed record ChatMessage(long Id, long ConversationId, long SenderId, DateTimeOffset SentAt, string Text);

[JsonSerializable(typeof(ChatMessage))]
internal partial class ChatJsonContext : JsonSerializerContext;
```

Bytes per message × messages per second gives bandwidth and storage; nanoseconds per serialization × messages per second gives cores. Measured once, both feed back into your estimates.

**A napkin script you can keep.** Since .NET 10, a single `.cs` file can be run directly with `dotnet run estimate.cs`, without a project file. That makes a personal "napkin" calculator — the formulas from this part with your own constants — a five-minute exercise, and a good way to practise the recipes until they're automatic (Exercise 1).

---

## Concept 33 — .NET memory arithmetic

Interviewers in .NET shops sometimes probe in-process caching or large in-memory structures: *"Could we just hold this in memory in the service?"* The answer depends on CLR overheads (Module 14) that are easy to estimate once you know them.

### 33a. The constants (64-bit runtime)

| Item | Approximate cost |
|---|---|
| Object header + method-table pointer | 16 bytes per object; minimum object size 24 bytes |
| Reference field | 8 bytes |
| `string` of n characters | ≈ 22 + 2n bytes, rounded up to a multiple of 8 (UTF-16) |
| Array | ~24 bytes overhead + elements (reference arrays: 8 bytes per slot) |
| `List<T>` | Backing array grows by doubling → up to ~2× slack after growth |
| `Dictionary<TKey,TValue>` entry | hash code (4) + next index (4) + key + value, plus a 4-byte bucket slot, plus free capacity from prime-sized resizing |
| Large Object Heap threshold | Objects of 85,000 bytes or more — collected with Gen 2, not compacted by default |

### 33b. Worked example: 10 million products in an in-process dictionary

`Dictionary<Guid, Product>` where `Product` has a `Guid` ID, a 40-character name, a `decimal` price, a `long` stock count and a `DateTimeOffset` updated timestamp.

| Component | Bytes per item |
|---|---|
| Dictionary entry: hash (4) + next (4) + `Guid` key (16) + reference to value (8) + bucket (4) | ~36 |
| `Product` object: header (16) + Guid (16) + string reference (8) + decimal (16) + long (8) + DateTimeOffset (16) | ~80 |
| Name string: 22 + 2 × 40 = 102 → rounded | ~104 |
| Dictionary slack after resizing (~20–50%) | ~10–20 |
| **Total** | **~230–240** |

> 10M × ~240 B ≈ **2.4 GB** of managed heap, almost all of it long-lived in Gen 2.

**So:** it *fits* — servers have plenty of memory — but every full Gen 2 collection must mark ~20 million objects (products plus their name strings), and you're paying for this in every instance of the service. The options follow from the arithmetic:

- **Shrink the objects** — store names as UTF-8 `byte[]` or intern repeated strings; use `record struct`s in arrays instead of class instances (one array object instead of 10M objects).
- **Reduce object count** — a struct-of-arrays layout or a `FrozenDictionary` built once for read-mostly data.
- **Move it out of process** — a shared Redis/Azure Managed Redis tier, with HybridCache keeping only the hottest slice in L1.
- **Configure the GC for it** — Server GC (with DATAS, the default dynamic heap adaptation since .NET 9), and container-aware limits such as `GCHeapHardLimit` (Module 14).

The interview-grade answer is the first line of the table plus one decision: *"About 2.5 GB per instance and 20 million objects on the heap — feasible, but I'd keep the hot 10% in-process via HybridCache and the rest in Redis, so instances stay small and GC stays cheap."*

---

# Part G — Worked examples

Each example states assumptions, runs the arithmetic at interview speed, and — most importantly — shows what each number **decides**. All arithmetic was checked in code; rounded values are what you'd say out loud.

## Concept 34 — URL shortener

**Assumptions** (consumer scale): 100M new short links a month; read:write 100:1; links kept 5 years; peak factor 3× normal, 10× for viral spikes.

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Write rate | 10⁸ ÷ 2.6 × 10⁶ s | **~40/s** avg, ~120/s peak | Writes are trivial. A single relational or document store handles them with no partitioning. |
| Read (redirect) rate | 100 × 40 | **~4,000/s** avg; ~12k–40k/s peak | Reads are the system. A cache in front of the store, and edge caching if analytics allow it. |
| Records after 5 years | 10⁸ × 12 × 5 | **6 × 10⁹** links | |
| Storage | 6 × 10⁹ × ~500 B (code, URL, owner, timestamps, index share) | **~3 TB** | Fits a single managed database comfortably. Don't spend deep-dive time on storage. |
| Hot set | Top 10M links × 500 B | **~5 GB** | One cache node holds the hot set; replicate for availability, not capacity. |
| Redirect bandwidth at 40k/s | 40k × ~500 B | ~20 MB/s ≈ **160 Mbps** | Bandwidth is irrelevant. |
| Click analytics | 10¹⁰ clicks/month × ~100 B | ~**1 TB/month** | Stream clicks asynchronously (Event Hubs) into an analytical store; never write them synchronously on the redirect path. |
| ID space (7 base-62 chars) | 62⁷ ≈ 3.5 × 10¹² | 6 × 10⁹ used ≈ **0.17%** | Seven characters is plenty. |
| Random-code collisions | Birthday bound: first collision expected after ~√(πN/2) ≈ √(5.5 × 10¹²) | **~2.4 million codes** | Collisions start almost immediately with random codes — enforce uniqueness with a unique constraint/conditional insert and retry, or use a coordination-free counter-based scheme (Module 37). |

**The design fork the estimate exposes.** If the product needs every click counted, redirects must return **302** with caching disabled (or carefully short-lived), so every click reaches your edge; a cacheable **301** offloads browsers and CDNs but loses clicks. That's a requirement question (Module 4) that the read-rate estimate makes urgent: at 40k/s, *who* serves the redirect determines the whole read path.

**Hot key check.** A viral link at 50,000 redirects a second is within one cache shard's ~100k ops/s, but uncomfortably close — and a celebrity tweet can go higher. Edge caching for a few seconds, or in-process L1 caching on the redirect service, removes the hot key entirely.

**The verdict to say out loud:** *"Storage and writes are small — one database is fine. The design is the read path: cache, edge, and getting analytics off the critical path."*

---

## Concept 35 — Chat at WhatsApp scale (continuing Module 4, Concept 33)

**Assumptions** (from the requirements step): 500M DAU; 50 messages/user/day; ~200 B per message stored; 30% of DAU online at peak; peak factor 3×; groups ≤ 500; average ~2 recipients and ~1.5 devices per recipient (≈3 deliveries per message); one-year server-side history; heartbeat every 30 s.

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Messages/day | 5 × 10⁸ × 50 | 2.5 × 10¹⁰ | Sanity: WhatsApp reported ~10¹¹ a day at its peak era — we're within the right order of magnitude |
| Message rate | 2.5 × 10¹⁰ ÷ 86,400; × 3 | ~290k/s avg; **~0.9M/s peak** | Partitioned write path — by conversation, which also gives per-conversation ordering |
| Delivery rate | 0.9M × ~3 | **~2.6M deliveries/s** | Delivery (fan-out to connections) is a separate tier from storage |
| Concurrent connections | 0.3 × 5 × 10⁸ | **~150M** | A dedicated connection tier |
| Gateway nodes | 150M ÷ 100k per node; × 1.5 for zone loss | ~1,500 → **~2,250** nodes | 100k per node is an operational choice (reconnect blast radius), not a hard limit |
| Memory per gateway | 100k × ~20–50 KB | ~2–5 GB | Comfortable; memory isn't the constraint |
| Heartbeats | 150M ÷ 30 s | **~5M/s** | **More heartbeats than messages.** Terminate them at the gateway (~3,300/s per node — trivial locally); never let each heartbeat write to a central presence store. Publish presence *changes* only. |
| Ingress bandwidth | 0.9M × 200 B | ~180 MB/s ≈ 1.4 Gbps | Not a concern across thousands of nodes |
| Storage | 2.5 × 10¹⁰ × 200 B | **~5 TB/day → ~1.8 PB/year** raw (×3 replicas ≈ 5.5 PB) | The cost problem (below) |

**The managed-database check — where estimation earns its keep.** Suppose the first instinct is Cosmos DB for messages.

- **Throughput:** 0.9M writes/s × ~5 RU ≈ **4.3M RU/s** — beyond the default 1M RU/s per container (a support request), ~430 physical partitions by throughput alone, and at ~$5.84 per 100 RU/s-month roughly **$250,000/month** for one region before zone redundancy (×1.25) or extra regions.
- **Storage:** a year of history is ~1.8 PB. At ~$0.25/GB-month that's **~$450,000/month** in storage — and the minimum-throughput rule (1 RU/s per GB stored) independently forces **≥1.8M RU/s** even if nobody ever read old messages. By storage, that's ~36,500 physical partitions of 50 GB.
- **Comparison:** the same 1.8 PB in hot Blob Storage is ~$33,000/month; compressed 4–10× into a columnar or packed format and moved to cool tiers, a few thousand.

**So:** recent history (say 30 days ≈ 150 TB) in a low-latency partitioned store, older history compacted and tiered to object storage with an index for on-demand loading. Or a self-managed wide-column store — Discord's published migration of trillions of messages to ScyllaDB is the canonical case study. Either way, **the estimate moved the deep dive from "which database" to "how history is tiered"**, which is where the money is.

**The verdict:** *"Messages are a partitioned-write problem; heartbeats are the connection tier's problem; history is a cost problem. The design has three tiers because the numbers have three shapes."*

---

## Concept 36 — News feed and the celebrity problem

**Assumptions:** 300M DAU; 10 feed loads/user/day; 0.5 posts/user/day; average 200 followers; timelines cached as the latest 800 post IDs (8 B each); peak 3×; a few accounts with 100M followers.

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Feed reads | 3 × 10⁹/day ÷ 86,400; × 3 | ~35k/s avg; **~100k/s peak** | Precomputed timelines: one cache read per feed load |
| Posts | 1.5 × 10⁸/day | ~1.7k/s avg; ~5k/s peak | The post store itself is small |
| Fan-out on write | 1.5 × 10⁸ × 200 | 3 × 10¹⁰ inserts/day → ~350k/s avg; **~1M/s peak** | Fan-out, not posting, is the write volume — 200× the post rate |
| Timeline cache | 3 × 10⁸ × 800 × 8 B | **~1.9 TB raw** | Several times that with per-entry overhead in rich structures (a sorted set per user) → a sharded cluster of tens of nodes; storing timelines as compact packed arrays keeps it near raw |
| Cache ops | ~1M inserts/s + 100k reads/s | Dozens of shards for ops alone | Pipelining/batching inserts matters more than memory |
| One celebrity post | 10⁸ followers × 1 | 10⁸ inserts ≈ **100 s of the entire cluster's peak capacity** | Fan-out-on-write fails for celebrities → **hybrid**: push for normal accounts, pull for accounts above a follower threshold, merged at read time |
| Read-time cost of pull | 100k feed loads/s × ~10 followed celebrities each | ~1M celebrity-timeline reads/s, concentrated on a few thousand keys | Those keys are hot by construction → in-process L1 caching of celebrity recent posts on the feed service |

**What this example teaches.** The total numbers say "big but ordinary"; the **distribution** says "push breaks." That's Concept 9c in action: you must estimate the hottest key, not just the sum.

---

## Concept 37 — Video platform: egress decides

**Assumptions:** 1M uploads/day averaging 10 minutes (10M minutes/day); originals ~10 Mbps; an encoding ladder totalling ~17 Mbps across renditions (≈8 + 5 + 2.5 + 1 + 0.5); peak 20M concurrent viewers at an average 5 Mbps; transcoding at ~5 CPU-minutes per minute of video for the full ladder (codec- and preset-dependent — measure).

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Upload ingress | 10⁷ min × 60 s × 10 Mbps ÷ 8 | ~750 TB/day ≈ **~70 Gbps average** | Uploads go straight to blob storage via resumable, chunked uploads with short-lived SAS tokens; app servers never touch the bytes |
| Encoded output | 17 Mbps ÷ 8 × 60 s | ~128 MB per minute of video → **~1.3 PB/day** | Plus originals: ~2 PB/day. Erasure coding and tiering are mandatory, not optimizations |
| Transcoding compute | 10⁷ min × 5 CPU-min ÷ 1,440 min/day | **~35,000 cores** continuously | A batch fleet on spot/low-priority capacity; encode expensive codecs only for videos that get views |
| Streaming egress | 2 × 10⁷ × 5 Mbps | **~100 Tbps** peak ≈ 12.5 TB/s ≈ **45 PB per hour** | Not servable from an origin, and not affordable at per-GB cloud egress: even at ~$0.05/GB that's ~$2M+ *per peak hour* |
| CDN offload | 95% hit ratio | Origin still ~5 Tbps | Multi-tier caching (edge → regional → origin shield) and, at the largest scale, caches embedded in ISP networks (Netflix's Open Connect is the public example) |

**So:** the architecture of a video platform is driven by **delivery economics** first and storage second. The upload, metadata and recommendation services are ordinary; the edge is the system. An interviewer who asks for a "YouTube design" and hears this estimate in minute ten knows the candidate will spend the deep dive in the right place.

---

## Concept 38 — Metrics platform: cardinality is the scale

**Assumptions:** 50,000 monitored pods; ~2,000 active time series per pod (metrics × label combinations); 15 s scrape interval; ~16 B per raw sample; ~1.37 B per compressed sample (Gorilla-style encoding); several KB of memory per active series in the ingest tier's in-memory head.

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Active series | 5 × 10⁴ × 2 × 10³ | **10⁸** | The scale unit is series, not hosts |
| Ingest rate | 10⁸ ÷ 15 s | **~6.7M samples/s** | Partitioned ingestion by series hash |
| Raw bytes | 6.7M × 16 B | ~107 MB/s | On the wire and in WAL — fine |
| Compressed storage | 6.7M × 1.37 B × 86,400 | **~0.8 TB/day** | Storage is cheap; it's not the problem |
| Ingest-tier memory | 10⁸ × several KB | **Hundreds of GB** | Many ingester shards; memory per series is the binding constraint |
| A dashboard panel: 10,000 series × 24 h at 15 s | 10⁴ × 5,760 | ~58M points per panel | Queries over long ranges need **downsampled rollups** (5-minute rollups → 288 points per series, ~20× fewer) |
| One bad label | Adding `user_id` with 1M values to a metric with 100 series | → **10⁸ new series** — doubling the platform | Cardinality budgets and label allow-lists are requirements; this is why Module 4 said "never user ID" as a metric tag |

**So:** for observability systems, the estimate that matters is **cardinality × resolution × retention**, and the design decisions are rollups, sharding by series, and cardinality controls. Hosts and requests per second are distractions.

---

## Concept 39 — Flash-sale ticketing: the write is small, admission is the system

**Assumptions:** 10M users arrive for 50,000 seats within ~10 minutes of on-sale, most in the first minute; clients poll status every 5 s if not throttled; a purchase session lasts ~3 minutes.

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Average arrivals | 10⁷ ÷ 600 s | ~17k/s; far higher in the first seconds | Arrival spikes need a waiting room, not autoscaling |
| Polling load if unthrottled | 10⁷ ÷ 5 s | **2M rps** of mostly "still waiting" | Serve waiting-room state from the edge (static or signed-token responses); never from the booking database |
| Seat claims | 5 × 10⁴ seats over ~2–5 minutes | **~170–400 claims/s**, with retries and contention perhaps ~1–2k attempts/s | A single primary handles the *writes*; the correctness problem is contention on hot rows, solved with conditional updates/holds (Module 3's ticketing deep dive) |
| Admission rate (Little's Law) | Target concurrent shoppers ≈ inventory: L = λW → λ = 50,000 ÷ 180 s | **~280 admits/s** | Admit roughly as many concurrent shoppers as there are seats; admitting faster only creates disappointed users and load |
| Seat-map reads | Every admitted shopper refreshing every few seconds | Tens of thousands of reads/s on one event | Section-level availability cached for ~1 s; exact seat state only on the hold attempt |

**So:** the estimate splits the problem cleanly — **writes fit on one database; the system design is admission control and edge-served reads**. The worst answer here is sharding the bookings table: it solves a problem the numbers say doesn't exist and ignores the 2M rps one that does.

---

## Concept 40 — Architect: the insurer's storm surge (continuing Module 4, Concept 35)

**Context:** an EU insurer modernizing claims. Normal volume ~2,000 claims a day; storms raise it 20× for a couple of days, concentrated into ~8 daytime hours. Each claim carries ~10 photos at ~3 MB. The vendor policy-administration system exposes a read API rated at ~20 requests/s, unavailable 01:00–05:00.

| Estimate | Arithmetic | Result | So… |
|---|---|---|---|
| Storm claim rate | 40,000 ÷ (8 × 3,600 s) | **~1.4 claims/s** | Transaction volume is trivial. **This is not a scale problem.** |
| Policy lookups at intake | 1.4/s × ~2 lookups | ~3 rps vs a 20 rps limit | Intake alone fits — *if* it's the only caller |
| Status-page refreshes (if each calls policy admin) | 40,000 claimants refreshing every ~30 s | **~1,300 rps vs 20 rps** | The real risk: user-facing reads must never hit policy admin. Replicate the needed policy data (nightly export plus change capture) into the claims platform |
| Photo upload volume | 40,000 × 30 MB | ~1.2 TB/day ≈ 42 MB/s ≈ **~330 Mbps** aggregate | Trivial for Blob Storage (60 Gbps default ingress) |
| Per-user upload time | 30 MB over a ~5 Mbps cellular uplink | **~48 s** | The bottleneck is the customer's phone, in a storm, on a congested network: resumable background uploads, on-device compression, upload directly to storage |
| Normal-year document storage | 2,000/day × 30 MB × 365 | ~22 TB/year | A few hundred euros a month in hot storage; less in cool tiers — cost is not a driver |

**The architect's conclusion, stated to stakeholders:**

> *"At storm peak we receive about one and a half claims a second. Raw scale isn't our risk — integration and resilience are. The two things that would actually hurt us are customers hammering a status page that calls the policy system, which allows only twenty requests a second, and photo uploads failing on congested mobile networks. So the budget goes into asynchronous intake, a local replica of policy data, and resumable uploads — not into a large compute footprint. Infrastructure cost is small enough that managed services are clearly right for a team of twelve."*

That paragraph is the most valuable output in this module: **an estimate that tells an organization where *not* to spend money**, and redirects effort to the risks the numbers reveal.

---

## Putting it together: how this shows up in interviews

### Common questions and model answers

**"How many requests per second do you expect?"**
Derive, don't ask: *"Assume 50M DAU and 20 requests per user per day — a billion a day, about 12,000 a second on average. With a 3× daily peak, ~35,000 at peak. Writes are maybe 10% — ~3,500 a second, which one relational primary handles, so I won't shard the write path."* The last clause is the point.

**"Do we need to shard the database?"**
Compare against both ceilings: *"Peak writes ~3,500 a second; Hyperscale's log rate of 100 MiB/s at ~2 KB per transaction caps around 50,000, so writes are a fraction of the ceiling. Data is ~4 TB after three years — well within one node. No sharding now; I'll choose keys that allow partitioning later."*

**"How much storage will we need?"**
Objects × size × retention × overhead × replicas, split by store: *"Rows are ~3 TB including indexes over five years; attachments dominate at ~400 TB in blob storage; telemetry is the third biggest line. Only the attachments drive design — tiering after 90 days."*

**"How many servers do we need?"**
Derive from CPU per request or a measured capacity, then add redundancy: *"~1 ms CPU per request → ~1,000 per core; 8-vCPU instances at 60% → ~5,000 rps each. 50,000 peak → 10 instances; ×1.5 to survive a zone loss → 15; plus deployment surge → ~18. I'd confirm the per-instance figure with a constant-arrival-rate load test."*

**"Will it fit in memory?"**
Working set × (value + overhead) × replicas, then compare with one node: *"Hot set ~40 GB including Redis overhead — one node plus a replica. If you meant in-process, a few GB per instance is feasible but means tens of millions of objects for the GC, so I'd keep only the hottest slice in L1."*

**"What's the latency between regions? Can we do synchronous replication to the US?"**
Physics, then a measured number: *"About 1 ms per 100 km of fibre round trip; West Europe to East US measures ~85 ms P50. Synchronous replication adds that to every write — fine for a low-rate ledger, not for an interactive path with a 100 ms budget."*

**"Why is that cold request so slow from Asia?"**
Count round trips: *"~170 ms each way-and-back, and a cold HTTPS request needs TCP, TLS and the request itself — three round trips, ~500 ms before the first byte. Edge termination with Front Door makes two of those round trips short; HTTP/3 and connection reuse help further."*

**"How many partitions do we need in Event Hubs / Cosmos DB / Kafka?"**
Max of throughput- and storage-driven counts, then the hot key: *"Cosmos: 150k RU/s ÷ 10k = 15 by throughput, 6 TB ÷ 50 GB = 120 by storage — storage wins, so each partition gets ~1,250 RU/s. Our largest tenant alone needs ~15,000 RU/s, which exceeds any single partition, so it needs a hierarchical key."*

**"How many concurrent connections — and is that a problem?"**
Little's Law, then per-node and per-port limits: *"2M connects an hour with 4-hour sessions → ~8M concurrent. At 100k per gateway, 80 nodes plus zone headroom. Heartbeats at 30 s are ~270k messages a second — terminate them at the gateway."*

**"What would this cost?"**
One or two unit costs and a decision: *"Throughput ~4M RU/s is roughly $250k a month; a year of history in Cosmos would be another ~$450k a month versus ~$33k in hot blob storage. So the cost decision is tiering history, not tuning RU/s."*

**"Your estimate assumes 50 messages a day. What if it's 500?"**
Sensitivity: *"Then the write path needs ~10× the partitions — same design, larger fleet. What would change the design is retention: keeping history forever makes the archive tier the main store."*

**"Isn't this over-engineered for our scale?"**
Let the numbers answer: *"At 40 writes a second and 3 TB, yes — a single database and a cache are the right design. The only part that needs scale thinking is the redirect read path."* Being willing to conclude "small" is a seniority signal.

**"What are the numbers every engineer should know?"**
Give the ladder by orders of magnitude and the ratios: *"Nanoseconds for caches and memory, ~20 µs for an NVMe read, ~0.5 ms within a zone, ~1–2 ms across zones, tens of ms across a continent, ~80 ms across the Atlantic. The important ratios: remote cache is ~1,000× slower than local memory; cross-region is ~10–100× cross-zone."*

### Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| Asks "what's the QPS?" | Derives it from users and actions, states assumptions, invites correction |
| Writes four big numbers, says "so it's a lot" | Compares each number with a capacity and states the decision |
| Uses averages | Applies a stated peak factor; sizes queues for bursts |
| Ignores skew | Checks the hottest key against one partition or shard |
| Forgets fan-out | Multiplies by followers, members, devices, shards |
| Forgets amplification | Counts internal calls, retries, polling, heartbeats |
| Mixes bits and bytes | Writes units on every number; converts before comparing |
| False precision | One or two significant figures, said out loud |
| 2015 hardware numbers | 2026 numbers: NVMe ~20 µs, terabytes of RAM per node, tens of Gbps NICs |
| Shards by reflex | Proves a single node is enough when it is |
| Partition count from throughput only | Also divides storage by per-partition storage |
| Treats latency as server time only | Counts round trips, handshakes and physics |
| Estimates everything | Picks the 3–5 numbers tied to the hard part |
| Long silent arithmetic | Rounds aggressively and narrates in four beats |
| Numbers never reappear | Refers back to them in the high-level design and deep dives |
| Cost ignored or a single number | Unit cost per component; ranges and sensitivities in architect rounds |
| Optimizes the wrong cost line | Estimates each component before optimizing one |
| Forgets the client side | Checks mobile uplinks, cold connections and outbound port limits |
| No sense of fragility | Names the assumption that would flip the decision |
| (Architect) Presents a point estimate | Low/base/high, runway to the next limit, and the plan to replace estimates with measurements |

---

## Practice exercises

**Exercise 1 — Build your napkin calculator.** Write a single-file C# script (runnable with `dotnet run napkin.cs` on .NET 10) containing the constants from Part B and C and functions for Concepts 17–25: rate from DAU, storage, bandwidth, Little's Law, fleet size with zone loss, partition count (max of throughput and storage), RU cost. Use it to verify the worked examples in Part G — then put it away and redo them mentally until you match within 2×.

**Exercise 2 — The latency ladder from memory.** On a blank sheet, write the 20 rows of Concept 10's table with orders of magnitude, then the six ratios in 10c. Check against the table. Repeat daily for a week; it should take under three minutes.

**Exercise 3 — Physics drill.** Pick ten Azure region pairs, estimate the round trip from great-circle distance (1 ms per 100 km), then look up the measured P50 in Microsoft's inter-region latency table. Record the ratio. You'll internalize both the floor and the typical inflation.

**Exercise 4 — Five-minute estimates.** For each prompt, produce the estimation block (Concept 26) in five minutes: Instagram-style photo sharing, Uber-style location updates, a global rate limiter, a web crawler for 1B pages a month, a notification system sending 1B pushes a day, Google Docs-style collaboration, an ad-click aggregator, Dropbox-style sync. For each, underline the one number that decides the most.

**Exercise 5 — Prove it's small.** Take three prompts that sound big ("design Stack Overflow," "design a company's internal expense system," "design a booking system for a national museum") and produce estimates that justify a single database and a handful of instances. Practise saying the conclusion confidently.

**Exercise 6 — Measure a real capacity.** In a small ASP.NET Core service with one endpoint that reads from a database and returns JSON: run a constant-arrival-rate k6 or Azure Load Testing script stepping from 500 to 10,000 rps; find the rate at which p99 breaches 200 ms; record CPU at that point; compute CPU ms per request. Then make one endpoint block synchronously on I/O and repeat — watch Little's Law and ThreadPool starvation in dotnet-counters.

**Exercise 7 — Measure RU and bytes.** Using the Cosmos DB emulator or a free-tier account, write items of 0.5, 1, 4 and 16 KB with your intended indexing policy and record `RequestCharge` for writes, point reads and a typical query. Replace "~5 RU per write" in your calculator with your measured values.

**Exercise 8 — Architect cost model.** For a system you know, build a one-page cost model: unit costs for compute, storage (by tier), database throughput, egress and telemetry; low/base/high volumes; runway to the first hard limit (log rate, partition throughput, SNAT ports, account request rate). Write a three-sentence recommendation to a non-technical budget holder.

### Self-scoring checklist for the estimation step

| # | Check | ✓ |
|---|---|---|
| 1 | Stated assumptions on their own line and invited correction | |
| 2 | Derived rates instead of asking for them | |
| 3 | Used a stated peak factor, not averages | |
| 4 | Included fan-out and amplification where relevant | |
| 5 | Checked the hottest key, not only totals | |
| 6 | Kept units throughout; no bits/bytes slip | |
| 7 | Rounded to one or two significant figures | |
| 8 | Compared every number with a current capacity or limit | |
| 9 | Each number ended with "so…" | |
| 10 | Proved at least one component is *not* a problem, when true | |
| 11 | Counted round trips for any cross-region or client-facing latency claim | |
| 12 | Finished in 2–4 minutes (45-minute round) | |
| 13 | Referred back to the numbers later in the design | |
| 14 | Named the assumption that would flip the decision | |
| 15 | (Architect) Gave ranges, runway and a measurement plan | |

---

## Free resources

### Estimation in design interviews

| Resource | What it covers |
|---|---|
| [Hello Interview — Delivery Framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery) | Where estimation sits, and the "only if it changes the design" stance |
| [Hello Interview — Mastering Estimation](https://www.hellointerview.com/blog/mastering-estimation) | Estimating to decide, with examples |
| [Hello Interview — Numbers to Know](https://www.hellointerview.com/learn/system-design/core-concepts/numbers-to-know) | Modern hardware limits and the stale-numbers trap (introduction free; remainder premium) |
| [ByteByteGo — Back-of-the-envelope Estimation](https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation) | The classic chapter: powers of two, latency table, availability, worked Twitter estimate |
| [The System Design Primer — Appendix](https://github.com/donnemartin/system-design-primer#appendix) | Powers-of-two table, latency numbers, handy conversions |
| [Google SRE Workbook — Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/) | Google's method: concrete resource arithmetic at every stage of a design |
| [interviewing.io — A Senior Engineer's Guide to the System Design Interview](https://interviewing.io/guides/system-design-interview) | Expectations by level, including estimation |
| [Tech Interview Handbook — System Design](https://www.techinterviewhandbook.org/system-design/) | Approach and curated resources |

### Latency numbers, old and new

| Resource | What it covers |
|---|---|
| [Latency Numbers Every Programmer Should Know (gist)](https://gist.github.com/jboner/2841832) | The classic table, for historical comparison |
| [Colin Scott — Interactive latency numbers by year](https://colin-scott.github.io/personal_website/research/interactive_latency.html) | How the numbers evolved |
| [Interactive Latency Numbers — 2026 NVMe/AI era](https://philbogle.github.io/interactive-latency-numbers/) | An updated interactive table with sourcing notes |
| [Latency numbers (2026 edition) gist](https://gist.github.com/andreasbros/87fec32cf97aa41a1cbb64cc4dbdcd43) | A referenced modern table |
| [Latency Numbers You Should Know in 2026](https://omarish.com/latency-numbers-you-should-know) | Modern table with human-scale column |
| [Peter Norvig — Teach Yourself Programming in Ten Years](https://norvig.com/21-days.html#answers) | The original source of the timing table |
| [Jeff Dean — Designs, Lessons and Advice from Building Large Distributed Systems (PDF)](https://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf) | The LADIS 2009 keynote with "numbers everyone should know" and back-of-envelope examples |
| [Jeff Dean — Stanford CS295 talk (PDF)](https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf) | Back-of-the-envelope design in practice |
| [Ulrich Drepper — What Every Programmer Should Know About Memory (PDF)](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) | Why the memory rows look the way they do |

### Napkin math and Fermi estimation

| Resource | What it covers |
|---|---|
| [Simon Eskildsen — napkin-math](https://github.com/sirupsen/napkin-math) | Measured base rates and a method for napkin math, with benchmarks |
| [Simon Eskildsen — Napkin Math newsletter](https://sirupsen.com/napkin) | Worked estimation problems from real systems |
| [Wikipedia — Fermi problem](https://en.wikipedia.org/wiki/Fermi_problem) | The method and its error behaviour |
| [Sanjoy Mahajan — Street-Fighting Mathematics (MIT OCW)](https://ocw.mit.edu/courses/18-098-street-fighting-mathematics-january-iap-2008/) | Estimation, dimensional analysis and approximation as skills — free open textbook |
| [Wikipedia — Rule of 72](https://en.wikipedia.org/wiki/Rule_of_72) | Doubling times for growth and runway |
| [Wikipedia — Birthday problem](https://en.wikipedia.org/wiki/Birthday_problem) | Collision estimates for random IDs |

### Networking and the physics of distance

| Resource | What it covers |
|---|---|
| [Azure network round-trip latency statistics](https://learn.microsoft.com/en-us/azure/networking/azure-network-latency) | Measured P50 RTT between Azure regions |
| [Test VM network latency (Azure)](https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-network/virtual-network-test-latency) | Measuring latency between your own VMs |
| [Microsoft global network](https://learn.microsoft.com/en-us/azure/networking/microsoft-global-network) | The backbone behind the region numbers |
| [Azure availability zones overview](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview) | Zone design and latency expectations |
| [Ilya Grigorik — High Performance Browser Networking](https://hpbn.co/) | Latency, bandwidth, TCP, TLS, HTTP/2 — free online |
| [HPBN — Building Blocks of TCP](https://hpbn.co/building-blocks-of-tcp/) | Handshakes, slow start, bandwidth-delay product |
| [RFC 6928 — Increasing TCP's Initial Window](https://www.rfc-editor.org/rfc/rfc6928) | The 10-segment initial window |
| [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446) | One-round-trip handshake, 0-RTT |
| [RFC 9000 — QUIC](https://www.rfc-editor.org/rfc/rfc9000) | Combined transport and crypto handshake |
| [Wikipedia — Bandwidth-delay product](https://en.wikipedia.org/wiki/Bandwidth-delay_product) | Window ÷ RTT throughput limits |
| [WonderNetwork — Global ping statistics](https://wondernetwork.com/pings) | Public internet round trips between cities |
| [Cloudflare Learning — What is latency?](https://www.cloudflare.com/learning/performance/glossary/what-is-latency/) | Accessible overview of latency sources |

### Hardware, storage and failure rates

| Resource | What it covers |
|---|---|
| [Backblaze — Drive Stats for 2025](https://www.backblaze.com/blog/backblaze-drive-stats-for-2025/) | Fleet-scale annualized failure rates |
| [Google — Failure Trends in a Large Disk Drive Population](https://research.google/pubs/failure-trends-in-a-large-disk-drive-population/) | Classic study of disk failures at scale |
| [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) | Utilization, saturation, errors for capacity analysis |
| [Azure managed disk types](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types) | IOPS, throughput and latency by disk type |
| [Azure VM sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview) | vCPU, memory, network and disk caps per size |
| [Facebook — Gorilla: A Fast, Scalable, In-Memory Time Series Database (PDF)](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf) | 1.37 bytes per data point; time-series sizing |

### Queueing, capacity and overload

| Resource | What it covers |
|---|---|
| [Wikipedia — Little's law](https://en.wikipedia.org/wiki/Little%27s_law) | The law and its proof sketch |
| [Marc Brooker — Latency Sneaks Up On You](https://brooker.co.za/blog/2021/08/05/utilization.html) | Utilization, queueing and why high utilization hurts tails |
| [Google SRE Book — Handling Overload](https://sre.google/sre-book/handling-overload/) | Capacity, client-side throttling, criticality |
| [Google SRE Book — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) | What happens when estimates are wrong |
| [Azure Well-Architected — Capacity planning](https://learn.microsoft.com/en-us/azure/well-architected/performance-efficiency/capacity-planning) | Capacity planning as a practice on Azure |
| [Amazon Builders' Library — Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) | Protecting capacity at peak |
| [Amazon Builders' Library — Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) | Retry amplification and how to bound it |

### Azure limits and pricing

| Resource | What it covers |
|---|---|
| [Azure subscription and service limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits) | The master list of quotas |
| [Cosmos DB — Service quotas and default limits](https://learn.microsoft.com/en-us/azure/cosmos-db/concepts-limits) | RU/s per partition, storage per partition, minimum RU/s rules |
| [Cosmos DB — Partitioning and horizontal scaling](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning) | Physical partition limits (10,000 RU/s, 50 GB) |
| [Cosmos DB — Request units](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units) | What drives RU cost |
| [Cosmos DB — Capacity planner](https://cosmos.azure.com/capacitycalculator/) | Workload-based RU and cost estimates |
| [Cosmos DB — Understand your bill](https://learn.microsoft.com/en-us/azure/cosmos-db/understand-your-bill) | Worked billing examples |
| [Event Hubs — Scaling](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability) | Throughput units, processing units, per-partition guidance |
| [Event Hubs — Quotas and limits](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas) | Partition and namespace limits by tier |
| [Service Bus — Quotas and limits](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas) | Message size, connection and throughput limits |
| [Azure Storage — Scalability targets for standard accounts](https://learn.microsoft.com/en-us/azure/storage/common/scalability-targets-standard-account) | Request rate, ingress, egress, capacity |
| [Blob Storage — Performance checklist](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-performance-checklist) | Partitioning, parallelism and throughput guidance |
| [Azure SQL — vCore resource limits (single databases)](https://learn.microsoft.com/en-us/azure/azure-sql/database/resource-limits-vcore-single-databases) | Log rate, IOPS, workers, sessions per tier |
| [Azure SQL — Hyperscale FAQ](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-frequently-asked-questions-faq) | 128 TB, log throughput |
| [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) | Current Redis offering and SKUs |
| [Load Balancer — SNAT for outbound connections](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-outbound-connections) | Default SNAT allocation and exhaustion |
| [Azure NAT Gateway overview](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-overview) | 64,512 ports per IP; scaling outbound |
| [Azure pricing calculator](https://azure.microsoft.com/en-us/pricing/calculator/) | Turning estimates into prices |
| [Azure bandwidth pricing](https://azure.microsoft.com/en-us/pricing/details/bandwidth/) | Egress and inter-region rates |
| [Azure Blob Storage pricing](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/) | Tier prices and operation charges |
| [Azure Cosmos DB pricing](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/) | Throughput and storage prices |

### .NET measurement and memory

| Resource | What it covers |
|---|---|
| [BenchmarkDotNet](https://benchmarkdotnet.org/) | Micro-benchmarks with allocation diagnostics |
| [dotnet-counters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters) | Live runtime metrics (ThreadPool, GC, requests) |
| [dotnet-trace](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) | Tracing CPU and events |
| [.NET GC fundamentals](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals) | Generations, heap behaviour |
| [.NET Large Object Heap](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap) | The 85,000-byte threshold and its consequences |
| [ASP.NET Core performance best practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices) | Async everywhere, pooling, avoiding blocking |
| [HttpClient guidelines for .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines) | Connection reuse and port exhaustion |
| [HybridCache in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) | L1/L2 caching for hot working sets |
| [Cosmos DB .NET SDK performance tips](https://learn.microsoft.com/en-us/azure/cosmos-db/performance-tips-dotnet-sdk-v3) | Connection modes, singleton client, RU efficiency |
| [Azure Load Testing](https://learn.microsoft.com/en-us/azure/load-testing/overview-what-is-azure-load-testing) | Measured capacity per instance with pass/fail criteria |
| [k6 — Open and closed models](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/) | Constant arrival rate vs virtual users; coordinated omission |
| [TechEmpower Framework Benchmarks](https://www.techempower.com/benchmarks/) | Framework overhead in context |
| [.NET Blog — Announcing `dotnet run app.cs`](https://devblogs.microsoft.com/dotnet/announcing-dotnet-run-app/) | File-based apps for your napkin calculator |

### Case studies where the numbers drove the design

| Resource | What it covers |
|---|---|
| [Discord — How Discord stores trillions of messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Chat history at scale; hot partitions; ScyllaDB migration |
| [WhatsApp — 1 million is so 2011](https://blog.whatsapp.com/1-million-is-so-2011) | Millions of connections on one server |
| [Slack Engineering — Real-time Messaging](https://slack.engineering/real-time-messaging/) | Connection tiers and fan-out |
| [Nick Craver — Stack Overflow: The Architecture, 2016 Edition](https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/) | How few servers a very large site needed |
| [InfoQ — Timelines at Scale (Twitter)](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/) | Fan-out on write, timeline caches, celebrities |
| [Netflix Open Connect](https://openconnect.netflix.com/en/) | Why video delivery moves to the edge |
| [Hello Interview — Bit.ly breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly) | Compare with Concept 34 |
| [Hello Interview — WhatsApp breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/whatsapp) | Compare with Concept 35 |
| [Hello Interview — FB News Feed breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/fb-news-feed) | Compare with Concept 36 |
| [Hello Interview — YouTube breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/youtube) | Compare with Concept 37 |
| [Hello Interview — Metrics Monitoring breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/metrics-monitoring) | Compare with Concept 38 |
| [Hello Interview — Ticketmaster breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ticketmaster) | Compare with Concept 39 |

### Cost as an engineering discipline

| Resource | What it covers |
|---|---|
| [FinOps Foundation — What is FinOps?](https://www.finops.org/introduction/what-is-finops/) | Unit economics and cost ownership |
| [Azure Cost Management overview](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management) | Measuring actual unit cost |
| [Azure Well-Architected — Cost Optimization](https://learn.microsoft.com/en-us/azure/well-architected/cost-optimization/) | Cost models and trade-offs |

---

## Quick-recall sheet

| If you need… | Remember… |
|---|---|
| The purpose | An estimate is a comparison: needed ÷ capacity per unit → decision |
| The precision | Right power of ten, within 2–3×; thresholds are ~10× apart |
| The method | Fermi: factors you can bound, geometric-mean centres, sanity-check |
| Powers | 2¹⁰ ≈ 10³; 2³² ≈ 4.3 × 10⁹; 62⁷ ≈ 3.5 × 10¹² |
| Bits/bytes | 1 Gbps ≈ 125 MB/s; network in bits, storage in bytes |
| Time | Day ≈ 10⁵ s (86,400); month ≈ 2.6 × 10⁶ s; year ≈ 3 × 10⁷ s; 730 h/month |
| Rates | 1M/day ≈ 12/s; 1B/day ≈ 12k/s; 1/s ≈ 32M/year |
| Peaks | Global ~2–3×; regional ~3–5×; B2B ~3–10×; events 10–100×+ |
| Latency ladder | L1 1 ns · DRAM ~100 ns · NVMe ~20 µs · in-zone RTT ~0.1–0.5 ms · cross-zone ~1–2 ms · same continent 10–30 ms · transatlantic ~80 ms · Europe–Asia 170–230 ms |
| Physics | ~1 ms round trip per 100 km of fibre; measured ≈ 1.3–2.3× the floor |
| Protocols | Cold HTTPS ≈ 3 RTT; warm ≈ 1; throughput per stream ≤ window ÷ RTT |
| Bandwidth | DRAM hundreds of GB/s · NVMe ~7–14 GB/s · NIC tens of Gbps · HDD ~200 MB/s |
| App server | rps per core ≈ 1,000 ÷ CPU-ms per request; run at 50–70% |
| Relational primary | Reads 10⁴–10⁵/s; durable writes 10³–10⁴/s; log rate caps writes (100 MiB/s ÷ 2 KB ≈ 50k/s) |
| Cache shard | ~10⁵ ops/s; check hot keys |
| Cosmos DB | 1 RU/KB read, ~5 RU/KB write; 10k RU/s and 50 GB per physical partition; 20 GB per logical; 1 RU/s per GB minimum; ~$5.84 per 100 RU/s-month |
| Event Hubs | 1 TU = 1 MB/s or 1,000 events/s in, 2 MB/s out; ~1 MB/s per partition; 40 TUs per Standard namespace |
| Blob Storage | 40k req/s, 60 Gbps in, 200 Gbps out per account (major regions) |
| SNAT | LB default ~1,024 ports/instance; NAT Gateway 64,512 per IP |
| Data sizes | Guid 16 B; .NET string ≈ 22 + 2n B; object header 16 B; chat message ~200 B; photo ~3 MB; 1080p ~5 Mbps |
| Failures | Disk AFR ~1.4% → 10k disks ≈ a failure every ~3 days |
| Traffic | DAU × actions ÷ 86,400 × peak × fan-out × amplification |
| Storage | Count × size × retention × index × replicas ÷ fill |
| Cache | Working set × (value + overhead); each nine of hit rate = 10× less backend |
| Little's Law | L = λ × W for threads, connections, ports, sessions |
| Fleet | Peak ÷ (capacity × utilization) × 1.5 for zone loss + surge |
| Partitions | max(throughput ÷ per-partition, storage ÷ per-partition); then hottest key |
| Latency budget | Serial sums, parallel max, 1 − 0.99ⁿ, count round trips first |
| Cost | Compute ~$30–40/vCPU-month · blob hot ~$0.02/GB-month · Cosmos storage ~$0.25/GB-month · egress ~$0.05–0.09/GB |
| Delivery | Assumptions line, one line per estimate with "so…", narrated in four beats, reused later |
| Seniority | Prove non-problems; name the assumption that flips the decision; ranges and runway for architects |

---

## Progress

Module 5 complete — and with it, **Phase 2: The System Design Method**. Module 3 gave you the seven-step skeleton; Module 4 the first five minutes; this module the numbers that connect requirements to structure.

How it connects forward:

- **Module 6 (scalability fundamentals)** formalizes the queueing knee, Little's Law and the Universal Scalability Law that Concepts 21–22 used as rules of thumb.
- **Modules 7–8 (consistency, replication, partitioning)** turn the cross-zone and cross-region ratios and the per-partition ceilings into replication and partitioning decisions.
- **Module 10 (caching)** develops the hit-rate lever, hot keys and HybridCache's L1 that Concept 20 sized.
- **Modules 14, 15 and 17** are the .NET depth behind Concept 33's memory arithmetic, Concept 21's ThreadPool trap and Concept 32's measurement tools.
- **Module 33 (cost, build-vs-buy)** extends Concept 31's cost models into executive conversations.
- **Module 37 (worked problems)** reuses Part G's estimates as the starting point for full designs.

From here on, every design you practise should include an estimation block that passes the self-scoring checklist — and at least once per mock, an estimate that proves something *doesn't* need to scale.
