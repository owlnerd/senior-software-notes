# Module 8 — Replication & Partitioning
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Module 7 described **guarantees**. This module supplies the **mechanisms** that produce them. Every consistency model you can offer is a consequence of two structural decisions:

- **Replication** — the *same* data on more than one node. Buys availability, read throughput, and locality. Costs you divergence, lag, and failover risk.
- **Partitioning (sharding)** — *different* data on different nodes. Buys write throughput and data volume. Costs you routing, rebalancing, cross-partition queries, and the loss of cheap transactions.

They are orthogonal and almost always combined: a partitioned system replicates each partition, so the unit of failover is a *replica set per shard* (Cosmos physical partition, Kafka partition with leader + followers, MongoDB shard, Cassandra token range with RF=3).

Three framings to carry through:

1. **Replication decisions are reversible; partitioning decisions are not.** You can add and remove replicas on a Tuesday afternoon. Changing a partition key is a data migration with a cutover plan. Say this out loud in interviews.
2. **Every partitioned system has a routing problem and a rebalancing problem.** Candidates who don't raise them have read about sharding, not run it.
3. **Hot partitions, not total load, are what actually kill partitioned systems.** The arithmetic of "10 nodes, 10× throughput" assumes uniformity. Real keys are Zipfian.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Replication vs partitioning | Same data in many places vs different data in different places. Different problems. |
| 2 | Single-leader replication | All writes through one node, ordered by it. 95% of production systems. |
| 3 | What flows over the wire | Statement / WAL / logical / trigger-based. Logical is what enables CDC and outbox tails. |
| 4 | Sync vs async vs semi-sync | You are choosing your RPO. Async = fast writes and a guaranteed data-loss window. |
| 5 | Replication lag | Lag is a user-visible variable; measure it in bytes *and* seconds. |
| 6 | Failover | Split brain, fencing, and lost writes. Auto-failover + async replication = silent data loss. |
| 7 | Bootstrapping a replica | Snapshot + log position. The same primitive that makes shard migration possible. |
| 8 | Multi-leader | Local-latency writes, and conflicts become your domain problem. |
| 9 | Leaderless (Dynamo) | Quorums plus read repair plus anti-entropy. Sloppy quorums void the guarantee. |
| 10 | Detecting concurrency | Version vectors and siblings; LWW is silent data loss. |
| 11 | Other topologies | Chain replication, consensus-based SMR, and shared-storage (Aurora/Hyperscale). |
| 12 | Read replicas in .NET | Routing, lag windows, and why replicas never help writes. |
| 13 | Why partition | Write throughput, volume, blast radius, and tenancy isolation. |
| 14 | Range partitioning | Range scans for free, sequential hot spots for free too. |
| 15 | Hash partitioning | Uniformity at the cost of range queries. |
| 16 | Compound keys | Hash the partition key, sort within it. The practical default. |
| 17 | Consistent hashing | Ring + virtual nodes; O(1/N) movement instead of O(1). |
| 18 | Rebalancing | Fixed partition count, dynamic splits, or per-node tokens. Pick one deliberately. |
| 19 | Request routing | Coordinator, routing tier, or partition-aware client. All need a topology source of truth. |
| 20 | Choosing a partition key | Cardinality, uniformity, query alignment, transaction boundary. The one-way door. |
| 21 | Secondary indexes | Local (scatter-gather reads) vs global (write amplification). |
| 22 | Cross-partition operations | Fan-out multiplies tail latency; uniqueness needs a single owner. |
| 23 | Resharding | Logical shards, dual writes, backfill, shadow reads, cutover. |
| 24 | Replication × partitioning | Shard = replica set; place replicas across failure domains; cells and shuffle sharding. |
| 25 | The .NET/Azure surface | Cosmos, Azure SQL, Postgres/Citus, Kafka, Redis Cluster, Orleans, EF Core's gap. |
| 26 | Operating a sharded fleet | Migrations, backups, per-partition telemetry, hot-partition detection. |

---

# Part A — Replication

## Concept 1 — The two axes, and why they're always combined

Draw this once and you'll never confuse them again:

```
REPLICATION (same data, N copies)        PARTITIONING (different data, N subsets)
┌──────┐ ┌──────┐ ┌──────┐               ┌──────────┐ ┌──────────┐ ┌──────────┐
│ A B C│ │ A B C│ │ A B C│               │ A..H     │ │ I..P     │ │ Q..Z     │
└──────┘ └──────┘ └──────┘               └──────────┘ └──────────┘ └──────────┘
 reads ×3, survives loss                  writes ×3, volume ×3
 writes ×1 (still one logical stream)      one copy — a node loss loses data

BOTH (the real world): every partition is a replica set
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ shard 1      │ │ shard 2      │ │ shard 3      │
│ L  F  F      │ │ L  F  F      │ │ L  F  F      │   L=leader F=follower
│ az1 az2 az3  │ │ az2 az3 az1  │ │ az3 az1 az2  │   leaders spread across AZs
└──────────────┘ └──────────────┘ └──────────────┘
```

**Replication buys** availability (survive node/AZ/region loss), read throughput (fan reads across copies), locality (serve from near the user), and operational freedom (take a replica out to patch, back up, or run analytics). It does **not** buy write throughput — every write still goes through one logical ordering point per object.

**Partitioning buys** write throughput, storage volume beyond one machine, smaller blast radius per failure, and per-tenant isolation. It does **not** buy availability by itself — an unreplicated shard is a single point of data loss.

The senior instinct from Module 6 applies: scale reads with replicas and cache, and only partition when write throughput or data volume genuinely exceeds the largest single node with headroom.

---

## Concept 2 — Single-leader replication

One replica is designated leader (primary, master, write region). All writes go to it; it appends them to a replication stream; followers apply the stream in order. Reads may be served by the leader or by followers.

This is the architecture of SQL Server Always On, Azure SQL, PostgreSQL streaming replication, MySQL, MongoDB replica sets, Kafka per partition, Cosmos DB within a write region, and etcd/ZooKeeper (with consensus-elected leaders). It dominates because it's the cheapest way to get a total order on writes per object, which is what makes uniqueness constraints, atomic increments, and compare-and-set possible.

**Why the leader matters conceptually.** The leader *is* the serialization point — Module 6's Amdahl serial fraction, in database clothing. Its consequences:

- Write throughput per object is bounded by one node (plus the cost of acknowledging replicas).
- Followers apply the log in the same order, which gives you **consistent prefix** reads for free (Module 7, Concept 14) — per partition.
- Failure of the leader is a *control-plane* event: someone must choose a new one, and that choice is where correctness is usually lost.

**Interview framing.** "Single-leader" should be your default proposal, with the follow-up stated pre-emptively: *"one leader per partition, so write scaling comes from more partitions, not from more leaders."* That sentence links Part A to Part B and is the single most useful line in the module.

---

## Concept 3 — What actually flows over the wire

Interviewers probe this because it distinguishes people who've configured replication from people who've drawn it.

| Format | How it works | Problems | Where you see it |
|---|---|---|---|
| **Statement-based** | Ship the SQL text; each follower re-executes it | Nondeterminism: `NOW()`, `RAND()`, `NEWID()`, triggers, auto-increment races. Divergence without errors. | MySQL's legacy `STATEMENT` binlog format |
| **Physical / WAL shipping** | Ship the write-ahead log: byte-level changes to pages | Tightly coupled to storage format and version — can't replicate across major versions or engines; no zero-downtime upgrade path | PostgreSQL streaming replication, SQL Server AG log blocks |
| **Logical (row-based)** | Ship logical row changes: "row with PK 7 in table Orders changed these columns" | Larger volume; needs a primary key / replica identity; DDL handling varies | MySQL `ROW` binlog, PostgreSQL logical replication / `pgoutput`, SQL Server CDC |
| **Trigger-based** | Application-level triggers write changes into a table that something tails | Slow, invasive, error-prone — but flexible and works anywhere | Legacy heterogeneous replication |

**Why logical replication matters for architecture, not just ops.** Logical change streams are the substrate for:

- **CDC pipelines** — Debezium on top of Postgres logical decoding or SQL Server CDC feeding Kafka; Cosmos DB's **change feed**; DynamoDB Streams.
- **The outbox alternative.** Module 7's dual-write problem has two fixes: transactional outbox (application-managed) or CDC (infrastructure-managed). CDC needs no application change but couples your event contract to your table schema. Knowing that trade is an architect-level answer.
- **Online migrations.** Logical replication between different schemas/versions is how you do zero-downtime major-version upgrades and how you backfill a new shard while the old one still serves traffic (Concept 23).
- **Read models.** CQRS projections (Module 23) are often just a logical replication consumer.

Mention the operational hazard: a **replication slot** in Postgres retains WAL until consumed; an abandoned slot (a CDC consumer that died) will fill the primary's disk and take the database down. That detail is a credibility marker.

---

## Concept 4 — Synchronous, asynchronous, semi-synchronous: you are choosing your RPO

The only question that matters: **does the leader acknowledge the client before or after replicas have the write?**

| Mode | Commit latency | Data loss on leader failure (RPO) | Availability effect |
|---|---|---|---|
| **Asynchronous** | Local commit only (~sub-ms to a few ms) | Everything not yet shipped — typically the lag window (ms to minutes) | Replica failure doesn't affect writes |
| **Synchronous (all)** | Slowest replica's RTT, every write | Zero, for acknowledged writes | Any replica down or slow blocks all writes |
| **Semi-sync / quorum** | Fastest *k* of *n* replicas | Zero if the surviving set includes an acknowledged replica | Survives *n − k* replica failures |

The middle row is why "synchronous replication to all replicas" is almost never the right configuration: you have multiplied the probability that *some* participant is slow, and every write now pays for it. The right shape is **synchronous to a quorum, asynchronous beyond it** — the basis of Raft/Paxos commit (Module 9).

**The numbers decide everything** (Module 7, Concept 8): within a region, cross-AZ quorum commit costs ~1–2 ms and is effectively free. Cross-region synchronous commit costs 70–150 ms and is a product decision, not a config flag.

**Concrete configuration you should be able to quote:**

- **PostgreSQL**: `synchronous_commit` takes `off`, `local`, `remote_write`, `on`, `remote_apply`. `on` means the standby has flushed WAL to disk; `remote_apply` means it has *applied* it — which is the only setting that gives you read-your-writes on that standby, at the cost of waiting for replay. `synchronous_standby_names = 'ANY 1 (s1, s2, s3)'` gives you quorum commit instead of naming one standby (which would otherwise be a new single point of failure).
- **SQL Server Always On**: `SYNCHRONOUS_COMMIT` vs `ASYNCHRONOUS_COMMIT` per replica. Up to eight secondaries, of which only a small number (three including the primary, in recent versions) may be synchronous-commit. **Automatic failover requires synchronous-commit** plus a healthy cluster quorum — this pairing is not optional and is the answer to "can we auto-fail-over to the DR region?" (No, if that region is async.)
- **Azure SQL**: Business Critical is a four-replica AG with synchronous commit inside the region; **active geo-replication** and failover groups are **asynchronous**, so cross-region failover has a real RPO (Microsoft publishes ~5 s as the expected data-loss bound for geo-replication). Hyperscale separates compute from a log service and page servers, which changes the replication story entirely (Concept 11).
- **Cosmos DB**: four replicas per physical partition with local quorum commit, plus cross-region replication whose synchrony is a function of the consistency level (Module 7, Concept 20).

**The interview line:** *"Synchronous within the region because it costs a millisecond and buys RPO 0; asynchronous across regions because it costs 80 ms; and because cross-region is async, failover there is a business decision with a documented data-loss window, not an automatic action."*

---

## Concept 5 — Replication lag, and how to measure it

Lag is not a failure; it's a variable. Its anomalies are exactly the session-guarantee violations from Module 7, Concept 13: read-your-writes, monotonic reads, and writes-follow-reads. The mechanism module's job is to tell you how to *see* it.

Measure lag in **three units**, because they answer different questions:

1. **Bytes/LSN distance** — how far behind, independent of traffic. The right alerting signal (it doesn't go quiet when traffic does).
2. **Time** — how stale a read is, which is what users experience.
3. **Apply vs receive lag** — a replica may have received WAL but not applied it (blocked by a long read, a lock, or single-threaded replay). Receive lag is a network problem; apply lag is a replica-capacity problem, and they have different fixes.

```sql
-- PostgreSQL, on the primary: per-standby state and lag
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- On a standby: how stale am I, in seconds?
SELECT now() - pg_last_xact_replay_timestamp() AS apply_delay;
```

```sql
-- SQL Server: AG health, redo queue, and estimated recovery time
SELECT ar.replica_server_name, drs.synchronization_state_desc,
       drs.log_send_queue_size, drs.redo_queue_size, drs.secondary_lag_seconds
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON ar.replica_id = drs.replica_id;
```

Common causes of lag spikes worth naming: a long-running analytical query on the replica blocking replay; single-threaded apply (Postgres replays WAL with one process; MySQL needs parallel appliers configured); a bulk write on the primary (an index rebuild, a large `DELETE`); network saturation; and the replica being a smaller SKU than the primary — a classic cost optimization that turns into an availability problem, because a replica that can't keep up is a replica you can't fail over to.

**Design consequence:** treat the lag metric as an SLI with a bound (Module 7's bounded staleness), alert on it, and make the routing layer lag-aware — if a replica exceeds the bound, remove it from the read pool rather than serving arbitrarily stale data.

---

## Concept 6 — Failover: the part that actually loses data

Everything about single-leader replication is simple until the leader dies. Failover has four steps and each one has a canonical failure.

1. **Detect.** Usually a timeout. Too aggressive → unnecessary failovers during a GC pause or network blip (each of which costs availability). Too lax → long outages. There is no correct value, only a trade you should state.
2. **Elect.** Choose the most up-to-date replica. This must be a **majority decision** or you get split brain. This is why production Postgres HA uses Patroni + etcd/Consul, and why SQL Server AGs require a Windows Server Failover Cluster quorum (often with a **witness**/file-share to break ties in a two-node deployment).
3. **Reconfigure.** Everything pointing at the old leader must be redirected: DNS/listener, connection strings, proxies (PgBouncer/HAProxy), and — the one people forget — application connection pools holding open sockets to the dead primary. .NET connection resiliency matters here: `SqlClient` and Npgsql need retry policies, and EF Core's `EnableRetryOnFailure` execution strategy exists precisely for the reconnect window.
4. **Fence the old leader.** If the old primary comes back believing it's still primary and clients still reach it, you have **two leaders accepting writes**. Fencing (STONITH, revoking storage leases, or fencing tokens — Module 9) is the only cure.

**The failure modes to name in an interview:**

- **Lost writes.** With async replication, the promoted replica is missing the tail of the log. Those writes are gone — and worse, if the old leader is later reattached without being rebuilt, its extra writes conflict with new ones. GitHub's well-documented 2018 incident is the canonical public example of what cross-region failover with a replication lag window costs.
- **Split brain.** Two leaders, divergent histories, manual reconciliation. Prevented only by quorum-based election plus fencing.
- **Auto-failover to an async replica is a decision to accept silent data loss.** For a ledger, make it manual with a runbook (Module 7's worked example). For a session store, automate it.
- **Downstream chaos.** Auto-increment IDs reused, caches holding values that no longer exist, message consumers replaying offsets that were rolled back. "What breaks downstream after failover?" is a great question to raise yourself.

---

## Concept 7 — Bootstrapping a replica (the primitive behind everything)

Adding a follower without downtime is the same pattern as migrating a shard, building a read model, or seeding a CDC pipeline — learn it once:

1. Take a **consistent snapshot** of the leader at a known log position (`pg_basebackup`, a Cosmos/SQL snapshot, a filesystem/volume snapshot).
2. Copy the snapshot to the new node.
3. Have the new node **request the log from that exact position** (Postgres: the LSN embedded in the backup; MySQL: the binlog coordinates or GTID).
4. **Catch up** — apply the backlog until lag is small.
5. **Join** the read pool / become eligible for failover.

The generalization is the migration playbook in Concept 23: *snapshot, copy, tail the log, verify, cut over.* If you can articulate steps 1–5 in the abstract, you can answer almost any "how do you migrate this live?" question.

Operational notes worth knowing: a replica must be able to fetch the WAL it missed, so either keep enough on the primary (`wal_keep_size`, a replication slot) or fetch from archive/blob storage; a slot guarantees retention but risks filling the disk. **Cascading replication** (a replica feeding other replicas) exists because fanning out WAL to 15 replicas costs the primary network bandwidth — one reason platforms cap replica counts.

---

## Concept 8 — Multi-leader replication

Multiple nodes accept writes for the same data and replicate to each other. Three legitimate use cases:

- **Multi-region active-active writes** — local write latency everywhere.
- **Offline-capable clients** — a mobile app or desktop client is a replica that accepts writes while disconnected (calendars, note-taking apps). This is multi-leader whether you call it that or not.
- **Real-time collaborative editing** — every client is a leader on a local copy; convergence via CRDTs or OT.

**The cost is conflicts**, and they are now a domain problem, not a database problem. Module 7, Concept 15 listed the resolution strategies (LWW, siblings + merge, CRDTs, application merge, structural avoidance). What Module 8 adds is the *mechanics*:

- **Topologies.** All-to-all (N² links, robust, but messages can arrive out of causal order — a row update arriving before the insert that created it, which is why dependency tracking or per-link ordering is required); circular and star (fewer links, but one node failing breaks the chain, and every hop adds lag).
- **Auto-increment and uniqueness are broken** across leaders. You need UUIDv7/ULID/Snowflake IDs (see Module 7's "turn a coordination problem into a naming problem"), or per-leader ID ranges.
- **Write skew across leaders can't be prevented.** Two leaders each independently approve the last seat. Only single ownership prevents it.
- **Cosmos DB multi-region writes** is the managed form: last-write-wins by a configurable timestamp path by default, or a **custom conflict-resolution stored procedure**, with an accessible conflict feed for anything unresolved. Knowing that the conflict feed exists — and that someone must monitor it — is the kind of detail that reads as experience.
- **SQL Server merge/peer-to-peer replication** exists and is generally a warning sign in a design review; teams adopt it to avoid re-architecting and inherit conflict reconciliation forever.

**The senior position:** prefer **structural conflict avoidance** — partition ownership by home region so each record has exactly one writer (Module 7, Concept 8's fourth pattern) — and reach for true multi-leader only where the domain is inherently conflict-tolerant (union-able carts, CRDT-shaped counters, collaborative documents).

---

## Concept 9 — Leaderless replication (Dynamo-style)

No leader. The client (or a coordinator node acting on its behalf) writes to *W* of *N* replicas and reads from *R* of *N*, and the system reconciles divergence in the background. Cassandra, Riak, and DynamoDB's internals are the canon; the Dynamo paper (2007) is the source text.

The four mechanisms you must be able to name:

1. **Quorum reads and writes.** `W + R > N` overlaps the read and write sets. Module 7, Concept 19 covers why this is necessary but not sufficient for linearizability.
2. **Read repair.** A read that sees divergent versions writes the winning value back to the stale replicas. This is *foreground* repair — it only fixes keys that are actually read. Rarely-read keys drift indefinitely.
3. **Anti-entropy.** Background comparison of replica contents, usually with **Merkle trees**: each replica builds a hash tree over its key ranges, two replicas exchange root hashes, and they descend only into subtrees that differ — so divergence is found in O(log n) transfers instead of a full scan. (Cassandra's `nodetool repair`; Dynamo's original design.) Repairs are expensive and are a major source of operational toil at scale — Discord's Cassandra war story is largely about this.
4. **Hinted handoff + sloppy quorums.** When a home replica is unreachable, the write is accepted by another node, which holds a "hint" and forwards it once the home node returns. This preserves write availability and **breaks the quorum overlap guarantee** — the *W* nodes that acknowledged may not be among the *N* home nodes, so `W + R > N` no longer implies you'll read the write. Tunable consistency is therefore "strong-ish until failure, which is precisely when you needed it."

**When leaderless is genuinely right:** high write availability across regions with conflict-tolerant data (time series, telemetry, feeds, message history), very high write volume, and a team that can operate repairs. **When it isn't:** anything with a global invariant, or a team that will be surprised by "eventually."

---

## Concept 10 — Detecting concurrent writes: version vectors and siblings

Once writes can be accepted in multiple places, you need to distinguish "newer" from "concurrent". Wall clocks cannot do this (skew, and no way to represent "concurrent").

**Per-key version numbers** work for a single leader: the leader increments a counter, clients send back the version they read, and a write with a stale version is either rejected (optimistic concurrency — EF Core's `rowversion`) or recorded as a sibling.

**Version vectors** (one counter per replica, per key) work for multiple replicas. Compare two vectors elementwise:

```
V1 = {A:2, B:1}   V2 = {A:2, B:3}   →  V1 < V2 (V2 dominates; it's strictly newer)
V1 = {A:3, B:1}   V2 = {A:2, B:3}   →  neither dominates → CONCURRENT → conflict
```

```csharp
// Version vector: merge is pointwise max; comparison detects concurrency.
public sealed record VersionVector(ImmutableDictionary<string, long> Counters)
{
    public VersionVector Increment(string replicaId) =>
        new(Counters.SetItem(replicaId, Counters.GetValueOrDefault(replicaId) + 1));

    public VersionVector Merge(VersionVector other) =>
        new(Counters.Keys.Union(other.Counters.Keys).ToImmutableDictionary(
            k => k,
            k => Math.Max(Counters.GetValueOrDefault(k), other.Counters.GetValueOrDefault(k))));

    public bool DominatesOrEquals(VersionVector other) =>
        other.Counters.All(kv => Counters.GetValueOrDefault(kv.Key) >= kv.Value);

    public bool IsConcurrentWith(VersionVector other) =>
        !DominatesOrEquals(other) && !other.DominatesOrEquals(this);
}
```

Terminology precision (a cheap signal): a **vector clock** tracks events across processes; a **version vector** tracks versions of a replicated *object*. Interviewers who know the difference notice.

**Siblings.** When two versions are concurrent, a Dynamo-style store can keep both and hand them to the application (Riak's siblings; Dynamo's shopping-cart union). This is honest but pushes real work into your domain layer.

**Tombstones.** Deletes must be replicated as *markers*, not as absence — otherwise a replica that never saw the delete resurrects the row during repair. Tombstones then need a grace period (Cassandra's `gc_grace_seconds`, default 10 days) before collection, and accumulating tombstones are a classic Cassandra performance pathology. "How do you delete data in an eventually consistent store?" is an excellent deep-dive question, and "tombstones, plus a GC window that must exceed your repair interval" is the answer.

CRDTs — types whose merge function makes conflicts impossible by construction — are Module 9.

---

## Concept 11 — Other replication architectures worth naming

- **Chain replication** (van Renesse & Schneider, 2004): replicas form a chain; writes enter at the head and propagate to the tail; reads are served by the tail. Strong consistency with simple recovery, and it separates read and write load cleanly. It shows up in object stores and in systems like Azure Storage's internals; naming it signals reading beyond the blog-post canon.
- **Consensus-based state machine replication**: the log itself is agreed by Raft/Paxos, so leader election and log commit are the *same* mechanism. This is etcd, ZooKeeper, CockroachDB ranges, Kafka's KRaft controller. Module 9.
- **Shared-storage / log-structured disaggregation**: the compute node doesn't replicate to other compute nodes at all — the *storage layer* replicates. Azure SQL **Hyperscale** (log service + page servers), AWS Aurora, Neon. Consequences: adding a read replica is nearly free and instant (no data copy), failover is fast, and storage durability is someone else's problem. In an architect round, "the platform already replicates at the storage layer, so my replication decision is only about read scale and RTO" is a strong, current answer.
- **Quorum-replicated storage under a single-writer database** (Cosmos physical partitions, most managed engines) — the case where you configure *intent* (consistency level) rather than topology.

---

## Concept 12 — Read replicas in .NET practice

Once you route reads to replicas, you've chosen eventual consistency for those reads (Module 7, Concept 21c). What Module 8 adds is the plumbing.

```csharp
// Npgsql multi-host: one cluster, two logical connections with different targets.
// Primary:
"Host=pg-1,pg-2,pg-3;Database=orders;Target Session Attributes=primary"
// Replicas (round-robins across standbys, honours Load Balance Hosts):
"Host=pg-1,pg-2,pg-3;Database=orders;Target Session Attributes=prefer-standby;Load Balance Hosts=true"
```

```csharp
// Azure SQL read scale-out is a connection-string flag:
"Server=...;Database=orders;ApplicationIntent=ReadOnly"
```

A workable EF Core pattern is two registered contexts over the same model, with the read one tracking-disabled, plus an explicit escape hatch for reads that must be fresh:

```csharp
builder.Services.AddDbContext<OrdersWriteContext>(o => o
    .UseNpgsql(cfg.GetConnectionString("primary"), n => n.EnableRetryOnFailure()));

builder.Services.AddDbContext<OrdersReadContext>(o => o
    .UseNpgsql(cfg.GetConnectionString("replica"), n => n.EnableRetryOnFailure())
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));

// Read path decides per operation, not per repository:
var order = mustBeFresh || await readRouter.MustUsePrimaryAsync(userId)
    ? await write.Orders.AsNoTracking().FirstAsync(o => o.Id == id)
    : await read.Orders.FirstAsync(o => o.Id == id);
```

Four things to say out loud when proposing read replicas:

1. **They don't help writes.** If the bottleneck is write throughput, replicas make it slightly worse (shipping the log costs the primary bandwidth and, if synchronous, latency).
2. **They shift the consistency problem to the routing layer.** Default to replicas, pin post-write reads to the primary for a short window, or use an LSN/session-token wait (Module 7, Concept 13).
3. **Replica count has limits.** Fan-out costs the primary; platforms cap it (Azure SQL Business Critical gives you a small fixed set; Hyperscale allows more named replicas because of storage disaggregation). Past that you need cascading replication, a cache tier, or a different read model.
4. **A replica sized smaller than the primary is not a failover target.** Cheap replicas are an availability liability.

---

# Part B — Partitioning

## Concept 13 — Why partition, and the vocabulary

You partition when one of four things exceeds one machine (with headroom):

- **Write throughput** — the only scaling problem replicas can't solve.
- **Data volume** — beyond what one node can store, back up, restore, index, or vacuum in an acceptable window. Restore time is the underrated driver: a 20 TB database with a 12-hour restore is an RTO problem even when performance is fine.
- **Blast radius** — one bad tenant, one bad query, one bad migration should not take down everyone.
- **Isolation / residency** — per-tenant performance isolation, or EU data staying in the EU.

Vocabulary equivalences (interviewers switch between them freely): *shard* (MySQL/Postgres/MongoDB, Vitess), *partition* (Cosmos, Kafka, Azure), *region*/*tablet* (Bigtable/HBase/Spanner), *vnode*/*token range* (Cassandra), *hash slot* (Redis Cluster), *vBucket* (Couchbase). And a critical disambiguation for .NET/SQL interviews: **SQL Server / PostgreSQL "table partitioning" is single-node partitioning** — splitting one table across filegroups or child tables for manageability and partition elimination. Useful (time-series retention via partition switch/detach), but it is *not* sharding. Say "horizontal partitioning across nodes" when you mean sharding, and know the difference cold.

---

## Concept 14 — Partitioning by key range

Assign contiguous key ranges to partitions (A–F, G–M, …), keeping keys sorted within each. Used by Bigtable/HBase, Spanner, CockroachDB, MongoDB ranged sharding, and any B-tree-ordered store.

**Wins:** efficient range scans (`WHERE date BETWEEN …`, "last 50 messages"), and boundaries can be chosen to match real data distribution, then **split and merge** dynamically as data grows.

**Loses:** sequential keys create hot spots. Partition by timestamp and all of today's writes land on one partition while the rest of the fleet idles — the single most common partitioning mistake, and one interviewers deliberately fish for. Auto-increment IDs have the same shape.

**Mitigations:** prefix the key with a higher-cardinality component (device ID, tenant ID) so time-ordering is *within* a partition rather than across the keyspace; or salt with a bucket number. Note the cost: you've traded a global time-range scan for a scatter-gather across buckets.

---

## Concept 15 — Partitioning by hash of key

Apply a hash function to the key and use the hash to assign a partition. Uniform distribution, sequential keys stop mattering, and no manual boundary tuning.

**The cost is ordering**: range queries over the partition key require touching every partition. You've optimized writes and point reads and given up scans.

Two rules:

1. **Use a stable hash.** `string.GetHashCode()` is randomized per process in .NET Core and later (Module 6, Concept 8g). Any routing decision must use `XxHash64`/`XxHash3` from `System.IO.Hashing`, MurmurHash, or a crypto hash — something identical across processes and across restarts.
2. **`hash(key) % N` is a trap.** Changing *N* remaps almost everything (Module 6, Concept 12.7). Use consistent hashing, a fixed logical partition count, or a directory — never naked modulo over the physical node count.

---

## Concept 16 — Compound keys: hash the partition, sort within it

The practical resolution of the range-vs-hash tension, and the model most modern stores actually use: a **partition key** (hashed, determines placement) plus a **sort/clustering key** (ordered within the partition).

- **Cassandra**: `PRIMARY KEY ((channel_id, bucket), message_id)` — the parenthesized part is the partition key; `message_id` sorts within it. This is exactly Discord's message schema, and the reason "give me the last 50 messages in this channel" is one sequential read.
- **DynamoDB**: partition key + sort key; queries within a partition can use range conditions on the sort key.
- **Cosmos DB**: partition key plus the item id, with hierarchical partition keys (up to three levels) adding sub-partitioning.
- **Kafka**: the message key determines the partition; offset order within it.

The design question becomes: *what is the largest set of rows I will need to read together, and can it be one partition?* If yes, you have a fast, cheap, transactionally-scoped access path. If no, you've signed up for scatter-gather. Module 7's corollary reappears here: **put data that must be read or written atomically in the same partition**, because a single-partition transaction is cheap and a cross-partition one needs consensus or 2PC.

---

## Concept 17 — Consistent hashing, in depth

Module 6 introduced consistent hashing for load balancing. Here it is as a *data placement* mechanism.

**The construction.** Hash the key space onto a ring (conceptually `[0, 2⁶⁴)`). Hash each node's identifier onto the same ring. A key belongs to the first node encountered walking clockwise from the key's position.

```
                          0 / 2⁶⁴
              N3-v7 ●────────┴────────● N1-v2
                   ╱                   ╲
        N2-v5 ●                            ● N2-v9
              │        hash ring           │   key "acct-4711" hashes here ──┐
        N1-v8 ●                            ● N3-v1  ← owner: first node       │
                   ╲                   ╱            clockwise from the key ◄──┘
              N2-v3 ●────────┬────────● N1-v4
                            2⁶³
```

**Why it matters.** Adding or removing a node moves only the keys in its immediate arc — about 1/N of the keyspace — instead of the ~(N−1)/N that modulo hashing would move. For a cache tier that's the difference between a 10% miss bump and a total miss storm; for a data tier it's the difference between moving 1 TB and moving 20 TB.

**Virtual nodes are mandatory.** With one ring position per node, arc sizes vary wildly (random points on a circle are not evenly spaced), so load is unbalanced and removing a node dumps its entire arc onto exactly one neighbour. Give each physical node *V* positions and the load's relative standard deviation falls as roughly 1/√V — so V ≈ 100 gives about 10% spread, V ≈ 256 about 6%. Vnodes also let you weight heterogeneous hardware (more positions = more data) and spread a departing node's load across many survivors.

```csharp
using System.IO.Hashing;
using System.Text;

public sealed class ConsistentHashRing<TNode> where TNode : notnull
{
    private readonly SortedList<ulong, TNode> _ring = new();
    private readonly Func<TNode, string> _id;
    private readonly int _virtualNodes;

    public ConsistentHashRing(Func<TNode, string> id, int virtualNodes = 128)
        => (_id, _virtualNodes) = (id, virtualNodes);

    private static ulong Hash(string s) => XxHash64.HashToUInt64(Encoding.UTF8.GetBytes(s));

    public void Add(TNode node)
    {
        for (int v = 0; v < _virtualNodes; v++)
            _ring[Hash($"{_id(node)}#{v}")] = node;   // stable across processes
    }

    public void Remove(TNode node)
    {
        for (int v = 0; v < _virtualNodes; v++)
            _ring.Remove(Hash($"{_id(node)}#{v}"));
    }

    /// First node clockwise from the key's position.
    public TNode GetOwner(string key)
    {
        if (_ring.Count == 0) throw new InvalidOperationException("Empty ring.");
        var h = Hash(key);
        var keys = _ring.Keys;                       // sorted; binary search the successor
        int lo = 0, hi = keys.Count - 1, idx = 0;
        while (lo <= hi)
        {
            int mid = (lo + hi) / 2;
            if (keys[mid] >= h) { idx = mid; hi = mid - 1; }
            else { idx = (lo = mid + 1) < keys.Count ? lo : 0; }
        }
        return _ring.Values[idx];
    }

    /// The N distinct physical nodes that should hold replicas of this key.
    public IReadOnlyList<TNode> GetPreferenceList(string key, int replicas)
    {
        var result = new List<TNode>(replicas);
        var h = Hash(key);
        var keys = _ring.Keys;
        int start = 0;
        while (start < keys.Count && keys[start] < h) start++;
        for (int i = 0; i < keys.Count && result.Count < replicas; i++)
        {
            var node = _ring.Values[(start + i) % keys.Count];
            if (!result.Contains(node)) result.Add(node);   // skip vnodes of the same host
        }
        return result;
    }
}
```

`GetPreferenceList` is the detail that separates a textbook answer from a working one: in a replicated ring, the *next N distinct physical nodes* clockwise hold the replicas (Dynamo's "preference list"), and you must skip additional vnodes belonging to a host you already picked — and, ideally, skip hosts in a failure domain you already used (Concept 24).

**The alternatives, and when each wins:**

| Scheme | Lookup cost | Movement on change | Notes |
|---|---|---|---|
| Modulo | O(1) | ~all keys | Never, for stateful data |
| Consistent hashing + vnodes | O(log(N·V)) | ~1/N | The default; weighting is easy |
| **Rendezvous (HRW)** | O(N) | minimal | Simplest correct implementation; fine for tens–hundreds of nodes; no ring state to maintain |
| **Jump consistent hash** (Lamping & Veach) | O(log N), no memory | minimal | Buckets are `0..N-1` only — can't remove an arbitrary node, so it fits "N logical shards" not "these named hosts" |
| **Maglev** | O(1) via lookup table | small, bounded | Very even; table build cost on change |
| **Bounded-load consistent hashing** | O(log n) amortized | ~1/N | Caps any node at a multiple of average load and forwards overflow — the fix for hot keys |

**The limitation to state before you're asked:** consistent hashing balances *keys*, not *load*. One celebrity key, one whale tenant, one viral channel still lands on one node. Bounded loads, key splitting, or a directory-based override for known-large keys are the mitigations (Concept 20).

---

## Concept 18 — Rebalancing strategies

When you add or remove nodes, data must move. Three strategies, and choosing one consciously is the signal.

**1. A fixed, large number of logical partitions.** Create (say) 1,024 partitions up front and assign many to each node. Adding a node moves whole partitions, never splits them; the key→partition mapping *never changes*. This is Kafka (partition count per topic), Elasticsearch (primary shards), Redis Cluster (16,384 hash slots), Couchbase (1,024 vBuckets), and the "logical shards" trick used by Notion and Figma over Postgres.
- **Pro:** operationally simple; rebalancing is a scheduling problem, not a data-model problem.
- **Con:** you must guess the count. Too few caps your maximum node count; too many means per-partition overhead (file handles, memory, metadata, per-partition query fan-out). Kafka's asymmetry is instructive: you can *add* partitions, but doing so changes `hash(key) % partitions` and therefore breaks key→partition affinity and per-key ordering for existing keys. Over-provision partitions early.

**2. Dynamic partitioning (split and merge).** Partitions split when they exceed a size/throughput threshold and merge when they shrink: HBase, Bigtable, MongoDB ranged shards, and **Cosmos DB physical partitions**. Adapts to data volume automatically and suits range partitioning. Con: split operations are moments of elevated latency and operational risk, and an empty database starts with one partition (hence "pre-splitting" for bulk loads).

**3. Partitioning proportional to nodes.** Each node claims a fixed number of random token ranges (Cassandra's `num_tokens` — 256 in the 3.x era, 16 by default in 4.x precisely because high vnode counts hurt repair and availability). Partition *count* grows with the cluster; sizes stay roughly stable.

**Automatic vs manual rebalancing.** Fully automatic rebalancing plus automatic failure detection is a dangerous combination: a node is declared slow, its data starts moving, the movement consumes network and I/O, more nodes look slow, and you've built a cascading-failure generator. The mature configuration is **automatic planning with human approval**, or at least strict rate limiting and concurrency caps on migrations. Saying this unprompted marks operational experience.

---

## Concept 19 — Request routing

Someone must answer "which node holds key K?" Three architectures:

1. **Any node routes** — the client hits any node; if that node doesn't own the key, it forwards (or redirects). Cassandra (coordinator node + gossip), Redis Cluster (`MOVED`/`ASK` redirects). Simple clients, extra hop.
2. **A routing tier** — a proxy that owns the topology: MongoDB's `mongos`, Vitess's `vtgate`, Figma's DBProxy, a Redis proxy, or YARP in front of shard-aware services. Centralizes the mapping (and the SQL rewriting and scatter-gather), at the cost of a hop and a component to scale.
3. **A partition-aware client** — the client caches the topology and connects directly to the owner. Cosmos DB's .NET SDK (direct mode caches the partition key range map), Cassandra token-aware drivers, Kafka producers/consumers (metadata fetch), Redis cluster-aware clients. Fastest; requires a smart client per language and a cache-invalidation story.

All three need a **source of truth for topology** and a way to propagate changes: a coordination service (ZooKeeper, etcd, Consul — Module 9), a gossip protocol, or the database's own metadata. The failure mode to name: a **stale routing cache**. Clients must handle "you asked the wrong node" gracefully (retry on `MOVED`/`NotOwner`/`410 Gone`) — the same hazard as a stale DNS cache in Module 6.

For .NET specifically: Cosmos's `CosmosClient` must be a **singleton** — it caches the partition map, connections, and routing state; constructing one per request is the single most common Cosmos performance bug and a well-known interview question.

---

## Concept 20 — Choosing a partition key: the one-way door

This is the highest-stakes decision in the module, and the one interviewers spend the most time on. Five criteria, in priority order:

1. **Cardinality** — many more distinct values than you have (or will have) physical partitions. `status` (5 values) or `country` (200, wildly skewed) are disqualified. Thousands to millions of values is the target.
2. **Uniformity of load** — both writes and reads. Note that *storage* uniformity and *throughput* uniformity are different: 50,000 tenants may be storage-balanced while three of them generate 60% of requests.
3. **Query alignment** — the key must appear in your dominant access patterns, or every read is a scatter-gather. Write the top five queries down *before* choosing.
4. **Transaction/consistency boundary** — anything that must be atomic or uniquely constrained should land in one partition. Design the model so the strong operations are single-partition.
5. **Growth per key value** — a single key value's data must stay under the platform's per-partition ceiling (Cosmos: 20 GB per logical partition; Cassandra guidance: keep partitions well under ~100 MB / 100k rows; DynamoDB: 10 GB per partition key for tables with LSIs).

**The hot-partition catalog** — recognize these instantly:

| Candidate key | Failure |
|---|---|
| Timestamp / date / auto-increment ID | All current writes on one partition |
| `tenantId` in a SaaS with whale tenants | One tenant exceeds the size or throughput ceiling |
| `channelId` / `roomId` / `postId` | A viral entity gets 1000× the traffic (Discord's exact problem) |
| `deviceType`, `region`, `status`, `country` | Insufficient cardinality |
| `customerId` for a marketplace with megasellers | Skew concentrated in a few values |

**Mitigations, in order of preference:**

- **Add a dimension (compound or hierarchical key).** `tenantId` → `tenantId/userId` or `tenantId/date`. Cosmos's **hierarchical partition keys** exist for exactly this: the first level keeps tenant locality and queries by prefix stay efficient, while subsequent levels let a single tenant exceed the 20 GB and 10,000 RU/s logical-partition ceiling. Prefer this over synthetic string concatenation, which loses prefix-query efficiency.
- **Bucket/salt the hot dimension.** `deviceId#<hour>#<bucket 0..15>`, with reads fanning out over the buckets. You've bought write spread with read fan-out — an explicit trade to state.
- **Isolate the whales.** Dedicated shard/container/database for the largest tenants, shared shards for the long tail. This is standard multi-tenant SaaS practice and doubles as a monetizable isolation tier.
- **Absorb the hot key elsewhere.** Cache, request coalescing (Discord's Rust data-services layer collapses duplicate concurrent reads into one query), write-behind aggregation, or a per-entity in-memory owner (Orleans grain) that batches.
- **Bounded-load routing** for cache-like tiers (Concept 17).

**The interview answer template:** *"Partition key `tenantId`, with `tenantId/entityId` hierarchical because we have a handful of tenants that will exceed the per-partition limit. It satisfies my three dominant queries, all of which are tenant-scoped, and it makes every transaction single-partition. The risk is tenant-level skew, which I'd detect with per-partition RU/throughput telemetry and handle by promoting large tenants to dedicated containers."* That names the key, the justification, the risk, the detection, and the remedy.

---

## Concept 21 — Secondary indexes: local vs global

A partitioned store partitions by *one* key. Querying by anything else needs an index, and there are exactly two shapes.

**Local index (document-partitioned).** Each partition indexes only its own documents. Writes are cheap and atomic (the index lives with the data). Reads by a non-partition-key attribute must **scatter-gather across every partition** — so query latency is driven by the *slowest* partition (Module 6's tail-at-scale arithmetic: 100 partitions, each slow 1% of the time → 63% of queries are slow). DynamoDB LSIs, Elasticsearch, MongoDB's per-shard indexes, Cosmos's per-partition indexes.

**Global index (term-partitioned).** The index itself is partitioned by the indexed term, so a lookup goes to one partition. Reads are fast; **writes now touch a different partition than the data**, so either the write becomes distributed (slow, and needs a transaction you don't have) or the index is updated asynchronously and is therefore **eventually consistent**. DynamoDB GSIs are explicitly eventually consistent for this reason.

**The .NET/Azure specifics:** Cosmos DB has **no global secondary index**. The idiomatic pattern is to build your own: use the **change feed** to project the same data into a second container partitioned by the other key — a materialized view maintained asynchronously, which you must then reason about as a replica with lag (Module 7). Azure's Index Table and Materialized View patterns document this shape. In SQL-land the analogue is a denormalized read table maintained by CDC or an outbox consumer.

The senior framing: **a global secondary index is a replica of your data with a different partition key, and every replica has a consistency model.** Once you say that, questions about GSIs answer themselves.

---

## Concept 22 — Cross-partition operations

Four categories, increasing in pain:

1. **Scatter-gather reads.** Query all partitions, merge results. Costs: tail latency (above), fan-out amplification (one client query becomes N backend queries), and pagination/sorting complexity (a global `ORDER BY … LIMIT 10` requires fetching 10 from each partition and merging). In Cosmos, a query without the partition key is charged and executed across every physical partition — the metric to watch is RU charge, and the fix is usually a second container keyed differently.

```csharp
// Bounded-parallelism scatter-gather with a per-shard timeout and partial results.
public async Task<(IReadOnlyList<Order> Results, IReadOnlyList<string> FailedShards)>
    QueryAllShardsAsync(Func<IShard, CancellationToken, Task<List<Order>>> query, CancellationToken ct)
{
    var results = new ConcurrentBag<Order>();
    var failed = new ConcurrentBag<string>();

    await Parallel.ForEachAsync(
        _shards,
        new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
        async (shard, token) =>
        {
            using var perShard = CancellationTokenSource.CreateLinkedTokenSource(token);
            perShard.CancelAfter(TimeSpan.FromMilliseconds(250));   // don't let one shard set the p99
            try
            {
                foreach (var o in await query(shard, perShard.Token)) results.Add(o);
            }
            catch (Exception ex) when (ex is OperationCanceledException or DbException)
            {
                failed.Add(shard.Name);                              // degraded harvest, full yield
            }
        });

    return (results.ToList(), failed.ToList());
}
```

Note what that code encodes: Module 7's **harvest vs yield**. Returning 19 of 20 shards' results with a "partial results" flag is usually better product behaviour than a 500, and saying so is an architect-level instinct.

2. **Aggregations.** Counts, sums, top-N across partitions. Do them in a stream processor or a pre-aggregated read model, not at query time. Approximate structures (HyperLogLog for distinct counts, t-digest for percentiles) merge cheaply across partitions and are often the right answer for dashboards.
3. **Global uniqueness.** A unique constraint cannot be enforced across partitions without coordination. Three answers, best first: make the key structurally partition-local (prefix with tenant/region); maintain a separate "claims" partition/table keyed *by the unique value* and insert there first (the index-table pattern, adjudicated by one partition's linearizable write); or coordinate (2PC/consensus).
4. **Cross-partition transactions.** 2PC couples availability and holds locks across the network; sagas with idempotency and compensation are the usual answer (Module 12). Some engines offer real distributed transactions (Spanner, CockroachDB, Citus, Cosmos within a logical partition only) — know which.

---

## Concept 23 — Resharding: the migration playbook

Nothing in Part B is harder than changing the partitioning of a live system. This is also the most likely *architect-round* exercise in the whole module, because it's brownfield, risky, and requires a plan rather than a diagram.

**The structural trick that saves you: logical shards.** Never map keys to *physical* nodes. Map keys to a large fixed number of **logical shards** (1,024 or 4,096), then map logical shards to physical nodes in a directory. Rebalancing becomes "move logical shard 719 from node 3 to node 7" — a data move with no key-mapping change and no application awareness. This is Notion's 480 logical shards over 32 physical Postgres instances, Figma's DBProxy logical-shard planner, Vitess's keyspace/shard model, and Redis Cluster's 16,384 slots. **If you propose sharding without logical shards, expect to be pushed on it.**

**The six-phase playbook:**

1. **Decide and instrument.** Pick the shard key (Concept 20), then verify it against *production* query logs, not assumptions. Identify every query that doesn't contain the key — each one is future work (scatter-gather, a read model, or a product change).
2. **Make the application shard-aware while still single-database.** Introduce the routing layer (or logical-shard views over the existing monolith) and run all traffic through it with one physical shard. This de-risks the code path before any data moves. Notion did this; Figma validated correctness with views on the unsharded database.
3. **Double-write (or CDC-replicate) to the new topology.** Write to old and new, with new-side failures logged but non-fatal at first. Alternatively, use logical replication/CDC so the new shards tail the old database — usually preferable, because it can't diverge on application bugs.
4. **Backfill.** Copy historical data in bounded batches, throttled, resumable, idempotent. Then **verify**: row counts, checksums per logical shard, and spot-level comparisons. Budget days-to-weeks and expect to run it more than once.
5. **Shadow reads.** Serve from old, read from new in parallel, compare, and report mismatches. This is where you find the bugs you'd otherwise find during cutover.
6. **Cut over, per shard or per table.** Flip reads first (reversible), then writes (needs a brief write freeze, a fence on the old path, or a lease to guarantee no split-brain writes). Keep the old path warm and a rollback plan written down. Notion's cutover was a five-minute scheduled maintenance window — a reminder that a short, honest downtime is often the *cheapest* correct answer, and that proposing it is a mark of judgment, not weakness.

**The cost you must state:** this is a multi-month program for a real system, it touches every team, and it is essentially irreversible once writes have cut over. That's why Module 6's advice — scale up, add replicas, cache, and defer sharding until the numbers force it — is not laziness but risk management. Alternatives to consider and reject explicitly: a bigger instance, archiving cold data, vertical/functional partitioning (move a few large tables to their own database — Figma's first move, cheaper than sharding), a managed elastic store (Cosmos, Citus, Hyperscale), or moving the offending workload out of the OLTP database entirely.

---

## Concept 24 — Replication × partitioning: placement, cells, and correlated failure

Once each shard is a replica set, *placement* becomes a design decision:

- **Spread replicas across failure domains** — AZs, racks, update domains. Cassandra's `NetworkTopologyStrategy`, Kafka's rack-aware assignment, Cosmos's zone redundancy. A replica set entirely inside one AZ provides no protection against the failure you most expect.
- **Spread leaders too.** If every shard's leader lands in AZ1, that AZ carries all write traffic and its loss triggers a fleet-wide election storm. Leader balancing is a real operational concern (CockroachDB and Kafka both have tooling for it).
- **Mind the replication factor arithmetic.** RF=3 across 3 AZs with quorum writes survives one AZ. RF=2 survives nothing gracefully (no majority). RF=3 in two AZs has a 2:1 split — losing the AZ with two replicas loses the quorum.
- **Correlated failure beats independent-failure math.** Nodes fail together: shared power, shared switch, the same bad deploy, the same poison query, the same expired certificate. Independence assumptions in availability arithmetic are the most common quantitative error in design interviews.

**Cells and shuffle sharding.** Module 6's Z-axis, revisited with a reliability lens. A **cell** (deployment stamp) is a complete, independent copy of the stack serving a subset of tenants; a bad deploy or noisy tenant is contained. **Shuffle sharding** goes further: assign each customer a random *pair* (or k-subset) of workers out of N, so two customers rarely share the same full set — with 8 workers and pairs, a single abusive customer degrades only the ~2 workers it touches, and the probability another customer shares *both* is small. It's a strikingly cheap blast-radius reduction and an excellent thing to bring up unprompted in an architect round.

---

# Part C — The .NET and Azure surface

## Concept 25 — Platform specifics you should know cold

**Cosmos DB** (near-certain material for .NET/Azure roles):

- **Logical partition** = all items sharing a partition key value. Hard limits: **20 GB** of storage, and because a logical partition never spans physical partitions, its throughput ceiling is that of one physical partition.
- **Physical partition** = the unit of compute and replication: roughly up to **50 GB** and **10,000 RU/s**, with four replicas. The service splits physical partitions automatically as data or throughput grows; you never address them directly, but you must reason about them because provisioned RU/s are divided across them (provision 60,000 RU/s over 6 physical partitions and any single partition can draw only 10,000 — the cause of "429s despite plenty of provisioned throughput").
- **Hierarchical partition keys** (up to three levels) let a first-level value exceed 20 GB / 10,000 RU/s while keeping prefix queries efficiently routed. The modern replacement for synthetic concatenated keys.
- **Change feed** is your global-secondary-index and materialized-view mechanism; the Change Feed Processor in the .NET SDK handles lease-based partition distribution across instances (which is also a nice example of Concept 19's routing problem solved with leases).
- **`CosmosClient` is a singleton.** Cross-partition queries cost RU proportional to partitions touched. Transactional batch operations are **single logical partition only**.

```csharp
// Hierarchical partition key: tenant first (locality), then user (spread within a whale tenant).
await database.CreateContainerIfNotExistsAsync(new ContainerProperties(
    id: "events",
    partitionKeyPaths: new List<string> { "/tenantId", "/userId" }));

// Point write: full key.
await container.CreateItemAsync(e, new PartitionKeyBuilder()
    .Add(e.TenantId).Add(e.UserId).Build());

// Prefix query: routed only to partitions holding this tenant, not fanned out globally.
using var it = container.GetItemQueryIterator<Event>(
    new QueryDefinition("SELECT * FROM c WHERE c.type = @t").WithParameter("@t", "login"),
    requestOptions: new QueryRequestOptions
    {
        PartitionKey = new PartitionKeyBuilder().Add(tenantId).Build()
    });
```

**Azure SQL / SQL Server:**

- **Elastic Database tools** (`ShardMapManager`, the Elastic Database Client Library) give you a directory-based shard map with key-range or list mappings, credential-scoped `OpenConnectionForKey`, **split-merge** tooling, and **elastic queries** for cross-shard reads. Old but real, and the only first-party sharding story for Azure SQL — worth naming even if you'd build your own router.
- **Elastic pools** solve a different problem: many databases sharing a resource budget (tenant-per-database multi-tenancy), which is partitioning by tenant with independent schemas and easy per-tenant restore.
- **Hyperscale** changes the calculus: up to 100 TB, near-instant replicas, fast restore — so "we must shard because of size" is often answerable with "or we move to Hyperscale."
- **Partitioned tables** (`CREATE PARTITION SCHEME`, `SWITCH`) = single-node manageability and partition elimination, not sharding. Don't conflate them; do use them for time-series retention.

**PostgreSQL:** declarative partitioning (single-node) for retention and pruning; **Citus** for real distributed tables (distribution column, reference tables, co-located joins) — the closest thing to "sharded Postgres that still speaks SQL"; logical replication for migrations; Patroni for HA.

**EF Core has no sharding.** This comes up and the honest answer wins: EF Core has no native shard router, so you implement one. The workable pattern:

```csharp
public interface IShardMap { string ResolveConnectionString(string shardKey); }

// Directory-based: shardKey -> logical shard (stable hash) -> physical connection (mutable map).
public sealed class HashShardMap(IOptionsMonitor<ShardTopology> topology) : IShardMap
{
    public string ResolveConnectionString(string shardKey)
    {
        var t = topology.CurrentValue;
        var logical = (int)(XxHash64.HashToUInt64(Encoding.UTF8.GetBytes(shardKey)) % (ulong)t.LogicalShards);
        return t.PhysicalByLogicalShard[logical];      // 1024 logical -> N physical
    }
}

// One context per shard, created per request from the resolved connection string.
public sealed class ShardedContextFactory(IShardMap map)
{
    public OrdersContext Create(string tenantId) =>
        new(new DbContextOptionsBuilder<OrdersContext>()
            .UseNpgsql(map.ResolveConnectionString(tenantId), o => o.EnableRetryOnFailure())
            .Options);
}
```

Then say the hard parts out loud: **migrations must run across all shards** (and be backward-compatible so shards can be mid-migration), `DbContext` pooling doesn't help across many connection strings, cross-shard queries need the scatter-gather code from Concept 22, and there is no cross-shard transaction — so aggregates must be shard-local. EF Core's multi-tenancy docs cover the per-tenant-database variant, which is the same thing with a coarser key.

**Kafka:** partition = ordering and parallelism unit. Key → partition by hash; ordering guaranteed *within* a partition only; consumer parallelism is capped by partition count; adding partitions breaks existing key→partition mapping. Design consequence: **the partition key is your ordering contract** — use the aggregate ID so all events for one entity stay ordered, and accept no global order.

**Redis Cluster:** 16,384 hash slots; multi-key operations must target one slot, which is why **hash tags** exist (`user:{1234}:cart` and `user:{1234}:profile` hash on `1234` and co-locate), and why you get `CROSSSLOT` errors otherwise. Lua scripts and transactions are single-slot.

**Azure Service Bus:** message **sessions** are partitioning applied to messaging — a session ID gives FIFO and single-consumer ownership per session, which is how you get per-entity ordering with a competing-consumer pool (and the messaging-tier answer to the "hosted service runs on every instance" problem from Module 6).

**Orleans:** grain identity *is* a partition key, and the grain directory is the routing tier. A grain is a single-threaded, in-memory owner of one entity — i.e. partitioned stateful ownership (Module 6, Concept 7), which gives you per-entity serialization without distributed locks. When an interviewer asks how to serialize writes to a hot aggregate in .NET, "a grain per aggregate, or a per-entity queue/session" is the idiomatic answer.

---

## Concept 26 — Operating a sharded, replicated fleet

The questions architect interviewers ask that pure design candidates miss:

- **Schema migrations across N shards.** Must be backward-compatible and idempotent, applied by an orchestrator with per-shard progress tracking, and safe to run while some shards are migrated and some aren't (expand/contract: add nullable column → backfill → dual-read → switch → drop).
- **Backup and restore per shard**, and *point-in-time* restore across shards — which is not globally consistent unless the platform guarantees it. "Can we restore to a single consistent moment across 32 shards?" is usually "no", and the mitigation is per-tenant restore semantics.
- **Per-partition telemetry, not just aggregates.** Fleet-average RU/CPU/latency hides the hot partition that's failing. You need per-partition-key metrics: Cosmos exposes **normalized RU consumption** per partition and partition-key statistics; Cassandra/Scylla expose per-partition-size and hot-partition tooling. Alert on the max, not the mean.
- **Capacity as a per-partition question.** Each partition has a ceiling; total provisioned capacity is meaningless if one key saturates its own partition.
- **Tenant lifecycle**: onboarding (which shard?), migrating a tenant between shards (the Concept 23 playbook in miniature — and worth building early, because you *will* need to move a whale), offboarding and deletion (GDPR erasure across shards and replicas and backups).
- **Runbooks**: failover, rebalance, hot-partition mitigation, and a stuck replication slot. Naming runbooks as deliverables is architect-track behaviour.

---

# Putting it together

## Worked example 1 — Multi-tenant usage metering on Cosmos DB

**Requirements.** SaaS platform, 50,000 tenants, ingesting usage events: 40,000 writes/sec at peak, 2 TB/month retained 13 months. Queries: (a) a tenant's usage for a billing period, (b) a tenant's per-user breakdown, (c) a platform-wide daily total. Three whale tenants generate ~50% of volume. Multi-region reads, single write region per tenant.

**Partition key.** Rejected: `/date` (all writes on one partition), `/eventType` (cardinality 12), `/tenantId` alone (whales blow past 20 GB and 10,000 RU/s). Chosen: **hierarchical `/tenantId` → `/userId`**, so query (a) is a prefix query confined to a tenant's partitions, (b) is a single logical partition, and whale tenants spread across many physical partitions. Whale tenants additionally get their own container so their throughput can't starve the long tail.

**Write path.** ~40k writes/sec × ~5 RU ≈ 200,000 RU/s, which the service spreads over ≥20 physical partitions; per-partition ceiling 10,000 RU/s means the key must spread writes across ≥20 partitions *evenly at every instant* — the reason `userId` is the second level rather than `date`. Autoscale throughput with a maximum derived from budget; 429s handled by the SDK's retry plus a client-side queue rather than by dropping events.

**Query (c), the platform-wide total,** is the cross-partition operation. Don't scatter-gather 20+ partitions per dashboard refresh: tail the **change feed** into a pre-aggregated `daily_totals` container keyed by `/date`, maintained by a Change Feed Processor. Aggregation is eventually consistent with a visible "as of" timestamp — Module 7's bounded staleness as a product decision.

**Retention.** 13 months at 2 TB/month can't live in a single container economically. Either month-per-container (drop the whole container at expiry — the cheapest delete there is) or TTL on items plus tiered export to blob/Synapse Link for cold analytics.

**Replication.** Single write region per tenant (their home region), read regions elsewhere, **session consistency** with tokens flowed through the API (Module 7, Concept 20). Billing runs read from the write region, because the invoice is money.

**Named bottlenecks.** Whale-tenant skew (detected via normalized RU per partition); the change-feed processor's lease distribution during scale events; cross-partition billing queries at month end (pre-aggregate ahead of the run); and the 20 GB logical-partition ceiling if a single `tenantId/userId` combination becomes hot — mitigated by adding a third level or bucketing.

## Worked example 2 — The brownfield question: shard a 4 TB Postgres orders monolith

**The setup.** ASP.NET Core + EF Core, one Postgres primary (largest available instance, 80% CPU at peak), two read replicas, 4 TB, growth 200 GB/month, 12k writes/sec at peak, and a restore time of 9 hours that the business has just discovered it hates.

**Step 1 — try not to shard.** Quantify what's actually saturated. Options that are cheaper and reversible: archive orders older than 18 months (often 60% of volume); declarative partitioning by month for pruning and cheap retention; move the three largest non-relational tables (events, audit, attachments) out — Figma's "vertical partitioning" move, which bought them years; add caching and a CQRS read model for the reporting queries that are actually driving CPU. State the threshold at which you'd reconsider: *"if writes exceed X or volume exceeds Y with the archive in place, we shard."*

**Step 2 — if sharding, choose the key.** `customerId` (or `tenantId`), because 90% of queries are customer-scoped and the order aggregate is customer-owned. The casualties: global reporting (moves to a warehouse/read model), admin search by order ID (needs a lookup table mapping order ID → logical shard — the index-table pattern), and any cross-customer transaction (none, as it turns out, after review).

**Step 3 — logical shards.** 1,024 logical shards, `stable_hash(customerId) % 1024`, mapped to 8 physical Postgres instances initially (128 logical each). Growth is "move logical shards", never "rehash".

**Step 4 — the routing layer.** An `IShardMap` + context factory as in Concept 25, with the topology in configuration that can be refreshed at runtime. All queries go through repository methods that require a customer ID; a Roslyn analyzer or code review rule forbids raw `DbContext` construction. Cross-shard reads go through the bounded scatter-gather helper with partial-result semantics.

**Step 5 — migrate per Concept 23.** Logical-shard views on the monolith first to validate the code path; then Postgres logical replication into the 8 new instances; backfill with checksums per logical shard; shadow reads for two weeks; cut over reads, then writes, one logical-shard group at a time, with a documented rollback. Accept a short write freeze per group rather than inventing a distributed-write protocol.

**Step 6 — operations, stated as deliverables.** Migration orchestrator across 8 (later N) instances; per-shard backup and PITR with explicit non-global-consistency caveats; per-shard dashboards alerting on max, not mean; a tenant-move tool; and runbooks for failover and rebalance.

**Step 7 — the honest trade.** This is 2–3 quarters of work for the team, and it is irreversible. That framing — cost, risk, reversibility, and the cheaper alternatives you rejected with numbers — *is* the architect answer.

---

## Common questions and what a strong answer contains

**"How would you scale the write path?"** Replicas don't help writes. More partitions do. Name the partition key, why it spreads writes evenly, and the per-partition ceiling.

**"Synchronous or asynchronous replication?"** Both, in different places: synchronous within the region (1–2 ms, RPO 0), asynchronous across regions (70–150 ms, RPO measured in seconds). Then note that async DR means failover is a business decision with a data-loss window, and that automatic failover requires synchronous commit.

**"The primary died — walk me through failover."** Detect (timeout trade-off), elect (majority, or you get split brain), reconfigure (DNS/listener, proxies, app pools, retry policy), fence the old leader. Then: what writes were lost, what caches and consumers are now wrong, and whether the old primary can be reattached or must be rebuilt.

**"How do you pick a shard key?"** Cardinality, load uniformity, query alignment, transaction boundary, per-key growth. Then name the skew risk, how you'd detect it, and the mitigation (hierarchical key, bucketing, whale isolation).

**"What's wrong with `hash(key) % N`?"** Changing N remaps ~everything. Consistent hashing with virtual nodes moves ~1/N; a fixed logical-shard count moves whole shards and never rehashes. Also: don't use `string.GetHashCode()` — it's per-process randomized in .NET Core+.

**"Why virtual nodes?"** Even arcs (load spread ~1/√V), weighting for heterogeneous hardware, and a departing node's load spreading across many survivors instead of one. Then the limitation: consistent hashing balances keys, not load, so hot keys still need bounded loads or splitting.

**"How do you query by something other than the partition key?"** Local index + scatter-gather (cheap writes, tail-latency-bound reads) or global index / second container keyed differently (fast reads, async writes, eventual consistency). In Cosmos, there are no GSIs — use the change feed to build a materialized view. Then: a secondary index is a replica, so it has a consistency model.

**"We need a global unique constraint across shards."** First try to make it structurally shard-local (prefix with tenant/region; ULID instead of a sequence). Otherwise a claims table in one partition adjudicating via a linearizable insert. 2PC is the last resort.

**"How do you reshard live?"** Logical shards, routing layer first, CDC/dual-write, throttled backfill with checksums, shadow reads, per-shard cutover with a rollback plan — and an explicit statement that a short scheduled downtime may be the cheapest correct option.

**"Cosmos: why are we getting 429s when we've provisioned 60,000 RU/s?"** Because RU/s divide across physical partitions and a single partition is capped near 10,000; the key is skewed, or a single logical partition is hot. Diagnose with normalized RU consumption per partition; fix with a better/hierarchical key or bucketing.

**"Kafka: we added partitions and ordering broke."** Key→partition is `hash(key) % partitions`; changing the count re-maps existing keys, so a key's history now spans two partitions and per-key order is gone. Over-provision partitions up front, or key by something you'll never reshard.

**"Table partitioning or sharding?"** Different things: partitioning is single-node manageability and pruning; sharding is horizontal distribution across nodes with routing, rebalancing, and no cross-shard transactions.

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "We'll add read replicas to handle more traffic." | Distinguishes read scaling from write scaling; names the write bottleneck. |
| Treating replication and partitioning as the same thing. | Same data vs different data; then combines them (shard = replica set). |
| "Sharded by user ID" with no justification. | Cardinality, uniformity, query alignment, transaction boundary, growth per key. |
| `hash(key) % nodeCount`. | Consistent hashing with vnodes, or a fixed logical-shard count with a directory. |
| Using `string.GetHashCode()` for routing. | A stable hash (`XxHash64`) — GetHashCode is per-process randomized. |
| Ignoring hot partitions. | Names the skew, the detection metric, and the mitigation before being asked. |
| Assuming automatic failover is free. | Explains sync-commit prerequisites, split brain, fencing, and lost writes. |
| Async cross-region replication with auto-failover for financial data. | Manual failover with a runbook and a stated RPO. |
| Forgetting routing and rebalancing entirely. | Picks a routing architecture and a rebalancing strategy explicitly. |
| Proposing sharding as the first scaling move. | Archives, partitions locally, caches, and moves large tables first; states the numeric threshold for sharding. |
| Hand-waving the migration. | Logical shards, CDC/dual-write, backfill with verification, shadow reads, per-shard cutover, rollback. |
| Cross-partition queries as an afterthought. | Quantifies fan-out tail latency and pre-aggregates via change feed / read models. |
| "We'll add a secondary index." | Local vs global, write amplification vs scatter-gather, and the eventual consistency that follows. |
| Assuming replicas fail independently. | Names correlated failure: shared AZ, bad deploy, poison query, certificate expiry. |
| No numbers. | Per-partition ceilings (20 GB / 10k RU/s), RTTs, lag bounds, restore times. |

---

## Practice exercises

**Exercise 1 — Build and measure a hash ring.** Implement the `ConsistentHashRing` above. Hash 1,000,000 keys across 10 nodes at V = 1, 4, 16, 64, 256 virtual nodes and plot the load distribution (max/mean and p99/mean). Then remove one node and measure the fraction of keys that move; compare against `hash % N`. Finally add a Zipfian key popularity distribution and observe that per-key balance does nothing for per-*request* balance — then implement bounded loads and re-measure. This one exercise covers Concepts 15, 17, and 20.

**Exercise 2 — Lose data on failover, on purpose.** With Docker Compose or .NET Aspire, run Postgres with one async standby. Write continuously from a .NET client recording every acknowledged commit. Kill the primary, promote the standby, and diff the acknowledged writes against what survived — that's your RPO, measured. Repeat with `synchronous_commit = on` and `synchronous_standby_names = 'ANY 1 (s1)'` and observe both the zero-loss result and the write-latency cost. Then repeat with `remote_apply` and measure read-your-writes on the standby.

**Exercise 3 — Shard a small EF Core app.** Take one of your projects, introduce `IShardMap` with 256 logical shards over 4 Postgres containers, route all reads and writes through a shard-aware repository, and implement the bounded scatter-gather for one cross-shard query with partial results. Then write the migration runner that applies EF migrations to all four shards and reports per-shard status. Finally, move logical shards 0–63 from node 1 to a new node 5 using logical replication, with verification — the miniature version of Concept 23.

**Exercise 4 — Provoke a hot partition in Cosmos.** Create two containers: one keyed `/date`, one keyed `/tenantId` with `/userId` as a second level. Load the same synthetic event stream (with a Zipfian tenant distribution) into both at a few thousand writes/sec. Compare RU charges, 429 rates, and the normalized RU consumption per partition metric in Azure Monitor. Then run a tenant-scoped query against each and compare RU cost — the difference between a prefix query and a fan-out is the whole lesson.

**Exercise 5 — Write the resharding plan (architect-round shape).** One page for Worked Example 2: the alternatives you'd try first with the numbers that would rule them out, the shard key and its casualties, the logical-shard scheme, the six migration phases with verification gates, the rollback plan at each phase, the operational deliverables, and the cost/duration estimate. This is very close to a real architect take-home.

**Exercise 6 — Five-minute verbal drill.** Timed, out loud: "Design the storage for a chat platform: 100M users, 10B messages/month, message history forever, search, read receipts, and per-channel ordering." Check afterwards that you named the partition key and its sort key, the per-partition size ceiling, the hot-channel mitigation, the replication factor and failure-domain placement, how search is served (a different partitioning of the same data), and what happens when a channel goes viral.

---

## Free resources

### Foundational papers (all free)

| Resource | What it covers | Why read it |
|---|---|---|
| [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — DeCandia et al., 2007 | Consistent hashing with vnodes, preference lists, sloppy quorums, hinted handoff, vector clocks, Merkle-tree anti-entropy | The single most load-bearing paper for this module; sections 4.1–4.8 are the mechanisms |
| [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/) — Chang et al., 2006 | Range-partitioned tablets, splits, a metadata routing hierarchy | The canonical range-partitioning design, and where "tablet" comes from |
| [Spanner: Google's Globally-Distributed Database](https://research.google/pubs/spanner-googles-globally-distributed-database/) | Paxos groups per shard, directory-based movement, TrueTime | Shows replication and partitioning composed properly at global scale |
| [Chain Replication for Supporting High Throughput and Availability](https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf) — van Renesse & Schneider, 2004 | Head/tail chain, strong consistency with simple recovery | The best-known alternative replication topology |
| [Consistent Hashing and Random Trees / Web Caching with Consistent Hashing](https://www8.org/w8-papers/2a-webserver/caching/paper2.html) — Karger et al. | The original consistent hashing construction and its caching motivation | The source; short and readable |
| [Consistent Hashing with Bounded Loads](https://research.google/blog/consistent-hashing-with-bounded-loads/) — Google Research ([arXiv](https://arxiv.org/abs/1608.01350)) | Capping per-node load and forwarding overflow | The practical answer to hot keys, with the Vimeo production story |
| [A Fast, Minimal Memory, Consistent Hash Algorithm](https://arxiv.org/abs/1406.2294) — Lamping & Veach (jump consistent hash) | O(log N), zero-memory bucket assignment | Elegant, and the right tool when buckets are `0..N-1` logical shards |
| [Maglev: A Fast and Reliable Software Network Load Balancer](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/) — Google | Lookup-table hashing with even distribution and minimal disruption | The third consistent-hashing family |
| [Slicer: Auto-Sharding for Datacenter Applications](https://www.usenix.org/conference/osdi16/technical-sessions/presentation/adya) — Adya et al., OSDI '16 | A general-purpose sharding control plane with load-aware rebalancing | Shows what "automatic rebalancing done properly" actually requires |
| [Scaling services with Shard Manager](https://engineering.fb.com/production-engineering/scaling-services-with-shard-manager/) — Meta | Shard assignment, constraints, and failure-domain-aware placement as a platform | The industrial counterpart to Slicer |
| [The Tail at Scale](https://research.google/pubs/the-tail-at-scale/) — Dean & Barroso | Why fan-out turns p99 into p50 | The quantitative argument against casual scatter-gather |
| [Why Logical Clocks Are Easy](https://queue.acm.org/detail.cfm?id=2917756) — Baquero & Preguiça, ACM Queue | Version vectors vs vector clocks, done carefully | Fixes the terminology most people get wrong |
| [Workload isolation using shuffle-sharding](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/) — Amazon Builders' Library | Assigning customers to overlapping worker subsets | Cheapest blast-radius reduction in the business |
| [Challenges with distributed systems](https://aws.amazon.com/builders-library/challenges-with-distributed-systems/) — Amazon Builders' Library | Partial failure, correlated failure, the operational reality | Good vocabulary for failover discussions |

### Explainers, courses, and books (free)

| Resource | What it covers |
|---|---|
| [Distributed Systems for Fun and Profit](http://book.mixu.net/distsys/single-page.html) — Mikito Takada | Free short book; chapters 4–5 cover replication (single-leader through leaderless) directly |
| [Martin Kleppmann's distributed systems lectures](https://martin.kleppmann.com/) | Free Cambridge lecture videos and notes on replication, causality, and consensus (the DDIA author) |
| [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.5840/) | Lecture notes and the canonical paper list, free |
| [CMU 15-445 Database Systems](https://15445.courses.cs.cmu.edu/) | Free lectures on storage, indexing, and distributed databases |
| [Jepsen: consistency models & analyses](https://jepsen.io/analyses) | Real engines tested against their replication claims — read the Cassandra, Mongo, and Postgres ones |
| [The System Design Primer — sharding & replication](https://github.com/donnemartin/system-design-primer) | Vocabulary-level overview with links |
| [PostgreSQL: Replication & log-shipping](https://www.postgresql.org/docs/current/warm-standby.html) | Streaming replication, synchronous standbys, cascading replication |
| [PostgreSQL: Logical replication](https://www.postgresql.org/docs/current/logical-replication.html) | Publications, subscriptions, replica identity — the CDC and migration substrate |
| [PostgreSQL: Declarative partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html) | Single-node partitioning, pruning, attach/detach |

### Case studies (the best material in the module)

| Resource | What it covers |
|---|---|
| [Herding elephants: sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion) | Logical shards over physical instances, the shard key decision, double-write migration, a five-minute cutover |
| [How Figma's databases team lived to tell the scale](https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/) | Vertical partitioning first, then horizontal sharding with DBProxy; "colos" for co-located joins |
| [How Figma built DBProxy for sharding Postgres](https://pganalyze.com/blog/5mins-postgres-figma-dbproxy-sharding-postgres) | A concise walkthrough of the routing tier, logical shard planning, and scatter-gather |
| [How Discord stores billions of messages](https://discord.com/blog/how-discord-stores-billions-of-messages) | Compound partition key (channel + time bucket), why partition sizing matters |
| [How Discord stores trillions of messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Hot partitions, repair toil, request coalescing, and the Cassandra → ScyllaDB migration |
| [Sharding & IDs at Instagram](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c) | Logical shards and ID generation that encodes the shard — the "naming instead of coordination" trick |
| [Schemaless: Uber's datastore](https://www.uber.com/blog/schemaless-rewrite/) | A sharding layer built over MySQL, with append-only cells |
| [Vitess documentation](https://vitess.io/docs/) | Keyspaces, vindexes, resharding workflows — the reference implementation of sharded MySQL |
| [Citus documentation](https://docs.citusdata.com/) | Distribution columns, co-located joins, reference tables for sharded Postgres |

### .NET and Azure documentation

| Resource | What it covers |
|---|---|
| [Partitioning and horizontal scaling in Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning) | Logical vs physical partitions, the 20 GB limit, partition key guidance |
| [Hierarchical partition keys](https://learn.microsoft.com/en-us/azure/cosmos-db/hierarchical-partition-keys) | Subpartitioning, prefix queries, escaping the per-logical-partition ceilings |
| [Cosmos DB change feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed) | Building materialized views and secondary-index equivalents |
| [Cosmos DB conflict resolution policies](https://learn.microsoft.com/en-us/azure/cosmos-db/conflict-resolution-policies) | Multi-region writes: LWW, custom stored procedures, the conflict feed |
| [Always On availability groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server) | Replica roles, listeners, quorum |
| [Availability modes (sync vs async commit)](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/availability-modes-always-on-availability-groups) | Why automatic failover requires synchronous commit |
| [Partitioned tables and indexes (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes) | Single-node partitioning — and why it isn't sharding |
| [Azure SQL: sharding with Elastic Database tools](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-scale-introduction) | Shard map manager, split-merge, elastic queries |
| [Shard map management](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-scale-shard-map-management) | Directory-based routing, `OpenConnectionForKey`, range vs list maps |
| [Azure SQL read scale-out](https://learn.microsoft.com/en-us/azure/azure-sql/database/read-scale-out) | `ApplicationIntent=ReadOnly` and replica lag characteristics |
| [Azure SQL Hyperscale architecture](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale) | Log service and page servers — replication moved into storage |
| [Npgsql failover and load balancing](https://www.npgsql.org/doc/failover-and-load-balancing.html) | Multi-host connection strings, `Target Session Attributes` |
| [EF Core: multi-tenancy](https://learn.microsoft.com/en-us/ef/core/miscellaneous/multitenancy) | Database-per-tenant and discriminator approaches — the closest EF Core gets to sharding |
| [Sharding pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding) / [Index Table](https://learn.microsoft.com/en-us/azure/architecture/patterns/index-table) / [Materialized View](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view) | Azure Architecture Center's vocabulary for exactly this module |
| [Multitenant data & storage approaches](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) | Tenant isolation models, sharding vs pooling, whale-tenant handling |
| [Kafka documentation — topics and partitions](https://kafka.apache.org/documentation/#intro_topics) | Ordering per partition, keys, consumer parallelism |
| [Redis cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) | 16,384 hash slots, hash tags, `MOVED`/`ASK` redirects |
| [Cassandra data modeling](https://cassandra.apache.org/doc/latest/cassandra/data_modeling/) | Partition key vs clustering columns, partition sizing, tombstones |
| [DynamoDB partition key design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html) | Write sharding, hot partitions, adaptive capacity, LSI/GSI trade-offs |
| [Azure Service Bus message sessions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions) | FIFO per session — partitioning applied to messaging |
| [Microsoft Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/overview) | Grains as partitioned stateful ownership, with a directory as the routing tier |
| [Debezium documentation](https://debezium.io/documentation/) | CDC from Postgres/SQL Server/MySQL for migrations and read models |
| [Patroni](https://patroni.readthedocs.io/en/latest/) | Quorum-based Postgres failover with etcd/Consul — fencing done properly |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Replication vs partitioning | Same data in many places vs different data in different places; combined as shard = replica set |
| Scaling writes | Replicas never help; more partitions do; name the per-partition ceiling |
| Sync vs async | Sync in-region (1–2 ms, RPO 0), async cross-region (70–150 ms, RPO seconds); auto-failover needs sync |
| Replication lag | Measure bytes and seconds, receive vs apply; treat the bound as an SLO and eject lagging replicas |
| Failover | Detect, elect by majority, reconfigure, fence; lost writes and split brain are the failure modes |
| Replication log formats | Statement / physical WAL / logical / trigger; logical is what CDC, migrations, and read models ride on |
| Multi-leader | Local writes, conflicts become domain logic; prefer single-writer-per-key by home region |
| Leaderless | Quorums + read repair + Merkle anti-entropy + hinted handoff; sloppy quorums void the overlap |
| Concurrent writes | Version vectors detect concurrency; LWW loses data; siblings push merging into the domain |
| Deletes in an eventually consistent store | Tombstones plus a GC window longer than the repair interval |
| Range vs hash partitioning | Range scans vs uniformity; compound key (hash the partition, sort within) is the practical answer |
| Consistent hashing | Ring + ~128 vnodes (spread ≈ 1/√V), ~1/N movement, preference list for replicas; balances keys, not load |
| `hash % N` | Remaps everything; use vnodes or a fixed logical-shard count with a directory |
| Rebalancing | Fixed logical partitions / dynamic splits / per-node tokens; automate planning, rate-limit movement |
| Routing | Coordinator, routing tier, or partition-aware client; all need a topology source of truth and stale-cache handling |
| Partition key choice | Cardinality, uniformity, query alignment, transaction boundary, per-key growth — then the skew mitigation |
| Hot partitions | Add a dimension (hierarchical key), bucket/salt, isolate whales, coalesce requests, bounded loads |
| Secondary indexes | Local = scatter-gather reads; global = async writes; a secondary index is a replica with a consistency model |
| Cross-partition queries | Fan-out multiplies tail latency; pre-aggregate via change feed; partial results (harvest vs yield) |
| Global uniqueness | Make it partition-local by naming, or adjudicate in one partition with a claims table |
| Resharding | Logical shards, routing first, CDC/dual-write, verified backfill, shadow reads, per-shard cutover, rollback |
| Placement | Replicas and leaders across failure domains; RF=3 over 3 AZs; correlated failure beats independence math |
| Blast radius | Cells/deployment stamps and shuffle sharding |
| Cosmos limits | 20 GB per logical partition; ~50 GB / 10k RU/s per physical partition; RU/s divide across partitions |
| Kafka | Ordering per partition only; key→partition by hash; adding partitions breaks key affinity |
| Redis Cluster | 16,384 slots; hash tags to co-locate; `CROSSSLOT` otherwise |
| EF Core and sharding | No native support: shard map + context factory, migrations across shards, no cross-shard transactions |

---

## Progress

Module 8 complete. Next: **Module 9 — Consensus & coordination** (Raft, Paxos, etcd/ZooKeeper, distributed locks, leases and fencing tokens, vector clocks, CRDTs) — which supplies the missing piece under everything above: *how* a group of replicas agrees on a leader and a log, why a distributed lock without a fencing token is unsafe, and how the weaker models track causality and converge without coordination at all.
