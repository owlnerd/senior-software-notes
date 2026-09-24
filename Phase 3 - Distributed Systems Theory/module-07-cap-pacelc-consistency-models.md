# Module 7 — CAP Theorem, PACELC, and Consistency Models
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Module 6 ended by making the application tier stateless and pushing every hard problem into the data tier. This module is the question that lands there: **once the same piece of data exists on more than one machine, what can you actually promise a reader?**

Every answer is a trade against latency, availability, and engineering complexity. The vocabulary for those trades — CAP, PACELC, linearizability, causal consistency, session guarantees, isolation levels — is precisely the vocabulary interviewers use to distinguish people who understand distributed systems from people who have memorized architecture diagrams. It is also the area where confident-sounding wrong answers are most common, because the industry folklore around CAP is genuinely bad.

Two framings to carry through the whole module:

1. **Consistency is a per-operation property, not a system label.** "Is your system CP or AP?" is a worse question than "what does *this* read need to guarantee?"
2. **CAP describes a narrow failure case; PACELC describes your Tuesday.** Partitions are rare. Cross-region latency is permanent.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Why consistency exists as a problem | One copy = no consistency question. Replication creates the possibility of disagreement. |
| 2 | The two C's | The C in ACID and the C in CAP are unrelated. Conflating them is a visible mistake. |
| 3 | CAP stated precisely | An impossibility result about a single linearizable register under asynchronous message loss. |
| 4 | The two-node proof | The entire theorem is one diagram: partition, write left, read right. |
| 5 | What CAP does not say | "CA" is vacuous, "P" isn't a choice, availability ≠ uptime, and labels don't survive contact with real systems. |
| 6 | Harvest and yield | Degradation is a spectrum: answer fewer requests, or answer with less data. |
| 7 | PACELC | If Partitioned: A or C. Else: L or C. The else-clause governs normal operation. |
| 8 | The physics of the else-clause | Strong consistency across regions costs a round trip you cannot argue with. |
| 9 | Two families of models | Single-object consistency models and transaction isolation levels are different axes. |
| 10 | Linearizability | Atomicity + recency + real-time order. The gold standard, and composable. |
| 11 | Sequential consistency | A total order respecting program order, with no real-time requirement. |
| 12 | Causal consistency | Happens-before preserved. The strongest model compatible with total availability. |
| 13 | Session guarantees | Read-your-writes, monotonic reads/writes, writes-follow-reads — what users actually notice. |
| 14 | Consistent prefix | You may see the past, never a shuffled past. |
| 15 | Eventual consistency | Convergence with no time bound and no ordering. Conflict resolution is the real design work. |
| 16 | Bounded staleness & PBS | Quantified lag turns "eventually" into an SLO. |
| 17 | The spectrum, assembled | One table mapping model → anomaly prevented → cost → implementation. |
| 18 | Transaction isolation levels | Snapshot isolation is not serializable; write skew is the canonical proof. |
| 19 | Quorums | W + R > N is necessary, not sufficient, for linearizability. |
| 20 | Cosmos DB's five levels | The best concrete artifact for calibrating the whole spectrum in an interview. |
| 21 | .NET in practice | Isolation levels, `TransactionScope`'s default trap, optimistic concurrency, replica routing, session tokens. |
| 22 | Choosing per operation | A decision framework, applied operation by operation to one system. |

---

## Concept 1 — Why consistency is a problem at all

With a single copy of data on a single machine, there is no consistency model to choose. Operations arrive, the machine orders them, and every reader sees the result of the last write. That is the single-copy illusion, and it is what every consistency model is trying to approximate, at varying cost.

Replication breaks it. You replicate for three reasons — availability (survive a node or region failure), read throughput (spread reads across copies), and locality (serve users near them) — and all three create the same hazard: at any instant, different copies may hold different values, because propagating a write takes time and can fail.

So the design question becomes: **how much of the single-copy illusion do you pay to preserve, and which operations get it?**

A useful mental model before any formalism: three clients, two replicas, one value. Client A writes `x = 2` to replica 1. Before replica 1's message reaches replica 2, client B reads from replica 1 (gets 2) and client C reads from replica 2 (gets 1). Nothing has failed. No bug exists. Yet B and C disagree about the present, and if B and C talk to each other — a user seeing their own comment vanish on refresh, a payment service seeing an order the order service says doesn't exist yet — the disagreement becomes a product bug. Every model below is a rule about which of these disagreements is permitted.

---

## Concept 2 — The two C's: ACID consistency vs CAP consistency

This is a small point that pays disproportionately in interviews, because getting it wrong is an instant tell.

- **The C in ACID** means *the database preserves application-level invariants*: declared constraints (foreign keys, uniqueness, check constraints) hold before and after every transaction. It is mostly a property the application defines and the database enforces. Joe Hellerstein and others have observed it's arguably the least interesting letter in ACID — it's largely a consequence of A, I, and D plus your constraints.
- **The C in CAP** means *linearizability*: a single-object recency guarantee about replicated data, defined in Concept 10. It has nothing to do with constraints.

They can vary independently. A single-node PostgreSQL instance gives you ACID-C and (trivially) CAP-C. A Cassandra cluster with `QUORUM` reads and writes can give you something close to CAP-C for single keys while offering no cross-row invariant enforcement at all. Spanner gives both. DynamoDB with eventually consistent reads gives neither in the strong sense.

If an interviewer says "so it's ACID, therefore consistent across replicas," that's an opening — not a trap to fall into.

---

## Concept 3 — CAP stated precisely

Eric Brewer's 2000 PODC keynote conjectured that a networked shared-data system cannot simultaneously provide consistency, availability, and partition tolerance. Seth Gilbert and Nancy Lynch formalized and proved it in 2002. Their definitions are narrow, and the narrowness is the whole story.

| Letter | Gilbert & Lynch's actual definition | Common misreading |
|---|---|---|
| **C** — Consistency | The system behaves as a single **atomic (linearizable) read/write register**: there exists a total order on all operations such that each appears to complete instantaneously, and reads return the last completed write. | "The data is accurate" / ACID-C / "no bugs" |
| **A** — Availability | **Every** request received by a **non-failing** node must eventually return a **non-error** response. Total availability. No timeouts, no 503s, no "try the other region." | "99.99% uptime" / an SLA |
| **P** — Partition tolerance | The network is allowed to **drop arbitrarily many messages** between nodes. | "We can survive a datacenter outage" |

Note what each definition excludes. **C** is about one object, not transactions across objects, and says nothing about isolation. **A** is an absolute liveness property over all non-failed nodes — a majority-quorum system is *not* available under this definition, because clients talking to the minority side get errors. **P** is not a feature you build; it's an assumption about the environment. Networks partition. Packets are lost. Therefore:

> **CAP is not "pick two of three." It is: when the network partitions, you must choose between consistency and availability for the affected data.**

Gilbert and Lynch also proved the more useful corollary: in the *partially synchronous* model (nodes have clocks, message delay has a bound), you can guarantee atomic consistency plus availability *in executions where no messages are lost* — which is exactly the practical regime that PACELC (Concept 7) is about.

---

## Concept 4 — The two-node proof, which you should be able to draw in ten seconds

The proof is trivially simple, which is part of why the theorem gets over-applied.

```
        ┌──────────┐         ╳ partition ╳        ┌──────────┐
 write  │ Node A   │ ─ ─ ─ ─ ─  (no msgs)  ─ ─ ─ ─│ Node B   │  read
 x=2 ──►│  x = 2   │                              │  x = 1   │──► ?
        └──────────┘                              └──────────┘
```

Two nodes, both holding `x = 1`. The network partitions. A client writes `x = 2` to node A, which acknowledges. Another client immediately reads `x` from node B. Node B has exactly two options:

1. **Return `x = 1`.** It answered, so it's available — but the read didn't see the last completed write, so the register isn't linearizable. Consistency lost.
2. **Block or return an error** until it can talk to A. Linearizability preserved — but a non-failing node failed to answer. Availability lost.

There is no third option, because by assumption no message can cross. That's the theorem. Note how much it *doesn't* cover: no transactions, no multiple keys, no notion of "how stale," no distinction between a 50 ms partition and a 5-hour one, and no way to express "B answers reads but refuses writes."

---

## Concept 5 — What CAP does not say (and why "CP vs AP" is a weak answer)

This is where a senior candidate separates from a mid-level one. The critique is well documented — Brewer's own *CAP Twelve Years Later* (2012) and Martin Kleppmann's *A Critique of the CAP Theorem* (2015) are the canonical sources — and knowing it is expected at staff/architect level.

**1. "CA" is not a meaningful category.** Since partitions happen whether you permit them or not, a system labeled CA is just a system that becomes incorrect or unavailable when partitioned, with no plan. Single-node databases are sometimes called CA, but a single node isn't a distributed system; the label is a category error. When someone says "our RDBMS is CA," what they mean is "we have one primary and we accept downtime."

**2. Availability in CAP is not availability in your SLA.** CAP-availability is binary, total, and assumes an asynchronous network with no timeouts. Real availability is a percentage measured over time and dominated by causes CAP says nothing about: bad deploys, expired certificates, schema migrations, cascading retries, capacity exhaustion, human error. A "CP" system with a 5-second leader election is more available in practice than an "AP" system whose conflict resolution silently corrupts data and requires a week-long reconciliation.

**3. Systems are not uniformly one letter.** The same database gives different guarantees per operation and per configuration:

- Cassandra with `ONE`/`ONE` is AP-ish; with `QUORUM`/`QUORUM` it's CP-ish for single keys; `LOCAL_QUORUM` changes the answer again per datacenter.
- MongoDB with `writeConcern: majority` + `readConcern: linearizable` is CP-ish; with `readPreference: secondary` it's AP-ish.
- Cosmos DB spans five levels, per account and per request.
- A "CP" quorum system is fully available on the majority side of a partition and unavailable only on the minority side — so it's not unavailable, it's unavailable *for some clients*.

**4. The proof's model is stricter than reality.** Kleppmann's core objection: CAP's availability requirement forbids *any* unavailability at *any* node, and its consistency requirement is specifically linearizability, so the theorem excludes almost everything interesting in between — bounded staleness, causal consistency, quorum systems, systems that keep serving reads but pause writes. He proposes reasoning instead about **delay sensitivity**: how does each operation's latency depend on network delay? Linearizable operations are inherently sensitive to network delay (they need a round trip to a quorum); causally consistent operations are not. That reframing is far more useful for design than a two-letter label.

**5. The interesting engineering is partition *management*, not partition *choice*.** Brewer's 2012 framing: detect the partition, enter an explicit degraded mode with reduced functionality, then recover and reconcile. Most of the work is in the third step — and "how do you reconcile?" is the follow-up question that separates people who have operated multi-region systems from people who have read about them.

**How to actually answer "is this CP or AP?" in an interview:** don't take the bait, and don't lecture either. Say something like: *"Per-operation. Checkout and inventory reservation need linearizable writes, so on a partition I'd fail them on the minority side rather than double-sell — that's CP for that path. Browsing the catalog, reading reviews, and rendering the cart summary stay available from a possibly-stale local replica — AP for that path. The system isn't one letter; the operations are."* That single sentence usually ends the theory portion of the round and moves you to design.

---

## Concept 6 — Harvest and yield: degradation as a spectrum

Before PACELC, Fox and Brewer's *Harvest, Yield and Scalable Tolerant Systems* (1999) offered a more operationally useful pair of dials than "available or not":

- **Yield** = the fraction of requests that get answered. (This is close to CAP-A, but continuous.)
- **Harvest** = the fraction of the data reflected in an answer.

You can trade them. A search index that has lost two of twenty shards can return 90% complete results with 100% yield (high yield, degraded harvest), or refuse to answer at all (perfect harvest, zero yield). Amazon's storefront during a partial failure prefers yield with reduced harvest: fewer recommendations, no "customers also bought," but the page renders and the buy button works.

This vocabulary is genuinely useful in architect rounds because it lets you describe **graceful degradation** precisely: "under a Redis outage we drop personalization (harvest) rather than returning 500s (yield); under a replica lag spike we serve stale prices with a staleness banner for browsing, but checkout reads from the primary."

---

## Concept 7 — PACELC: the formulation you should actually design with

Daniel Abadi's 2012 PACELC formulation (*Consistency Tradeoffs in Modern Distributed Database System Design: CAP is only part of the story*) fixes CAP's biggest practical omission. Read it as a conditional:

> **if (P)artition: choose (A)vailability or (C)onsistency**
> **(E)lse: choose (L)atency or (C)onsistency**

The insight is that CAP only constrains you during a failure that is, in most systems, rare — while the else-clause constrains you **all the time**. If your replicas are in different regions and you require every read to reflect every write, every write must be acknowledged by remote replicas before returning; that cost is paid on every single request, partition or no partition. That is why Abadi argued the consistency/latency trade-off influenced the design of Dynamo, Cassandra, PNUTS, and Riak *more* than the consistency/availability trade-off did.

The four cells, with the classifications usually cited (treat them as discussion starters; several are debated, and tunable systems occupy more than one cell):

| Class | Meaning | Commonly cited examples |
|---|---|---|
| **PA/EL** | Sacrifices consistency in both regimes; optimizes for staying up and being fast | Dynamo, Cassandra, Riak (default configs), DynamoDB eventually-consistent reads |
| **PC/EC** | Refuses to compromise consistency, ever; pays availability and latency | VoltDB/H-Store, BigTable/HBase, Spanner (as usually classified), etcd/ZooKeeper |
| **PC/EL** | Normally favors latency (local reads, async replication), but on a partition favors consistency | PNUTS (Abadi's motivating example), Cosmos DB in the middle levels |
| **PA/EC** | Strong consistency in normal operation, but abandons it to stay up under partition | MongoDB is often placed here (contested) |

**Why PACELC is the better interview tool.** It forces you to name the normal-operation trade-off, which is where the actual money and user experience live. A strong sentence: *"Partitions between our regions happen maybe twice a year; a cross-region round trip happens 40,000 times a second. I'm going to spend most of my design budget on the else-clause."*

Abadi later extended the point in his writing on Spanner and Calvin: Spanner is best described as PC/EC — it never gives up consistency, and in the normal case it pays the latency of a Paxos round trip (plus TrueTime commit-wait). Brewer's own *Spanner, TrueTime and the CAP Theorem* paper concedes Spanner is technically CP but argues its availability is so high (Google's private network, fast failover) that it is "effectively CA" in practice. That is a good example of the correct senior instinct: the label is technically CP, the engineering reality is that partitions on a well-run private backbone are rare enough that the trade rarely bites.

---

## Concept 8 — The physics of the else-clause

The else-clause has a hard floor set by the speed of light, and quoting real numbers here is one of the cheapest ways to sound experienced. (Module 5's latency table is the source; these are the ones relevant to replication.)

| Hop | Round-trip order of magnitude |
|---|---|
| Same rack / same AZ | ~0.25–0.5 ms |
| Cross-AZ, same region | ~0.5–2 ms |
| Cross-region, same continent (e.g. West Europe ↔ North Europe) | ~10–30 ms |
| Transatlantic (Europe ↔ US East) | ~70–90 ms |
| Trans-Pacific (US West ↔ Asia) | ~100–150 ms |
| Antipodal (Europe ↔ Australia) | ~250–300 ms |

Now apply them:

- **Single-region quorum write** (3 replicas across AZs, majority ack): adds ~1–2 ms. Essentially free. This is why "strong consistency is expensive" is false *within* a region.
- **Cross-continent synchronous replication**: a linearizable write from Europe to a quorum spanning US East costs ~80 ms *minimum*, before any processing. If a user action triggers three sequential such writes, you've spent a quarter of a second on physics alone.
- **Linearizable reads** also need coordination — either route to the leader (paying the distance to the leader's region) or read a quorum (paying the distance to the second-nearest replica). Reading "locally but linearizably" from an arbitrary region is not a thing you can buy.
- **Local reads from an async replica**: ~1 ms, and stale by whatever the replication lag is.

This yields the single most reusable line in a multi-region design round: **"Where do the writes go?"** Four answers, each with a consequence:

| Pattern | Write latency | Read latency | Consistency reality |
|---|---|---|---|
| Single write region, async read replicas | Local for one region, remote for others | Local everywhere | Stale reads; replica lag is a user-visible variable |
| Single write region, sync replication to all | Slowest region's RTT, every write | Local everywhere | Strong-ish, but write latency and availability degrade with each region added |
| Multi-write regions (active-active) | Local | Local | Conflicts are now yours to resolve — LWW, CRDTs, or app merge |
| Partitioned by home region (data residency by user) | Local for the owning region | Local for the owning region | Strong within a partition; cross-partition operations are the hard part |

The fourth is the answer that most impresses in interviews and that most real global SaaS products actually use: partition users/tenants by home region, keep each tenant's data authoritative in exactly one region, replicate read-only copies elsewhere, and handle the rare cross-region interaction explicitly.

---

## Concept 9 — Two different families of "consistency" (the map)

People collapse two distinct research traditions into one word, and untangling them is a genuine seniority signal.

**Family 1 — Distributed/shared-memory consistency models.** About *single objects* and *concurrent operations*, inherited from multiprocessor memory models. Members: linearizable, sequential, causal, PRAM, session guarantees, eventual. The question they answer: *what values may a read return, given the operations that have happened?*

**Family 2 — Transaction isolation levels.** About *multiple objects* grouped into transactions, inherited from database theory. Members: serializable, snapshot isolation, repeatable read, read committed, read uncommitted. The question they answer: *what interleavings of multi-operation transactions may be observed?*

They are orthogonal axes, and the crossover facts matter:

- **Linearizability is not serializability.** Linearizability is single-object and imposes real-time order. Serializability is multi-object and imposes *no* real-time order — a serializable system may execute your transaction as if it happened earlier, or reorder two non-overlapping transactions. Peter Bailis's *Linearizability versus Serializability* is the canonical short read.
- **Strict serializability** = serializability + real-time order. It's the multi-object generalization of linearizability, and it's what Spanner and CockroachDB (for its default) aim at. It is the strongest practically-offered guarantee.
- **Linearizability is composable (a "local" property).** If every object in your system is individually linearizable, the composed system is linearizable. Serializability is not composable in that way — combining two serializable stores does not give you a serializable whole, which is exactly why "each microservice has an ACID database" does not add up to correctness across services (Module 12's distributed-transaction problem).

Jepsen's consistency-model map (jepsen.io/consistency) draws the lattice of both families with implication arrows, and it's the single best reference to internalize. Aphyr's *Strong consistency models* post is the readable narrative version.

---

## Concept 10 — Linearizability

**Definition.** Every operation appears to take effect atomically at a single instant between its invocation and its response, and the resulting total order is consistent with real time: if operation A returned before operation B was invoked, then A precedes B in the order. Herlihy and Wing formalized it in 1990.

Three consequences worth stating explicitly:

- **Recency.** A read returns the value of the most recent completed write. No staleness at all.
- **Atomicity.** No one ever observes a half-applied operation.
- **Real-time order.** Once a write is acknowledged, *every* subsequent read from *any* client sees it (or a later value). This is what makes it feel like "one copy."

**What it's worth.** Linearizability is what lets you build things that would otherwise be impossible: uniqueness constraints, distributed locks and leader election, compare-and-set, atomic counters that must not lose increments, and any invariant where two concurrent actors must not both believe they succeeded. If an interviewer asks "what breaks without strong consistency?", the good answers are always of this shape: *two people book the last seat; a username is registered twice; a lock is held by two nodes; an account is overdrawn.*

**What it costs.**

1. **Coordination on every operation** — a quorum round trip or a hop to the leader, so latency is bounded below by network delay (this is Kleppmann's delay-sensitivity point).
2. **Availability under partition** — the minority side must refuse service.
3. **Throughput ceiling** — all operations on an object funnel through one ordering mechanism, which is Amdahl's serial fraction from Module 6 in a new costume.

**How it's implemented.** Single-leader replication with synchronous log commit to a majority (Raft/Paxos, Module 9); quorum reads and writes with read-repair or the ABD two-phase read; leases plus fencing tokens for lock-like behavior; or globally synchronized clocks with uncertainty intervals (Spanner's TrueTime + commit-wait, ~7 ms of deliberate waiting to guarantee external consistency).

**A subtlety worth knowing.** Plain quorum overlap does *not* give linearizability by itself — see Concept 19.

---

## Concept 11 — Sequential consistency

All operations appear in *some* total order that respects each individual process's program order — but that order need not match real time. If you write `x = 2` and, a full second later on a different client, someone reads `x`, sequential consistency permits them to read the old value, as long as no one ever observes an ordering that contradicts any process's own sequence.

Almost no database advertises sequential consistency (it's a memory-model concept; the CPU you're running on offers something weaker). It matters in interviews for one reason: it isolates the ingredient that makes linearizability expensive. Linearizability = sequential consistency + real-time recency, and **the real-time recency is the part that forces coordination with the outside world**. That's why "sequentially consistent" systems are still delay-sensitive but "causally consistent" ones are not.

---

## Concept 12 — Causal consistency

**Definition.** Operations that are causally related are seen by everyone in the same order; concurrent (causally unrelated) operations may be seen in different orders by different observers. Causality here is Lamport's happens-before, which arises three ways: one operation follows another in the same thread of execution; a read is followed by a write (the write may depend on what was read); or transitive closure of the first two.

**Why it's the important middle of the spectrum.** Causal consistency is (informally) **the strongest consistency model achievable in an always-available, convergent system** — this is the result from Mahajan, Alvisi and Dahlin, and it's the reason causal+ shows up in COPS, Eiger, Antidote, and most research on highly available data stores. Under partition, a causally consistent store can keep accepting reads and writes on both sides and reconcile later, because it never promised recency — only that it won't show you an effect without its cause.

**What it prevents, concretely.** The examples are the ones that show up in real bug reports:

- You delete your boss from an album's ACL, then post the photo. Under causal consistency no replica shows the photo without the ACL change (the classic Facebook example). Under eventual consistency, they might.
- A comment thread where the reply appears before the comment it replies to.
- A chat where "never mind, I fixed it" arrives before "the build is broken."

**What it doesn't prevent.** Anything requiring global agreement: uniqueness, "only one winner," non-negative balances. Two users on opposite sides of a partition can both claim the last seat, because those two writes are *concurrent* — no causal relationship exists between them — and causality has nothing to say about them.

**Implementation.** Metadata that tracks dependencies: Lamport timestamps, vector clocks / version vectors, dependency lists, or per-partition sequence numbers. Module 9 covers the mechanics. The practical cost is metadata size and the bookkeeping to garbage-collect it, which is why full causal consistency is rarer in production than the weaker session guarantees below.

---

## Concept 13 — Session guarantees: the four that users actually notice

Doug Terry and colleagues defined these in the Bayou project (1994), and they're **client-centric** rather than data-centric: they promise things about *one client's view over time* rather than about global ordering. This makes them cheap, and it makes them the most practically relevant models on the whole spectrum — most "eventual consistency ruined my UX" stories are a violation of one of these four.

| Guarantee | Promise | The bug you get without it |
|---|---|---|
| **Read your writes** (read-after-write) | A client always sees its own prior writes | User edits their profile, reload shows the old name. User posts a comment, refresh and it's gone. The single most reported replication-lag bug. |
| **Monotonic reads** | A client never sees time go backwards: later reads never return older state than earlier reads | A user refreshes and a message appears, refreshes again and it vanishes (two different replicas with different lag) |
| **Monotonic writes** | A client's writes are applied in the order it issued them | Update a document twice; the earlier version wins because it arrived at a replica later |
| **Writes follow reads** (session causality) | If a client reads value V then writes W, everyone who sees W also sees V | Someone answers a question, and the answer is visible on a replica where the question isn't |

**Session consistency** is the usual bundle of all four, scoped to a session. It requires **stickiness**: the client must be routed consistently (to one replica, or with a token carrying its position in the log). Bailis calls this *sticky availability* — session guarantees are highly available only if a client can keep talking to the same logical replica; if it loses stickiness, the system must either block or violate the guarantee. That's the same affinity trade-off from Module 6, Concept 9, appearing at the data tier.

**How you implement read-your-writes without going strong everywhere** (this is a great concrete answer):

1. **Route-to-primary window:** after a write, send that user's reads to the primary for N seconds (or until confirmed applied). Cheap, crude, effective.
2. **Read your own writes by key:** route reads of *the object you just wrote* to the primary; everything else goes to replicas.
3. **Log-position tokens:** the write returns a position (LSN, session token, commit timestamp); the client sends it back, and the replica waits until it has applied at least that position. This is what Cosmos DB session tokens and PostgreSQL LSN-based routing do. It's the best answer because it scopes the cost to exactly the requests that need it.
4. **Write-through the read path:** update the local cache/read model synchronously on write so the user's own view is correct even if the replica lags.

---

## Concept 14 — Consistent prefix reads

A reader may lag behind, but never sees writes out of order: it observes some *prefix* of the write history. You may see the past; you never see a shuffled past.

This is the guarantee you get naturally from applying a replication log in order, and it's what prevents the "reply before comment" class of anomaly without any dependency tracking — provided the writes are in the same log (same partition). It breaks at partition boundaries: two writes to different shards have no relative order, so a reader can see shard B's later write and shard A's earlier one. That's a subtle and interview-worthy point: **consistent prefix is per-partition unless you do extra work**, which is one reason "put related data in the same partition" (Module 8) is a consistency decision, not just a performance one.

---

## Concept 15 — Eventual consistency, and the conflict resolution nobody plans for

**Definition (liveness only).** If no new updates are made to an object, all replicas eventually converge to the same value. Werner Vogels's *Eventually Consistent* is the canonical industry statement.

Note the two absences: **no time bound** ("eventually" can be milliseconds or, after a partition heals, hours) and **no ordering guarantee** at all. Eventual consistency is remarkably weak as written — a system that returns arbitrary values for a year and then converges satisfies it. That's why practitioners almost always mean *eventual consistency plus some session guarantees plus a convergence rule*, and why "we'll use eventual consistency" is an incomplete answer. The follow-up is always: **"and how do you resolve conflicts?"**

The options:

| Strategy | How it works | The catch |
|---|---|---|
| **Last write wins (LWW)** | Attach a timestamp; highest wins | Silent data loss for concurrent writes, and correctness depends on clock sync. Cassandra uses client-supplied timestamps; clock skew means the "last" write may not be the latest. Riak's authors argue LWW is almost always a bug in disguise. |
| **Highest version / vector clocks + siblings** | Detect concurrency, keep all versions, hand them to the application | The application must merge (Riak siblings, Dynamo's shopping cart union). Real work, real correctness. |
| **CRDTs** | Data types whose merge is commutative, associative, idempotent, so convergence is automatic | Limited to expressible types (counters, sets, registers, sequences); some (OR-Set, RGA) carry tombstones/metadata growth. Module 9. |
| **Application-level merge** | Domain logic decides (union carts, max of read-counts, business rule for price) | Requires someone to actually think about every field |
| **Avoid conflicts structurally** | Single writer per key (partition ownership, home region), append-only event logs instead of mutable state | The best answer when you can get it |

**The "shopping cart" canon:** Dynamo's famous example. Two replicas each receive an "add item" during a partition. LWW loses one item — a customer-visible bug. Union-merge keeps both — the failure mode becomes a re-appearing deleted item, which Amazon judged acceptable versus losing a sale. That trade, stated out loud, is a perfect senior answer to "what do you do about conflicts?"

---

## Concept 16 — Bounded staleness and probabilistic staleness

"Eventually" is unquantified; **bounded staleness** quantifies it: reads may lag the latest write by at most *k* versions or *t* time, whichever comes first. This converts a vague risk into an SLO you can design against and monitor — and a recovery point objective (RPO), since the bound on lag is also the bound on data loss if you lose the primary region.

Enforcement requires teeth: if lag exceeds the bound, the system must either block reads or throttle writes. Cosmos DB does the latter (Concept 20), which is a good detail to know — *the bound is maintained by slowing writers, so you trade write availability for read freshness.*

**Probabilistically Bounded Staleness (PBS)**, from Bailis et al. at Berkeley, is the empirical cousin: rather than a worst-case bound, it models the *probability* that a read returns a value at most *k* versions or *t* milliseconds stale, based on measured replica latency distributions. Their well-known result was that in practice, partial-quorum systems returned consistent values far more often than their weak guarantees implied — e.g. Dynamo-style configurations were consistent ~99.9% of the time within a few milliseconds. The lesson for interviews: **weak guarantees usually behave well and occasionally behave terribly; design for the tail, not the mode.** Cosmos DB exposes a PBS metric in Azure Monitor for exactly this reason.

---

## Concept 17 — The spectrum, assembled

Strongest to weakest, with what each buys and what it costs. This table is worth being able to reproduce from memory.

| Model | Guarantee | Prevents | Delay-sensitive? | Available under partition? | Typical implementation | Systems |
|---|---|---|---|---|---|---|
| **Strict serializability** | Linearizability for multi-object transactions | Everything below | Yes | No | Consensus + synchronized clocks / 2PC over Paxos groups | Spanner, CockroachDB (default), FoundationDB |
| **Linearizable** | Single-object atomic + real-time recency | Stale reads, lost updates on one key, split-brain | Yes | No (minority side blocks) | Raft/Paxos leader, quorum + read repair, leases | etcd, ZooKeeper, Cosmos DB *Strong*, DynamoDB strongly-consistent reads, Mongo `linearizable` |
| **Sequential** | One total order respecting each client's program order | Reordering of a client's own ops globally | Yes | No | Total-order broadcast | Mostly theoretical / memory models |
| **Bounded staleness** | Lag ≤ *k* versions or *t* time; prefix preserved | Unbounded staleness; gives an RPO | Partly | Reads yes; writes throttled | Track lag, throttle writers | Cosmos DB *Bounded staleness* |
| **Causal (causal+)** | Happens-before order preserved; concurrent ops may diverge then converge | Effect-before-cause anomalies | **No** | **Yes** | Vector clocks, dependency tracking | COPS, Eiger, AntidoteDB, MongoDB causally-consistent sessions |
| **Session (RYW + monotonic + WFR)** | Per-client coherent view | The four anomalies in Concept 13 | No (with stickiness) | Sticky-available | Session tokens / LSN waits / sticky routing | Cosmos DB *Session* (default), Mongo causal sessions, most app-level solutions |
| **Consistent prefix** | Readers see a prefix of the write order | Out-of-order observation within a partition | No | Yes | Ordered log application | Cosmos DB *Consistent prefix*, read replicas of a single-leader DB |
| **Eventual** | Convergence, someday | Nothing, formally | No | Yes | Async replication + a merge rule | Cassandra `ONE`, DynamoDB default reads, Cosmos *Eventual*, S3 (historically), DNS |

Two things to note out loud when you draw this: the **delay-sensitive column is the real cost line** (everything above causal requires a round trip that scales with your topology), and **causal consistency is the boundary** — it's the strongest model that stays on the available, low-latency side of that line.

---

## Concept 18 — The other axis: transaction isolation levels

Your interviewer for a .NET role is at least as likely to probe this axis, because it's where day-to-day SQL Server/PostgreSQL/EF Core work lives.

**The anomalies** (the vocabulary matters more than the ANSI level names):

| Anomaly | What happens |
|---|---|
| **Dirty read** | Read uncommitted data that may roll back |
| **Dirty write** | Overwrite another transaction's uncommitted write |
| **Non-repeatable read (read skew)** | Read the same row twice in one transaction, get different values |
| **Phantom read** | A query's result set gains rows on re-execution because another transaction inserted matching rows |
| **Lost update** | Two read-modify-write cycles interleave; one update disappears (the classic counter increment) |
| **Read skew across rows** | See a state that never existed as a whole (transfer between accounts observed half-applied) |
| **Write skew** | Two transactions each read a set, each decide their write is legal, and together they violate an invariant that neither violated alone |

**Write skew is the one to be able to explain cold**, because it's the proof that snapshot isolation ≠ serializable. The canonical example: a hospital requires at least one doctor on call. Alice and Bob are both on call. Both simultaneously open a transaction, both read "2 doctors on call," both conclude it's fine to go off call, both write. Under snapshot isolation both commit — neither saw the other's write, and no row was written twice. Now zero doctors are on call. Nothing prevented it because the conflict was on a *predicate*, not a row. The same shape covers double-booking a meeting room, allowing a claim on the last unit of stock twice, and permitting two usernames when you check-then-insert.

**The levels, and what they actually do** (ANSI SQL's definitions are notoriously implementation-shaped — Berenson et al.'s *A Critique of ANSI SQL Isolation Levels* is the classic takedown):

| Level | Prevents | Still permits | Implementation |
|---|---|---|---|
| **Read uncommitted** | Dirty writes | Dirty reads and everything else | No read locks |
| **Read committed** | Dirty reads/writes | Non-repeatable reads, phantoms, lost updates, write skew | Short read locks, or MVCC snapshot per statement (RCSI) |
| **Repeatable read** (ANSI) | + Non-repeatable reads | Phantoms (in theory), write skew | Held read locks; in MySQL/InnoDB it's actually snapshot isolation |
| **Snapshot isolation** | + Read skew, phantoms for reads, lost updates (with first-committer-wins) | **Write skew**, some read-only anomalies | MVCC: transaction reads a consistent snapshot at start |
| **Serializable** | Everything | — | 2PL with range locks (SQL Server `SERIALIZABLE`), SSI (PostgreSQL `SERIALIZABLE`), or deterministic execution (Calvin/VoltDB) |

**Engine defaults you should know by heart for a .NET interview:**

- **SQL Server (boxed)**: default `READ COMMITTED` implemented with **locking** — readers block writers and vice versa. `READ_COMMITTED_SNAPSHOT ON` switches to MVCC-based RCSI (row versions in tempdb), which usually removes a whole class of blocking problems.
- **Azure SQL Database**: ships with **RCSI enabled by default** — a genuine behavioral difference from on-prem SQL Server that trips teams during migration.
- **SQL Server `SNAPSHOT`** isolation: opt-in per database (`ALLOW_SNAPSHOT_ISOLATION`), gives true snapshot isolation with update-conflict errors (error 3960) instead of blocking.
- **SQL Server `SERIALIZABLE`**: real serializability via range locks (2PL) — correct, but deadlock- and blocking-prone under contention.
- **PostgreSQL**: default `READ COMMITTED` (MVCC); `REPEATABLE READ` is actually snapshot isolation; `SERIALIZABLE` is **SSI** (Serializable Snapshot Isolation) — optimistic, so it aborts transactions with `40001 serialization_failure` and *requires your application to retry*.
- **Oracle**: `SERIALIZABLE` is snapshot isolation, not serializable. A good trivia question and a real source of production bugs.

**The crossover point back to Family 1:** serializability says nothing about real-time order, so a serializable database can still show you stale data across replicas. You need *strict* serializability (or linearizable reads) for "my write is definitely visible."

---

## Concept 19 — Quorums: why W + R > N is necessary but not sufficient

You'll cover replication mechanics in Module 8, but the consistency implication belongs here.

With *N* replicas, requiring *W* acknowledgements per write and *R* responses per read, the condition **W + R > N** guarantees that the read set and the write set overlap in at least one replica, so a read *can* observe the latest write. Common configurations: N=3, W=2, R=2 (balanced); W=N, R=1 (fast reads, slow/fragile writes); W=1, R=N (fast writes).

But overlap alone does not give you linearizability. The failure modes to name:

1. **Reading the newest value isn't enough — you must recognize it.** The reader gets two versions and needs a version/timestamp order to pick the right one. With wall-clock timestamps and clock skew, it can pick wrong.
2. **No read repair ⇒ non-monotonic reads.** Read 1 hits replicas {A,B} and sees the new value; read 2 hits {B,C}... fine. But if the quorum system doesn't write back the winning value (ABD's second phase) or repair the stale replica, later reads with different quorums can go backwards. Linearizability demands the read *also* writes back.
3. **Sloppy quorums destroy the overlap.** Dynamo-style systems, when replicas are unreachable, accept writes at *other* nodes (hinted handoff). Now W nodes acknowledged, but they may not be among the N "home" nodes, so W + R > N no longer implies overlap. Cassandra/Riak's tunable consistency is therefore "strong-ish, until failure — precisely when you needed it."
4. **Concurrent writes still conflict.** Quorums order nothing by themselves; two concurrent writes need a conflict resolution rule (Concept 15).
5. **Partial write failures leave ambiguity.** A write that reached 1 of 3 nodes and then failed is neither applied nor rolled back; the value may surface later.

So the honest statement: **quorums give you a tunable staleness/availability dial and the *possibility* of strong reads; linearizability requires quorums plus version ordering plus read repair (or, more simply, consensus).** That's exactly why Module 9 exists.

---

## Concept 20 — Cosmos DB's five consistency levels: the best concrete calibration

If you're interviewing for .NET/Azure roles, this is near-certain material, and it's also the best teaching artifact on the market because Cosmos exposes the whole spectrum as a configuration knob with published costs and TLA⁺ specifications.

| Level | Guarantee | Read cost (RU) | Read latency | Write behavior |
|---|---|---|---|---|
| **Strong** | Linearizable reads — always the latest committed write | 2× | Higher (quorum read) | Synchronously committed to a majority in every region |
| **Bounded staleness** | Consistent prefix + lag ≤ *K* versions or *T* time | 2× | Higher (quorum read) | Local majority; writes throttled if lag exceeds the bound |
| **Session** (default) | Read-your-writes, monotonic reads/writes, writes-follow-reads, consistent prefix — **within a session** | 1× | Low (single replica) | Local majority |
| **Consistent prefix** | Never see out-of-order writes; may be stale | 1× | Low | Local majority |
| **Eventual** | Convergence, no order guarantee | 1× | Lowest | Local majority |

The details that demonstrate you've actually used it:

- **Session is the default**, and it's the right default: it's the cheapest level that doesn't produce user-visible weirdness.
- Strong and bounded staleness reads are served from two replicas in a four-replica set to guarantee consistency, while session, consistent prefix, and eventual use single-replica reads — so read throughput for the two strong levels is half that of the others for the same RU cost. Write RU charges are identical across levels.
- For a single-region account the minimum K/T for bounded staleness is 10 write operations or 5 seconds; for multi-region accounts it's 100,000 write operations or 300 seconds, and that value is effectively the minimum RPO for the data.
- Strong consistency across regions more than 5,000 miles apart is blocked by default because of the write latency, and accounts with multiple write regions cannot use strong consistency at all — a distributed system can't offer RPO 0 and RTO 0 simultaneously. This is CAP/PACELC as a product constraint, which makes it an excellent thing to cite.
- **Session tokens are the mechanism, and they're your job in a multi-instance app.** Each write returns a session token; the SDK caches it per client instance. If a user's request lands on a different ASP.NET Core instance (which it will — see Module 6), that instance has a different token and can serve stale data. The fix is to flow the token: return it to the client (cookie/header) and pass it back into `ItemRequestOptions.SessionToken`.
- Consistency can be overridden per SDK instance or per request, but traditionally only *relaxed* — to move from weaker to stronger you had to change the account default. The newer `ReadConsistencyStrategy` (in the .NET SDK from v3.46 and the Java SDK from v4.69) lets you request a stronger read consistency per read without changing the account configuration. That's a current detail most candidates won't have.
- Azure Monitor exposes replication latency between regions and the PBS metric, so "how stale are we really?" is measurable rather than theoretical.

Flowing a session token through an ASP.NET Core app:

```csharp
// Write path: capture the token the write produced.
var response = await container.CreateItemAsync(order, new PartitionKey(order.CustomerId));
httpContext.Response.Cookies.Append("cosmos-session", response.Headers.Session,
    new CookieOptions { HttpOnly = true, IsEssential = true, Secure = true });

// Read path: replay it so this instance doesn't read behind the user's own write.
var token = httpContext.Request.Cookies["cosmos-session"];
var read = await container.ReadItemAsync<Order>(
    id, new PartitionKey(customerId),
    new ItemRequestOptions { SessionToken = token });

// Per-request relaxation for a genuinely tolerant read (e.g. a dashboard count):
var cheap = await container.ReadItemAsync<Order>(
    id, new PartitionKey(customerId),
    new ItemRequestOptions { ConsistencyLevel = ConsistencyLevel.Eventual });
```

---

## Concept 21 — Consistency in .NET: the concrete practice

### 21a. Isolation level control

```csharp
// EF Core: explicit isolation level per transaction.
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Snapshot);
// ... work ...
await tx.CommitAsync();
```

**The `TransactionScope` trap, which is a real interview question.** `System.Transactions.TransactionScope` defaults to **`IsolationLevel.Serializable`**. On SQL Server that means range locks, which means blocking and deadlocks that the author never asked for and rarely understands. Always pass options explicitly:

```csharp
var options = new TransactionOptions
{
    IsolationLevel = IsolationLevel.ReadCommitted,
    Timeout = TimeSpan.FromSeconds(30)
};

using var scope = new TransactionScope(
    TransactionScopeOption.Required, options, TransactionScopeAsyncFlowOption.Enabled);
// ... work ...
scope.Complete();
```

(`TransactionScopeAsyncFlowOption.Enabled` is required for the ambient transaction to survive `await` — another classic bug.) Note also that distributed transactions via MSDTC returned in .NET 7, Windows-only; reaching for them across services is almost always the wrong answer versus a saga (Module 12).

### 21b. Optimistic concurrency: the right default for lost updates

You usually don't need serializable isolation to prevent lost updates — you need a version check. EF Core makes this declarative:

```csharp
public class Product
{
    public int Id { get; set; }
    public int StockOnHand { get; set; }

    [Timestamp]                       // SQL Server rowversion
    public byte[]? Version { get; set; }
}

// Npgsql equivalent, using PostgreSQL's system column:
// modelBuilder.Entity<Product>().UseXminAsConcurrencyToken();

try
{
    product.StockOnHand -= quantity;
    await db.SaveChangesAsync();      // UPDATE ... WHERE Id = @id AND Version = @original
}
catch (DbUpdateConcurrencyException ex)
{
    // Zero rows affected ⇒ someone else changed it. Reload, re-decide, retry.
    await ex.Entries.Single().ReloadAsync();
}
```

Three patterns to have opinions about:

1. **Optimistic (rowversion + retry)** — best default for user-edited entities; conflicts are rare and a retry is cheap.
2. **Atomic conditional update** — for counters and stock, skip read-modify-write entirely and let the database do it in one statement, so there's nothing to lose:

```csharp
// Single atomic statement; the WHERE clause is the invariant.
var rows = await db.Products
    .Where(p => p.Id == id && p.StockOnHand >= quantity)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.StockOnHand, p => p.StockOnHand - quantity));

if (rows == 0) throw new InsufficientStockException(id);
```

3. **Pessimistic locking** — `SELECT ... FOR UPDATE` / `WITH (UPDLOCK, ROWLOCK)` via raw SQL when contention is high and retries would thrash. Say out loud that it trades throughput and deadlock risk for predictability.

For write skew specifically, optimistic row versions don't help (no row is doubly written). The fixes are: `SERIALIZABLE`/SSI with retry, materializing the conflict (insert a row into a `Reservations` table with a unique constraint so the database adjudicates), or serializing the decision through a single owner (a partition/actor per aggregate — Orleans grains, per-entity queues).

### 21c. Read replicas and the read-your-writes problem in ASP.NET Core

Once you route reads to replicas, you have chosen eventual consistency for those reads, whether or not you said so.

- **Azure SQL Database read scale-out**: add `ApplicationIntent=ReadOnly` to the connection string to hit a replica. Replication is asynchronous, so lag is real.
- **PostgreSQL**: Npgsql supports multiple hosts plus `Target Session Attributes=primary | prefer-standby | read-only`, so you can keep two connection strings (or one multi-host string with different attributes) and pick per operation.
- **The pattern that works:** default reads to replicas; after a write, pin that user's reads to the primary for a short window, or capture a log position and wait for it.

```csharp
// Minimal read-your-writes: a short primary-affinity window per user.
public sealed class ReadRouter(IDistributedCache cache)
{
    public Task MarkWrittenAsync(string userId) =>
        cache.SetStringAsync($"rw:{userId}", "1",
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(10) });

    public async Task<bool> MustUsePrimaryAsync(string userId) =>
        await cache.GetStringAsync($"rw:{userId}") is not null;
}
```

Better, when the store exposes it: return the commit position with the write and have the replica wait for it. PostgreSQL's `pg_current_wal_lsn()` on the primary and `pg_last_wal_replay_lsn()` on the standby let you implement exactly this; Cosmos session tokens are the managed version.

### 21d. Caching is a consistency decision

Everything in Module 10 applies, but state the coupling now: an `IMemoryCache` per instance means *N* independent replicas of your data with no invalidation protocol. `HybridCache`'s L1 means one instance can serve a value that another instance already invalidated. A cache is a replica; giving it a TTL is choosing bounded staleness; choosing write-through versus invalidate-on-write is choosing your consistency model. The interview-ready phrasing: *"The cache TTL is my staleness bound, and I picked 30 seconds because the business said price changes may take up to a minute to appear but must never appear on the payment page — so the payment path reads through to the primary."*

### 21e. The dual-write problem

Writing to a database and then publishing an event (or updating a cache, or calling another service) is **two writes with no atomicity**. Crash in between and your system is permanently inconsistent — a state no consistency model will save you from, because the inconsistency is *between* stores. The fix is to make it one write plus asynchronous propagation: the **transactional outbox** (write the event into the same transaction, ship it with a relay) or change data capture. Module 11 covers it; mention it the moment a design has "save to DB, then publish to Service Bus."

---

## Concept 22 — Choosing: a per-operation decision framework

The senior move is to refuse to answer "what consistency model does the system use?" and instead answer per operation. Four questions per operation:

1. **What invariant must hold?** Uniqueness, non-negative balance, single-winner → you need linearizable writes or a single owner. "Roughly correct count" → eventual is fine.
2. **Who observes a violation, and what does it cost?** A user seeing their own stale profile (annoying, common) vs two customers sold the same seat (refunds, support, trust).
3. **Is it reversible?** If an occasional conflict can be compensated after the fact (cancel and apologize, re-ship, credit the account), you can often take availability over consistency. Hotels and airlines overbook on purpose. Bank ledgers don't.
4. **What's the latency budget and where's the user?** If the operation is on a p99-200 ms path and the leader is 80 ms away, linearizable reads are already half your budget.

Worked application — a multi-region .NET commerce platform (US East, West Europe, Southeast Asia), Cosmos DB or PostgreSQL + Redis:

| Operation | Requirement | Choice | Why |
|---|---|---|---|
| Browse catalog | High availability, low latency | Eventual / CDN + local replica, TTL 60 s | Stale product copy is harmless; availability is revenue |
| Product price on the product page | Bounded staleness | Cache TTL 30 s, replica read | A minute-old price is acceptable for browsing |
| Price at checkout | Linearizable read | Primary read, or re-validate at authorization | Charging the wrong price is a legal/trust problem |
| Add to cart | Availability, RYW | Session consistency; cart is user-partitioned | User must see their own cart; conflicts merge by union |
| Inventory reservation | Linearizable, single-winner | Single write region per SKU-partition + atomic conditional update | Overselling is the invariant; this is the one place to pay |
| Place order | Strict serializability for the order aggregate | Single-region write, transactional outbox for downstream | Money; also the point where the saga starts |
| Payment capture | Idempotent + linearizable | Idempotency keys with a uniqueness constraint | Retries must not double-charge; the constraint needs linearizability |
| Order history | Consistent prefix + RYW | Read model with session token | User must see the order they just placed; ordering must not scramble |
| Product reviews / ratings | Eventual | Async aggregation, CRDT counter | Nobody notices a 2-minute-old average |
| "X people viewing this" | Eventual, best effort | Local counters, approximate | Correctness is decorative |
| User login / session | RYW + monotonic | Session; short-lived JWT with local validation | Stale permissions are a security issue → short TTL + revocation list |
| Password / permission change | Linearizable | Primary write, propagate with an explicit invalidation | Security invariant; must not be served from stale cache |

Notice the shape of the result: **two or three operations need strong guarantees; everything else doesn't.** That's the answer interviewers want — targeted strength, not a blanket choice. And the corollary: design the *data model* so the strong operations are single-partition, because a linearizable operation confined to one partition is cheap, while one spanning partitions needs consensus or 2PC.

A last framing worth knowing for architect rounds: the research direction called **coordination avoidance** (Bailis et al.) and the **CALM theorem** (Hellerstein and Alvaro) formalize when you can skip coordination entirely. The practical test is **invariant confluence**: if merging any two valid states always yields a valid state, the invariant can be maintained without coordination. "Total is non-negative" is *not* invariant-confluent (two valid deductions merge into an overdraft), so it needs coordination. "Set of cart items" *is*, so it doesn't. Also useful: Bailis's HAT result — read committed, read atomic, monotonic atomic view, and causal/session guarantees are achievable with high availability; snapshot isolation, serializability, and preventing lost update or write skew are **not**. That table tells you exactly which correctness properties cost you availability.

---

## Putting it together: how this shows up in interviews

### The five-minute set piece

If you get "explain CAP," a strong answer is roughly 60 seconds and moves immediately to design:

> "CAP is a narrow result: for a single linearizable register on an asynchronous network, when a partition occurs you must choose between answering requests and answering correctly. Partition tolerance isn't optional, so it's really a binary choice during a failure. The more useful formulation is PACELC, which adds the else-clause: even with no partition, you're trading consistency against latency on every request, because a linearizable operation needs a round trip to a quorum or a leader. In practice I don't label a system CP or AP — I go operation by operation. Here, inventory reservation needs linearizability so I'd fail writes on the minority side during a partition; catalog browsing gets a stale local read so it stays up. Then the real work is the degraded mode and the reconciliation."

### Worked example: a multi-region ledger with a global uniqueness constraint

**The problem.** Global payments platform, three regions, requirements: no double-spend, every transaction has a globally unique idempotency key, users see their own balance immediately, and the balance page must render in under 200 ms worldwide.

**Step 1 — Classify the operations.** Balance *display* tolerates staleness. Balance *mutation* does not. Idempotency-key uniqueness is a hard global invariant. Transaction history needs consistent prefix + RYW.

**Step 2 — Kill the global invariant if you can.** A globally unique key doesn't need global coordination if you make the key *structurally* unique: prefix it with the region or the account ID, so uniqueness is enforced within one partition owned by one region. This is the highest-value move in the whole design — you've turned a coordination problem into a naming problem. (Same trick: ULIDs/Snowflake IDs instead of a global sequence.)

**Step 3 — Partition by home region.** Each account is homed in exactly one region and all mutations for it are routed there. Within the region, writes are linearizable via a Raft/quorum commit across AZs, costing ~1–2 ms — cheap. Cross-region reads get async replicas. Balance display reads locally (stale by the replication lag, typically <100 ms) with the user's own recent writes handled by a session token so they see their own transfers immediately.

**Step 4 — Handle the cross-region case explicitly.** An account-to-account transfer between accounts homed in different regions cannot be one linearizable transaction without a cross-region round trip. Two options, and naming both is the senior signal: a **two-phase reservation saga** (debit-pending in region A, credit in region B, confirm, with a compensating credit on failure) — available, eventually consistent, requires idempotency and reconciliation; or a **strictly serializable cross-region transaction** (Spanner-style) — correct, simple to reason about, and ~150 ms slower. For a ledger I'd argue for the saga with a reconciliation job and a hard invariant check, because the latency is user-visible and the compensation is well-defined in accounting terms (reversals are normal in finance).

**Step 5 — Degraded mode.** If region B is partitioned: accounts homed in B are unavailable for mutation from A (CP for those) — return an explicit "temporarily unavailable," not a stale write. Reads of B-homed accounts from A continue from the replica with a staleness indicator. Write availability is restored by failover, which is an RPO decision: with async replication you may lose the last few hundred milliseconds of writes, so the ledger needs a synchronous quorum within region and an explicit decision about whether cross-region failover is automatic (fast, possible data loss) or manual (slow, safe). For money: manual, with a documented runbook.

**Step 6 — Name your bottleneck.** The per-account write path (one owner, serialized) is the Amdahl fraction; hot accounts (a merchant receiving 10k payments/sec) need batching or per-account sharding of the ledger with periodic rollup.

### Common questions and what a strong answer contains

**"Explain CAP."** As above: precise definitions, P isn't a choice, the real choice is per-operation during a partition, then pivot to PACELC.

**"Is Cosmos DB / Cassandra / Mongo CP or AP?"** "Configurable, per operation." Then give the specific knob: Cosmos's five levels; Cassandra's `ONE`/`QUORUM`/`LOCAL_QUORUM` plus the sloppy-quorum caveat; Mongo's `writeConcern`/`readConcern`/`readPreference`.

**"What's the difference between linearizability and serializability?"** Single-object + real-time order vs multi-object + any order. Strict serializability is both. Mention that linearizability composes and serializability doesn't, which is why per-service ACID databases don't give you cross-service correctness.

**"Why is snapshot isolation not serializable?"** Write skew, with the on-call-doctors or last-seat example, and the fix options (SSI with retries, materialize the conflict as a row with a unique constraint, or serialize through one owner).

**"Our users say their profile edit doesn't show up."** Read-your-writes violation from replica lag. Then the four mitigations from Concept 13, and pick the log-position token because it scopes the cost.

**"Eventual consistency is fine here, right?"** Ask what converges and how. Then: what's the merge rule, what's the staleness bound, which session guarantees do we still need, and what does the user see during the window.

**"How do you prevent double-booking the last seat across two regions?"** Don't try to do it consistently in two places. Own each inventory partition in one region (single writer), make the decrement an atomic conditional update, and either fail or queue requests from other regions. If the business tolerates it, overbook deliberately and compensate — and say that out loud as a business decision, not a technical failure.

**"What's the cost of strong consistency?"** Latency bounded by network delay (give the numbers), availability of the minority side under partition, throughput limited by the ordering mechanism, and double the read cost in systems like Cosmos. Then note that *within* a single region it's cheap and the real cost appears only across regions.

**"Would you use distributed transactions?"** Within a single database, yes, a plain transaction. Across services, no: 2PC couples availability (any participant down blocks everyone) and holds locks across the network. Use sagas with idempotency and compensation, plus an outbox for the dual-write problem (Module 12).

### Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "Pick two of three." | Explains that P isn't optional, so it's C-or-A *during a partition*, per operation. |
| Using ACID-C and CAP-C interchangeably. | Distinguishes them in one sentence and moves on. |
| Labeling the whole system CP or AP. | Classifies operations, and names which ones pay for strength. |
| "We'll use eventual consistency." | Specifies the convergence rule, the staleness bound, and which session guarantees are retained. |
| Treating linearizability and serializability as synonyms. | Knows the single-object/real-time vs multi-object/reordering distinction. |
| Assuming snapshot isolation is safe for invariants. | Reaches for write skew immediately and offers three fixes. |
| Believing W + R > N means strong consistency. | Adds version ordering, read repair, and the sloppy-quorum caveat. |
| Ignoring conflict resolution in active-active. | Names LWW's data loss, vector clocks/siblings, CRDTs, and structural avoidance. |
| Forgetting that caches and read replicas are replicas. | Treats TTL as a staleness SLO and routes critical reads to the primary. |
| "Save to the DB then publish the event." | Flags the dual-write problem and reaches for an outbox. |
| Claiming strong consistency is always too slow. | Distinguishes intra-region (~1–2 ms, cheap) from cross-region (~80–150 ms, expensive). |
| No numbers. | Quotes RTTs, lag bounds, and RU multipliers to justify each choice. |

---

## Practice exercises

**Exercise 1 — Break read-your-writes on purpose.** Run an ASP.NET Core API against PostgreSQL with one primary and one streaming replica (Docker Compose or .NET Aspire). Route all reads to the replica, add an artificial 500 ms replication delay (`recovery_min_apply_delay`), and click through a profile edit. Then fix it three ways: primary-affinity window, per-entity primary read, and LSN-token wait (`pg_current_wal_lsn()` on write, poll `pg_last_wal_replay_lsn()` on read). Measure the added latency of each. This is the single most interview-relevant exercise in the module.

**Exercise 2 — Reproduce write skew, then defeat it.** With EF Core against PostgreSQL, write a test that runs two concurrent transactions implementing "at least one doctor must remain on call" under `REPEATABLE READ` (snapshot isolation). Confirm both commit and the invariant breaks. Then fix it three ways: switch to `SERIALIZABLE` and handle `40001` with a retry policy (Polly, Module 25); materialize the constraint as a row with a unique index; and serialize through a single owner. Compare throughput under contention with BenchmarkDotNet or a simple load loop.

**Exercise 3 — Walk the Cosmos spectrum.** Create a Cosmos DB account with two regions. Write a small .NET console app that writes an item and immediately reads it from the secondary region at each of the five consistency levels, 1,000 times, recording RU charge, latency, and staleness rate. Then run it again while dropping the session token on the read path, and watch session consistency degrade. Compare your measured staleness against the account's PBS metric in Azure Monitor.

**Exercise 4 — A consistency matrix for your own project.** Take one of your ambitious personal projects, enumerate every read and write operation, and fill in the Concept 22 table: invariant, observer, reversibility, latency budget, chosen model, mechanism. Anything you can't justify with an invariant should be eventual. You'll usually find that fewer than 20% of operations need strong guarantees — and the exercise of proving that is exactly what an architect round tests.

**Exercise 5 — Read a Jepsen analysis properly.** Pick one (etcd, MongoDB, Cassandra, Redis Raft, PostgreSQL) from jepsen.io/analyses and write a one-page summary: what guarantee was claimed, what anomaly was found, what the minimal counterexample history was, and how the vendor responded. You'll absorb more about consistency models from one careful read of an analysis than from a week of blog posts, and being able to say "Jepsen found X in version Y" about a database you're proposing is a strong credibility marker.

**Exercise 6 — Five-minute verbal drill.** Timed, out loud: "Design the consistency model for a global ride-hailing app: driver location updates, ride matching, surge pricing, payment, ride history." Check afterward that you said *per-operation*, named where writes go, gave at least two real latency numbers, identified the one operation that needs linearizability (matching a driver to a rider — a single-winner problem), and described the degraded mode.

---

## Free resources

### The foundational papers (all free)

| Resource | What it covers | Why read it |
|---|---|---|
| [Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services](https://www.comp.nus.edu.sg/~gilbert/pubs/BrewersConjecture-SigAct.pdf) — Gilbert & Lynch, 2002 | The actual CAP proof, with the precise definitions | Nine pages; reading the real definitions permanently inoculates you against folklore |
| [CAP Twelve Years Later: How the "Rules" Have Changed](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) — Eric Brewer, 2012 | Brewer walking back "2 of 3"; partition management; harvest/yield | The author of the conjecture explaining how it's misused |
| [A Critique of the CAP Theorem](https://martin.kleppmann.com/2015/09/17/critique-of-the-cap-theorem.html) — Martin Kleppmann, 2015 ([arXiv:1509.05393](https://arxiv.org/abs/1509.05393)) | Definitional problems; the delay-sensitivity alternative | The rigorous critique, and the framework worth adopting instead |
| [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) — Kleppmann | Why the labels don't fit real systems | Short; gives you the exact language for the interview answer |
| [Consistency Tradeoffs in Modern Distributed Database System Design (PACELC)](https://cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) — Daniel Abadi, 2012 | The PACELC formulation and system classification | Six pages, and the single most useful reframing in this module |
| [Problems with CAP, and Yahoo's little known NoSQL system](https://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html) — Abadi | The blog post that became PACELC, via PNUTS | The clearest motivation for the else-clause |
| [Spanner, TrueTime and the CAP Theorem](https://research.google/pubs/spanner-truetime-and-the-cap-theorem/) — Brewer, 2017 | Why Spanner is technically CP but "effectively CA" | Great model for how to talk about theory vs operational reality |
| [Linearizability: A Correctness Condition for Concurrent Objects](https://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf) — Herlihy & Wing, 1990 | The original definition, and composability | The source; skim sections 1–3 |
| [Eventually Consistent](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) — Werner Vogels | Eventual consistency, session guarantees, quorum trade-offs from Amazon's perspective | The industry's foundational statement, and it names the session guarantees |
| [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — DeCandia et al., 2007 | Sloppy quorums, vector clocks, hinted handoff, shopping-cart merge | The canonical AP system; the conflict-resolution discussion is the valuable part |
| [Highly Available Transactions: Virtues and Limitations](https://arxiv.org/abs/1302.0309) — Bailis et al., 2013 | Exactly which isolation/consistency levels are achievable with high availability | The definitive map of what availability costs you in correctness |
| [Coordination Avoidance in Database Systems](https://arxiv.org/abs/1402.2237) — Bailis et al., 2014 | Invariant confluence: when coordination is provably unnecessary | The architect-level frame for "do we actually need to coordinate?" |
| [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/abs/cs/0701157) — Berenson, Bernstein, Gray et al., 1995 | Why the ANSI levels are underspecified; snapshot isolation introduced | Where write skew and the modern vocabulary come from |
| [Replicated Data Consistency Explained Through Baseball](https://www.microsoft.com/en-us/research/publication/replicated-data-consistency-explained-through-baseball/) — Doug Terry, MSR | Six consistency levels mapped to who needs what in a baseball game | The best intuition pump in the literature; ideal for explaining this to non-specialists |

### The best explainers

| Resource | What it covers |
|---|---|
| [Jepsen: Consistency Models](https://jepsen.io/consistency) | The definitive map of both families with implication arrows; individual pages per model |
| [Strong consistency models](https://aphyr.com/posts/313-strong-consistency-models) — Kyle Kingsbury | The readable narrative walk from linearizability down to eventual |
| [Jepsen analyses](https://jepsen.io/analyses) | Real databases tested against their claims; etcd, Mongo, Cassandra, Postgres, Redis |
| [Linearizability versus Serializability](http://www.bailis.org/blog/linearizability-versus-serializability/) — Peter Bailis | The clearest short treatment of the distinction |
| [Distributed Systems for Fun and Profit](http://book.mixu.net/distsys/single-page.html) — Mikito Takada | Free short book; chapters 2–5 cover consistency, time, and replication |
| [Martin Kleppmann's distributed systems lecture series](https://martin.kleppmann.com/) | Free Cambridge lecture videos and notes (linked from his site), covering causality, consistency, and consensus |
| [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.5840/) | Free lecture notes and paper list; the standard graduate treatment |
| [CMU 15-445 Database Systems](https://15445.courses.cs.cmu.edu/) | Free lectures on concurrency control, MVCC, and isolation |
| [Hermitage](https://github.com/ept/hermitage) — Kleppmann | Executable test suite showing what each engine's isolation levels *actually* do — the fastest way to stop trusting documentation |
| [Serializability, linearizability, and locality](https://www.cockroachlabs.com/blog/consistency-model/) — Cockroach Labs | A careful engineering treatment of the composability point |
| [Elle](https://github.com/jepsen-io/elle) | The transactional-anomaly checker Jepsen uses; the README is a great anomaly taxonomy |

### .NET, Azure, and engine documentation

| Resource | What it covers |
|---|---|
| [Consistency levels in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels) | The five levels, RU and throughput implications, K/T minimums, region constraints |
| [Manage consistency levels in Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-manage-consistency) | Per-request overrides, `ReadConsistencyStrategy`, session-token handling, the PBS metric |
| [azure-cosmos-tla](https://github.com/Azure/azure-cosmos-tla) | TLA⁺ specifications of all five Cosmos levels — rare and excellent for building precise intuition |
| [SET TRANSACTION ISOLATION LEVEL (T-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql) | SQL Server's levels, including `SNAPSHOT` and `READ_COMMITTED_SNAPSHOT` |
| [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) | Read committed, repeatable read (= SI), and SSI, with the retry requirement spelled out |
| [PostgreSQL SSI wiki page](https://wiki.postgresql.org/wiki/SSI) | Worked write-skew examples and how SSI detects them |
| [EF Core: Concurrency conflicts](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) | `rowversion`, concurrency tokens, `DbUpdateConcurrencyException`, resolution strategies |
| [EF Core: Transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions) | Explicit transactions, isolation levels, `TransactionScope` interop, execution strategies |
| [TransactionScope](https://learn.microsoft.com/en-us/dotnet/api/system.transactions.transactionscope) | Including the `Serializable` default and `TransactionScopeAsyncFlowOption` |
| [Azure SQL Database read scale-out](https://learn.microsoft.com/en-us/azure/azure-sql/database/read-scale-out) | `ApplicationIntent=ReadOnly`, replica lag characteristics |
| [Npgsql failover and load balancing](https://www.npgsql.org/doc/failover-and-load-balancing.html) | Multi-host connection strings and `Target Session Attributes` for primary/replica routing |
| [DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) | Eventually vs strongly consistent reads, and their cost difference |
| [MongoDB read concern](https://www.mongodb.com/docs/manual/reference/read-concern/) / [write concern](https://www.mongodb.com/docs/manual/reference/write-concern/) | `majority`, `linearizable`, causally consistent sessions |
| [Data Consistency Primer](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn589800(v=pandp.10)) — Microsoft patterns & practices | Microsoft's own strong-vs-eventual guidance, written for architects |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| CAP | Precise definitions; P isn't optional; the real choice is C-or-A during a partition, per operation |
| "Is it CP or AP?" | "Per operation" — then classify two operations in opposite directions |
| The better framework | PACELC: else-clause = latency vs consistency, paid on every request |
| Why strong consistency costs | Delay sensitivity: a round trip to a quorum or leader; ~1–2 ms in-region, ~80–150 ms cross-region |
| ACID-C vs CAP-C | Invariants vs linearizability; unrelated |
| Linearizability | Atomic + recent + real-time order; single object; composable |
| Serializability | Multi-object, any order; strict serializability adds real time |
| The strongest available model | Causal consistency; session guarantees are the practical subset |
| What users actually notice | Read-your-writes and monotonic reads |
| Eventual consistency | Convergence with no bound; the real question is the merge rule |
| Quantifying staleness | Bounded staleness (k versions / t seconds) = your RPO; PBS to measure reality |
| Snapshot isolation | Not serializable — write skew; three fixes |
| Quorums | W + R > N is necessary, not sufficient; version order + read repair, and sloppy quorums break it |
| Cosmos DB | Five levels, session is the default, strong costs 2× RU and is blocked >5,000 miles |
| Multi-region | "Where do the writes go?" — then partition by home region |
| Conflicts in active-active | LWW loses data; vector clocks/siblings, CRDTs, or structural single-writer |
| Caches and replicas | Both are replicas; TTL is a staleness SLO |
| DB write + event publish | Dual-write problem → transactional outbox |

---

## Progress

Module 7 complete. Next in the curriculum: **Module 8 — Replication & partitioning** (leader-follower, multi-leader, quorum reads/writes, sharding, consistent hashing), which supplies the mechanisms this module has been describing the guarantees of. After that, **Module 9 — Consensus & coordination** (Raft, Paxos, etcd/ZooKeeper, distributed locks, vector clocks, CRDTs) explains how linearizability is actually built and how the weaker models track causality.
