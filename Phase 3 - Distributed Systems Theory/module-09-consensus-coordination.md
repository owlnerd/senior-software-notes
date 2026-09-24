# Module 9 — Consensus & Coordination
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Module 8 gave you the mechanisms of replication, and quietly deferred one question every time it came up: **who decides?** Who decides which replica is the leader after a failover. Who decides whether a write is committed. Who decides which of two nodes owns partition 17. Who decides that this background job runs on exactly one instance.

That question is **consensus**, and it is the only primitive in distributed systems you cannot fake. Caching, sharding, retries, queues — all of those you can approximate with effort and get away with. Agreement you either have or you don't, and the failures when you don't have it are the ones that corrupt data rather than slow it down.

This module has four jobs:

1. **Teach the theory precisely enough to reason with** — what consensus formally requires, what FLP forbids, why quorums are majorities, and why a "distributed lock" is a different animal from `lock (obj)`.
2. **Teach Raft cold, and Paxos well enough to compare.** Raft is the algorithm you will be asked about. Paxos is the vocabulary you need to sound literate when Spanner or Chubby comes up.
3. **Make you dangerous about locks and leases**, because "how would you make sure only one instance does X?" is one of the two or three most common senior .NET interview questions in existence, and the majority of candidates answer it unsafely.
4. **Teach the opposite instinct**: coordination is the most expensive thing in your architecture, so the best architects spend most of their effort *avoiding* it — logical clocks, CRDTs, escrow, single-writer ownership.

Four framings to carry through the module:

1. **Consensus is a control-plane tool, not a data-plane tool.** Put it where decisions are rare and important (leadership, membership, configuration, shard assignment). Never in the path of every user request if you can avoid it. Systems that violate this rule have a throughput ceiling measured in low thousands of ops/sec per group.
2. **You cannot distinguish a slow node from a dead node.** Every safety bug in this module derives from someone forgetting this. It's why leases expire, why fencing tokens exist, and why "the lock said I have it" is not the same as "I have it".
3. **Any sentence containing "exactly one" is a request for consensus.** Exactly one leader, exactly one order, exactly one delivery, exactly one owner of this email address. Recognize the shape, name the primitive, then decide whether you actually need it or can weaken the requirement.
4. **The senior move is usually to remove the need for coordination, not to coordinate better.** Partition ownership, idempotency, monotonic data types, and per-node budgets are cheaper, faster, and more available than any lock you can buy.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What consensus is | Agreement + integrity + termination; leader election, total order broadcast, atomic commit, CAS and locks are all the same problem |
| 2 | The system model | Design for asynchrony (safety), rely on partial synchrony (liveness); monotonic clocks only |
| 3 | FLP impossibility | No deterministic async algorithm is both always-safe and always-live; this is why timeouts exist |
| 4 | Failure detectors | Timeouts are the only tool; slow is indistinguishable from dead; GC pauses are the .NET version |
| 5 | Byzantine faults | 3f+1 for malicious nodes; inside your trust boundary, crash-fault tolerance is the right choice |
| 6 | Quorums and sizing | N = 2f+1, quorum intersection is the safety property; 3/5/7, never even; witnesses and learners |
| 7 | Replicated state machines | Agree on a log, apply deterministically; leader election and commit become the same mechanism |
| 8 | Paxos (single-decree) | Prepare/promise, accept/accepted; a proposer adopts the highest already-accepted value |
| 9 | Multi-Paxos | Elect a distinguished proposer, skip phase 1, one round trip per entry |
| 10 | Raft | Terms, randomized elections, log matching, leader completeness, and the term-safety commit rule |
| 11 | Raft's hard parts | Linearizable reads (ReadIndex/lease), snapshots, membership changes, learners |
| 12 | The family tree | Zab, Viewstamped Replication, Flexible Paxos, EPaxos — what each buys |
| 13 | Multi-group consensus | Consensus doesn't scale writes; N Raft groups do. This is Spanner/Cockroach/Cosmos |
| 14 | BFT | PBFT, 3f+1, blockchains; checksums handle corruption, not BFT |
| 15 | 2PC is not consensus | 2PC needs *all* participants; it blocks on coordinator failure; Paxos-Commit fixes it |
| 16 | The cost of consensus | 1 RTT to a majority per commit; ~1–3 ms in-region, 70–150 ms cross-region; batch and pipeline |
| 17 | ZooKeeper | Zab, znodes, ephemeral + sequential, watches, sessions; reads are local and can be stale |
| 18 | etcd | Raft + MVCC revisions + leases + watch-from-revision; Kubernetes' brain; fsync-bound |
| 19 | Consul and gossip | SWIM for membership scale, Raft for the decisions; two layers, two purposes |
| 20 | What belongs in it | Small, low-rate, critical metadata; not a queue, not a database, not your cache |
| 21 | Chubby's lessons | Coarse-grained locks, sequencers, caching with invalidation, and the human failure modes |
| 22 | A lock is not a mutex | Pauses and delays give you two holders; mutual exclusion across machines needs the resource's help |
| 23 | Leases and fencing tokens | Time-bounded locks plus a monotonic token the resource checks. The correct answer |
| 24 | Redlock | The Kleppmann/antirez argument, and why you should be able to summarize both sides |
| 25 | The ladder of answers | Don't need it → idempotent → OCC → lease + fence → lock service → consensus group |
| 26 | Optimistic concurrency in .NET | RowVersion, ETags, If-Match, advisory locks, SKIP LOCKED |
| 27 | .NET implementations | DistributedLock, Blob leases, HandleLostToken, Orleans, Kubernetes Lease, Hangfire/Quartz |
| 28 | Exactly-once | Doesn't exist on the wire; idempotency keys + dedup + effectively-once processing |
| 29 | Logical time | Lamport (order), vector clocks (concurrency detection), HLC (both, compactly), TrueTime (real time) |
| 30 | CRDTs | Merge functions that are commutative, associative, idempotent; no coordination, no invariants |
| 31 | Coordination avoidance | CALM: monotonic ⇒ coordination-free. Invariant confluence tells you which invariants cost you |
| 32 | Escrow / reservations | Split a numeric invariant into per-node budgets; the practical face of coordination avoidance |
| 33 | Control plane vs data plane | Consensus for metadata, replication for data, idempotency for everything else |
| 34 | The .NET/Azure surface | Blob leases, Cosmos ETags, Service Bus locks, Orleans, Service Fabric, Dapr, KRaft |
| 35 | Operating consensus | Disk fsync latency, leader-election rate as an SLI, quorum loss recovery, backup/restore hazards |

---

# Part A — The problem consensus solves

## Concept 1 — What consensus actually is, and the five problems that are one problem

**The formal problem.** Each process proposes a value. The protocol must satisfy:

| Property | Statement | Class |
|---|---|---|
| **Uniform agreement** | No two processes decide differently | Safety |
| **Integrity / validity** | A decided value was proposed by some process; no process decides twice | Safety |
| **Termination** | Every process that does not crash eventually decides | Liveness |

The split matters more than the definitions. **Safety properties must hold always, including during partitions, clock chaos, and arbitrary message delay. Liveness is allowed to be conditional.** Every real consensus system is built on exactly that asymmetry: Raft and Paxos never violate agreement, and openly give up on termination while the network is behaving badly enough to prevent a leader from being elected. When you say "the minority side of a partition becomes unavailable" (Module 7's CP choice), this is the mechanism.

**The five problems that are the same problem.** This is the single most useful piece of vocabulary in the module, because it lets you recognize consensus in disguise:

| Problem | Phrased as consensus |
|---|---|
| **Leader election** | Agree on which node is leader for term *t* |
| **Total order broadcast** (atomic broadcast) | Agree, for each log position *i*, which message occupies it |
| **Linearizable compare-and-set** | Agree on the sequence of CAS outcomes on a register |
| **Atomic commit** | Agree on commit vs abort for a transaction |
| **Distributed lock / lease** | Agree on who holds the lock for this interval |

They are **reducible to each other**: total order broadcast gives you consensus (propose by broadcasting, decide the first delivered message), and consensus gives you total order broadcast (run one consensus instance per log position). Herlihy's wait-free hierarchy says the same thing in shared-memory terms: compare-and-swap has **consensus number ∞**, which is why every distributed coordination primitive you've ever used is ultimately a CAS against a replicated register.

**Also in this family, one step up:** uniqueness constraints ("exactly one account with this email"), single-activation guarantees (Orleans grains), "only one instance of this cron job", partition ownership, and cluster membership. All consensus. When a candidate says "we'll just use a flag in Redis", they have proposed an unreplicated, unlinearizable consensus implementation.

**Interview framing to say out loud:** *"That requirement is a consensus problem — we need agreement on a single value across nodes that can fail. So either I use a system that implements consensus properly, or I weaken the requirement so I don't need agreement at all. Let me look at whether I can weaken it first, because consensus costs a round trip to a majority on every decision."* That sentence, delivered unprompted, is a staff-level signal.

---

## Concept 2 — The system model: what you are allowed to assume

Every algorithm in this module is only correct relative to a model. Knowing the three axes lets you answer "what if the network does X?" without guessing.

**Timing model:**

| Model | Assumption | Reality |
|---|---|---|
| **Synchronous** | Known bounds on message delay, processing time, clock drift | False in any real network; assuming it produces split brain |
| **Asynchronous** | No bounds on anything | True, but too weak: FLP says deterministic consensus is impossible |
| **Partially synchronous** | Bounds exist but are unknown, or hold only after some unknown stabilization time (GST) | The model everything real is designed for (Dwork, Lynch & Stockmeyer, 1988) |

**The sentence to memorize:** *"Raft and Paxos are safe in the asynchronous model and live under partial synchrony."* Safety never depends on timing; progress does.

**Failure model:**

- **Crash-stop** — a node halts and never returns. Easiest, and the model of most textbook proofs.
- **Crash-recovery** — a node halts and comes back, possibly having forgotten anything not on stable storage. This is the real model, and it is why Raft persists `currentTerm`, `votedFor`, and log entries with an fsync *before* responding. A node that forgets its vote can vote twice in the same term and elect two leaders.
- **Omission / partition** — messages lost or a subset of nodes unreachable. Includes the nasty **asymmetric partition** (A can reach B, B cannot reach A) and the **gray failure** (a node responds slowly enough to be useless but fast enough to look alive).
- **Byzantine** — arbitrary behavior including malice. Concept 5.

**Channel model:** networks give you *fair-loss* links (a message retried infinitely often eventually arrives). You build *reliable* links on top with retries and sequence numbers, which immediately means **duplicates**, which immediately means **idempotency is not optional**. Concept 28.

**Clocks — and the .NET detail that matters.** Two kinds:

- **Time-of-day clock** (`DateTime.UtcNow`, `DateTimeOffset.UtcNow`): synchronized by NTP, can jump forward, can jump **backward**, can be stepped by an operator. Never use it to measure a duration.
- **Monotonic clock** (`Stopwatch`, `Environment.TickCount64`): only moves forward, no absolute meaning, not comparable across machines.

**Leases must be measured on a monotonic clock**, and the holder must use a *conservative* estimate of its own remaining validity (renew at one-third of the lease, treat the lease as expired earlier than the server does). Using `DateTime.UtcNow` for lease arithmetic means an NTP step can make a node believe it still holds a lease that has expired. This is a concrete, checkable, senior-level detail, and it makes a great answer to "what could go wrong with your leader election?"

```csharp
// Wrong: wall-clock arithmetic for a lease.
private DateTime _leaseExpiresUtc;
bool StillHeld() => DateTime.UtcNow < _leaseExpiresUtc;   // NTP step ⇒ silent split brain

// Right: monotonic, with a safety margin for clock rate error and scheduling delay.
private long _acquiredAtTicks;                  // Stopwatch.GetTimestamp()
private static readonly TimeSpan Lease  = TimeSpan.FromSeconds(15);
private static readonly TimeSpan Margin = TimeSpan.FromSeconds(3);

bool StillHeld() =>
    Stopwatch.GetElapsedTime(_acquiredAtTicks) < Lease - Margin;
```

`Stopwatch.GetElapsedTime(long)` (.NET 7+) is the allocation-free way to do this without holding a `Stopwatch` instance.

---

## Concept 3 — FLP: the impossibility result you should be able to state calmly

**Fischer, Lynch & Paterson (1985):** in an asynchronous system with even a single crash failure, **no deterministic protocol can guarantee both safety and termination** for consensus.

**Intuition for the proof** (enough to discuss, not to reproduce): the protocol has *bivalent* configurations — states from which both decisions are still reachable. The adversary (the message scheduler) can always find a way to deliver messages in an order that moves from one bivalent configuration to another, forever. Because the system is asynchronous, a process that hasn't spoken is indistinguishable from a process that has crashed, so the protocol can never safely proceed without it.

**What FLP does not say.** It does not say consensus is impossible in practice, or that it's usually slow, or that it fails often. It says no algorithm can *guarantee* termination in the worst case. Every production system lives comfortably inside this result.

**The three escape routes, all of which you use daily:**

| Escape | How | Where you see it |
|---|---|---|
| **Partial synchrony** | Assume timing bounds eventually hold; use timeouts | Raft, Paxos, Zab, etcd, ZooKeeper |
| **Randomization** | Break symmetry probabilistically; terminate with probability 1 | Ben-Or; Raft's randomized election timeouts are a practical cousin |
| **Failure detectors** | Assume an oracle that eventually stops suspecting correct processes (Chandra & Toueg: ◇W is the weakest detector sufficient for consensus) | Every heartbeat mechanism in your stack |

**How to use FLP in an interview.** Not as trivia. Use it to explain a design consequence: *"FLP is why every consensus system exposes a timeout you have to tune, and why that tuning is a genuine trade-off: short election timeouts detect a dead leader fast but cause spurious elections under load, and every spurious election is a write outage for the duration. In etcd, leader-election rate is a first-class alerting metric for exactly this reason."*

---

## Concept 4 — Failure detectors: you cannot distinguish slow from dead

There is no way, over a network, to tell a crashed node from a slow node, a partitioned node, or a node whose process is paused. The only mechanism available is a **timeout**, and a timeout is a guess.

The two properties a failure detector can have:

- **Completeness** — every crashed node is eventually suspected.
- **Accuracy** — no correct node is ever suspected.

In an asynchronous system you cannot have both perfectly. Practical detectors are *eventually* accurate: they may produce false positives, and the system must be safe when they do.

**The cost of each direction:**

| Timeout too short | Timeout too long |
|---|---|
| False positives: spurious leader elections, lease stolen from a live holder, unnecessary failovers | Long unavailability after real failures: clients blocked for the timeout on every request |
| Flapping under load — a GC pause or disk stall looks like death | Extended split-brain window in systems without fencing |

**Better detectors exist.** The **phi-accrual failure detector** (Hayashibara et al.) outputs a continuous suspicion level from the recent distribution of heartbeat inter-arrival times instead of a boolean, letting each consumer pick its own threshold. Cassandra and Akka use it; it's worth naming.

**The .NET-specific version of this problem.** Your process can pause for reasons that have nothing to do with the network:

- A blocking gen-2 GC on a large heap. Well-tuned server workloads see sub-100 ms; pathological ones (huge LOH, fragmentation, Workstation GC on a big heap) see seconds.
- ThreadPool starvation (Module 15's territory) delaying your renewal timer's continuation by seconds even though the machine is healthy.
- Hypervisor-level pauses: live migration, VM snapshot, noisy-neighbor CPU steal.
- A container hitting its memory limit and being throttled, or a laptop being suspended.

So the timeline that produces split brain is entirely realistic: you hold a 15-second lease, you pause for 20 seconds, another node takes the lease, you resume and finish writing. **Your lease-renewal code being correct does not save you. Only the resource rejecting your stale write saves you.** That is Concept 23, and this is why it's the most important thing in the module.

**Membership as voting.** Because detectors are unreliable, mature systems don't let one node's opinion kill another. Orleans' cluster membership requires a configurable number of independent suspicion votes (`NumVotesForDeathDeclaration`, default 2) written to the membership table before a silo is declared dead; SWIM-based systems (Consul/Serf) use indirect probes through other members before declaring a node failed. "One node's timeout should not be sufficient to evict another node" is a good architectural rule to state.

---

## Concept 5 — Byzantine faults, and why you probably don't need them

**Two Generals.** Two armies must attack simultaneously; the only channel is a messenger who may be captured. There is no protocol that guarantees both generals know the other has agreed. The consequence you use every day: **over an unreliable channel, exactly-once delivery is impossible.** You can have at-most-once (send and don't retry) or at-least-once (retry), and you convert at-least-once into an *effectively*-once outcome with idempotency. Concept 28.

**Byzantine Generals** (Lamport, Shostak & Pease, 1982). If up to *f* participants may behave arbitrarily — lying, sending different values to different peers, colluding — agreement requires **n ≥ 3f + 1** with unauthenticated messages. Signed messages relax the bound for some formulations, and **PBFT** (Castro & Liskov, 1999) made it practical: 3f+1 replicas, a primary, and a three-phase protocol, tolerating f malicious nodes with good throughput.

**When it's relevant:**

- **Blockchains and multi-party systems** where participants are mutually distrusting and money is at stake. Nakamoto consensus (probabilistic, open membership) and BFT protocols (HotStuff, Tendermint) live here.
- **Cross-organization consensus** — a consortium where no single operator is trusted.

**When it is not relevant, and this is the answer you want:** inside a single trust boundary with operator-controlled nodes, you use **crash-fault-tolerant** consensus. BFT triples your replica count and adds phases to defend against a threat model where an attacker with node-level access would have far easier paths anyway.

**Don't confuse Byzantine faults with corruption.** Bit rot, bad NICs, buggy firmware, and truncated writes produce *wrong data* without malice. The defence is end-to-end checksums, not BFT: page checksums in the storage engine (`data_checksums` in Postgres, `PAGE_VERIFY CHECKSUM` in SQL Server), CRCs on records in the replication log, checksums on backups, and verification during replica bootstrap. Amazon's Builders' Library piece on integrity is the reference. Saying *"corruption is a checksum problem, not a consensus problem"* neatly separates two things candidates often blur.

---

## Concept 6 — Quorums, cluster sizing, and the arithmetic you should have cold

A **quorum** is any set large enough that any two quorums intersect. Intersection is the entire safety argument: if every decision requires a majority, and any two majorities share a member, that member's memory prevents two contradictory decisions.

**Majority quorum:** with N nodes, quorum = ⌊N/2⌋ + 1, and the cluster tolerates f = ⌊(N−1)/2⌋ failures. So **N = 2f + 1**.

| N | Quorum | Failures tolerated | Notes |
|---|---|---|---|
| 1 | 1 | 0 | Not a cluster. Fine for dev. |
| 2 | 2 | 0 | **Worse than 1**: either node's failure blocks everything |
| 3 | 2 | 1 | The default. Survives one node or one AZ |
| 4 | 3 | 1 | Same tolerance as 3, more cost and more latency. Never do this |
| 5 | 3 | 2 | Survives two failures, or one failure during a rolling upgrade |
| 7 | 4 | 3 | Rare; used where membership changes are frequent or failure rates high |

**Why even numbers are strictly worse:** adding the 4th node doesn't raise tolerance (still 1) but raises the quorum size (2→3), so you now need more nodes healthy and more responses per commit. Every extra voter is a node that can slow you down without increasing what you survive.

**The latency consequence.** A commit waits for the ⌈N/2⌉-th fastest follower response. Going from 3 to 5 nodes means waiting for 2 of 4 instead of 1 of 2 — slightly more robust to a single slow node, but a larger fan-out and more fsyncs per commit. This is why control-plane clusters are 3 or 5 and never 15: **consensus clusters trade throughput for tolerance, and the exchange rate gets bad quickly.**

**Placement is the real decision** (Module 8, Concept 24). Three nodes in one AZ tolerate one node loss and zero AZ losses. Three nodes in three AZs in one region tolerate an AZ loss at ~1–2 ms commit cost. Three nodes in three *regions* tolerate a region loss at 70–150 ms per commit, and the number of regions matters more than the number of nodes: **two failure domains can never form a majority safely**, which is why two-region active-active with strong consistency is not a thing, and why a third region holding a single witness is such a common pattern.

**Witnesses, learners, and non-voting replicas** — the vocabulary for cheating this arithmetic productively:

- **etcd learners**: non-voting members that receive the log, used to bootstrap a new member without endangering quorum, then promoted.
- **CockroachDB non-voting replicas**: serve local follower reads without participating in the quorum, so read locality doesn't cost write latency.
- **Witness / file-share witness** (SQL Server WSFC, Azure SQL's quorum sets): a tiebreaker vote holding no data, so you get odd-numbered quorum with two data copies.
- **Flexible Paxos** (Howard et al., 2016): quorums must only intersect *across phases* — |Q1| + |Q2| > N. That lets you shrink the steady-state accept quorum (fast commits) at the price of a larger, slower leader-election quorum. It's the theoretical basis for several "fast path" designs, and naming it signals real reading.

**Sizing answer to have ready:** *"Three voting members across three availability zones, with five if we need to survive a failure during maintenance. Learners for new members so bootstrap doesn't endanger quorum. Cross-region I'd keep the consensus group in one region and replicate asynchronously outward, unless the requirement is genuinely RPO zero across regions — in which case I need three regions and I'll pay 100 ms per commit, and that must be a stated product decision."*

---
# Part B — The algorithms

## Concept 7 — Replicated state machines: the log is the system

Before any algorithm, internalize the architecture all of them implement.

```
     client
       │  command: "SET x = 5"
       ▼
  ┌─────────┐        replicate & agree        ┌─────────┐   ┌─────────┐
  │ LEADER  │ ─────────────────────────────►  │FOLLOWER │   │FOLLOWER │
  │ log:    │                                 │ log:    │   │ log:    │
  │ 1 SET a │                                 │ 1 SET a │   │ 1 SET a │
  │ 2 DEL b │                                 │ 2 DEL b │   │ 2 DEL b │
  │ 3 SET x │ ◄── committed once a majority   │ 3 SET x │   │ 3 ...   │
  └────┬────┘     has it durably              └────┬────┘   └────┬────┘
       ▼ apply in order                            ▼             ▼
  ┌─────────┐                                 ┌─────────┐   ┌─────────┐
  │  state  │  identical on every replica      │  state  │   │  state  │
  └─────────┘  because same log + determinism  └─────────┘   └─────────┘
```

**The theorem in one line:** if every replica starts in the same state and applies the *same commands in the same order*, and the state machine is **deterministic**, every replica ends in the same state. So replication reduces to agreeing on a log — total order broadcast — which reduces to consensus (Concept 1).

Three consequences worth saying out loud:

1. **Leader election and log commit become the same mechanism.** In Module 8's single-leader replication they were separate: the database replicated the log, and something else (an operator, Patroni, a cluster manager) picked the leader. In consensus-based replication the protocol does both, which is exactly why it can't lose committed writes on failover and Module 8's async-replication failover can.
2. **Determinism is a real constraint on your state machine.** No `DateTime.UtcNow`, no `Random`, no `Guid.NewGuid()`, no dictionary iteration order, no floating-point differences, no reading external services inside the apply step. If the command needs a timestamp or an ID, the **leader** generates it and puts it *in the log entry*. This is the single most common bug when engineers build their own replicated state machine — and the same discipline event-sourcing systems require (Module 23).
3. **The log is also your audit trail, your CDC feed, and your recovery mechanism.** Which is why event sourcing, Kafka, and Raft look so similar when you squint: a durable ordered log with deterministic consumers.

---

## Concept 8 — Paxos, single-decree: the algorithm to understand, not implement

Lamport's Paxos (1989/1998, restated in *Paxos Made Simple*, 2001) solves: agree on **one** value among proposers and acceptors, tolerating crashes and arbitrary delays.

**Roles:** proposers (suggest values), acceptors (vote and remember), learners (observe the outcome). In practice one process plays all three.

**Phase 1 — Prepare.** A proposer picks a globally unique, monotonically increasing **proposal number** *n* (typically `(counter, nodeId)` so comparison is total) and sends `Prepare(n)` to acceptors.

An acceptor receiving `Prepare(n)`:
- If *n* is greater than any proposal number it has previously promised, it **promises** not to accept anything numbered below *n*, and replies with the highest-numbered proposal it has already accepted, if any (`(n_accepted, value_accepted)`).
- Otherwise it ignores or NACKs.

**Phase 2 — Accept.** If the proposer hears promises from a majority:
- If **any** promise carried an already-accepted value, the proposer **must** propose the value from the highest-numbered such promise. It does not get to propose its own value.
- If none did, it may propose its own.

It sends `Accept(n, v)`. Acceptors accept unless they've promised a higher number. When a majority accepts, *v* is **chosen**, forever.

**The whole point is that one rule**: *adopt the highest-numbered accepted value you learn about.* Because any two majorities intersect, a new proposer cannot miss a value that was already chosen, so it is forced to re-propose it. That's the safety argument in one sentence, and being able to give it is the difference between "I've read about Paxos" and "I understand Paxos".

**Why two phases.** Phase 1 establishes "no one older than me can still win, and here is what may already have been decided." Phase 2 commits. You cannot fuse them without losing the ability to safely recover from a failed proposer mid-flight.

**Liveness failure to name:** two proposers can duel — each `Prepare` invalidates the other's in-flight `Accept` — forever. FLP in the flesh. The fix is to elect a single distinguished proposer, which is Multi-Paxos.

---

## Concept 9 — Multi-Paxos, and the gap between paper and production

A log needs a decision *per position*. Running full single-decree Paxos per entry costs two round trips each. **Multi-Paxos** elects a stable **distinguished proposer** (leader) that runs Phase 1 **once** for all future positions, then only needs Phase 2 (one round trip) per entry. Steady state: client → leader → majority of followers → commit. One RTT to a majority, which is the same steady state as Raft.

The parts the original paper leaves to you, and which is why "Paxos" in the wild means a family of systems rather than one algorithm: leader election mechanics, log holes and out-of-order commits, membership changes, snapshotting, and read-only queries.

**"Paxos Made Live" (Chandra, Griesemer & Redstone, 2007)** is the best engineering paper in this module and belongs on your reading list. Google's team reports what it actually took to make Paxos into Chubby: handling disk corruption, master leases, group membership, snapshots, an expression of the algorithm as a state machine to make it testable, and the discovery that the hard parts were fault-injection testing and operational tooling rather than the protocol. The line to remember: *the algorithm is a small fraction of the work.* That is the right answer to "would you implement Raft yourself?"

---

## Concept 10 — Raft, in the depth an interview requires

Ongaro & Ousterhout, 2014, *In Search of an Understandable Consensus Algorithm*. Same guarantees as Multi-Paxos, designed for comprehensibility, and the algorithm behind etcd, Consul, CockroachDB, TiKV, MongoDB's election protocol, Kafka's KRaft controller, RabbitMQ quorum queues, Neo4j, InfluxDB, and Redpanda. **If you know one consensus algorithm, know this one.**

Raft decomposes into three subproblems: **leader election**, **log replication**, **safety**.

### State

Each node is `Follower`, `Candidate`, or `Leader`. Persistent state, fsynced before responding to any RPC:

| Field | Meaning | Why persistent |
|---|---|---|
| `currentTerm` | Logical clock; monotonic | A node that forgets its term can accept stale leaders |
| `votedFor` | Candidate voted for in this term | A node that forgets its vote can vote twice ⇒ **two leaders** |
| `log[]` | Entries, each with `term` + command | Committed entries must survive restart |

Volatile: `commitIndex`, `lastApplied`; on leaders, `nextIndex[]` and `matchIndex[]` per follower.

**Terms** are the central abstraction: time is divided into terms, each beginning with an election. At most one leader per term. Every RPC carries a term; **a node seeing a higher term immediately becomes a follower**, and a node receiving a request with a lower term rejects it. That single rule is how deposed leaders discover they've been deposed.

### Leader election

- Followers expect `AppendEntries` (heartbeats) from the leader. No contact within the **election timeout** ⇒ become candidate.
- Candidate: increments `currentTerm`, votes for itself, sends `RequestVote(term, lastLogIndex, lastLogTerm)` to all.
- A node grants its vote if: the candidate's term is ≥ its own, it hasn't already voted this term, **and the candidate's log is at least as up to date as its own** (compare last log term, then index). That last clause is the **Election Restriction**, and it is what makes committed entries safe.
- Majority of votes ⇒ leader; immediately starts heartbeating to suppress further elections.
- Split vote ⇒ nobody gets a majority, timeouts fire again. **Randomized election timeouts** (the paper uses 150–300 ms) make repeated splits exponentially unlikely. This is Raft's practical answer to FLP.

### Log replication

Client command → leader appends to its log → `AppendEntries` to followers (carrying `prevLogIndex`/`prevLogTerm`) → follower rejects unless its log matches at `prevLogIndex`, in which case the leader decrements `nextIndex` and retries until they converge, then overwrites the follower's conflicting suffix.

This yields the **Log Matching Property**: *if two logs contain an entry with the same index and term, the logs are identical in all entries up through that index.* It's proved by induction from the consistency check, and it's the property that makes Raft's recovery so much simpler than Paxos's log holes.

An entry is **committed** when the leader has replicated it to a majority. The leader advances `commitIndex`, applies to the state machine, replies to the client, and piggybacks `commitIndex` on subsequent heartbeats so followers apply too. **Raft has no holes**: entries commit in order, which trades a little throughput (a single slow follower can't be skipped for a *commit*, though it can be lagged) for enormous conceptual simplicity.

### Safety — the two subtleties that separate real understanding from a summary

**1. The Election Restriction.** A candidate whose log is missing committed entries cannot win, because a committed entry is on a majority, any winning candidate needs votes from a majority, and those majorities intersect — so at least one voter has the entry and will refuse a candidate whose log is behind. Hence the **Leader Completeness Property**: a leader's log contains all committed entries from previous terms. Consequence: **Raft leaders never overwrite or delete their own entries; they only append.** All repair happens on followers.

**2. The commit rule for entries from previous terms.** A new leader may find an entry from an earlier term replicated on a majority and be tempted to mark it committed. **Raft forbids this** (§5.4.2), because a subsequent sequence of failures can still cause that entry to be overwritten. Instead, a leader only counts *its own term's* entries toward commitment; earlier entries commit *implicitly* when an entry from the current term commits above them.

The practical upshot: a new leader appends a **no-op entry** in its own term immediately on election, and only then can it safely learn its commit index and start serving reads. If you can state this — "the new leader commits a no-op to establish its commit index, because the majority-replication rule alone is unsafe for entries from earlier terms" — you are demonstrating that you read the paper rather than a blog summary. It is also the reason there is a brief write unavailability after every election.

### Numbers to quote

| Parameter | Typical | Constraint |
|---|---|---|
| Heartbeat interval | 50–150 ms (etcd default 100 ms) | ≪ election timeout |
| Election timeout | 150–300 ms randomized (etcd default 1000 ms) | ≫ broadcast time, ≪ MTBF |
| Time to recover from leader failure | ~1 × election timeout + election round trip | Half a second to a couple of seconds in practice |
| Steady-state commit | 1 RTT to a majority + fsync | ~1–3 ms in-region; fsync often dominates |

The paper's rule is `broadcastTime ≪ electionTimeout ≪ MTBF`. Cross-region clusters break the first inequality, which is why etcd's tuning guide tells you to raise both when running across regions, and why nobody runs a latency-sensitive Raft group across continents.

---

## Concept 11 — Raft's hard parts: reads, snapshots, and membership

The parts of Raft that generate the best interview follow-ups, all from Ongaro's thesis rather than the short paper.

### Linearizable reads — the trap almost everyone falls into

**"Just read from the leader" is not linearizable.** A deposed leader that hasn't noticed yet will happily serve stale data. Three correct options:

| Technique | Mechanism | Cost |
|---|---|---|
| **Log read** | Put the read through the log as an entry | Full commit cost; simplest to reason about |
| **ReadIndex** | Leader records `commitIndex`, commits a no-op if needed, exchanges heartbeats with a majority to confirm it's still leader, waits for `applied ≥ readIndex`, then reads locally | 1 RTT, no disk write. The standard |
| **Lease read** | Leader holds a time-based lease from the last successful heartbeat round and reads locally with no network at all | Zero RTT, but correctness depends on bounded clock **drift** (not synchronized clocks) |

etcd exposes exactly this choice: linearizable reads (default, ReadIndex-based) vs serializable reads (`--consistency=s` / `Serializable` option), which read locally from any member and may be stale. **A cheap way to make etcd faster is also a cheap way to introduce a subtle consistency bug**, which is a great thing to raise unprompted.

**Follower reads** are a real design (CockroachDB's follower reads, ZooKeeper's local reads): you serve reads from a follower and accept bounded staleness, or from a closed timestamp in the past. It's the same Module 7 spectrum reappearing inside a CP system.

### Log compaction and snapshots

The log grows forever, so replicas periodically snapshot the state machine, record the last included index/term, and discard the prefix. Consequences:

- A follower that has fallen behind the leader's snapshot cannot be caught up with `AppendEntries`; it needs an `InstallSnapshot` transfer. This is Module 8's "bootstrapping a replica" primitive in a consensus dress.
- Snapshotting is expensive and must not stall the apply loop; implementations use copy-on-write or fork-like tricks.
- Operationally: etcd's `--snapshot-count`, and ZooKeeper's snapshot + transaction-log split, with the recommendation to put the transaction log on a **dedicated, fast device** because it is fsynced on every write.

### Membership changes

Naively swapping the member set can produce two disjoint majorities (old set and new set) and therefore two leaders. Two safe approaches:

- **Joint consensus** (the paper): a transitional configuration `C_old,new` where decisions require majorities in *both* configurations, then a switch to `C_new`.
- **Single-server changes** (the thesis, and what etcd does): add or remove exactly one voter at a time, which guarantees old and new majorities intersect.

Practical rules to state: **never add two members at once**; **add a new member as a learner first**, let it catch up, then promote (etcd's learner feature exists because adding a slow, empty member to a 3-node cluster momentarily makes quorum depend on a node that can't serve); and **remove the failed member before adding the replacement** when recovering from a failure, because a 3-node cluster with one dead node has no tolerance left and a 4-node cluster with one dead node needs 3 of 4 healthy.

---

## Concept 12 — The family tree: Zab, VR, Flexible Paxos, EPaxos

You need to recognize these names and say one sentence each.

| Algorithm | Origin | Distinguishing feature |
|---|---|---|
| **Viewstamped Replication** | Oki & Liskov, 1988; *VR Revisited*, 2012 | Predates Paxos in publication of the RSM shape; views ≈ terms. Raft is closer to VR than to Paxos |
| **Zab** | Yahoo!, powers ZooKeeper | Primary-order broadcast: preserves the order a *single* primary issued writes, which Paxos doesn't guarantee; optimized for high read:write ratios and fast recovery |
| **Multi-Paxos** | Lamport et al. | Leader + phase-1 elision; allows log holes and out-of-order commit, so higher throughput, more complexity |
| **Raft** | Ongaro & Ousterhout, 2014 | Strong leader, no holes, understandable; randomized elections |
| **Flexible Paxos** | Howard, Malkhi & Spiegelman, 2016 | Quorums need only intersect across phases (size of Q1 + size of Q2 > N). Fast common-case commits, slower elections |
| **EPaxos** | Moraru et al., 2013 | Leaderless; non-interfering commands commit in one round trip; clients talk to the nearest replica. Great for WAN, complex dependency tracking |
| **Nakamoto / PoW, PoS** | Bitcoin, Ethereum | Open membership, probabilistic finality, Byzantine. Different problem class |
| **HotStuff / Tendermint** | 2018–19 | Modern BFT with linear message complexity; the basis of several chains |

**The comparison to make if asked "Paxos or Raft?"**: *"Functionally equivalent for our purposes. Raft constrains the design — strong leader, no log holes, ordered commit — which costs a little throughput and buys a lot of understandability, testability and operability. Multi-Paxos and its descendants relax those constraints for throughput and WAN latency. I'd use an off-the-shelf Raft implementation, because the algorithm is the easy part; the hard parts are fault injection, snapshotting, membership changes and operational tooling."*

---

## Concept 13 — Multi-group consensus: how consensus scales

**The objection every good interviewer raises:** "consensus requires every write to reach a majority, so the whole system is capped at one leader's throughput. How do you scale?"

**The answer: you don't scale a consensus group. You run many of them.** One consensus group per partition — exactly Module 8's "shard = replica set", with consensus supplying the replication:

| System | Unit | Consensus per unit |
|---|---|---|
| **Spanner** | Directory/split within a tablet | One Paxos group per split, with leaders spread across zones |
| **CockroachDB / TiKV** | Range (a contiguous key span) | One Raft group per range; thousands per node ("Multi-Raft"), with batched heartbeats to keep the overhead sane |
| **Cosmos DB** | Physical partition | A replica set per partition, quorum-committed |
| **Kafka** | Partition | Not Raft — leader/ISR replication for data, with a single Raft group (KRaft) for the *controller* metadata |
| **Kubernetes** | Whole cluster | One etcd cluster. Which is why etcd is the scaling limit of a cluster |

Two consequences worth volunteering:

1. **Cross-group operations lose the easy guarantees.** A transaction spanning two Raft groups needs 2PC *over* the groups (Spanner and CockroachDB both do exactly this: 2PC where each participant is itself a Paxos/Raft group, so the coordinator's decision is itself durably replicated — Concept 15). Cheap within a group, expensive across. Which is Module 8's partition-key lesson restated: **co-locate what must be transactional.**
2. **Multi-Raft has real overhead**: per-group heartbeats, timers, and leader placement. Implementations batch heartbeats across groups and coordinate leader placement so one node isn't leader for every hot range. "How do you keep 50,000 Raft groups from spending all their CPU on heartbeats?" is a lovely deep-dive question, and "batch the heartbeats per peer, not per group, and balance leadership" is the answer.

---

## Concept 14 — Where BFT actually appears in a normal architecture

Short section, because the honest answer is "rarely". But be ready with:

- **You don't run PBFT internally.** Crash-fault tolerance plus authentication (mTLS), authorization, audit logging, and checksums covers the realistic threat model.
- **Where you do think about Byzantine-ish behavior:** untrusted clients (never trust a client-supplied sequence number, timestamp or total), multi-tenant isolation, and third-party participants in a workflow. The mitigations are authentication and validation, not consensus protocols.
- **Where BFT is the right tool:** permissioned ledgers across organizations, and public blockchains. If your interview touches these, the vocabulary is 3f+1, finality (probabilistic vs deterministic), and the fact that open-membership consensus needs an external cost function (PoW/PoS) to prevent Sybil attacks.

---

## Concept 15 — Two-phase commit is not consensus

This distinction is a frequent, high-value interview discriminator.

**2PC:** coordinator sends `Prepare`; each participant durably promises it *can* commit (locks held, WAL flushed) and votes yes/no; coordinator writes the decision to its own durable log and sends `Commit`/`Abort`; participants apply and acknowledge.

| | Consensus (Paxos/Raft) | Atomic commit (2PC) |
|---|---|---|
| Decision requires | A **majority** of nodes | **Unanimity** — any participant can veto |
| Tolerates failed participants | Yes, up to f | No: a failed participant blocks the decision |
| Coordinator failure | Elect a new one; the log survives | **Blocks**, with locks held, until the coordinator returns |
| Availability | Improves with more nodes | Degrades with more participants |

**The blocking window is the real problem.** If the coordinator dies after participants have prepared, those participants hold locks on rows they cannot release: they cannot unilaterally commit (someone may have voted no) or abort (someone may have been told to commit). These are **in-doubt transactions**, and in practice they require operator intervention. Adding participants multiplies the probability of it happening.

**3PC** adds a pre-commit phase to make it non-blocking, but requires a synchronous network model and bounded failure detection, so it isn't used in practice. The fix that *is* used is **Paxos-Commit / consensus-backed 2PC**: replicate the coordinator's decision log with consensus so coordinator failure is survivable. That's exactly what Spanner and CockroachDB do, and why they can offer cross-shard ACID transactions without the classic 2PC availability collapse.

**.NET specifics worth knowing:**

- `System.Transactions.TransactionScope` enlists resource managers, and escalates from a lightweight local transaction to a **distributed transaction** via MSDTC when a second durable resource enlists.
- Distributed transactions were absent from .NET Core and **returned in .NET 7 — Windows-only**, and require explicitly enabling `TransactionManager.ImplicitDistributedTransactions = true`. On Linux, `TransactionScope` across two databases throws `PlatformNotSupportedException`. Knowing this is a concrete, current, differentiating detail.
- Message queues do not enlist usefully in practice: Azure Service Bus supports transactions only *within* one entity (plus the "send via" pattern), not across a queue and a SQL database. So the answer to "transactionally write to the DB and publish an event" remains the **transactional outbox** (Module 7, Concept 22), not 2PC.
- The architectural alternative for long-running, cross-service work is the **saga** with compensating actions (Module 12). The one-liner: *"2PC trades availability for atomicity and holds locks across the network; sagas trade atomicity for availability and make compensation a domain concern. For anything crossing service boundaries I choose sagas plus idempotency; for two tables in the same database I use one local transaction and no coordination at all."*

---

## Concept 16 — The cost of consensus, in numbers

The estimation skills from Module 5, applied to the most expensive primitive you have.

**Per committed entry, steady state with a stable leader:**

1. Client → leader: 1 network hop (0 if the client happens to be co-located with the leader).
2. Leader: append + **fsync** to its own disk.
3. Leader → followers → leader: 1 RTT to the ⌈N/2⌉-th fastest, each follower fsyncing.
4. Leader → client: the response.

| Deployment | Commit latency | What dominates |
|---|---|---|
| Single AZ, NVMe | ~0.5–2 ms | fsync (~0.1–1 ms NVMe, 5–10 ms spinning/network disk) |
| Three AZs, one region | ~1–3 ms | Cross-AZ RTT (~0.5–1 ms) + fsync |
| Three regions, continental | ~30–60 ms | Speed of light |
| Three regions, intercontinental | ~100–250 ms | Speed of light, and this is a product decision |

**Throughput.** A naive one-entry-at-a-time implementation is limited to roughly `1 / commit_latency` ops/sec — a few hundred to a couple of thousand. Real implementations recover an order of magnitude or two with **batching** (many client commands per log entry / per fsync — group commit) and **pipelining** (send the next `AppendEntries` without waiting for the previous ack). Published etcd guidance lands in the low tens of thousands of small writes per second on good hardware, and it is **fsync-bound**, which is why "put etcd on dedicated fast local SSDs and never on a network file share" is standard advice.

**The three architectural conclusions:**

1. **Never put consensus on the per-item hot path.** A "distributed lock per request" design is a design with a hard ceiling around a few thousand RPS and a latency floor you cannot optimize away. Batch, or partition so each partition has its own group, or eliminate the coordination (Part E).
2. **Consensus latency is the floor for any strongly consistent operation.** Module 7's "strong consistency costs a round trip to a majority" is this number. You now know where it comes from.
3. **Disk quality is a consensus SLA.** fsync latency directly sets your commit latency; a noisy-neighbor EBS volume makes leaders flap because heartbeats miss their deadline while the apply loop waits on the disk. This is the single most common cause of real etcd/ZooKeeper incidents.

---
# Part C — Coordination services

You will almost never implement consensus. You will *use* a system that has, and the interview question is whether you know what those systems guarantee, what they cost, and what you must not do with them.

## Concept 17 — ZooKeeper: the original, and still the best-documented recipes

Hunt et al., *ZooKeeper: Wait-free coordination for internet-scale systems* (USENIX ATC 2010). Consensus via **Zab**. Still under Kafka (pre-4.0), HBase, Solr, Flink, Hadoop, and a huge amount of enterprise Java.

**Data model:** a hierarchical namespace of **znodes** (`/app/config/db`), each holding a small payload (default limit 1 MB, controlled by `jute.maxbuffer`) plus a version number. Two flags do all the interesting work:

- **Ephemeral** — the znode is deleted automatically when the creating client's **session** ends (explicit close, or session timeout with no heartbeat). This is the primitive behind liveness-linked ownership: a crashed process's lock disappears without anyone reaping it.
- **Sequential** — ZooKeeper appends a monotonically increasing counter to the name (`/lock/n-0000000017`). This gives you an ordered queue of waiters *and* a monotonic number you can use as a fencing token.

**Watches:** one-shot, client-side triggers on a znode's data or children. One-shot is important: after firing you must re-register, and between fire and re-register you can miss changes, so watches are a notification that something changed, not a change *feed*. You always re-read state after a watch.

**Guarantees, precisely:**

| Guarantee | Detail |
|---|---|
| Linearizable **writes** | All writes go through the leader and are totally ordered |
| **FIFO client order** | A single client's operations are applied in the order issued |
| **Reads are local, and may be stale** | Any server can serve a read from its own replica — fast, but a client can read data older than a write it knows committed |
| `sync()` | Forces the connected server to catch up to the leader, so `sync()` then read gives you a fresh read |

That third row is the exam question. **ZooKeeper is not linearizable for reads by default.** The classic anomaly: client A writes config, tells client B out-of-band, B reads and sees the old value because it's connected to a lagging follower. `sync()` + read is the fix, and "ZooKeeper reads are sequentially consistent, not linearizable, unless you `sync()` first" is a high-signal sentence.

**The recipes** (the ZooKeeper docs' own list, and worth reading in full):

- **Leader election**: every candidate creates an ephemeral sequential znode under `/election`; the lowest sequence number is the leader; each node watches only the znode immediately *before* its own.
- **Locks**: identical structure under `/lock`.
- **The herd effect, and its fix**: if every waiter watches *the lock node itself*, then releasing it wakes all N waiters, who all stampede the server, and N−1 fail. Watching only your immediate predecessor turns a thundering herd into a linked-list handoff — one notification per release. Being able to explain this is a strong sign you've built one of these.
- **Group membership**: ephemeral znodes under `/members`, watch children.
- **Configuration**: a znode, a watch, and re-read on notification.
- **Barriers and double barriers**, **two-phased commit**, and **queues** — documented, and mostly things you shouldn't build on ZooKeeper at volume (Concept 20).

**Operational notes:** an ensemble of 3 or 5; the transaction log should be on its own fast disk (it's fsynced per write); an "observer" role exists for read scaling without voting; session timeouts must exceed realistic GC pauses or you get spontaneous ownership loss.

---

## Concept 18 — etcd: the one to know for modern infrastructure

Raft-based, gRPC, and the persistence layer for Kubernetes — which means it is the coordination service most likely to matter in a modern .NET/cloud interview.

**Data model:** a flat, sorted, **MVCC** key-value space. Every mutation increments a cluster-wide **revision**, and old revisions remain readable until compaction. This single design choice makes etcd much more useful than ZooKeeper for programmatic coordination:

| Feature | What it gives you |
|---|---|
| **Revisions** | Read the store at a past revision; watch *from* a revision so you never miss an event across a reconnect |
| **Watch from revision** | A real change stream, not a one-shot trigger. The basis of Kubernetes informers/controllers |
| **Leases** | A TTL object; keys attached to a lease are deleted when it expires. `KeepAlive` renews. This is the ephemeral-node equivalent, but the TTL is explicit and per-lease |
| **Txn (mini-transactions)** | Compare-and-swap generalized: `If(compare...) Then(ops...) Else(ops...)`, applied atomically. Compare on value, version, create-revision or mod-revision |
| **Election & Lock APIs** | Server-side recipes, so you don't hand-roll the sequential-node dance |

**The txn primitive is the important one**, because it is a linearizable compare-and-set — the universal coordination primitive of Concept 1. Almost every etcd-based pattern is "CAS on a key, with a lease attached":

```
# Leader election, conceptually:
Txn().If(CreateRevision("/leader") == 0)
     .Then(Put("/leader", myId, WithLease(leaseId)))
     .Else(Get("/leader"))
```

**Fencing comes free**: the `mod_revision` (or the lease's key revision) of the leader key is a cluster-wide monotonically increasing number. Carry it with every side effect and have the resource reject anything older. Kubernetes uses the same idea with `resourceVersion` and optimistic concurrency on every object update.

**Hard limits to quote** (defaults, check your version):

| Limit | Default | Why it exists |
|---|---|---|
| Backend storage quota | **2 GB** (`--quota-backend-bytes`), max recommended **8 GB** | Mean time to recovery: a new member must be able to catch up in seconds, not hours. When exceeded, etcd raises a `NOSPACE` alarm and goes **read-only** until you compact, defrag, and clear the alarm |
| Max request size | **1.5 MB** (`--max-request-bytes`) | Every request goes through Raft; large entries stall the log |
| Max ops per txn | **128** (`--max-txn-ops`) | Bounds the size of a single replicated entry |
| Default heartbeat / election | 100 ms / 1000 ms | Tunable for high-latency links |

The read-only-on-quota-exceeded behaviour is a genuinely famous outage shape: a Kubernetes cluster whose etcd fills up stops accepting all writes, and recovery requires `etcdctl compact`, `etcdctl defrag`, and `etcdctl alarm disarm`. Naming that sequence is a credible ops signal.

**Consistency knob:** reads are linearizable by default (ReadIndex); `Serializable` reads are served locally from whatever member you're talking to and may be stale. Same trade as ZooKeeper's local reads, but opt-in rather than default — a better default, and a good comparison to draw.

**.NET client:** `dotnet-etcd` (gRPC-based, community) is the practical option; there's no first-party Microsoft client, because in the Microsoft ecosystem the equivalent jobs are done by Blob leases, Cosmos, Service Bus, and Azure App Configuration. Worth saying: *"on Azure I would not deploy etcd just for coordination — I'd use a Blob lease or a Cosmos conditional write, because the platform already gives me a linearizable CAS and I don't want to operate a consensus cluster."*

---

## Concept 19 — Consul, and the two-layer pattern

Consul is worth knowing because it illustrates an architectural pattern you should be able to name: **gossip for scale, consensus for decisions.**

- **Membership and health** use **SWIM**-style gossip (via Serf): eventually consistent, O(1) load per node regardless of cluster size, indirect probing before declaring failure, scales to thousands of nodes.
- **The authoritative data** (KV store, service catalog writes, sessions, ACLs) goes through a **Raft** group of 3–5 servers.

The lesson generalizes: **don't run consensus over your whole fleet.** Run consensus over a small set of coordinators and use cheap, eventually-consistent protocols for the wide fan-out. Cassandra's gossip + per-key quorums, Kafka's KRaft controller + ISR data replication, and Orleans' gossip-accelerated membership table are the same shape.

Consul also has **sessions** (its ephemeral-ownership concept, bindable to health checks) and a lock/leader-election API on top, and supports multi-datacenter federation where each DC has its own Raft group and WAN gossip joins them. That last point is the right shape for multi-region coordination generally: **per-region consensus, cross-region asynchronous.**

---

## Concept 20 — What belongs in a coordination service, and what must never

The most common way teams hurt themselves with ZooKeeper/etcd is by using them as a database or a queue.

**Belongs there:**

- Leader election and ownership (who owns partition 17, who runs the singleton job)
- Cluster membership and service registry
- Configuration and feature flags that must be consistent
- Shard/partition assignment tables (the routing source of truth from Module 8, Concept 19)
- Distributed locks for rare, coarse-grained operations (a migration, a nightly rebuild)
- Small monotonic counters or sequencers, at low rates

**Does not belong there:**

| Anti-pattern | Why it fails | Use instead |
|---|---|---|
| Work queue at high throughput | Every enqueue/dequeue is a Raft commit; you get thousands of ops/sec at best, and you burn the storage quota | Service Bus, RabbitMQ, Kafka, SQS |
| Per-request locking | Consensus round trip per request; ceiling and latency floor | Partition ownership, optimistic concurrency, idempotency |
| Storing blobs / large values | 1.5 MB request cap, 2 GB total, and every value replicated to every node and read into memory | Blob storage, with a pointer in etcd |
| Session state or a cache | Wrong cost model entirely; you're paying for linearizability on data that tolerates staleness | Redis, distributed cache (Module 10) |
| Metrics / event history | Unbounded growth, quota exhaustion, read-only cluster | Time-series store |
| One giant cluster for everything | Blast radius: the coordination service is the one component whose failure stops *everything* | Separate clusters per concern; cells (Module 8, Concept 24) |

**The rule to state:** *"A coordination service holds a small amount of critical metadata that changes rarely and must be agreed. If the data is large, hot, or tolerant of staleness, it belongs somewhere else. And because everything depends on it, I size its blast radius deliberately and make sure the data plane can survive its unavailability for a while — degrade to the last-known-good configuration rather than fail."*

That last clause is the architect-level point: **your data plane should keep serving with cached coordination state when the coordination service is down.** Kubernetes does this (kubelets keep running pods when the API server is unavailable); Consul clients cache; good service meshes cache endpoints. A design where etcd being down means all traffic stops has converted a control-plane dependency into a data-plane dependency.

---

## Concept 21 — Chubby: the lessons paper

Burrows, *The Chubby lock service for loosely-coupled distributed systems* (OSDI 2006). Google's lock service, built on Paxos, five replicas, master lease. Read it for the design decisions, which are still the best-argued in the literature:

- **Coarse-grained locks, deliberately.** Locks are expected to be held for hours or days (leadership, ownership), not milliseconds. This keeps load low and makes lock-server unavailability survivable. If you find yourself designing fine-grained distributed locks, the paper's argument is that you've made an architectural mistake.
- **It became a naming service more than a lock service.** Most traffic was clients using it as a consistent store of small configuration and as DNS-replacement. The team notes that a lock service was easier for engineers to adopt than a Paxos library, which is exactly why etcd/ZooKeeper exist as services rather than libraries.
- **Sequencers = fencing tokens.** Chubby's lock holders can request an opaque **sequencer** to pass to a resource server, which validates it against the lock service (or a cached copy). The paper explicitly discusses the alternative, **lock-delay**, where after an unclean loss the server refuses to hand the lock to anyone for a grace period. Both appear in Concept 23; Chubby is where they're best explained.
- **Caching with invalidation, not TTL.** Clients cache aggressively and the master *invalidates* on change, preserving strict consistency. Nice counterpoint to Module 10's TTL-based caching.
- **KeepAlives and session leases** carry the liveness signal, with the lease-handoff design allowing a client to survive a master failover.
- **The human failure modes** are the most quoted part: developers ignoring availability implications of a dependency on Chubby, using it for high-volume publishing, and a lack of understanding of session semantics. Google's response included quotas and review of new usage patterns. That is an architecture-governance lesson, not a technical one, and it plays extremely well in an enterprise-architect interview: *"the failure mode of a shared coordination service is usually organizational — teams take a dependency on it without understanding the availability coupling, so you need both quotas and review."*

---
# Part D — Locks, leases, and fencing tokens

This is the part of Module 9 you are most likely to be asked about in a .NET interview, in the form: *"you have three instances of a service behind a load balancer, and this job must run on exactly one. How?"* Most candidates answer with a lock. The complete answer is below.

## Concept 22 — A distributed lock is not a mutex

`lock (obj)` in C# is enforced by the runtime with hardware support: the lock and the thing it protects live inside one memory space under one scheduler, and the CLR guarantees that a thread holding the lock is a thread that is running.

A distributed lock has none of those properties. The lock lives in one process (Redis, SQL, etcd, a blob), the protected resource lives in another (your database, your file, your payment API), and the holder lives in a third. **Nothing connects the three.** The lock service's belief about who holds the lock and the resource's experience of who is writing to it are independent facts.

**The canonical failure, worth drawing on a whiteboard:**

```
 Client 1                  Lock service              Storage
    │── acquire ──────────────►│                        │
    │◄── granted (TTL 15s) ────│                        │
    │                          │                        │
  [ GC pause / VM freeze /     │                        │
    ThreadPool starvation      │                        │
    for 20 seconds ]           │                        │
    │                     lease expires                 │
    │                          │◄── acquire ── Client 2 │
    │                          │─── granted ──►│        │
    │                          │               │── write(x=2) ──►│
    │  ...resumes, still       │                        │
    │  believing it holds it   │                        │
    │────────────────── write(x=1) ────────────────────►│   ← corruption
```

Both clients wrote. The lock service did nothing wrong. Client 1's code did nothing wrong. The data is now wrong.

**Every "cause" of this is unavoidable in general:** a gen-2 GC pause, a hypervisor live-migration, CPU steal, a page fault storm, a container CPU throttle, a delayed timer callback under ThreadPool starvation, a network delay that makes the renewal arrive after expiry, or a clock adjustment. Concept 4's mantra: **slow is indistinguishable from dead**, and the lock service *must* eventually give the lock to someone else or a crashed holder would block the resource forever.

**So the conclusion, stated crisply:** a distributed lock alone gives you **efficiency** (usually only one worker does the work, so you don't waste effort) but not **correctness** (mutual exclusion at the resource). If a second holder would merely duplicate harmless work, a lock is fine. If a second holder would corrupt data, you need the resource to participate. That efficiency/correctness split is Kleppmann's framing and it is the cleanest way to answer the question.

---

## Concept 23 — Leases and fencing tokens: the correct answer

**A lease is a lock with a TTL.** It solves the "crashed holder blocks forever" problem, which is why every practical distributed lock is a lease. It does *not* solve the two-holders problem; it *creates* it, because expiry is decided by the lease server's clock while the holder may still be mid-operation.

**A fencing token is what makes it safe.** The lock service issues a **monotonically increasing number** with each grant. The holder passes the token with every operation on the protected resource. **The resource remembers the highest token it has seen and rejects anything lower.**

```
 Client 1 gets lease, token = 33
 Client 1 pauses
 Lease expires; Client 2 gets lease, token = 34
 Client 2 writes with token 34  →  resource records 34, accepts
 Client 1 resumes, writes with token 33  →  resource sees 33 < 34, REJECTS
```

Now correctness does not depend on any clock, any timeout tuning, or any assumption about pauses. It depends only on the token being monotonic and the resource checking it. This is the single most important pattern in the module.

**Where you get a monotonic token for free:**

| System | Token |
|---|---|
| **etcd** | The lease's key `mod_revision`, or the revision returned by the winning `Txn` |
| **ZooKeeper** | The `zxid` or the sequential znode's counter; Curator exposes it |
| **Chubby** | An explicit "sequencer" |
| **Azure Blob Storage** | The lease ID (opaque, not monotonic — see below), plus `If-Match` ETag on every write |
| **Cosmos DB** | The item `_etag`, used with `IfMatchEtag` |
| **SQL Server / Postgres** | `rowversion` / `xmin`, or a sequence in the same transaction |
| **Kubernetes** | `metadata.resourceVersion` on the object, plus `Lease.spec.leaseTransitions` |

**And where the resource can't check a token** — a third-party API, an SMTP server, a file share with no conditional write, a payment gateway — you have three fallbacks, in order of preference:

1. **Idempotency key** at the resource, so a duplicate operation has no additional effect (Concept 28). The payment-gateway answer.
2. **Lock-delay / grace period**: after an unclean lease loss, wait longer than any plausible pause before granting to anyone. Chubby's `lock-delay`; it converts a correctness problem into a longer outage, which is often the right trade.
3. **Self-fencing with a conservative margin**: the holder stops work when its *own* monotonic clock says the lease is nearly expired, with a margin exceeding your worst observed GC pause. This is weaker than fencing (a pause can exceed any margin), but combined with an alert on pauses longer than the margin it is a defensible engineering position. Say explicitly that it's a probabilistic mitigation, not a guarantee — that honesty is the signal.

**Azure Blob leases are worth a paragraph of their own** because they're the most common .NET leader-election substrate. A blob lease is exclusive for 15–60 seconds (or infinite), and while held, any write to the blob must carry the lease ID. So the *blob itself* is properly fenced — the storage service enforces it. But the lease ID is a GUID, not a monotonic counter, so **it does not fence anything other than that blob.** If you use the lease to decide "I'm the leader" and then write to SQL, you must carry your own monotonic epoch: keep an epoch number *inside* the leased blob, increment it on each acquisition, and have SQL store and compare `last_epoch`. Also note `HandleLostToken`-style cancellation (Concept 27) and that the renewal must be conservative on a monotonic clock (Concept 2).

---

## Concept 24 — Redlock: the argument you should be able to summarize from both sides

A short, famous, and genuinely instructive disagreement, and a good interview topic because it rewards nuance rather than a side.

**Redlock** (proposed by Redis's author, antirez) is a distributed lock over N independent Redis masters: acquire `SET key uuid NX PX ttl` on a majority within a small fraction of the TTL, and you hold the lock; release by comparing the value and deleting via a Lua script.

**Kleppmann's critique** (*How to do distributed locking*, 2016):
- The algorithm's safety depends on **bounded clock error and bounded process pauses**, which are timing assumptions the asynchronous model doesn't grant — a clock jump on one node or a long GC pause makes two holders possible.
- Crucially, **for correctness you need fencing tokens regardless**, and Redlock's random UUID is not monotonic, so it cannot fence.
- Therefore: for efficiency, a single Redis instance with a TTL is enough; for correctness, use a real consensus system *and* fencing tokens.

**antirez's reply** (*Is Redlock safe?*): the clock requirement is about bounded drift rather than synchronized time, and can be mitigated; pause-related failures affect essentially every lease-based system; and the fencing critique applies equally to any lock without resource-side checks. His point about the fencing token is that if the resource can enforce a monotonic check, you often don't need the lock's exclusivity at all — you need the CAS.

**The senior answer:** *"Both are right about different things. Redlock's added complexity buys availability, not safety; the safety gap is fencing, and that gap exists with ZooKeeper too if the resource doesn't check the token. So my decision rule is: if the lock is an optimization, one Redis with a TTL and a value check is fine and simple. If the lock is load-bearing for correctness, I need a linearizable store for the lock and a monotonic token the resource validates — and I'd first ask whether I can restructure the problem so no lock is needed."* That reframing — **the important decision is efficiency vs correctness, not which lock library** — is what an interviewer is listening for.

One concrete practical note: even for the efficiency case, never `DEL` a lock key unconditionally. Compare the value first, atomically, or you will delete someone else's lock after your own expired:

```lua
-- release.lua: only delete if we still own it
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

---

## Concept 25 — The ladder of answers

When asked "how do you make sure only one X happens?", walk *down* this ladder out loud. Getting to the right rung, and showing why the cheaper rungs were rejected, is the entire signal.

| Rung | Technique | When it's the right answer |
|---|---|---|
| **0. Don't need it** | Make the operation naturally single-owner: partition the work by key and let each partition have one owner (Kafka consumer group, Service Bus sessions, Orleans grain, shard ownership) | Most of the time. Ownership by partition scales; a global lock doesn't |
| **1. Idempotent + at-least-once** | Let it run twice; make the second run a no-op via an idempotency key or a natural unique constraint | Anything you can make idempotent. Cheapest correctness in distributed systems |
| **2. Optimistic concurrency (CAS)** | Conditional write: `WHERE rowversion = @v`, `If-Match: etag`, etcd `Txn`. Loser retries or drops | Contention is low and conflicts are detectable. No lock service needed at all |
| **3. Queue-based serialization** | One partition/session per key, ordered processing, single consumer per partition | Per-entity serialization at high throughput |
| **4. Lease + fencing token** | Time-bounded ownership plus a monotonic token the resource validates | Coarse-grained ownership: singleton jobs, partition assignment, leader-only work |
| **5. Lock service** | etcd/ZooKeeper/Consul lock, or `DistributedLock` over SQL | Rare, coarse operations where you want mutual exclusion as a first-class concept |
| **6. Your own consensus group** | Embed Raft in your service | Almost never. You are building a database |

**The two sentences that do the most work in an interview:**

> *"First I'd try to remove the need for mutual exclusion, by partitioning ownership so that only one node is ever responsible for a given key."*

> *"If I do need it, I'd use a lease rather than a lock, and I'd carry a fencing token into the resource, because a lease can expire while the previous holder is still paused."*

---

## Concept 26 — Optimistic concurrency: the primitive you'll actually use

Rung 2 deserves depth, because in .NET work it's the correct answer far more often than a lock, and interviewers like it precisely because it's unglamorous and correct.

**EF Core — row version:**

```csharp
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }

    [Timestamp]                       // SQL Server: rowversion, maintained by the engine
    public byte[]? Version { get; set; }
}

// Or, provider-agnostic / Postgres:
modelBuilder.Entity<Order>().Property(o => o.Version).IsRowVersion();
modelBuilder.Entity<Order>().UseXminAsConcurrencyToken();   // Npgsql: the system xmin column
```

EF Core adds the token to the `WHERE` clause and checks the affected row count:

```sql
UPDATE Orders SET Status = @p0 WHERE Id = @p1 AND Version = @p2;
```

Zero rows affected ⇒ `DbUpdateConcurrencyException`. The important part is what you do next, and "retry the whole operation" is usually the wrong answer:

```csharp
try { await db.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var current = await entry.GetDatabaseValuesAsync(ct);
    if (current is null) { /* deleted by someone else — domain decision */ }
    else
    {
        // Merge deliberately: whose intent wins, per property?
        entry.OriginalValues.SetValues(current);   // then re-apply *your* intent
    }
}
```

The senior point: **a concurrency exception is a domain event, not a transient fault.** Blindly retrying re-applies a decision made against stale data (the classic bug: "set status = Shipped" retried after someone cancelled the order). Decide per-field whether to merge, reject, or ask the user. Also note that a concurrency token only protects the rows you loaded — it does nothing about *write skew* across rows (Module 7, Concept 19), which needs a real isolation level or an explicit lock.

**Pessimistic options in the database, which are often better than any distributed lock:**

| Primitive | Use |
|---|---|
| `SELECT ... FOR UPDATE` | Lock rows you're about to modify; blocks other writers |
| `SELECT ... FOR UPDATE SKIP LOCKED` | **The queue-in-a-database pattern**: N workers each grab distinct rows with no coordination. Postgres, MySQL 8+, Oracle. This is how you build a competing-consumers worker pool with only your existing database |
| `sp_getapplock` / `sp_releaseapplock` (SQL Server) | A named, session- or transaction-scoped application lock. A genuine distributed lock, backed by a system you already run and already monitor |
| `pg_advisory_lock` / `pg_try_advisory_xact_lock` | The Postgres equivalent; transaction-scoped variants release automatically on commit/rollback, which removes the leak risk |

**Say this out loud when it applies:** *"Before adding Redis or ZooKeeper as a dependency purely for locking, I'd check whether the database I already have gives me what I need — `sp_getapplock` or a Postgres advisory lock is a real lock with a real owner, it's included in my existing HA and monitoring story, and it dies with the connection. One fewer system to operate."* Interviewers consistently reward reducing moving parts.

**Cosmos DB / Azure Storage ETags:**

```csharp
// Cosmos: conditional replace — fails with 412 if someone else changed it
var read = await container.ReadItemAsync<Order>(id, new PartitionKey(tenantId), cancellationToken: ct);
read.Resource.Status = OrderStatus.Shipped;
try
{
    await container.ReplaceItemAsync(
        read.Resource, id, new PartitionKey(tenantId),
        new ItemRequestOptions { IfMatchEtag = read.ETag }, ct);
}
catch (CosmosException e) when (e.StatusCode == HttpStatusCode.PreconditionFailed)
{
    // someone else won; re-read and decide
}
```

`PatchItemAsync` with a filter predicate, and `TransactionalBatch` (atomic, but only within a single logical partition key) are the other two tools. Note the link back to Module 8: **Cosmos gives you atomicity only inside a partition key, so your partition key choice determines what you can do without coordination.**

---

## Concept 27 — .NET implementations, and what each actually guarantees

### `DistributedLock` (madelson) — the default answer for "I need a distributed lock in .NET"

A well-maintained library exposing mutexes, reader-writer locks and semaphores over multiple backends: `DistributedLock.SqlServer`, `.Postgres`, `.MySql`, `.Oracle`, `.Redis`, `.Azure` (blob leases), `.ZooKeeper`, `.FileSystem`, `.WaitHandles`.

```csharp
// DI registration
services.AddSingleton<IDistributedLockProvider>(
    _ => new PostgresDistributedSynchronizationProvider(connectionString));

// Usage
public sealed class NightlyRebuild(IDistributedLockProvider locks, ILogger<NightlyRebuild> log)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var @lock = locks.CreateLock("nightly-rebuild");
        await using var handle = await @lock.TryAcquireAsync(TimeSpan.Zero, ct);
        if (handle is null) { log.LogInformation("Another instance holds the lock; skipping."); return; }

        // Critical: link work to lease loss.
        using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct, handle.HandleLostToken);
        await DoWorkAsync(linked.Token);
    }
}
```

Two details that make this a strong answer rather than a library name-drop:

- **`HandleLostToken`** is cancelled if the library detects the lock was lost (connection dropped, lease not renewed). Linking your work's `CancellationToken` to it is the difference between "I called a lock library" and "I understood what a lease is". It's still best-effort — detection isn't instantaneous — so it complements fencing, not replaces it.
- **Backend choice changes the semantics.** SQL Server `sp_getapplock` and Postgres advisory locks die with the connection (good liveness, dependent on connection health). Redis is TTL-based with renewal. Azure blob leases are 15–60 s with renewal. ZooKeeper is session/ephemeral-based. Knowing that the *backend* determines your failure mode, not the API, is the point.

### Azure Blob lease leader election — the Azure-native pattern

This is what Azure Functions' singleton/host locks, WebJobs, and most Azure-hosted `BackgroundService` singletons use under the hood.

```csharp
public sealed class BlobLeaseLeader(BlobClient blob, ILogger<BlobLeaseLeader> log)
{
    private static readonly TimeSpan LeaseDuration = TimeSpan.FromSeconds(30);
    private static readonly TimeSpan RenewEvery    = TimeSpan.FromSeconds(10);  // ≤ 1/3 of the lease

    public async Task RunAsLeaderAsync(Func<long, CancellationToken, Task> work, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var leaseClient = blob.GetBlobLeaseClient();
            BlobLease? lease = null;
            try { lease = await leaseClient.AcquireAsync(LeaseDuration, cancellationToken: ct); }
            catch (RequestFailedException e) when (e.Status == 409) // LeaseAlreadyPresent
            {
                await Task.Delay(TimeSpan.FromSeconds(5), ct);
                continue;
            }

            // Bump and read the fencing epoch stored *inside* the leased blob.
            long epoch = await IncrementEpochAsync(leaseClient.LeaseId, ct);
            log.LogInformation("Became leader with epoch {Epoch}", epoch);

            using var leadership = CancellationTokenSource.CreateLinkedTokenSource(ct);
            var renewal = RenewAsync(leaseClient, leadership, ct);
            try { await work(epoch, leadership.Token); }   // work receives its fencing token
            finally
            {
                await leadership.CancelAsync();
                try { await renewal; } catch (OperationCanceledException) { }
                try { await leaseClient.ReleaseAsync(cancellationToken: CancellationToken.None); } catch { }
            }
        }
    }

    private async Task RenewAsync(BlobLeaseClient leaseClient, CancellationTokenSource leadership, CancellationToken ct)
    {
        using var timer = new PeriodicTimer(RenewEvery);
        try
        {
            while (await timer.WaitForNextTickAsync(ct))
                await leaseClient.RenewAsync(cancellationToken: ct);
        }
        catch (Exception ex)
        {
            log.LogWarning(ex, "Lease renewal failed; surrendering leadership");
        }
        finally
        {
            await leadership.CancelAsync();   // stop the work the instant renewal fails
        }
    }
}
```

What to point out about this code in an interview:

1. Renewal at **one third** of the lease, so two consecutive failures don't lose it.
2. Renewal failure **cancels the work**, immediately and unconditionally.
3. The work receives an **epoch**, and every side effect it performs must carry that epoch to a resource that rejects stale epochs. Without step 3 this is an efficiency lock, not a correctness lock.
4. `ReleaseAsync` on the way out so the next instance takes over in milliseconds instead of waiting for the lease to expire. Graceful handoff is a real availability feature.
5. Missing on purpose from the sample, and good to mention: jitter on the acquire retry to avoid a herd, and `HandleLostToken`-style propagation into every downstream call.

### The rest of the .NET/platform surface

| Mechanism | What it gives you | Caveat to name |
|---|---|---|
| **Azure Functions singleton / WebJobs** | Blob-lease-based single-instance execution, `[Singleton]` attribute, `SingletonScope.Function/Host` | Efficiency-level guarantee; your function must still be idempotent |
| **Service Bus sessions** | FIFO + exclusive lock per session ID — per-entity serialization without any lock service | Session lock also expires; `RenewSessionLockAsync` for long work; a session lock loss mid-batch means redelivery |
| **Service Bus peek-lock** | Per-message lock (default 30 s, max 5 min), `RenewMessageLockAsync`, `MaxAutoLockRenewalDuration` on the processor | At-least-once. Duplicates are guaranteed eventually; dedup or idempotency required |
| **Kafka consumer groups** | Exactly one consumer per partition at a time via group coordination | Rebalances can briefly overlap processing; commit offsets after side effects and make them idempotent |
| **Orleans grains** | Single-activation per grain ID — coordination by *placement*, the cleanest form of rung 0 | Single activation is best-effort: during silo failures/membership churn, two activations can briefly exist. For correctness use a version/etag in the grain's persisted state |
| **Microsoft Orleans reminders/`IGrainTimer`** | Durable scheduled work owned by a grain, so no cluster-wide singleton needed | Reminder firing is at-least-once |
| **Service Fabric** | Reliable Services with quorum-replicated reliable collections; a stateful partition has one primary | Its own replication/federation protocol, not Raft |
| **Kubernetes `Lease` + client-go leader election** | Lease objects in `coordination.k8s.io`, typical 15 s lease / 10 s renew deadline / 2 s retry | **Not fenced**: safety relies on the losing leader stopping voluntarily. The Kubernetes docs say so explicitly |
| **Hangfire** | `DisableConcurrentExecution` filter, distributed locks in its storage | Lock is storage-backed; timeouts mean duplicate execution is possible |
| **Quartz.NET clustering** | DB-backed job locks so only one node fires a trigger | Requires the clustered `JobStoreTX`; misfire handling is your problem |
| **Dapr** | Distributed lock building block, actors with single-activation, state store ETags | Adds a sidecar dependency; the lock has the same lease semantics as everything else here |
| **`IHostedService` on multiple replicas** | Nothing at all. It runs on every replica | This is the default and it surprises people in production |

---

## Concept 28 — Exactly-once doesn't exist; idempotency is the real answer

Two Generals (Concept 5) says you cannot guarantee exactly-once *delivery* over an unreliable channel. What you can achieve is **exactly-once effect**, sometimes called *effectively-once*: at-least-once delivery plus deduplication or idempotent operations.

**The three mechanisms:**

1. **Idempotency keys.** The caller generates a unique key per logical operation; the server records `(key → result)` and returns the stored result on a repeat. This is how Stripe's API works, and the right pattern for any command endpoint.

```csharp
// Insert-first idempotency: the unique index is the coordination primitive.
// No lock, no consensus — the database's linearizable unique constraint does the work.
public async Task<PaymentResult> ChargeAsync(string idempotencyKey, ChargeRequest req, CancellationToken ct)
{
    try
    {
        db.IdempotencyRecords.Add(new IdempotencyRecord {
            Key = idempotencyKey, RequestHash = Hash(req), Status = "in-progress" });
        await db.SaveChangesAsync(ct);                      // unique index on Key
    }
    catch (DbUpdateException e) when (IsUniqueViolation(e))
    {
        var existing = await db.IdempotencyRecords.FindAsync([idempotencyKey], ct);
        if (existing!.RequestHash != Hash(req)) throw new IdempotencyConflictException();
        return existing.Status == "complete"
            ? Deserialize(existing.Result!)
            : throw new OperationInProgressException();      // 409; client retries later
    }

    var result = await gateway.ChargeAsync(req, idempotencyKey, ct);  // pass it downstream too
    // ... persist result + mark complete in one transaction
    return result;
}
```

2. **Deduplication windows.** The infrastructure remembers recent message IDs: Azure Service Bus duplicate detection (`RequiresDuplicateDetection` plus a `DuplicateDetectionHistoryTimeWindow`, based on `MessageId`), or an inbox table in your own database. Bounded window means bounded memory and bounded protection — a redelivery after the window is not caught, so an inbox table with a longer retention is stronger.

3. **Transactional read-process-write inside one system.** Kafka's transactions plus `read_committed` give exactly-once *within* Kafka (consume → produce → commit offsets atomically). It does not extend to your database or a third-party API. Be precise about that boundary; "Kafka gives exactly-once" is a half-truth that a good interviewer will probe.

**Naturally idempotent operations** are the cheapest form: `SET x = 5` rather than `x += 1`; upsert by natural key rather than insert; state transitions guarded by the current state (`UPDATE ... WHERE status = 'Pending'` — the affected-row count tells you whether you were first). Designing commands to be idempotent by construction removes coordination from your architecture entirely, which is the theme of Part E.

**The sentence to have ready:** *"I treat every message and every command as at-least-once, and I make the handler idempotent — usually with an idempotency key backed by a unique index, sometimes by making the operation naturally idempotent. That's cheaper and more available than trying to achieve exactly-once delivery, which is impossible anyway."*

---
# Part E — Coordinating less

Parts B–D were about doing coordination correctly. This part is about needing less of it, which is where architect-level judgement lives. The framing: **coordination is a latency and availability tax; pay it only where an invariant genuinely requires it.**

## Concept 29 — Logical time: Lamport clocks, vector clocks, HLC, TrueTime

Physical clocks disagree. So systems that need ordering build it from causality instead.

### Lamport clocks (1978)

Each node keeps a counter. Increment on every local event; attach it to every message; on receive, set `counter = max(local, received) + 1`. Order events by `(counter, nodeId)`.

- **Gives you:** a *total* order consistent with causality. If a → b (a happened before b), then `L(a) < L(b)`.
- **Doesn't give you:** the converse. `L(a) < L(b)` does **not** imply a → b; they may be concurrent. So Lamport clocks cannot *detect* conflicts, only order them arbitrarily-but-consistently.
- **Use for:** tie-breaking, LWW registers where you accept the loss, sequencing where any consistent order will do.

### Vector clocks / version vectors

A vector of counters, one per node. Compare element-wise: `V1 < V2` if every element ≤ and at least one <; if neither dominates, the events are **concurrent**.

- **Gives you:** conflict *detection* — the thing Lamport clocks can't do. Module 8, Concept 10 covered the mechanics and the Dynamo shopping-cart case.
- **Costs:** size O(number of writers), plus pruning problems when writers come and go. This is why *version vectors* (one entry per replica, not per client) are the practical form, and why Baquero & Preguiça's *Why Logical Clocks Are Easy* is the reference for getting the terminology right.

### Hybrid Logical Clocks (HLC) — the modern default

Kulkarni et al., 2014. A timestamp of `(physical_time, logical_counter)` that tracks physical time closely (so it's human-meaningful and comparable to wall-clock timestamps) while preserving causality even when clocks are skewed: on receive, take the max of local physical time and the received timestamp, bumping the logical counter on ties.

- **Gives you:** causally-consistent, monotonically increasing, ~wall-clock-accurate timestamps with constant size and **no coordination**.
- **Where it's used:** CockroachDB, MongoDB (cluster time), YugabyteDB, and Cosmos DB's internals. If asked "how would you timestamp events across a cluster without a central sequencer?", HLC is the right answer, and naming it is a strong signal.

### TrueTime and commit-wait (Spanner)

Google gave up on pure logical time and instead built a clock infrastructure (GPS + atomic clocks per datacenter) whose API returns an *interval* `[earliest, latest]` with a bounded uncertainty ε (the paper reports ε up to a few milliseconds, sawtoothing as clocks re-synchronize).

The trick is **commit-wait**: a transaction picks a commit timestamp, then *waits out the uncertainty* before releasing locks and acknowledging. That wait guarantees the timestamp is in the past for every observer, which buys **external consistency** (strict serializability) globally.

The lesson to state: **Spanner turned a correctness problem into a latency problem by making clock uncertainty explicit and bounded.** Systems without TrueTime-grade clocks either accept weaker guarantees (CockroachDB's uncertainty restarts) or pay a coordination round trip. This is a terrific example of "buy a hardware property to simplify a distributed algorithm."

| Mechanism | Detects concurrency? | Size | Real-time meaning? | Needs coordination? |
|---|---|---|---|---|
| Lamport clock | No | O(1) | No | No |
| Vector / version vector | **Yes** | O(replicas) | No | No |
| HLC | Partially (via causality) | O(1) | **Approximately** | No |
| TrueTime + commit-wait | N/A (avoids conflicts) | O(1) | **Yes, bounded** | Special hardware + a wait |
| Central sequencer / consensus | N/A | O(1) | Yes | **Yes — a round trip** |

---

## Concept 30 — CRDTs: data types that don't need agreement

A **Conflict-free Replicated Data Type** is a data type whose merge operation is **commutative, associative, and idempotent**, so replicas that receive the same updates in any order, any number of times, converge to the same state — with **zero coordination**. Shapiro, Preguiça, Baquero & Zawirski (2011) formalized them.

Two flavours:

- **State-based (CvRDT)**: replicas ship their whole state (or a delta); merge is a **join** in a lattice, and state only moves "upward" (monotonically). Robust to duplicate and reordered messages; heavier on the wire, which δ-CRDTs mitigate by shipping only deltas.
- **Operation-based (CmRDT)**: replicas ship operations, which must be commutative; requires **reliable causal broadcast** (exactly-once, causal order) from the transport. Lighter messages, stronger transport requirement.

**The catalogue** you should be able to sketch:

| Type | Merge rule | Note |
|---|---|---|
| **G-Counter** (grow-only) | Per-replica counts; merge = element-wise max; value = sum | The canonical example |
| **PN-Counter** | Two G-Counters (increments, decrements); value = P − N | Can't enforce "never negative" — see below |
| **G-Set** | Union | Adds only |
| **2P-Set** | Add-set ∪ tombstone-set; removed is forever | Can't re-add |
| **LWW-Register** | Highest timestamp wins | Simple; loses data on concurrency |
| **MV-Register** | Keeps all concurrent values | Pushes merging to the application (Dynamo siblings) |
| **OR-Set** (observed-remove) | Tag each add with a unique id; remove only the observed tags | Correct add/remove semantics; metadata grows |
| **RGA / Logoot / YATA / Treedoc** | Sequence CRDTs with dense identifiers | Collaborative text editing: Yjs, Automerge |

```csharp
// G-Counter: a lattice join. Merge is commutative, associative, idempotent.
public sealed record GCounter(ImmutableDictionary<string, long> Counts)
{
    public static GCounter Empty => new(ImmutableDictionary<string, long>.Empty);

    public GCounter Increment(string replicaId, long by = 1) =>
        new(Counts.SetItem(replicaId, Counts.GetValueOrDefault(replicaId) + by));

    public GCounter Merge(GCounter other)
    {
        var b = Counts.ToBuilder();
        foreach (var (id, v) in other.Counts)
            b[id] = Math.Max(b.GetValueOrDefault(id), v);    // max, not sum — idempotence
        return new GCounter(b.ToImmutable());
    }

    public long Value => Counts.Values.Sum();
}
```

The `Math.Max` is the whole idea: re-delivering the same update twice cannot double-count, because taking a max is idempotent. Summing per-replica maxima recovers the total.

**The three costs you must name, because this is where candidates over-sell CRDTs:**

1. **Metadata and tombstone growth.** OR-Sets accumulate tags; sequence CRDTs accumulate identifiers for deleted characters. Garbage collection requires knowing what every replica has seen, which is... coordination, usually amortized and off the critical path.
2. **No invariants across replicas.** A PN-Counter cannot guarantee "balance ≥ 0" or "at most 100 tickets sold", because each replica would have to know about the others' decrements. **Any invariant over a *sum* or a *limit* fundamentally requires coordination** — or the escrow trick in Concept 32. This is the most important limitation and it connects directly to Concept 31.
3. **Semantics are fixed by the type, not the domain.** "Concurrent add and remove: which wins?" is decided by which CRDT you picked, and users will notice if that decision doesn't match their expectation.

**Where you'll meet them:** Redis Enterprise Active-Active (CRDT-based geo-replication), Riak data types, Azure Cosmos DB's custom conflict resolution (you can hand-write a merge procedure, which is CRDT thinking even if it isn't a formal CRDT), Automerge/Yjs for collaborative editing, Akka.NET **Distributed Data** (`ORSet`, `GCounter`, `LWWRegister` shipped as first-class .NET types over a gossip protocol), and application-level designs where you store a set of events and fold them.

**The .NET answer for "build Google Docs-style collaboration":** the two families are **CRDTs** (Yjs/Automerge, peer-friendly, no server authority needed) and **OT (operational transformation)** (a central server transforms concurrent ops; what Google Docs actually uses, and Figma's multiplayer is a custom LWW-per-property design over a central server). Naming all three and noting that a server-authoritative design lets you use simpler algorithms is a complete answer.

---

## Concept 31 — Coordination avoidance: CALM and invariant confluence

Two results that give you a principled way to answer "does this need coordination?"

**The CALM theorem** (Hellerstein; proved by Ameloot et al.): *a program has a consistent, coordination-free distributed implementation **if and only if** it is **monotonic*** — meaning new inputs only ever add to the output, never retract it.

- **Monotonic, so coordination-free:** set union, counting-up, "has this ever happened?", max/min, append-only logs, most read models, reachability queries, any aggregation that only accumulates.
- **Non-monotonic, so coordination-requiring:** anything that can be *invalidated* by a later fact. Counting to a final total (you must know there's no more input), "is this the *only* one?", uniqueness constraints, "did everyone vote?", negation, deletion followed by a query.

This is the deepest useful idea in the module. The practical technique it hands you: **make your operations monotonic and you can drop the coordination.** Prefer append-only event logs over mutable state; prefer "add to a set" over "set a value"; make deletes into monotonic tombstone-adds; replace "is this the only order?" with "collect all orders, and resolve later".

**Invariant confluence** (Bailis et al., *Coordination Avoidance in Database Systems*, VLDB 2015) makes it operational: an invariant is **I-confluent** with respect to a set of operations if merging any two I-valid divergent states yields an I-valid state. If yes, you can execute without coordination. The paper works through actual SQL constraints:

| Invariant | Coordination needed? |
|---|---|
| Per-record check constraint (`amount > 0`) | **No** — each replica can enforce it locally |
| Foreign key, on insert | **No** (with care around concurrent parent delete) |
| Foreign key, with cascading delete | **Yes** — delete + insert can interleave badly |
| `UNIQUE` constraint / auto-increment | **Yes** — two replicas can each accept "the only one" |
| `SUM(balance) ≥ 0` / inventory limits | **Yes** — and this is the escrow case |
| Monotonic status transitions (Pending → Shipped, never back) | **No**, if the transition lattice is a join |

**How to use this in an interview.** When asked to design something at scale, enumerate the invariants first and classify them: *"Most of these invariants are per-record and I-confluent, so they can be enforced locally at each replica with no coordination. Two of them — email uniqueness and the inventory limit — are not, so those are the only places I'll pay for coordination, and I'll handle each differently: uniqueness by making it partition-local through the key design, and inventory with per-region escrow."* That is precisely the reasoning senior/architect rubrics are looking for, and almost nobody does it unprompted.

---

## Concept 32 — Escrow and reservations: buying coordination-freedom for numeric limits

The pattern that dissolves the hardest non-I-confluent invariant (a global limit) into local decisions.

**The idea:** partition the *budget* rather than coordinating the *decisions*. If you have 1,000 tickets and 4 regions, give each region 250. Each region sells locally with no cross-region coordination, at local latency, and stays correct globally because the sum of budgets never exceeds the limit. When a region runs low it requests a transfer — a rare, coordinated operation rather than a per-request one.

```csharp
// Escrow: a per-node budget, replenished rarely. Coordination cost amortized over N sells.
public sealed class EscrowAllocator(IBudgetStore store, string nodeId)
{
    private long _local;                      // tickets this node may sell without asking anyone

    public async ValueTask<bool> TryReserveAsync(long qty, CancellationToken ct)
    {
        if (Interlocked.Add(ref _local, -qty) >= 0) return true;   // fast path: no network at all
        Interlocked.Add(ref _local, qty);                          // put it back

        // Slow path: coordinate once for a whole new block (a conditional write, not a lock).
        var block = await store.TryTakeBlockAsync(nodeId, blockSize: 100, ct);
        if (block == 0) return false;                              // globally exhausted
        Interlocked.Add(ref _local, block);
        return await TryReserveAsync(qty, ct);
    }
}
```

**Where you already use this without naming it:** database sequence caching (`CACHE 100` on a Postgres sequence, SQL Server `SEQUENCE ... CACHE`, Hi/Lo ID generation in EF Core), distributed rate limiters that hand out token batches, Kafka producer ID blocks, Snowflake/ULID ID schemes that avoid a central sequencer entirely, and DynamoDB/Cosmos RU budgets.

**The trade-offs to state:** stranded capacity (a region with idle budget while another is exhausted), so you need a rebalancing or return policy; a harder story for exact "sold out" moments; and over-selling is impossible but *under*-selling is possible, which is usually the acceptable direction. Mention **overbooking as a business decision** too — airlines deliberately choose eventual consistency with compensation because coordination costs more than the occasional bump.

**The general principle, which generalizes far beyond counters:** *"Convert a global invariant into a set of local invariants whose conjunction implies it."* Per-region write ownership (Module 7), per-partition uniqueness through key design (Module 8), and escrow are all instances. This sentence is worth memorizing verbatim.

---

## Concept 33 — Where consensus belongs: control plane vs data plane

The synthesis of the module, and the answer to most "how does this fit together" questions.

```
┌──────────────────────── CONTROL PLANE (consensus) ─────────────────────────┐
│  small, critical, low-rate decisions — strongly consistent                 │
│  • who is leader / who owns partition 17    • cluster membership           │
│  • shard map & routing table                • config, feature flags        │
│  • schema version, migration state          • lock / lease grants          │
│  3 or 5 nodes · Raft · ~1–3 ms commits · thousands of ops/sec ceiling      │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │ pushes decisions down; data plane CACHES them
               ▼
┌──────────────────────── DATA PLANE (no consensus) ─────────────────────────┐
│  high-rate user traffic — as weakly coordinated as the invariants allow    │
│  • partition-owned writes (one owner ⇒ no lock needed)                     │
│  • idempotent handlers + dedup (at-least-once everywhere)                  │
│  • optimistic concurrency per record (local, I-confluent invariants)       │
│  • CRDT / monotonic structures where merge is definable                    │
│  • escrow budgets for limits          • fencing tokens on every side effect│
│  millions of ops/sec · survives control-plane outage on cached state       │
└────────────────────────────────────────────────────────────────────────────┘
```

**Real systems, mapped:**

| System | Control plane | Data plane |
|---|---|---|
| **Kubernetes** | etcd + API server (Raft) | kubelets keep running pods when the API server is down |
| **Kafka (KRaft)** | Controller quorum (Raft) for metadata | Per-partition leader/ISR replication; producers/consumers unaffected by controller elections |
| **Cosmos DB** | Partition map, replica set membership | Quorum reads/writes within a partition |
| **Spanner / CockroachDB** | Placement driver, range metadata | Per-range Paxos/Raft; 2PC across ranges only when needed |
| **Service Fabric** | Federation ring + failover manager | Stateful partition primaries |
| **Orleans** | Membership table + gossip | Grain calls routed by a directory; grains own their state |
| **Your app** | Blob lease / etcd / SQL applock for ownership | Idempotent handlers, per-entity OCC, partitioned queues |

**The three rules:**

1. **Consensus decides; replication carries.** Never route user data through a consensus group that doesn't have to be there.
2. **Cache control-plane decisions in the data plane and keep serving on stale ones**, with a bounded staleness and fencing so a stale decision can't corrupt anything.
3. **Failure of the control plane should degrade change, not traffic.** "New deployments and failovers stop; existing traffic continues" is a good outcome. "Everything stops" means you built a data-plane dependency and called it a control plane.

---

# Part F — Operating it

## Concept 34 — What breaks consensus clusters in production

The operational knowledge that separates "I read the Raft paper" from "I have been paged for etcd".

| Failure mode | Symptom | Why | Mitigation |
|---|---|---|---|
| **Slow disks** | Leader elections every few minutes; write latency spikes | Every Raft commit fsyncs; a slow or throttled volume delays the apply loop and heartbeats | Dedicated local NVMe; never network file shares; `etcd_disk_wal_fsync_duration_seconds` p99 as an SLI |
| **Noisy neighbours / CPU steal** | Random elections | Heartbeat deadlines missed | Dedicated nodes; CPU reservations; raise heartbeat/election timeouts |
| **Cross-region membership** | Constant instability | broadcastTime ≪ electionTimeout violated | One group per region; async replication between; or dramatically raised timeouts, accepting slow failover |
| **Quota exhaustion** | Cluster goes **read-only**, `NOSPACE` alarm | 2 GB default backend quota | Auto-compaction (`--auto-compaction-retention`), periodic defrag, alert at 60% of quota |
| **Keyspace bloat from watches/events** | Memory growth, slow reads | Unbounded history; too many watchers | Compaction policy, bounded watch counts, don't store history here |
| **Quorum loss** | Cluster completely unavailable for writes | Lost ⌈N/2⌉ members | Restore from snapshot on one node with `--force-new-cluster`, then re-add members. **This is a data-loss-risk operation** and must be rehearsed |
| **Restoring a backup onto a live cluster** | Split brain, divergent history | Two clusters believe they're authoritative | Runbook discipline; distinct cluster IDs; fencing at the client tier |
| **Clock jumps** | Lease-read anomalies; weird timestamps | Lease reads depend on bounded drift | Monitor NTP offset; prefer ReadIndex over lease reads if drift isn't controlled |
| **Membership mistakes** | Accidental quorum loss during maintenance | Adding two members, or adding before removing a dead one | One change at a time; learners first; remove-then-add during recovery |

**Metrics that should be on the dashboard, and naming them is a credibility marker:** leader changes per hour (should be ~0), `has_leader`, proposal commit/apply/failed rates, `wal_fsync` and `backend_commit` p99, DB size vs quota, watcher count, and slow-read/apply warnings in the logs.

**Backups:** snapshot regularly (`etcdctl snapshot save`), verify by *restoring* into a throwaway cluster (an unverified backup is a rumour), and store the snapshot outside the failure domain of the cluster.

## Concept 35 — Designing for coordination-service failure

State these as design requirements, not afterthoughts:

1. **Bounded dependency**: the data plane must have a documented behaviour for "coordination service unreachable" — usually "continue on last-known-good for up to T, then degrade to read-only or shed load".
2. **No fate-sharing**: don't put the coordination cluster's own dependencies (DNS, certificates, the container platform) in a cycle with it. Circular dependencies during cold start are one of the classic ways a whole platform fails to recover; Amazon's Builders' Library on static stability is the canonical reading.
3. **Static stability**: pre-provision so that failover requires no new resources and no control-plane calls. A failover that needs to *scale up* during an incident is a failover that fails during an incident.
4. **Rehearsal**: leader kill, quorum loss, and restore-from-snapshot must all be practised drills with a written runbook. If you've never lost quorum on purpose, you don't know your RTO.

---
# Putting it together

## Worked example 1 — "Only one instance should run this job" (the .NET interview classic)

**Prompt:** an ASP.NET Core service runs on 5 pods. Every 15 minutes it must reconcile pending payments against the provider and mark them settled. Today it runs on all 5 and occasionally double-settles. Fix it.

**Step 1 — Clarify the requirement before reaching for a lock.** Is the harm *duplicated work* (efficiency) or *duplicated effect* (correctness)? Double-settlement is a correctness problem involving money, so the answer must be fenced or idempotent, not merely "usually one".

**Step 2 — Try to eliminate coordination (rung 0/1).** The strongest design here doesn't use leader election at all:

- Make the settlement operation **idempotent** at the boundary: `UPDATE Payments SET Status='Settled', SettledAt=@now WHERE Id=@id AND Status='Pending'`. The affected-row count tells the worker whether it was the one that transitioned the payment; zero means someone else already did it, and it stops. Any call to the provider carries an **idempotency key** derived from the payment ID so a duplicate call is a no-op provider-side.
- **Partition the work** so concurrent workers don't collide: claim batches with `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 100` (Postgres) or an `UPDATE TOP (100) ... OUTPUT` claim pattern (SQL Server). Now five pods process disjoint payments, with no lock service, five times the throughput, and no single point of failure.

**Say this explicitly:** *"My first choice is to make the work claimable and idempotent rather than to elect a leader, because that gets correctness from the database I already trust, and it scales to five workers instead of wasting four."* Many candidates never consider this, and it is the better design.

**Step 3 — When a singleton genuinely is required** (a non-parallelizable reconciliation, a third-party API that rate-limits per account, a stateful rebuild), use a lease with fencing:

- **Lease**: Azure Blob lease (30 s, renewed every 10 s) or `sp_getapplock` on the existing SQL Server, or an etcd lease if you already run etcd.
- **Epoch**: on each acquisition, atomically increment an epoch stored in the leased blob (or a SQL row). That's your fencing token.
- **Fencing at the resource**: every write from the job includes `WHERE @epoch >= LastEpoch`, and the job's first act is `UPDATE JobOwnership SET LastEpoch=@epoch WHERE JobName='settlement' AND LastEpoch < @epoch` — zero rows means a newer leader already exists and this instance exits immediately.
- **Renewal failure cancels work**: link the work's `CancellationToken` to renewal failure (`HandleLostToken`, or the pattern in Concept 27).
- **Graceful release** on shutdown so handoff takes milliseconds, plus `IHostApplicationLifetime` wiring so a rolling deployment doesn't cause a 30-second gap.

**Step 4 — Name the residual risks yourself.** This is where the interview is won:

1. A 40-second GC pause exceeds the lease; the epoch check in the database is what stops the corruption, not the lease.
2. The job holds the lease but is *stuck* (deadlock, hung HTTP call). The lease renews happily and nothing progresses. Add a **liveness signal separate from the lease** — a heartbeat with a progress counter, and an alert on "leader present but no progress in N minutes". This failure is more common in practice than split brain, and almost no candidate raises it.
3. Reconciliation itself must be resumable, because a leadership change mid-run will happen. Checkpoint per batch.
4. Clock: renewal arithmetic on `Stopwatch`, not `DateTime.UtcNow`.

**Step 5 — Observability.** Leadership transitions per hour, current leader identity, epoch value, lease-renewal failures, time-since-last-successful-run as the real SLI (the user-facing invariant is "reconciliation ran within 20 minutes", not "someone holds the lease").

---

## Worked example 2 — Global inventory for a flash sale (the coordination-avoidance question)

**Prompt:** 3 regions, 200,000 concurrent users, 10,000 units of one item. Must not oversell. Latency target: 100 ms p99 for "add to cart".

**Step 1 — Name the invariant and classify it.** `sum(reserved) ≤ 10,000` is a global aggregate limit: **not I-confluent** (Concept 31), so it fundamentally requires coordination *unless restructured*. Everything else (per-user cart validity, per-item price) is per-record and local.

**Step 2 — Reject the naive designs, with reasons and numbers.**

- *A distributed lock per purchase*: consensus round trip per request. At ~2 ms in-region and ~120 ms cross-region, the global-lock design caps at a few hundred to a few thousand serialized ops/sec and blows the latency budget for two of three regions. Rejected on arithmetic, not taste.
- *A single strongly-consistent row in one region*: same problem plus a single point of failure, though it's the right answer for 10 units instead of 10,000.
- *Eventually-consistent per-region counters*: oversells, because each region independently believes there's stock.

**Step 3 — Escrow.** Split 10,000 into per-region budgets proportional to forecast demand (say 5,000 / 3,000 / 2,000), then within each region split again into per-node blocks of 100.

- A reservation is a **local, in-memory decrement** — no network, microseconds, trivially within budget.
- A node exhausting its block takes another from the regional pool with a **single conditional write** (Cosmos `IfMatchEtag` / etcd `Txn` / SQL `UPDATE ... WHERE Remaining >= 100`). One coordinated operation per 100 sales: coordination cost amortized 100:1.
- Regional exhaustion triggers a **rebalance** request against a small consensus-backed global pool. Rare, so it can be slow.
- Unsold budget is **returned** on a timer, and expired cart reservations return to the local block — this is the compensating path that keeps stranded inventory bounded.

**Step 4 — Correctness argument, stated plainly.** The sum of all budgets never exceeds 10,000 because budgets are only created by conditional writes against the global pool, and each conditional write decrements it atomically. Local decrements can only reduce a budget. Therefore total reservations ≤ 10,000. **Overselling is impossible; underselling (stranded stock) is possible and bounded by the number of nodes × block size.** Note out loud that you chose the direction of the error deliberately.

**Step 5 — The parts that make it a senior answer.**

- **Idempotency**: "add to cart" carries a client-generated request ID; a retry doesn't consume two units.
- **Reservation TTL**: carts expire in 10 minutes and return units. Reservation, not sale.
- **Fencing on the budget block**: a node that pauses and resumes must not sell from a block that's been reclaimed — so blocks carry a lease and a generation number, and the reservation write checks it.
- **Degradation**: if the global pool is unreachable, nodes keep selling from existing blocks and stop when exhausted. Availability degrades to "sold out" rather than "error", which is the correct product behaviour.
- **The business conversation**: at some point, ask whether slight overselling with compensation (a voucher) is cheaper than the engineering. Airlines already answered this. Raising it shows you understand that consistency requirements are negotiated, not given.

---

## Common questions and what a strong answer contains

**"What is consensus?"** Agreement + integrity + termination; safety unconditional, liveness conditional. Then immediately: leader election, total order broadcast, atomic commit, linearizable CAS and distributed locking are the same problem.

**"Explain Raft."** Terms as a logical clock; randomized election timeouts; RequestVote with the log-up-to-date restriction; AppendEntries with the consistency check giving the Log Matching Property; commit on majority; leaders only append. Then the two subtleties: the election restriction is why committed entries survive, and entries from earlier terms can't be committed by majority count alone (hence the no-op).

**"Paxos vs Raft?"** Equivalent guarantees. Raft constrains (strong leader, no holes, ordered commit) for understandability and operability; Multi-Paxos and descendants relax for throughput and WAN behaviour. Then: I'd use an existing implementation; Paxos Made Live shows the algorithm is the small part.

**"Why a majority? Why not 2 of 4?"** Quorum intersection is the safety property. N = 2f+1; even sizes raise the quorum without raising tolerance. And: two failure domains can never form a majority, so cross-region strong consistency needs three.

**"How do you do a linearizable read from a Raft cluster?"** Not by reading from the leader — it may be deposed. ReadIndex (confirm leadership with a heartbeat round, then read locally), a lease read (zero RTT, assumes bounded drift), or put the read through the log. Then mention etcd's linearizable vs serializable read options as the concrete instance.

**"Design distributed leader election."** Lease + fencing token + renewal at 1/3 + cancel-on-renewal-failure + graceful release + the resource validating the epoch. Then the residual risks: pauses beyond the lease, a live-but-stuck leader, resumability across handoff.

**"Is a distributed lock safe?"** Only for efficiency unless the resource validates a fencing token. Explain the pause timeline. Then the ladder: eliminate the need, idempotency, OCC, lease + fence, lock service.

**"Redis or ZooKeeper for locking?"** Wrong axis. The question is whether the lock is load-bearing for correctness. If yes, you need a linearizable store *and* fencing. If no, one Redis with `SET NX PX` and a compare-then-delete release script is fine and simpler. Summarize Redlock's critique fairly.

**"Only one instance of this background job — how?"** Worked example 1. Lead with "can I make it claimable and idempotent instead?"

**"2PC or saga?"** 2PC requires unanimity, holds locks across the network, and blocks on coordinator failure; consensus-backed coordinators (Spanner, CockroachDB) fix the blocking. Sagas trade atomicity for availability and make compensation explicit. Cross-service: saga plus idempotency. Same database: one local transaction. Also: MSDTC is Windows-only and needs opting in on .NET 7+.

**"How does a consensus system scale writes?"** It doesn't — you run many groups, one per partition, and co-locate what must be transactional. Then note the cost: cross-group transactions need 2PC over the groups.

**"Can you get exactly-once delivery?"** No (Two Generals). You get at-least-once plus idempotency, or at-most-once. Kafka transactions give exactly-once *within* Kafka only. Mechanisms: idempotency keys with a unique index, dedup windows, naturally idempotent operations.

**"When would you use CRDTs?"** When the merge is definable and there's no cross-replica invariant: counters, sets, presence, collaborative text, offline-first clients. Then the three costs: metadata/tombstone growth, no global invariants, and semantics fixed by the type.

**"How do you order events across services without a central sequencer?"** HLC for causally-consistent near-wall-clock timestamps; vector clocks if you need to *detect* concurrency; Lamport if any consistent total order will do; TrueTime-style commit-wait if you must have external consistency and can bound clock error.

**"What happens when etcd/ZooKeeper is down?"** Writes to the control plane stop; the data plane should keep serving from cached decisions with bounded staleness and fencing. Then the operational answers: quorum-loss recovery from snapshot, and why the 2 GB quota can silently make the cluster read-only.

**"Does your design have a split-brain risk?"** Walk the three places it can appear: two leaders (prevented by majority election + persisted votes), two lock holders (prevented by fencing, not by the lock), and two clusters (prevented by runbook discipline around restores).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "We'll use a distributed lock" and moving on | Distinguishes efficiency from correctness, then adds a fencing token or removes the need for the lock |
| Assuming a lease can't expire while you hold it | Walks the GC-pause timeline and explains why only the resource can prevent the double write |
| "Read from the leader for strong consistency" | ReadIndex or lease read; names the deposed-leader stale-read window |
| Explaining Raft as only "elect a leader, replicate a log" | Adds the election restriction and the previous-term commit rule (the no-op) |
| "Quorum means most of the nodes" | N = 2f+1, quorum intersection as the safety argument, never even-sized, two regions can't form a majority |
| Calling 2PC consensus | Unanimity vs majority; blocking coordinator; in-doubt transactions; consensus-backed coordinators as the fix |
| Putting consensus in the request path | Control plane vs data plane; quantifies the RPS ceiling and the latency floor |
| Using a coordination service as a queue or a database | Names the quota/request limits and the read-only-on-NOSPACE failure; routes high-rate data elsewhere |
| One giant etcd/ZooKeeper cluster for everything | Blast radius, separate clusters per concern, and a data plane that survives control-plane outage |
| `DateTime.UtcNow` for lease arithmetic | Monotonic clock plus a conservative margin; mentions NTP steps |
| Retrying a `DbUpdateConcurrencyException` blindly | Treats it as a domain event; merges or rejects per field |
| "Exactly-once with Kafka/Service Bus" | At-least-once plus idempotency; exactly-once only within one system's transactional boundary |
| Proposing CRDTs as a general consistency solution | Names the invariant limitation and the metadata growth; reaches for escrow when there's a limit |
| Never questioning the consistency requirement | Enumerates invariants, classifies them as local vs global, and pays for coordination only where required |
| Ignoring what happens when the lock service is down | States the degradation behaviour and whether it's fail-open or fail-closed, with the business consequence |
| Only considering split brain, not the stuck leader | Adds a progress-based liveness signal independent of the lease |
| No numbers | 1 RTT to a majority; ~1–3 ms in-region; 70–150 ms cross-region; ~2 GB etcd quota; 15–60 s blob lease |

---

## Practice exercises

**Exercise 1 — Implement Raft leader election and log replication.** Use the [Raft paper's Figure 2](https://raft.github.io/raft.pdf) as the spec and build it in C# with an in-process simulated network you control (message delay, drop, reorder, partition). Implement `RequestVote` and `AppendEntries` only, with persisted `currentTerm`/`votedFor`. Then write tests that assert the invariants: at most one leader per term, Log Matching, and that a committed entry is never lost. Then deliberately break the election restriction and watch a committed entry disappear — that experiment teaches more than re-reading the paper. Budget: a weekend. There's no faster way to be able to discuss Raft with authority.

**Exercise 2 — Cause split brain, then fence it.** Two console apps competing for an Azure Blob lease (or Postgres advisory lock), each incrementing a counter in a SQL table when it believes it's the leader. Force the failure by pausing one process (`kill -STOP` on Linux, or `GC.TryStartNoGCRegion` abuse / a `Thread.Sleep` inside the critical section, or simply attaching a debugger and breaking) for longer than the lease. Observe the double write. Then add an epoch column and the `WHERE LastEpoch < @epoch` guard, repeat, and observe the stale writer being rejected. **This is the single most valuable exercise in the module** because it converts Concept 23 from a fact you know into a thing you've seen.

**Exercise 3 — Measure the cost of consensus.** Run a 3-node etcd cluster in Docker (or `dotnet-etcd` against `etcd` in .NET Aspire). Benchmark: single-key writes/sec and p99; the same with batching; linearizable vs serializable reads; then add `tc netem` delay of 50 ms between two members and re-measure. Kill the leader while writing continuously and measure the write-unavailability window; then raise heartbeat/election timeouts and measure again. You'll end up with your own version of the Concept 16 table, which means you can quote numbers you've personally measured.

**Exercise 4 — The claimable-work queue.** Build a competing-consumers worker over Postgres using `SELECT ... FOR UPDATE SKIP LOCKED`, with visibility timeouts, retry counts, and a dead-letter table. Run 8 workers against 100,000 jobs and verify every job executed exactly once *in effect* (idempotency key + a results table). Then do the same with a single distributed lock around the whole queue and compare throughput. The ratio is the argument for rung 0 of Concept 25's ladder.

**Exercise 5 — CRDTs and their limits.** Implement `GCounter`, `PNCounter`, `LWWRegister`, and `ORSet` in C# as immutable records with a `Merge`. Property-test the three laws (commutativity, associativity, idempotence) with FsCheck. Then attempt to add the invariant "counter never goes below zero" and articulate precisely why you can't — that's Concept 30's limitation, discovered rather than memorized. Finally implement the escrow allocator from Concept 32 and show that it enforces the limit at the cost of possible under-allocation.

**Exercise 6 — The architect write-up.** One page, in the shape of a real design review: for a system you know, enumerate every invariant, classify each as local/I-confluent or global/coordination-requiring, and for each global one state the mechanism you'd use (key design to make it partition-local, escrow, consensus, or a negotiated business compensation). Then list every place your current architecture uses a lock and decide which ladder rung it should actually be on. This is close to an architect-round take-home and it will probably find a real bug in something you own.

---

## Free resources

### Foundational papers (all free)

| Resource | What it covers | Why read it |
|---|---|---|
| [In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf) — Ongaro & Ousterhout, 2014 | The whole algorithm, with Figure 2 as a complete spec | **The single most important read in this module.** Figure 2 and §5.4 are the interview material |
| [Diego Ongaro's PhD thesis: Consensus — Bridging Theory and Practice](https://github.com/ongardie/dissertation) | Raft plus everything the paper omits: membership changes, log compaction, client interaction, **linearizable read-only queries (ReadIndex/lease)** | Where the "hard parts" of Concept 11 are properly specified |
| [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) — Lamport, 2001 | Single-decree and Multi-Paxos, in 11 pages | The canonical statement; short enough to read twice |
| [The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) — Lamport, 1998 | The original, in Greek-parliament allegory | Historical, and genuinely fun once you know the algorithm |
| [Paxos Made Live — An Engineering Perspective](https://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf) — Chandra, Griesemer & Redstone, 2007 | What it took to ship Paxos inside Chubby: testing, disk corruption, snapshots, tooling | **The best "theory vs practice" paper in distributed systems.** Quote it when asked whether you'd implement consensus yourself |
| [Impossibility of Distributed Consensus with One Faulty Process (FLP)](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) — Fischer, Lynch & Paterson, 1985 | The impossibility result | Read the statement and the intuition; you don't need the full proof |
| [Consensus in the Presence of Partial Synchrony](https://groups.csail.mit.edu/tds/papers/Lynch/jacm88.pdf) — Dwork, Lynch & Stockmeyer, 1988 | The model every real system uses | The formal home of "safe when async, live when synchronous enough" |
| [The Chubby lock service](https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf) — Burrows, 2006 | Coarse-grained locking, sequencers, caching with invalidation, and the organizational failure modes | The best treatment of locks-as-a-service, including why fencing exists |
| [ZooKeeper: Wait-free coordination](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf) — Hunt et al., 2010 | znodes, sessions, watches, the guarantee set, and the local-read design | Explains *why* reads aren't linearizable by default |
| [Zab: High-performance broadcast for primary-backup systems](https://marcoserafini.github.io/papers/zab.pdf) — Junqueira, Reed & Serafini, 2011 | ZooKeeper's atomic broadcast and primary order | The "what's different from Paxos" answer for ZooKeeper |
| [Viewstamped Replication Revisited](https://pmg.csail.mit.edu/papers/vr-revisited.pdf) — Liskov & Cowling, 2012 | The RSM protocol that predates Paxos in publication | Useful for the family-tree question |
| [Flexible Paxos: Quorum intersection revisited](https://arxiv.org/abs/1608.06696) — Howard, Malkhi & Spiegelman, 2016 | Quorums need only intersect across phases | Elegant, short, and a strong signal if you can state the result |
| [There Is More Consensus in Egalitarian Parliaments (EPaxos)](https://www.cs.cmu.edu/~dga/papers/epaxos-sosp2013.pdf) — Moraru et al., 2013 | Leaderless consensus, one-RTT commits for non-interfering commands | The WAN-optimized end of the design space |
| [Practical Byzantine Fault Tolerance](https://pmg.csail.mit.edu/papers/osdi99.pdf) — Castro & Liskov, 1999 | 3f+1 BFT made fast | The BFT reference |
| [The Byzantine Generals Problem](https://lamport.azurewebsites.net/pubs/byz.pdf) — Lamport, Shostak & Pease, 1982 | The 3f+1 bound | Read the problem statement at minimum |
| [Spanner: Google's Globally-Distributed Database](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf) | Paxos groups per shard, TrueTime, commit-wait, 2PC over Paxos | The best worked example of consensus composed at scale |
| [Coordination Avoidance in Database Systems](https://www.vldb.org/pvldb/vol8/p185-bailis.pdf) — Bailis et al., VLDB 2015 | Invariant confluence, with real SQL constraints classified | **The paper that makes Concept 31 actionable** |
| [Keeping CALM: When Distributed Consistency Is Easy](https://arxiv.org/abs/1901.01930) — Hellerstein & Alvaro, 2019 | Monotonicity ⟺ coordination-freedom | The cleanest statement of the deepest idea here |
| [A Comprehensive Study of CRDTs](https://inria.hal.science/inria-00555588/document) — Shapiro, Preguiça, Baquero & Zawirski, 2011 | The full catalogue of CvRDTs and CmRDTs with proofs | The reference; skim the catalogue, read the definitions |
| [Conflict-free Replicated Data Types (the short paper)](https://inria.hal.science/hal-00932836/document) | The concise version | Read this one first |
| [Logical Physical Clocks (HLC)](https://cse.buffalo.edu/tech-reports/2014-04.pdf) — Kulkarni et al., 2014 | Hybrid logical clocks | Short, practical, and used by every modern distributed database |
| [Time, Clocks, and the Ordering of Events](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) — Lamport, 1978 | Happens-before and logical clocks | The origin of everything in Concept 29 |
| [Unreliable Failure Detectors for Reliable Distributed Systems](https://www.cs.cornell.edu/home/sam/FDpapers/CT96-JACM.pdf) — Chandra & Toueg, 1996 | The failure-detector hierarchy; ◇W as the weakest sufficient detector | The formal answer to "why timeouts?" |
| [The φ Accrual Failure Detector](https://www.researchgate.net/publication/29682135_The_ph_accrual_failure_detector) — Hayashibara et al., 2004 | Continuous suspicion levels instead of a boolean | The detector Cassandra and Akka use |
| [SWIM: Scalable Weakly-consistent Infection-style Membership](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf) — Das, Gupta & Motivala, 2002 | Gossip membership with indirect probing | The other half of Concept 19 |

### Explainers, courses, and interactive material (free)

| Resource | What it covers |
|---|---|
| [The Secret Lives of Data — Raft visualization](http://thesecretlivesofdata.com/raft/) | An animated, narrated walkthrough of Raft elections and replication. Start here before the paper |
| [raft.github.io](https://raft.github.io/) | The official Raft page: the interactive cluster simulator, the paper, and a list of ~100 implementations |
| [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — Kleppmann | The fencing-token argument, with the pause timeline diagram. **Required reading for Part D** |
| [Is Redlock safe?](http://antirez.com/news/101) — antirez | The other side of the argument |
| [Distributed Locks with Redis](https://redis.io/docs/latest/develop/use-cases/patterns/distributed-locks/) | The Redlock algorithm as specified, plus the single-instance pattern and the release script |
| [Martin Kleppmann's distributed systems lecture series](https://www.youtube.com/playlist?list=PLeKd45zvjcDFUEv_ohr_HdUFe97RItdiB) + [free lecture notes](https://www.cl.cam.ac.uk/teaching/2122/ConcDisSys/dist-sys-notes.pdf) | Cambridge course covering logical time, broadcast, consensus, Raft, and linearizability. The best free lecture material on this module's content |
| [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.5840/) | Lecture notes and the canonical paper list; the Raft labs are the gold-standard exercise |
| [Distributed Systems for Fun and Profit](http://book.mixu.net/distsys/single-page.html) — Takada | Free short book; chapter 4 covers replication and consensus at exactly the right level |
| [Jepsen: consistency models](https://jepsen.io/consistency) and [analyses](https://jepsen.io/analyses) | Formal definitions plus real systems tested against their claims. Read the [etcd/ZooKeeper](https://aphyr.com/posts/291-jepsen-zookeeper) and [Redis](https://aphyr.com/posts/283-jepsen-redis) posts |
| [Notes on Distributed Systems for Young Bloods](https://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/) — Jeff Hodges | Hard-won operational wisdom; the "coordination is expensive" instinct in prose |
| [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) | Useful companion to Module 7, and clarifies what consensus actually gives you |
| [TLA+ Video Course](https://lamport.azurewebsites.net/video/videos.html) — Lamport | Specifying and model-checking concurrent algorithms. Raft, etcd, and Cosmos DB all have TLA+ specs |
| [etcd: Raft implementation notes & tuning](https://etcd.io/docs/latest/tuning/) | Heartbeat/election tuning, disk and network requirements — the operational half of Concept 34 |
| [ZooKeeper Recipes and Solutions](https://zookeeper.apache.org/doc/current/recipes.html) | Leader election, locks, barriers, queues — including the herd-effect fix |
| [Amazon Builders' Library: Leader election in distributed systems](https://aws.amazon.com/builders-library/leader-election-in-distributed-systems/) | When to use leader election, fencing, and the failure modes. Written by people who operate it |
| [Amazon Builders' Library: Static stability using Availability Zones](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/) | Why failover shouldn't depend on the control plane |
| [Amazon Builders' Library: Challenges with distributed systems](https://aws.amazon.com/builders-library/challenges-with-distributed-systems/) | Partial failure and the "slow vs dead" problem in production language |
| [Google SRE Book: Managing Critical State (Distributed Consensus)](https://sre.google/sre-book/managing-critical-state/) | Chapter 23 — how many replicas, where to put them, consensus performance, and monitoring. **The most directly interview-useful free chapter on this topic** |

### .NET and Azure documentation

| Resource | What it covers |
|---|---|
| [DistributedLock (madelson) — README](https://github.com/madelson/DistributedLock) | Locks, reader-writer locks and semaphores over SQL Server, Postgres, MySQL, Oracle, Redis, Azure blobs, ZooKeeper, files; `HandleLostToken` and provider semantics |
| [Azure Blob lease (REST) ](https://learn.microsoft.com/en-us/rest/api/storageservices/lease-blob) and [BlobLeaseClient](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.specialized.blobleaseclient) | 15–60 s or infinite leases, renew/change/break semantics, and lease-conditional writes |
| [Leader Election pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/leader-election) — Azure Architecture Center | The pattern, with the blob-lease implementation and its caveats |
| [Optimistic concurrency in EF Core](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) | `[Timestamp]`, `IsConcurrencyToken`, `DbUpdateConcurrencyException`, and resolution strategies |
| [Optimistic concurrency in Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/database-transactions-optimistic-concurrency) | ETags, `IfMatchEtag`, 412 handling, transactional batch within a partition key |
| [Cosmos DB conflict resolution policies](https://learn.microsoft.com/en-us/azure/cosmos-db/conflict-resolution-policies) | LWW vs custom merge procedures vs the conflict feed — CRDT thinking in a managed service |
| [`sp_getapplock` (T-SQL)](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-getapplock-transact-sql) | A real distributed lock in a database you already run; principals and lock owners |
| [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS) and [`SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE) | Session vs transaction-scoped locks; the claimable-work-queue pattern |
| [Azure Service Bus message sessions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions) and [message locks / lock renewal](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement) | Per-entity serialization and per-message leases — coordination you already own |
| [Service Bus duplicate detection](https://learn.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection) | The dedup-window mechanism, and its bounded protection |
| [Idempotent APIs and the `Idempotency-Key` header](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-mission-critical/mission-critical-data-platform) / [Stripe's idempotency docs](https://docs.stripe.com/api/idempotent_requests) | The industry-standard shape for exactly-once effects |
| [Microsoft Orleans: cluster management](https://learn.microsoft.com/en-us/dotnet/orleans/implementation/cluster-management) | The membership protocol, suspicion votes, and why single activation is best-effort |
| [Orleans grain placement & single activation](https://learn.microsoft.com/en-us/dotnet/orleans/grains/grain-placement) | Coordination by placement — rung 0 of the ladder, as a framework feature |
| [Azure Functions singleton / `[Singleton]`](https://github.com/Azure/azure-webjobs-sdk/wiki/Singleton) | Blob-lease-based single-instance execution and its scopes |
| [Kubernetes Leases & leader election](https://kubernetes.io/docs/concepts/architecture/leases/) | The `Lease` object, typical timings, and the explicit statement that it isn't fenced |
| [etcd API and concurrency (Lock/Election) docs](https://etcd.io/docs/latest/learning/api/) | Txn semantics, leases, watch-from-revision, and the server-side election/lock recipes |
| [dotnet-etcd](https://github.com/shubhamranjan/dotnet-etcd) | The practical .NET etcd client, if you do run etcd |
| [Dapr distributed lock building block](https://docs.dapr.io/developing-applications/building-blocks/distributed-lock/distributed-lock-api-overview/) | A portable lease API, with the same expiry semantics as everything else here |
| [Saga distributed transactions pattern](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga) | The alternative to 2PC across services (previews Module 12) |
| [Distributed transactions in .NET 7+](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/7.0/distributed-transaction-support) | MSDTC support, Windows-only, and the opt-in flag |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What consensus is | Agreement + integrity + termination; safety unconditional, liveness conditional |
| Consensus in disguise | Leader election, total order broadcast, atomic commit, linearizable CAS, locks, uniqueness — all one problem |
| System model | Safe when asynchronous, live under partial synchrony; crash-recovery means fsync the term and the vote |
| FLP | No deterministic async algorithm is both always safe and always live; hence timeouts, hence election tuning |
| Failure detection | You can't distinguish slow from dead; completeness vs accuracy; GC pauses are the .NET version |
| Quorums | N = 2f+1, intersection is the safety property, never even, two regions can't form a majority |
| Raft | Terms, randomized elections, election restriction, log matching, leaders only append, no-op for previous-term commits |
| Linearizable reads | ReadIndex or lease read, never a naive leader read; etcd's linearizable vs serializable option |
| Membership changes | One voter at a time, learners first, remove-before-add during recovery |
| Paxos vs Raft | Equivalent guarantees; Raft constrains for understandability; Paxos Made Live says the algorithm is the small part |
| Scaling consensus | Many groups, one per partition; cross-group needs 2PC over the groups |
| 2PC | Unanimity not majority; blocks on coordinator failure with locks held; consensus-backed coordinator is the fix |
| Cost of consensus | 1 RTT to a majority + fsync: ~1–3 ms in-region, 70–150 ms cross-region; batch and pipeline |
| ZooKeeper | Zab, ephemeral + sequential znodes, one-shot watches; **reads are local and stale unless you `sync()`** |
| etcd | Raft + MVCC revisions + leases + txn CAS; 2 GB quota, 1.5 MB requests; read-only on NOSPACE |
| Coordination service hygiene | Small critical metadata only; not a queue; data plane survives on cached decisions |
| Distributed locks | Efficiency vs correctness; a lease can expire mid-operation; only the resource can enforce exclusion |
| Fencing tokens | Monotonic number carried to the resource, which rejects anything older. The correct answer |
| Redlock | Adds availability, not safety; the gap is fencing; decide by whether correctness depends on the lock |
| The ladder | Don't need it → idempotent → OCC → partitioned queue → lease + fence → lock service → own consensus |
| .NET locking | `DistributedLock` + `HandleLostToken`; blob leases renewed at 1/3; `sp_getapplock`; `SKIP LOCKED` |
| Optimistic concurrency | `[Timestamp]`/`IsRowVersion`, `IfMatchEtag`; a concurrency exception is a domain event, not a transient fault |
| Exactly-once | Impossible on the wire; at-least-once + idempotency key on a unique index; Kafka's guarantee is internal only |
| Singleton background job | Prefer claimable + idempotent work; else lease + epoch + cancel-on-renewal-failure + progress liveness signal |
| Logical time | Lamport orders, vector clocks detect concurrency, HLC does both compactly, TrueTime buys external consistency with a wait |
| CRDTs | Commutative/associative/idempotent merge; no coordination; no global invariants; watch tombstone growth |
| Coordination avoidance | CALM: monotonic ⟺ coordination-free; invariant confluence classifies your constraints |
| Global limits | Escrow: convert a global invariant into local invariants whose conjunction implies it; overselling impossible, understocking bounded |
| Architecture | Consensus in the control plane, replication in the data plane, idempotency everywhere |
| Operating it | fsync p99 and leader-changes-per-hour as SLIs; compaction and defrag; rehearsed quorum-loss recovery |

---

## Progress

Module 9 complete. You now have the full Module 7–9 arc: **guarantees** (consistency models), **mechanisms** (replication and partitioning), and **agreement** (consensus and coordination). Together they cover the theory portion of almost any distributed-systems deep dive.

Next in the curriculum: **Module 10 — Caching strategy** (cache-aside, write-through/write-back, CDNs, invalidation, Redis patterns, stampede protection). It connects directly to this module in two ways: a cache is a replica with a consistency model (Module 7), and cache invalidation is the coordination problem you meet most often in day-to-day .NET work — with stampede protection being a distributed lock question in disguise, which you can now answer properly.
