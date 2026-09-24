# Module 12 — Data Storage Deep Dive
*Phase 3: Distributed Systems Theory · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **a database is not a place you put data — it is a bundle of guarantees, and every guarantee is paid for by a physical layout on disk and a concurrency-control mechanism in memory.**

That reframing does a lot of work. "SQL vs NoSQL" stops being a branding question and becomes: *which access patterns do I want to be cheap, and which invariants am I willing to enforce in application code instead of the engine?* "Should I add an index?" stops being a tuning tip and becomes: *I am buying read seeks with write amplification and lock surface.* "Can we do this in one transaction?" stops being a coding detail and becomes *a boundary that determines your service decomposition* — which is exactly why this module ends where Module 11 ended, at sagas.

Everything from Phase 3 lands here. Module 7's consistency models are the *distributed* half of a picture whose *local* half is isolation levels — and confusing the two is one of the most common vocabulary failures in senior interviews. Module 8's partitioning is what a shard key is. Module 9's consensus is what makes a distributed commit possible and what makes two-phase commit *not* an instance of it. Module 10's cache is a denormalized replica, which is exactly what a secondary index and a materialized view also are. Module 11's outbox exists precisely because you rejected a distributed transaction.

This module has seven jobs:

1. **Put a storage engine in your head.** B+ trees, LSM trees, the write-ahead log, the buffer pool, pages. Not so you can implement one, but because every performance answer you give — why this index helps, why that write pattern is slow, why the p99 spiked at 3 a.m. — is downstream of these four mechanisms. Candidates who can say *why* a random-GUID clustered key is expensive outscore candidates who merely know it is.
2. **Make indexing a design skill, not a tuning reflex.** Selectivity, column order, covering, statistics, the write cost, and the specific senior move of asking "what does this index let the planner *stop* doing?"
3. **Install isolation levels correctly and permanently.** Dirty read, non-repeatable read, phantom, lost update, read skew, write skew — and the ability to say which levels prevent which, and to recognize write skew in a design before it becomes an incident.
4. **Make SQL-vs-NoSQL a set of axes rather than a side.** Data model, query flexibility, transaction scope, scale-out shape, schema evolution, operational cost. Then apply it to the Azure lineup concretely.
5. **Complete the 2PC vs saga comparison** started in Module 11, Concept 21 — with the actual protocol, the actual blocking failure, why XA persists in enterprise .NET, and why Spanner's "2PC" isn't the thing you were warned about.
6. **Cover operating data, not just designing it.** Migrations at scale, connection limits, read replicas, backups you've actually restored, multi-tenancy, and cost — this is the half that separates architect answers from senior-IC answers.
7. **Build the judgment to say "one Postgres instance."** The most valuable sentence in many data design reviews is "this fits on one machine for the next four years, and here's the arithmetic."

Seven framings to carry through:

1. **The storage engine decides what is cheap.** B-tree or LSM, row or column, clustered or heap — this is a decision about which operations get sequential I/O.
2. **An index is a denormalized, transactionally-maintained copy of your data sorted differently.** Everything true about replicas (Module 8) and caches (Module 10) is true about indexes: they cost writes, they can be stale in distributed systems, and they exist to make one access pattern cheap.
3. **Isolation is about concurrency on one node; consistency is about visibility across nodes.** Serializable ≠ linearizable. Knowing the difference is a reliable seniority tell.
4. **The transaction boundary is an architectural boundary.** Decide what must be atomic *before* you decide what is a service. Aggregate first, then service, then transaction — never the reverse.
5. **Schema flexibility is never free; it is relocated.** Schema-on-read moves validation from DDL into every reader, forever, including the readers you'll write in three years.
6. **Every distributed transaction question is really "can I avoid needing one?"** Usually yes — by moving the boundary, by idempotency, or by accepting a compensatable window.
7. **Volunteer the numbers.** Row size × rows, index size, working set vs RAM, IOPS, RU/s, drain and rebuild times. Data questions are the easiest place in a design interview to be concrete, and the most damaging place to be vague.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Storage as latency arbitrage | Every engine design is an attempt to turn random I/O into sequential I/O and to keep the working set in RAM |
| 2 | Pages, rows, heaps | The page (8 KB in SQL Server and Postgres) is the unit of I/O, caching, and often locking — rows are an interpretation of bytes inside it |
| 3 | B+ trees | Sorted, high-fanout, 3–4 levels for hundreds of millions of rows; in-place updates, page splits, fill factor |
| 4 | LSM trees | Memtable + immutable SSTables + compaction; sequential writes, read amplification, bloom filters, compaction as the real cost |
| 5 | B-tree vs LSM, RUM | Read, Update, Memory — optimize two, pay in the third; the choice predicts write throughput, space, and p99 shape |
| 6 | WAL, durability, group commit | Durability is "the log is fsynced," not "the page is written"; group commit amortizes fsync; checkpoints bound recovery |
| 7 | Buffer pool & working set | The only cache number that matters is working set vs RAM; a cache-miss cliff is the most common "sudden" DB slowdown |
| 8 | Row vs column, OLTP vs OLAP | Row = fetch whole entities; column = scan few attributes over many rows with 10× compression; don't run both on one schema |
| 9 | The arithmetic | Row size × rows, index size, working set, IOPS, RU/s — the numbers that turn opinions into decisions |
| 10 | What an index is | A sorted copy answering three questions: can it seek, can it cover, can it deliver order |
| 11 | Clustered vs nonclustered | The clustered key is the row's address; random keys cause splits and fragmentation; UUIDv7 / sequential GUIDs fix it |
| 12 | Composite indexes | Equality columns first, then range, then order-by; leftmost prefix; one composite usually beats three singles |
| 13 | Covering indexes & INCLUDE | Eliminating the key lookup is often a 10–100× win; INCLUDE adds leaf payload without widening the key |
| 14 | Statistics & cardinality estimation | Bad plans are usually bad estimates; skew, stale stats, correlated predicates, and non-sargable expressions |
| 15 | Parameter sniffing | One plan cached for many parameter shapes; RECOMPILE, OPTIMIZE FOR, plan guides, Query Store forcing |
| 16 | Specialized indexes | Filtered, hash, columnstore, GIN/GiST, full-text, spatial, vector (HNSW/DiskANN) — and when each earns its keep |
| 17 | The write cost of indexes | Every index multiplies write I/O, log volume, and lock surface; fragmentation and maintenance windows are real |
| 18 | Local vs global secondary indexes | In a partitioned store, an index is either per-partition (scatter-gather reads) or globally repartitioned (async, eventually consistent writes) |
| 19 | Diagnosing a slow query | The narrative: measure → plan → estimates vs actuals → the physical cause → the fix, in that order |
| 20 | ACID, precisely | Atomicity is all-or-nothing, Consistency is your invariants (not the engine's), Isolation is the interesting one, Durability is fsync + replication |
| 21 | The anomaly catalogue | Dirty write/read, non-repeatable read, phantom, lost update, read skew, write skew — name them on sight |
| 22 | The isolation ladder | RU / RC / RR / Snapshot / Serializable, plus the ANSI critique; snapshot ≠ serializable, and write skew is why |
| 23 | MVCC mechanics | Postgres xmin/xmax + vacuum + bloat; SQL Server version store / PVS; readers don't block writers, at a storage cost |
| 24 | Locking | 2PL, granularity, intent locks, escalation at ~5000 locks, and SQL Server 2025 optimized locking (TID + LAQ) |
| 25 | Deadlocks | Detection and victim selection; consistent ordering, short transactions, covering indexes; always retry |
| 26 | Optimistic concurrency in .NET | rowversion / xmin as concurrency token, DbUpdateConcurrencyException, and the three resolution strategies |
| 27 | Serializable & SSI | True serializability via 2PL or SSI (Postgres); write skew is the anomaly that justifies it |
| 28 | Isolation vs consistency | Serializable is about order of transactions; linearizable is about visibility of a single object in real time — orthogonal axes |
| 29 | Long transactions | The single most common self-inflicted database outage: held locks, version-store growth, pool exhaustion |
| 30 | What NoSQL meant | Not "no SQL" — dropping the relational model *and* cross-entity transactions to buy partition-friendly scale-out |
| 31 | The families | Relational, document, key-value, wide-column, graph, time-series, search, vector — with the access pattern each is built for |
| 32 | Normalization & aggregates | Normalize for write correctness, denormalize for read cost; the aggregate is the unit of atomicity and the unit of storage |
| 33 | Access-pattern-driven design | In NoSQL you enumerate queries first and design storage backwards; single-table design and its trade-offs |
| 34 | Cosmos DB modeling | Partition key, 20 GB logical / 50 GB and 10k RU/s physical partition, hierarchical keys, RU arithmetic, change feed |
| 35 | Schema evolution without DDL | Schemaless means "schema enforced by every reader"; version fields, tolerant readers, lazy migration, backfills |
| 36 | Distributed SQL | Spanner, CockroachDB, YugabyteDB, TiDB, Aurora DSQL: SQL + horizontal scale + serializable, paid for in commit latency |
| 37 | Polyglot persistence | Each additional store multiplies operational surface; the bar is a *first-class* access pattern, not a preference |
| 38 | "Just use Postgres" | The arithmetic that justifies one node, and the senior move of stating when it stops being true |
| 39 | OLTP/OLAP separation | Don't run analytics on the transactional store; CDC, medallion architecture, Fabric/Synapse, and the staleness contract |
| 40 | The atomic commit problem | Distributed atomic commit needs unanimity, which is why it blocks — consensus needs a majority, which is why it doesn't |
| 41 | Two-phase commit | Prepare/commit, the in-doubt window, coordinator failure, the held locks, and why 3PC didn't save it |
| 42 | XA, MSDTC, and .NET | TransactionScope promotes to MSDTC, Windows-only, opt-in since .NET 7; know this before you propose it |
| 43 | 2PC over consensus | Spanner/Percolator: each participant is a replicated group, so 2PC's blocking problem is removed rather than tolerated |
| 44 | Sagas, completed | ACD not ACID: no isolation, compensations are new facts, semantic locks and the four countermeasures |
| 45 | TCC and reservations | Try-Confirm-Cancel: make the first step a reservation with a TTL — the pattern that makes sagas feel safe |
| 46 | Outbox + idempotency | The default answer for cross-service writes; one local transaction plus at-least-once delivery plus a dedup key |
| 47 | The decision framework | Move the boundary → single transaction → reservation → saga → 2PC, in that order of preference |
| 48 | The scaling ladder | Query/index → cache → vertical → read replicas → functional split → partitioning → sharding; skip nothing |
| 49 | Read replicas | Replication lag is a consistency decision; read-your-writes via sticky reads, LSN tokens, or write-path reads |
| 50 | Partitioning vs sharding | Same table split within one engine vs data split across engines; partition elimination, sliding windows, resharding pain |
| 51 | Connection pooling | Postgres connections are processes; SQL Server pools per connection string; serverless compute breaks both assumptions |
| 52 | Zero-downtime migration | Expand/contract: add nullable → dual-write → backfill → switch reads → stop writes → drop; never a blocking DDL at peak |
| 53 | Multi-tenancy | Silo / bridge / pool: isolation, noisy neighbours, per-tenant restore, and the migration fan-out cost |
| 54 | Backup, PITR, RPO/RTO | An untested restore is not a backup; geo-replication is not a backup; state the two numbers before the strategy |
| 55 | Lifecycle & cost | Retention, TTL, archival tiers, soft delete's hidden costs, and the cost model per platform |
| 56 | Azure SQL | DTU vs vCore, General Purpose / Business Critical / Hyperscale, serverless, elastic pools, failover groups |
| 57 | SQL Server 2025 | Native JSON type, vector type + DiskANN indexes, regex, optimized locking, change event streaming |
| 58 | Cosmos DB product surface | RU/s vs serverless, autoscale, indexing policy, TTL, transactional batch, change feed, GSIs in preview |
| 59 | The rest of the Azure data lineup | Azure Database for PostgreSQL, Azure DocumentDB, Managed Instance, Storage/Table/Blob, Fabric |
| 60 | ADO.NET and EF Core at architecture level | Pooling, retries on transient faults, DbContext lifetime, tracking, compiled queries, and where the ORM stops |
| 61 | Data security | TDE, Always Encrypted, RLS, dynamic data masking, PII classification, crypto-shredding for GDPR |
| 62 | Observability for data | The four SLIs, Query Store, wait statistics, pg_stat_statements, RU charge, and tracing DB spans |
| 63 | Anti-patterns | EAV, shared database, GUID clustered keys, soft-delete everywhere, SELECT *, over-indexing, the DB as a queue |
| 64 | Choosing a database | The decision table and the two questions that resolve most of it |
| 65 | Migrating a live database | Strangler fig for data: dual-write, backfill, shadow read, compare, cut over, keep the rollback |
| 66 | When to say no | To a new store, to sharding, to event sourcing, to a distributed transaction, and to "let's denormalize it" |

---

# Part A — The physical layer

## Concept 1 — Storage as latency arbitrage

Recall the latency ladder from Module 5, because every storage engine design is an attempt to exploit it:

| Operation | Order of magnitude | Relative |
|---|---|---|
| L1 cache reference | ~1 ns | 1× |
| Main memory reference | ~100 ns | 100× |
| NVMe SSD random read | ~50–150 µs | ~1,000,000× |
| SSD sequential read, 1 MB | ~100–300 µs | — |
| Rotational disk seek | ~5–10 ms | ~10,000,000× |
| Same-datacenter round trip | ~0.5 ms | — |
| Cross-region round trip (EU↔US) | ~80–150 ms | — |

Three consequences that explain almost every engine design decision:

**1. Random I/O is the enemy.** A B-tree seek touching 4 pages costs 4 random reads if they're not cached. An LSM tree exists because turning those writes into one sequential append is worth accepting extra work on the read path. A columnstore exists because scanning one column of 100M rows sequentially beats reading 100M whole rows randomly.

**2. RAM is the real database.** If your working set fits in the buffer pool, your database is an in-memory database with a durability mechanism. If it doesn't, every query is at the mercy of storage latency. This is why "the query got slow and nothing changed" is nearly always "the table grew past the point where its hot pages fit in memory" (Concept 7).

**3. Anything crossing a network is three orders of magnitude more expensive than anything in memory, and a cross-region commit is five.** This is the entire cost of distributed transactions (Part E) and the entire reason Module 7's PACELC "else latency" clause matters commercially.

**The interview-grade sentence:** *"Storage engines are latency arbitrage — they trade CPU and space, which are cheap, for random I/O, which is not. When I evaluate a design I'm asking which operations became sequential and which stayed random."*

---

## Concept 2 — Pages, rows, and heaps: the unit of I/O

Databases do not read rows. They read **pages**.

- **SQL Server:** 8 KB page; 8 contiguous pages = a 64 KB extent; the maximum in-row size of a row is 8,060 bytes, with overflow for LOB and variable-length columns.
- **PostgreSQL:** 8 KB page by default; rows larger than roughly a quarter of a page get pushed out to **TOAST** (compressed and/or chunked out-of-line storage).
- **InnoDB:** 16 KB page by default.

Why this matters in an interview answer:

**Rows per page drives everything.** A 200-byte row gives ~40 rows per 8 KB page; a 2 KB row gives ~4. Widening rows by adding columns you rarely read multiplies the I/O of every scan over that table by 10×. This is a concrete, quantitative argument for moving a large `nvarchar(max)` description or a blob out of a hot table — and it's a far better answer than "for cleanliness."

**A heap is a table without a clustered index.** Rows are placed wherever there's space; the row's physical address is a `RID` (file:page:slot). Heaps are fine for staging and write-once tables; they're bad for anything with updates and range scans, because there's no ordering to exploit and forwarded records accumulate.

**Fill factor and page splits.** When you insert into the middle of a full page in an ordered structure, the engine splits the page — allocating a new one, moving half the rows, and fixing up pointers. That's extra I/O, extra log, and permanent fragmentation. Fill factor reserves free space to postpone that. This is the mechanism behind the GUID primary key problem (Concept 11).

**Page-level everything.** Pages are the unit of I/O, the unit of buffer-pool caching, the unit of checksums, often the unit of locking, and the unit of compression. When someone says "row-level locking," they still mean rows tracked within pages, and escalation goes row → page → table for a reason (Concept 24).

---

## Concept 3 — B+ trees: the read-optimized default

Nearly every relational engine's primary structure is a **B+ tree**: a balanced, sorted, high-fanout tree where all the data lives in the leaves and internal nodes carry only keys and pointers.

**Fanout is the whole trick.** With an 8 KB page and, say, 20 bytes per (key, pointer) entry, an internal node holds roughly 400 children. Then:

- 2 levels ≈ 160,000 rows
- 3 levels ≈ 64 million rows
- 4 levels ≈ 25 billion rows

So a point lookup in a very large table costs **3–4 page reads**, and the top levels are essentially always in the buffer pool, so in practice it's often **one** physical read. That is the number to quote: *"a B-tree seek is 3–4 logical reads, one of which might be physical."*

**Properties you should be able to name:**

- **Sorted leaves, linked together.** Range scans (`BETWEEN`, `ORDER BY` on the key, `>`) are sequential once you've found the start. This is why a B-tree index serves both equality and range queries, and why sort order in a composite index matters (Concept 12).
- **In-place update.** A B-tree mutates pages where they sit. That's good for read locality and bad for write amplification: every logical write touches the WAL *and*, eventually, a full page write.
- **Page splits and merges** keep it balanced; splits are the write-path cost and the fragmentation source.
- **Write amplification of roughly 2×+**: the WAL record plus the page. Postgres additionally writes **full-page images** to WAL after a checkpoint to protect against torn pages, which is why write volume spikes right after checkpoints.

**When a B-tree hurts:** very high write rates with random key distribution (splits everywhere), or workloads where the index is much larger than RAM and every insert touches a cold page.

---

## Concept 4 — LSM trees: the write-optimized alternative

A **Log-Structured Merge tree** takes the opposite position: make writes purely sequential, and pay for it on reads and in background work.

**The write path:**

1. Write to a WAL (durability).
2. Insert into an in-memory sorted structure — the **memtable**.
3. When the memtable fills, flush it to disk as an immutable, sorted **SSTable**.
4. Background **compaction** merges SSTables, discarding superseded versions and tombstones.

**The read path:** check the memtable, then SSTables newest-to-oldest, until you find the key. Two mechanisms keep that from being catastrophic:
- **Bloom filters** per SSTable answer "definitely not here" in one memory probe, so most files are skipped.
- **Sparse block indexes** and per-file min/max key ranges narrow the search further.

**The three amplifications** (the vocabulary that scores):
- **Write amplification** — one logical write becomes several physical writes as data is rewritten by compaction. Leveled compaction: high write amplification, low space and read amplification. Size-tiered: the reverse.
- **Read amplification** — a point read may touch several files; a range scan must merge across levels.
- **Space amplification** — old versions and tombstones occupy space until compacted.

**The operational signature to recognize:** LSM stores have excellent average write throughput and *lumpy p99s*, because compaction competes for I/O with the foreground workload. "Our writes are fast but p99 latency spikes every few hours" is a compaction story. Also: **delete is a write** (a tombstone), so heavy-delete workloads on LSM stores can grow rather than shrink until compaction catches up.

**Who uses it:** RocksDB and LevelDB (the engines under many systems), Cassandra, ScyllaDB, HBase, and — worth knowing for the .NET ecosystem — **CockroachDB's Pebble** and the storage layer of many distributed SQL engines. SQL Server's in-memory and columnstore delta structures borrow related ideas.

---

## Concept 5 — B-tree vs LSM, and the RUM conjecture

The clean framing is the **RUM conjecture** (Athanassoulis et al., 2016): you can optimize for any two of **R**ead overhead, **U**pdate overhead, and **M**emory (space) overhead, at the expense of the third. A B-tree optimizes read and space; an LSM optimizes update and space (with compaction tuning shifting the trade); an index-everything approach optimizes read at the cost of update and space.

| | B+ tree | LSM tree |
|---|---|---|
| Write path | In-place update, random page writes | Sequential append + background merge |
| Write throughput | Good | **Excellent** |
| Point read | **Excellent** (3–4 reads, sorted) | Good (bloom filters, but multiple files) |
| Range scan | **Excellent** (linked leaves) | Fair (merge across levels) |
| Space | Fragmentation from splits, fill factor | Old versions until compaction; compresses well |
| Latency shape | Predictable | Spiky under compaction |
| Delete | In-place | Tombstone (a write) |
| Natural fit | OLTP, mixed read/write, strong secondary indexing | Write-heavy ingestion, time-series, key-value at scale |

**The senior move in an interview** isn't reciting this table — it's using it diagnostically: *"You're describing 200k writes/second of append-mostly device telemetry with time-range reads. That's an LSM shape, and I'd expect compaction to be the operational risk we tune for, not the write path."*

---

## Concept 6 — The write-ahead log, durability, and group commit

**The WAL rule:** before a page change is written to the data files, the log record describing it must be durably on disk. This is what makes crash recovery possible and what "D" in ACID actually means.

**Commit means "the log is fsynced," not "the data is written."** Data pages are written lazily by checkpoints and background writers. Recovery replays the log: redo committed work, undo uncommitted work (the classic **ARIES** algorithm — analysis, redo, undo — worth naming).

Four mechanisms to have ready:

**1. Group commit.** `fsync` costs are dominated by device round-trips, so engines batch concurrent commits into one flush. This is why throughput can rise with concurrency even when individual commit latency doesn't improve — and why a benchmark with one connection tells you almost nothing about commit capacity.

**2. Durability knobs are the classic silent-data-loss setting.** `synchronous_commit = off` in Postgres, delayed durability in SQL Server, `innodb_flush_log_at_trx_commit = 2` in MySQL — each trades a window of recent committed transactions for throughput. Exactly parallel to Kafka's `acks=1` from Module 11. Say the window out loud: *"we'd lose up to N milliseconds of committed transactions on a host failure."*

**3. Checkpoints bound recovery time.** More frequent checkpoints mean shorter recovery and more steady-state I/O. This is a real RTO lever (Concept 54), and in Postgres the full-page-write behaviour after a checkpoint is a common cause of periodic write spikes.

**4. Durability in a cluster is replication, not fsync.** A single-node fsync doesn't survive the node. Synchronous replication (SQL Server Always On synchronous-commit, Postgres `synchronous_standby_names`, Azure SQL Business Critical) means the commit waits for a replica — trading Module 7's PACELC "else latency" for a smaller RPO. Azure SQL Hyperscale is the interesting case: the **log service** is the durability boundary, and page servers catch up asynchronously.

**The interview-grade sentence:** *"Durability is a property of the log, not the data files. A commit returns when the log record is durable on enough independent failure domains — which is why the real durability question is 'how many replicas acknowledged,' not 'was it written to disk.'"*

---

## Concept 7 — The buffer pool and the working set

The **buffer pool** (Postgres: shared buffers plus the OS page cache; SQL Server: the buffer pool) caches pages in RAM. Reads hit memory when the page is resident and storage when it isn't. Writes are made in memory and to the log; dirty pages are flushed later.

**The only number that matters is working set vs RAM.** Your working set is the set of pages actually touched by your workload in a given window — usually a small fraction of total data, because access is skewed (Module 10, Concept on hot keys). A 2 TB database with a 40 GB working set runs beautifully on a 64 GB machine. The same database with a full-table-scan report running hourly does not.

**The cliff.** Buffer-pool hit ratio doesn't degrade linearly. It sits at 99.9% and then falls off as the working set outgrows RAM, and query latency jumps by the ratio of storage latency to memory latency — roughly three orders of magnitude for the pages that now miss. This is the single most common explanation for "the database got slow and nobody deployed anything."

**What to say in an interview:** *"Before I add a cache or a read replica I'd check whether the working set still fits in the buffer pool, because if it doesn't, both of those are treating a symptom. The fix might be a covering index that reduces the pages touched, or moving a cold column out of a hot table."* That answer demonstrates you understand the mechanism rather than the playbook.

**Related:** Postgres's double caching (shared buffers plus OS page cache) is why the classic guidance is ~25% of RAM for shared buffers rather than 80%; SQL Server manages its own pool and expects the majority of RAM. Knowing that distinction is a nice, concrete tell.

---

## Concept 8 — Row stores, column stores, and the OLTP/OLAP split

**Row store:** all columns of a row stored contiguously. Reading one whole entity is one page read. Ideal when you fetch entities by key and write them whole — i.e., OLTP.

**Column store:** each column stored separately, in compressed, encoded segments. Reading three columns over 500M rows touches only those columns' segments, which compress 5–10× because a column is homogeneous (run-length, dictionary, bit-packing). Combined with vectorized/batch-mode execution and segment elimination via min/max metadata, aggregates run one to two orders of magnitude faster than the same query against a row store — and correspondingly, single-row lookups and updates are poor.

| | Row store | Column store |
|---|---|---|
| Unit of locality | The row | The column segment |
| Great at | Point lookup, insert/update/delete of whole entities | Scans and aggregates over few columns, many rows |
| Compression | Modest | **5–10×**, because values in a column are homogeneous |
| Typical use | OLTP | OLAP, reporting, analytics |
| In the Microsoft stack | Clustered/nonclustered B-tree indexes | Clustered & nonclustered **columnstore** indexes, Fabric/Synapse, Parquet in the lake |

**The HTAP middle ground** is worth naming: a **nonclustered columnstore index on an OLTP table** lets you run analytic queries against live data without a second system, at the cost of write overhead and a delta-store/tuple-mover to manage. SQL Server and Azure SQL support this; TiDB's TiFlash is the distributed version; Cosmos DB's **analytical store** plus Fabric mirroring is the Azure NoSQL version.

**The senior position:** don't run serious analytics on the transactional store by default. State the reason in resource terms — *analytics is scan-heavy and will evict the OLTP working set from the buffer pool, which is why the "one report" slows down everything else* — and then propose the boundary: replica, columnstore index, or CDC into a lake (Concept 39).

---

## Concept 9 — The arithmetic you should have reflexive

Five calculations. Do them out loud; they're the cheapest credibility you can buy in a design round.

**1. Table size.** `rows × average row size × (1 + overhead)`. Use ~1.2–1.5× for page overhead and fill factor.
> 500M orders × 300 bytes ≈ 150 GB of data, call it ~200 GB with overhead. Add indexes.

**2. Index size.** A nonclustered index costs roughly `(key columns + clustered key + INCLUDE columns) × rows`, plus overhead.
> An index on `(CustomerId, OrderDate)` over 500M rows with a `bigint` clustered key: (8 + 8 + 8) ≈ 24 bytes → ~12 GB, ~15 GB with overhead. Now say it: *"five such indexes cost more than the table."*

**3. Working set.** Estimate the fraction of rows touched in a typical hour, times row size, plus the index pages for those rows. Compare against RAM. This is the single most decision-relevant number in a data design.

**4. IOPS and throughput.** `writes/sec × pages dirtied per write` for the write path; for reads, `queries/sec × logical reads × (1 − buffer hit ratio)`. Azure managed disks and Azure SQL tiers have published IOPS ceilings; hitting them looks exactly like a slow application.

**5. Cosmos DB RU math.** 1 RU ≈ a point read of a 1 KB item. A 1 KB write ≈ 5 RU with default indexing. So:
> 2,000 writes/s × 5 RU + 10,000 reads/s × 1 RU ≈ 20,000 RU/s. At ~10,000 RU/s per physical partition, that's a minimum of 2 physical partitions on throughput alone — and a reminder that a single logical partition is capped at 10,000 RU/s, so a hot key can't be fixed by buying more RU/s.

**Two more numbers worth memorizing:**
- A single well-tuned SQL Server or Postgres node on modern hardware handles **thousands to tens of thousands** of simple OLTP transactions per second, with data volumes in the low TB, comfortably. That's the yardstick for "do we actually need to shard?"
- A cross-region synchronous commit costs **at least one round trip** — 80–150 ms EU↔US. That number alone kills most globally-synchronous designs, and being the person who says it early is the point.

---

# Part B — Indexing

## Concept 10 — What an index actually is, and the three questions

An index is **a second copy of a subset of your data, sorted by a different key, maintained transactionally by the engine.** That definition is worth internalizing because it makes the costs obvious: it consumes space, it must be updated inside every write transaction, it adds lock surface, and — in a distributed store — it may be *asynchronously* maintained and therefore stale (Concept 18).

When you evaluate whether an index helps a query, ask three questions in order:

1. **Can it seek?** Is there a prefix of the index key that the query's predicates constrain, so the engine can navigate the B-tree to a starting point instead of scanning? A predicate that can be used this way is called **sargable** (Search ARGument-able).
2. **Can it cover?** Does the index contain every column the query needs, so no lookup back to the table is required?
3. **Can it deliver order?** Does the index's sort order match the query's `ORDER BY`/`GROUP BY`, so the sort operator disappears?

Almost every index design conversation is these three questions. A senior answer names which one an index is buying.

**Sargability, concretely.** These prevent a seek:
- Wrapping the column in a function: `WHERE YEAR(OrderDate) = 2026` → use `WHERE OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01'`.
- Leading wildcards: `WHERE Name LIKE '%smith'` (a trailing wildcard is fine).
- Implicit conversions: comparing an `nvarchar` parameter to a `varchar` column — a classic production incident, because the plan silently becomes a scan.
- `OR` across different columns, which often forces a scan or index union.
- Arithmetic on the column: `WHERE Price * 1.2 > 100`.

**Computed/generated columns + an index on them** are the escape hatch when the expression is unavoidable. PostgreSQL supports expression indexes directly (`CREATE INDEX ON orders (lower(email))`), which is cleaner.

---

## Concept 11 — Clustered vs nonclustered, and why key choice is a physical decision

**Clustered index:** the table *is* the index. Leaf level = the data rows, stored in key order. One per table. The clustered key is therefore the physical ordering *and* the row's logical address.

**Nonclustered index:** a separate structure whose leaves contain the index key plus a pointer back to the row — in SQL Server, the clustered key (or the RID for a heap). So every nonclustered index silently carries the clustered key in every leaf row.

Three consequences that come up constantly:

**1. A wide clustered key inflates every nonclustered index.** Clustering on a `uniqueidentifier` (16 bytes) rather than a `bigint` (8 bytes) adds 8 bytes to every row of every nonclustered index. Over 500M rows and five indexes, that's ~20 GB of pure overhead — memorize that shape of calculation.

**2. A random clustered key causes page splits and fragmentation.** Inserting `NEWID()` values lands rows at random positions in the ordered leaf level, splitting full pages, fragmenting the index, and destroying the sequential-insert locality that keeps the tail of the index in cache. This is *the* classic .NET data-modelling mistake, because `Guid.NewGuid()` is the path of least resistance for a distributed ID.

**The fixes, in order of preference:**
- **`bigint` identity/sequence** where you don't need client-side ID generation.
- **UUIDv7** (time-ordered UUIDs) — now first-class: `Guid.CreateVersion7()` exists in .NET 9+, and **PostgreSQL 18 added a native `uuidv7()` function** precisely because random UUIDs are hostile to B-tree inserts.
- **`NEWSEQUENTIALID()`** in SQL Server for server-generated sequential GUIDs.
- **Keep the random GUID as the logical key but cluster on something sequential** — e.g. cluster on an `identity` column, put a unique nonclustered index on the GUID. This is the pragmatic answer when the GUID is part of the public contract.

**3. The key lookup tax.** A nonclustered index seek that finds 50,000 rows and must look each one up in the clustered index performs 50,000 random reads. The optimizer knows this, which is why it will often *ignore* your index and scan instead — the "tipping point," typically somewhere around 1–5% of the table depending on row size. If your index is being ignored, the answer is usually to make it covering (Concept 13), not to add a hint.

**PostgreSQL differs in an important way:** its tables are heaps and *all* indexes are secondary, pointing at a physical tuple ID (`ctid`). There's no clustered index (the `CLUSTER` command is a one-time physical reorder). The corresponding optimization is the **index-only scan**, which requires the visibility map to show the page as all-visible — which is why vacuum affects read performance in Postgres (Concept 23).

---

## Concept 12 — Composite indexes: column order, prefixes, and selectivity

A multi-column index is sorted by the first column, then the second within equal firsts, and so on — like a phone book sorted by (last name, first name). That single mental model answers most composite-index questions.

**The leftmost-prefix rule.** An index on `(A, B, C)` can seek on `A`, `(A, B)`, and `(A, B, C)`. It cannot seek on `B` alone or `(B, C)` — those require a scan of the whole index at best. So an index on `(A, B, C)` makes a separate index on `(A)` redundant, and a separate index on `(A, B)` redundant. Consolidating redundant indexes is one of the highest-value, lowest-risk database optimizations, and a good thing to have a story about.

*Nuance worth knowing:* **PostgreSQL 18 added B-tree "skip scan"**, which lets the planner use a multicolumn index when the leading column has low cardinality by skipping through its distinct values — so the leftmost-prefix rule is softening in Postgres for that specific case. SQL Server has long had a related trick in the form of index skip scans for some plans, but the rule remains the right default assumption.

**The ordering rule:** **equality columns first, then the range column, then the columns you sort by.** For:
```sql
SELECT OrderId, Total FROM Orders
WHERE CustomerId = @c AND Status = 'Open' AND OrderDate >= @d
ORDER BY OrderDate DESC;
```
the index is `(CustomerId, Status, OrderDate DESC) INCLUDE (Total)`. Equality on the first two narrows to a contiguous range; the range predicate on the third is satisfied by walking that range; the `ORDER BY` is free because the index is already in that order. Putting `OrderDate` before `Status` breaks all three properties at once.

**Selectivity, defined properly.** Selectivity is the fraction of rows a predicate keeps; high selectivity = few rows. Put the more selective columns first *among the equality columns*, but the equality/range/order rule dominates: a range column placed before an equality column stops the equality column from narrowing anything.

**One composite beats three singles.** Three single-column indexes require the engine either to pick one (and filter the rest) or to do an index intersection — both worse than a single well-ordered composite. The "I added an index per column in the WHERE clause" pattern is a mid-level tell.

---

## Concept 13 — Covering indexes and INCLUDE

A **covering index** contains every column a query references, so the engine never touches the base table. This turns a seek + N random lookups into a single range scan — routinely a 10–100× improvement, and the most reliably impressive optimization you can describe in an interview.

**`INCLUDE` (SQL Server) vs widening the key.** Included columns live only in the **leaf** level, not in the internal nodes. So they add payload without widening the key, without affecting sort order, and without the size hit in the upper tree levels. They cannot be seeked on, only returned. Use `INCLUDE` for columns in the `SELECT` list; use key columns for anything in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY`.

PostgreSQL supports the same thing: `CREATE INDEX ... INCLUDE (...)` (covering indexes since PG 11).

**The trade-off to state out loud:** a covering index is a denormalized copy of those columns. Every update to an included column now updates the index too. Covering a 200-byte set of columns over a 500M-row table means a ~100 GB index. The senior framing is: *"I'd cover this query because it runs 5,000 times a second and the columns are rarely updated. I wouldn't cover the one that runs hourly."*

---

## Concept 14 — Statistics, cardinality estimation, and why plans go wrong

The optimizer is a cost model driven by **estimates of how many rows each operator will produce**. Nearly all catastrophic plans are estimation failures, not optimizer bugs. If you can say that sentence and then name the four common causes, you're well past mid-level.

**How it estimates:** histograms over column values (SQL Server: up to 200 steps; Postgres: `default_statistics_target`, 100 buckets by default), plus density/distinct-count information.

**Where it goes wrong:**

1. **Stale statistics.** SQL Server's auto-update threshold is proportional to table size, so very large tables can go a long time without an update; a table that grows steadily can have statistics that say "10,000 rows" when it has 10 million. Ascending-key columns (`OrderDate`) are the classic case: the newest range falls outside the histogram, and the estimate is 1 row.
2. **Correlated predicates.** `WHERE City = 'Paris' AND Country = 'France'` — the engine multiplies the two selectivities as if independent, and underestimates by orders of magnitude. Fixes: multi-column statistics (SQL Server), `CREATE STATISTICS ... (dependencies, ndistinct)` (Postgres extended statistics).
3. **Skew.** A `TenantId` column where one tenant has 60% of rows. The average estimate is right and every individual plan is wrong. This connects straight to Module 8's hot-partition problem — skew is the same enemy at a different layer.
4. **Opaque expressions.** Local variables, scalar UDFs, table variables (historically estimated at 1 row), and multi-statement TVFs all defeat estimation.

**The diagnostic move:** compare **estimated vs actual rows** in an actual execution plan. A 1000× discrepancy at some operator is the root cause; everything downstream (a nested loop where a hash join belonged, a memory grant spill to tempdb) is a symptom. Saying *"I'd look for the operator where estimated and actual diverge, because everything after that point is the optimizer making good decisions from bad information"* is a strong, specific answer.

---

## Concept 15 — Parameter sniffing and plan stability

The engine compiles a plan using the parameter values from the *first* execution and caches it for reuse. If the data is skewed, the cached plan can be catastrophically wrong for other parameter values: a plan optimized for `@TenantId = <tiny tenant>` (nested loops + seeks) applied to `<huge tenant>` will do 10 million lookups.

The signature: *"the same query is fast for most users and terrible for one, and restarting the service or clearing the cache temporarily fixes it."* Recognizing that description instantly is a genuine senior signal.

**The remedies, with their costs:**

| Remedy | What it does | Cost |
|---|---|---|
| `OPTION (RECOMPILE)` | Fresh plan every execution | CPU per execution; fine for infrequent, expensive queries |
| `OPTIMIZE FOR (@p = value)` | Pin the plan to a representative value | You must know the representative value, forever |
| `OPTIMIZE FOR UNKNOWN` | Use average density instead of the sniffed value | Mediocre for everyone; sometimes exactly right |
| Split the query | Separate code paths for the big/small cases | Application complexity, but the most honest fix |
| Query Store plan forcing | Force a known-good plan | Operational, reversible, auditable — the modern default |

**Modern SQL Server / Azure SQL adds automatic help** worth naming: Query Store, **automatic plan correction** (detects regressions and reverts), and Intelligent Query Processing features such as **Parameter Sensitive Plan optimization** (multiple cached plans for different parameter buckets) and adaptive joins/memory-grant feedback. Knowing these exist — and that the modern answer is "let Query Store manage it, and investigate why the data is skewed" — reads as current.

PostgreSQL's analogue: prepared statements switch to a **generic plan** after five executions (`plan_cache_mode` controls it), producing the same class of problem.

---

## Concept 16 — Specialized index types

Beyond the B-tree, know what exists and the one question each answers.

| Index type | Answers | Notes |
|---|---|---|
| **Filtered / partial** | "Index only the rows I query" | `WHERE IsActive = 1`, or `WHERE DeletedAt IS NULL` — far smaller, cheaper to maintain. The correct answer to soft-delete bloat. Postgres: partial indexes |
| **Unique** | "Enforce an invariant" | An index *is* the enforcement mechanism for uniqueness — this is the substrate for idempotency keys (Module 11, Concept 9) |
| **Hash** | "Equality only, fastest" | No ranges, no ordering. Postgres hash indexes; SQL Server memory-optimized hash indexes; the bucket-count sensitivity matters |
| **Columnstore** | "Aggregate over millions of rows" | Clustered (the table is columnar) or nonclustered (analytics alongside OLTP); batch-mode execution; segment elimination |
| **GIN / GiST (Postgres)** | "Index inside a composite value" | GIN for JSONB, arrays, full-text; GiST for ranges, geometry, nearest-neighbour |
| **Full-text** | "Search words, not values" | Inverted index; ranking, stemming, stop words. SQL Server FTS, Postgres `tsvector`, or an external engine (Azure AI Search, Elasticsearch) |
| **Spatial** | "Near / within" | R-tree or grid decomposition; SQL Server spatial indexes, PostGIS |
| **Vector** | "Nearest neighbour in embedding space" | HNSW (graph), IVF (clustered), **DiskANN** (disk-resident). Approximate: you trade recall for latency |
| **Bitmap** | "Low-cardinality columns, analytic ANDs" | Oracle/analytics engines; not a SQL Server/Postgres OLTP tool |

**Vector search is now first-class in the Microsoft stack, and it's worth being current here.** SQL Server 2025 (GA November 2025) and Azure SQL ship a native `VECTOR` data type stored in an optimized binary format but exposed as JSON arrays, with `VECTOR_DISTANCE` supporting cosine, Euclidean and dot-product metrics, plus approximate **DiskANN-based vector indexes**. SQL Server 2025 also supports half-precision (`float16`) vectors to halve storage. **EF Core 10 has full support for the vector type and `VECTOR_DISTANCE()`**, and Cosmos DB's vector search support in EF Core 10 is no longer experimental. In Postgres, the equivalent is `pgvector` with HNSW or IVFFlat indexes.

**The judgment layer, which matters more than the feature list:** a dedicated vector database earns its place when vector search *is* the product; an in-database vector index earns its place when embeddings are an attribute of records you already store relationally and you want filters, joins, and transactions over them without a second system to keep in sync. Say *that*, and the feature knowledge lands as judgment rather than trivia.

---

## Concept 17 — The write cost of indexes, fragmentation, and maintenance

**Every index is a tax on every write.** An insert into a table with six indexes is seven B-tree insertions, seven sets of log records, seven sets of latches and locks, and potentially seven page splits. An update touches only the indexes containing the changed columns — which is a specific argument for `INCLUDE`-ing rarely-updated columns and not hot ones.

**Numbers to have:** a well-indexed OLTP table often has indexes totalling **1–3× the size of the table's data**. Write throughput degrades roughly linearly with index count. "We have 14 indexes on this table" is a finding, not a detail.

**Finding the waste.** Unused and duplicate indexes are pure cost. SQL Server: `sys.dm_db_index_usage_stats` (user seeks/scans/lookups vs updates — an index with 0 reads and 4M updates is a liability). Postgres: `pg_stat_user_indexes.idx_scan`. Mentioning these DMVs by name signals you've actually done this.

**Fragmentation** — logical order diverging from physical order, plus low page density — hurts scans and wastes buffer pool. It matters much less on SSDs than the folklore suggests; **low page density (wasted space in the buffer pool) is the effect that still matters**. Reorganize (online, incremental) vs rebuild (rebuilds the structure, updates statistics; `ONLINE = ON` in Enterprise/Azure SQL, `REINDEX CONCURRENTLY` in Postgres).

**The maintenance point an architect makes:** index maintenance on a large table generates enormous log volume, which affects replication lag, backup size, and geo-replication bandwidth. "We rebuild indexes nightly" is a decision with a blast radius, not a checkbox.

**Adding an index to a live large table** is itself a migration: `CREATE INDEX ... WITH (ONLINE = ON)` on SQL Server, `CREATE INDEX CONCURRENTLY` on Postgres (which takes longer, can't run in a transaction, and can leave an invalid index if it fails). Knowing that `CREATE INDEX` without `CONCURRENTLY` takes a lock that blocks writes for the duration is the kind of specific operational knowledge interviewers probe for.

---

## Concept 18 — Local vs global secondary indexes in a partitioned store

This is the concept that connects indexing to Module 8, and it's frequently the difference between a mid and a senior answer about NoSQL.

Once data is partitioned by key `K`, an index on a different attribute `A` has exactly two possible implementations:

**Local (partition-local) index.** Each partition indexes its own rows by `A`. Writes are cheap and atomic — the index entry lives in the same partition as the row. Reads by `A` alone must **scatter-gather** across every partition, so query cost grows with partition count, and tail latency becomes the slowest partition's latency (Module 6's fan-out problem).

**Global index.** A separate structure partitioned by `A`, spanning the whole dataset. Reads by `A` hit one partition. But a write now touches two partitions — so it is either a distributed transaction (expensive, Part E) or **asynchronous and eventually consistent** (the near-universal choice).

| | Local | Global |
|---|---|---|
| Write cost | Cheap, atomic with the row | Cross-partition; usually async |
| Consistency | Strongly consistent with the row | **Eventually consistent** |
| Read by indexed attribute | Scatter-gather across N partitions | Single-partition |
| Examples | DynamoDB LSI, Cassandra secondary indexes, Cosmos DB's per-partition index | DynamoDB GSI, Cosmos DB **global secondary indexes (preview)** |

**Cosmos DB, concretely.** Cosmos automatically indexes every property within a partition (an indexing policy controls this), so a query filtered by partition key is cheap and a query without it is a cross-partition fan-out billed accordingly. Cosmos's global-secondary-index feature — the evolution of the former materialized views feature, currently in **public preview** — creates a read-only container automatically synchronized from the source with a *different partition key*, and is explicitly **eventually consistent with the source regardless of the account's consistency level**. Microsoft exposes a *catchup gap in minutes* metric for it, which is the staleness bound you should ask about by name.

**The reframing that scores:** *"A global secondary index is a replica with a different partition key, maintained asynchronously — so it's Module 10's cache with the engine doing the invalidation. Which means I have to ask the same question I'd ask of any cache: what's the staleness bound, and is any invariant depending on it?"*

---

## Concept 19 — Diagnosing a slow query: the narrative

Interviewers ask "a query got slow, what do you do?" to see whether you have a method or a bag of tricks. Use this order, out loud:

1. **Establish the measurement.** Slow for whom, since when, and by which metric — p50 or p99? Is the query slow, or is the *system* slow and this query is the visible victim? (Waiting on a lock, on I/O, on CPU, on a memory grant.)
2. **Find the actual plan and the actual waits.** SQL Server: Query Store, `sys.dm_exec_query_stats`, wait statistics. Postgres: `pg_stat_statements`, `EXPLAIN (ANALYZE, BUFFERS)`. The `BUFFERS` output tells you whether you're I/O-bound or cache-resident — do not skip it.
3. **Compare estimated vs actual rows** (Concept 14). Find the first operator where they diverge.
4. **Name the physical cause.** Scan where a seek was possible; key lookups past the tipping point; a sort spilling to tempdb; a nested loop over a large outer input; a hash spill; lock waits; a non-sargable predicate; an implicit conversion; parameter sniffing.
5. **Fix at the cheapest level that works.** Rewrite the predicate → update statistics → add or widen an index → change the plan (Query Store forcing) → change the schema → change the architecture. In that order. Reaching for "let's add a cache" or "let's shard" at step 1 is the anti-signal.
6. **Verify and guard.** Re-measure, and add the regression test or Query Store baseline that will tell you if it comes back.

**The line that sells it:** *"I'd want to know whether this is a plan problem, a data-volume problem, or a concurrency problem, because those have three different fixes and the plan tells me which one I have."*

---

# Part C — Transactions, isolation, and concurrency

## Concept 20 — ACID, stated precisely

Most candidates can expand the acronym. Few can say what each letter is actually *for*, which is what gets scored.

**A — Atomicity.** All-or-nothing. Note what it is *not*: it is not about concurrency. Atomicity exists so that a failure part-way through leaves no partial effects — it's an **abortability** guarantee. The mechanism is the undo log.

**C — Consistency.** The odd one out, and the one to have an opinion about. In ACID, "consistency" means *your* application invariants hold (credits equal debits, an order always has a customer). The database can enforce *some* of them — constraints, foreign keys, uniqueness — but most are the application's responsibility. As Kleppmann notes, C was arguably added to make the acronym work. Saying that, briefly, is a good signal — and it's a useful bridge to the point that **ACID's C and CAP's C are entirely different things** (Module 7).

**I — Isolation.** The interesting letter. Textbook isolation means serializability: concurrent transactions produce a result equivalent to *some* serial order. In practice virtually nobody runs serializable by default, so isolation is a **spectrum with named anomalies** (Concepts 21–22).

**D — Durability.** Committed data survives a crash. Single-node durability = the WAL is fsynced. Distributed durability = enough replicas acknowledged (Concept 6). "Durable" without saying "on how many failure domains" is an incomplete answer.

**The senior framing:** *"Atomicity and durability are about failure. Isolation is about concurrency. Consistency is about my domain. Most production incidents I've seen in this area are isolation problems, because that's the letter people quietly weaken for performance."*

---

## Concept 21 — The anomaly catalogue

Learn the *anomalies*, and the isolation levels fall out of them rather than the other way around. Each of these has a two-transaction story; be able to tell it in three lines.

| Anomaly | The story | Prevented by |
|---|---|---|
| **Dirty write (P0)** | T2 overwrites a value T1 has written but not committed | Every level (write locks held to commit) |
| **Dirty read (P1)** | T2 reads a value T1 wrote but hasn't committed; T1 then rolls back | Read Committed and above |
| **Non-repeatable read (P2)** | T1 reads a row twice and gets different values because T2 committed in between | Repeatable Read / Snapshot and above |
| **Phantom read (P3)** | T1 runs the same range query twice and gets *new rows* because T2 inserted into the range | Snapshot (for reads) / Serializable |
| **Lost update** | T1 and T2 both read `balance = 100`, both write `balance − 10`; one update vanishes | Explicit locking (`SELECT ... FOR UPDATE`/`UPDLOCK`), atomic write (`SET balance = balance - 10`), compare-and-set, or Serializable |
| **Read skew (A5A)** | T1 reads account A before T2's transfer and account B after it; the total looks wrong | Snapshot isolation and above |
| **Write skew (A5B)** | T1 and T2 each read a set, each verify a constraint still holds, each write a *different* row, and together they break the invariant | **Serializable only** |

**Write skew deserves its own paragraph**, because it's the one that catches good engineers and the one interviewers use to separate "knows the levels" from "understands them."

> Two doctors are on call. The rule: at least one must remain on call. Alice and Bob each open a transaction, each query "how many doctors are on call?" (answer: 2), each conclude it's safe to go off call, each update *their own row*. Both commit. Zero doctors are on call. No dirty read, no lost update, no phantom read on the row each wrote — and **snapshot isolation does not prevent it**, because each transaction wrote a different row and neither write conflicts.

Real-world versions: double-booking the last seat, two concurrent withdrawals that each pass an overdraft check, allocating the same inventory unit twice, two "primary" flags being set. If a design has the shape *read a set → check an invariant → write something in that set*, write skew is live and you should name it.

**The four fixes:** (1) Serializable isolation; (2) materialize the conflict — introduce a row that both transactions must lock (the shift row, the inventory row); (3) explicit locking of the read set (`SELECT ... FOR UPDATE`); (4) a uniqueness/check constraint that makes the invariant the engine's problem. Being able to offer four options with trade-offs is a strong answer.

---

## Concept 22 — The isolation ladder, and the ANSI critique

| Level | Dirty read | Non-repeatable read | Phantom | Write skew | Typical mechanism |
|---|---|---|---|---|---|
| Read Uncommitted | possible | possible | possible | possible | No read locks |
| **Read Committed** | prevented | possible | possible | possible | Short read locks, or read latest committed version (RCSI/Postgres) |
| Repeatable Read | prevented | prevented | possible* | possible | Read locks held to commit |
| **Snapshot** | prevented | prevented | prevented | **possible** | MVCC: read a consistent point-in-time snapshot |
| **Serializable** | prevented | prevented | prevented | prevented | 2PL + range/predicate locks, or SSI |

\* Postgres's Repeatable Read is implemented as snapshot isolation, so it *does* prevent phantoms. This kind of per-engine divergence is exactly why the ANSI table is a starting point, not a contract.

**Defaults you should know cold, because interviewers ask:**
- **SQL Server (on-premises, default):** Read Committed using *locking* — readers block on writers. This surprises people coming from Postgres, and it's the reason so many SQL Server codebases are peppered with `NOLOCK`.
- **Azure SQL Database:** **Read Committed Snapshot Isolation (RCSI) is on by default** — readers don't block writers. Enabling RCSI on an on-premises database is one of the highest-leverage configuration changes available, at the cost of version-store space in tempdb.
- **PostgreSQL:** Read Committed, implemented with MVCC; readers never block writers.
- **Oracle:** Read Committed via MVCC; no dirty reads possible at all.
- **Cosmos DB:** a completely different axis — five *distributed* consistency levels (Module 7), with transactional scope limited to a logical partition.

**The ANSI critique.** The 1995 paper *A Critique of ANSI SQL Isolation Levels* (Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil) is the primary source and worth knowing by name. Its argument: the ANSI definitions are phrased in terms of a few specific phenomena, are ambiguous, and don't characterize real implementations — in particular, snapshot isolation, which was already widespread, doesn't fit the ladder at all. It sits "between" Repeatable Read and Serializable and permits write skew. Citing that paper when explaining why "snapshot is not serializable" is a genuinely senior move.

**`NOLOCK` deserves a direct answer**, because it shows up constantly in .NET codebases. `WITH (NOLOCK)` = Read Uncommitted. It permits dirty reads and, less well known, **can return the same row twice or skip rows entirely** during page splits and allocation-order scans. The correct advice: use RCSI instead; reserve `NOLOCK` for genuinely approximate diagnostics. Being able to say "it doesn't just risk uncommitted data, it risks missing and duplicated rows" is a small detail that reliably registers.

---

## Concept 23 — MVCC: how snapshot isolation is actually built

**Multi-Version Concurrency Control**: a write creates a *new version* of a row rather than overwriting it, and each transaction reads the versions visible as of its snapshot. The headline property — **readers don't block writers and writers don't block readers** — is the single biggest concurrency win available in an OLTP system. The cost is storing and cleaning up old versions.

**PostgreSQL.** Versions live in the table itself. Each tuple carries `xmin` (creating transaction) and `xmax` (deleting/superseding transaction); visibility is computed from the snapshot's transaction IDs. Consequences to know:
- **Dead tuples accumulate** and must be reclaimed by **`VACUUM`** (usually autovacuum). If vacuum can't keep up — typically because of long-running transactions holding back the visibility horizon — you get **table and index bloat**: disk usage and buffer-pool waste growing without the row count growing.
- **`VACUUM` also maintains the visibility map**, which is what enables **index-only scans**. So vacuum is a read-performance feature, not just housekeeping.
- **Transaction ID wraparound** is the doomsday scenario; modern versions handle it well but it's a real historical outage class and knowing the term reads as operational experience.
- **HOT (heap-only tuple) updates** avoid touching indexes when no indexed column changed and the new version fits on the same page — which is why `fillfactor` on update-heavy tables matters in Postgres.

**SQL Server.** Versions go to a **version store**, historically in `tempdb`, and with **Accelerated Database Recovery (ADR)** into the in-database **Persistent Version Store (PVS)**. Consequences:
- Under RCSI/Snapshot, every row gets a 14-byte versioning tag, and tempdb sizing becomes a real capacity concern.
- **A long-running transaction prevents version cleanup** — the same failure mode as Postgres bloat, wearing a different costume.
- ADR additionally makes rollback of huge transactions near-instant and shortens crash recovery, because undo works from the version store rather than by scanning the log.

**RCSI vs SNAPSHOT in SQL Server** is a distinction worth having crisp, because it's a common probe:
- **RCSI** (`READ_COMMITTED_SNAPSHOT ON`): each *statement* sees a snapshot as of the statement's start. No code changes; the default Read Committed level simply stops blocking.
- **SNAPSHOT** (`ALLOW_SNAPSHOT_ISOLATION ON` + `SET TRANSACTION ISOLATION LEVEL SNAPSHOT`): the whole *transaction* sees one snapshot as of its start, and update conflicts raise error 3960, which your code must handle and retry.

---

## Concept 24 — Locking: 2PL, granularity, escalation, and optimized locking

**Two-Phase Locking (2PL)** — not to be confused with two-phase *commit* (Concept 41), and confusing them is a classic slip. 2PL: a transaction acquires locks in a growing phase and releases them only in a shrinking phase (in practice, at commit — "strict 2PL"). Shared locks for reads, exclusive for writes. Held-to-commit locking is how classic serializability is implemented.

**Granularity and intent locks.** Locks exist at row, page, and table level. To make "is anything in this table locked?" cheap, engines take **intent locks** (IS, IX, SIX) on the parent — so a table-level lock request can be rejected without scanning every row lock. Knowing why intent locks exist (to make hierarchical lock checks O(1)) is a nice depth signal.

**Lock escalation.** Row locks cost memory. SQL Server escalates to a table lock at roughly **5,000 locks** on a single object in a single statement, which converts a "big update" into a blocking event for the whole table. The classic mitigation is **batching**: update in chunks of a few thousand rows with a commit between, which also keeps the log and the version store bounded. That batching pattern is worth having as a ready answer to "how would you update 50 million rows in production?"

**Optimized locking (SQL Server 2025 / Azure SQL) is the current-knowledge item here.** It has two components:
- **TID (transaction ID) locking:** each row is labelled with the last transaction ID that modified it, and a *single* lock on the TID protects all rows the transaction modified. Row and page locks are still taken but released immediately after each row is modified, instead of being held to commit.
- **Lock After Qualification (LAQ):** predicates are evaluated against the latest committed version *without* taking a lock, and the lock is taken only just before a qualifying row is modified. LAQ requires RCSI.

The results worth quoting: far less lock memory, **lock escalation becomes unlikely** because locks are released before the threshold is crossed, and some deadlock classes disappear. It requires **Accelerated Database Recovery** to be enabled first, it's **always on in Azure SQL Database** and opt-in per database in SQL Server 2025 (`ALTER DATABASE ... SET OPTIMIZED_LOCKING = ON`). Mentioning this — and that RCSI is a prerequisite for the full benefit — is a strong "I keep current" signal in any Microsoft-stack interview.

---

## Concept 25 — Deadlocks

A cycle in the waits-for graph. The engine detects it (SQL Server runs a deadlock monitor roughly every 5 seconds; Postgres checks after `deadlock_timeout`, default 1s) and kills a victim, chosen by estimated rollback cost — the loser gets error 1205 in SQL Server, `40P01` in Postgres.

**The four causes, in frequency order:**
1. **Inconsistent access order.** T1 updates Orders then Inventory; T2 does the reverse. The fix is a convention: always touch tables (and rows) in a fixed order.
2. **Lock upgrades.** Read with a shared lock, then update the same row; two transactions doing this deadlock on the upgrade. `UPDLOCK` on the read (or `SELECT ... FOR UPDATE`) takes the stronger lock up front.
3. **Missing indexes.** Without an index, an update scans and locks far more rows than it needs to, so unrelated transactions collide. **A missing index is one of the most common hidden causes of deadlocks**, and saying so is a good, non-obvious answer.
4. **Long transactions.** More time holding locks = more opportunity to intersect.

**Always retry.** Deadlock victims are a normal, expected outcome in a concurrent system — exactly like Module 11's transient message failures. Retry with a small randomized backoff, bounded attempts, and only for the deadlock/serialization error codes. In .NET:
- `SqlException.Number == 1205`, or `PostgresException.SqlState == "40P01"` / `"40001"` (serialization failure).
- **EF Core's `EnableRetryOnFailure()`** (the SQL Server execution strategy) covers transient faults — but note the important caveat: with a **user-initiated transaction**, EF can't retry automatically and throws; you must wrap the work in `strategy.ExecuteAsync(...)` yourself. That detail is a real .NET interview differentiator.
- Polly (Module 25) for the general policy.

**The design-level answer:** the best deadlock fix is usually shorter transactions and a consistent ordering convention, not a smarter retry.

---

## Concept 26 — Optimistic concurrency in .NET and EF Core

This is Module 7's optimistic-concurrency thread, now grounded.

**Pessimistic:** lock the row while you think (`SELECT ... FOR UPDATE` / `WITH (UPDLOCK, HOLDLOCK)`). Correct, simple, and it does not survive a user-think-time gap — nobody should hold a lock across an HTTP round trip.

**Optimistic:** read without locking, and at write time verify nothing changed. This is the right default for anything with a user in the loop.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int StockLevel { get; set; }

    [Timestamp]                       // SQL Server rowversion
    public byte[]? Version { get; set; }
}
```
EF Core issues `UPDATE ... WHERE Id = @id AND Version = @originalVersion`; if zero rows are affected, it throws `DbUpdateConcurrencyException`. On PostgreSQL the idiom is the system column: `modelBuilder.Entity<Product>().UseXminAsConcurrencyToken()`. For a non-`rowversion` approach, `.IsConcurrencyToken()` on a business column (e.g. `LastModifiedUtc`) works and is portable.

**The three resolution strategies — name all three, because "just retry" is the mid-level answer:**
1. **Store wins** — discard the user's change, reload, tell them.
2. **Client wins** — overwrite, having refreshed the original values from the database.
3. **Merge** — reconcile field by field; only fields both parties changed are a true conflict. This is the best UX and the most work, and it's the same idea as Module 9's CRDT reasoning at application level.

**The distinction that scores:** *"Optimistic concurrency prevents lost updates on a single row. It does not prevent write skew, because write skew involves writing a different row from the one you read. If the invariant spans rows, I need either a materialized conflict row, an explicit lock on the read set, or serializable isolation."* That single sentence ties Concepts 21, 22, and 26 together and is exactly the level of synthesis interviewers are listening for.

**Also worth having:** for pure counters and stock decrements, the best answer isn't concurrency control at all — it's an atomic, conditional write: `UPDATE Inventory SET Qty = Qty - @n WHERE Id = @id AND Qty >= @n`, then check rows affected. One statement, no race, no retry loop. Reaching for this before reaching for a lock is a good instinct to display.

---

## Concept 27 — Serializable, SSI, and when to actually use it

Two ways to get real serializability:

**1. Two-phase locking with range locks.** SQL Server's Serializable takes key-range locks to prevent phantoms. Correct, and it blocks — concurrency drops, deadlocks rise.

**2. Serializable Snapshot Isolation (SSI).** Postgres's `SERIALIZABLE` (since 9.1, from the Cahill/Röhm/Fekete work). It runs snapshot isolation but tracks read/write dependencies between concurrent transactions, and aborts a transaction when it detects a dangerous structure that could produce a non-serializable outcome. Optimistic rather than blocking: no extra locks, but **transactions can fail at commit with a serialization error and must be retried**. Your application therefore *must* have a retry loop — that requirement is the practical cost, and naming it is the sign you've used it.

**When serializable is the right answer:** a genuine cross-row invariant (the on-call constraint, the overdraft check, seat allocation, "at most one active X per Y") where materializing the conflict is awkward, and the transaction rate on that path is low enough to absorb aborts. Financial and scheduling logic is the archetype.

**When it isn't:** globally, as a default, "to be safe." The senior answer is scoped: *"I'd run Read Committed / RCSI as the default, and use serializable — or an explicit lock, or a uniqueness constraint — on the specific paths with a cross-row invariant. Blanket serializable buys correctness I mostly don't need at a throughput price I pay everywhere."*

---

## Concept 28 — Isolation vs consistency: the vocabulary trap

This is the trap that catches strong candidates, and getting it right is disproportionately valuable.

- **Serializability** is an **isolation** property: the outcome of concurrent *multi-object transactions* equals *some* serial order. It says nothing about *which* order, or about real time.
- **Linearizability** is a **consistency** (recency) property: operations on a *single object* appear to take effect atomically at some point between invocation and response, consistent with real time. This is CAP's "C" (Module 7).

They're orthogonal. A system can be serializable but not linearizable (a serializable snapshot of an old state is a valid serial order, just a stale one). A system can be linearizable but not serializable (a single-object register with no transactions at all).

**Strict serializability** is both: transaction serializability *plus* real-time ordering. Spanner's "external consistency" is this (Concept 43).

**The sentence to have ready:** *"Serializable is about the ordering of transactions; linearizable is about the recency of a single object. Spanner gives you both and pays for it with commit wait against TrueTime bounds; Postgres SSI gives you serializability on one node; Cosmos DB's Strong level gives you linearizability on a single item, not multi-item serializability."* That one answer tours Modules 7, 9, and 12 simultaneously.

---

## Concept 29 — Long transactions: the self-inflicted outage

If you take one operational lesson from Part C, take this one. Long-running transactions are the most common way a healthy database becomes an incident, and the failure is always multi-symptom:

- **Locks held** for the duration → blocking chains → application timeouts → retries → more load.
- **Version cleanup blocked** → Postgres bloat / SQL Server version-store growth in tempdb → disk pressure and buffer-pool waste.
- **WAL retained** (replication slots, log truncation) → log growth, replication lag, backup growth.
- **Connections held** → pool exhaustion (Concept 51) → the application fails *everywhere*, not just on that path.

**The patterns that cause them:**
1. **Calling out over the network inside a transaction** — an HTTP call, a broker publish, a blob upload. This is the number-one cause, and it's exactly the dual-write anti-pattern of Module 11 in a different costume. Rule: *no network I/O of unbounded latency inside an open transaction.*
2. **Waiting on a human.** Opening a transaction on page load and committing on save.
3. **Unbatched bulk work.** One `UPDATE` over 50M rows.
4. **`await` on something slow while a `DbContext` transaction is open**, including a `SemaphoreSlim` or a lock in your own code.
5. **Idle in transaction** — a code path that opens a transaction and returns early without committing or disposing.

**What to monitor:** longest-running transaction age, oldest snapshot / `xmin` horizon, blocking chain length, version-store size, and `idle in transaction` counts. These are the equivalent of Module 11's "age of the oldest message" — the leading indicator, not the lagging one.

**The sentence:** *"I treat transaction duration as an SLI. A transaction that stays open for seconds isn't slow, it's a availability risk, because it converts one slow operation into blocking, bloat, and pool exhaustion at the same time."*

---

# Part D — Data models: SQL vs NoSQL, properly

## Concept 30 — What "NoSQL" actually meant, and the axes that matter

"NoSQL" is a terrible name for a real idea. The movement (c. 2007–2012, driven by the Dynamo and Bigtable papers) gave up two things to buy one:

**Given up:** (1) the relational model with arbitrary joins and a declarative query language; (2) cross-entity ACID transactions.
**Bought:** horizontal scale-out with predictable per-operation cost, because every operation is routed to one partition by a key.

That's the entire trade, and stating it that way immediately outclasses "SQL is structured, NoSQL is flexible."

**The axes that actually decide the choice.** When you're asked "SQL or NoSQL," answer on these, not on the label:

| Axis | Question to ask |
|---|---|
| **Data model** | Do entities have a natural aggregate boundary, or are relationships the point? |
| **Query flexibility** | Are the access patterns known and stable, or will analysts ask questions you can't predict? |
| **Transaction scope** | What must be atomic together? Is it inside one aggregate or across many? |
| **Scale shape** | Does scale come from data volume, write throughput, geographic distribution, or connection count? Each has a different answer. |
| **Consistency requirement** | Per-item strong? Cross-item? Cross-region? (Module 7's ladder.) |
| **Schema evolution** | Who owns the schema and how many writers are there? |
| **Operational cost** | Who runs it at 3 a.m., and what's the hiring market for that skill? |

**The reframe that scores:** *"The relational model's superpower isn't tables, it's that it lets you ask questions you didn't anticipate when you designed the schema. A document store's superpower is that a single key lookup returns an entire aggregate in one I/O. If my access patterns are fixed and partition-aligned, I'm paying for flexibility I won't use; if they aren't, I'll be rebuilding the storage model every quarter."*

---

## Concept 31 — The families, and the access pattern each is built for

| Family | Physical idea | Built for | Examples | The cost |
|---|---|---|---|---|
| **Relational** | Normalized rows, B-tree indexes, a cost-based optimizer | Ad-hoc queries, cross-entity invariants, transactions | SQL Server, PostgreSQL, MySQL, Oracle | Scale-out requires sharding; joins across a shard boundary are expensive |
| **Document** | Aggregate stored as one JSON/BSON value, partitioned by key | Fetch/store a whole aggregate in one operation | Cosmos DB for NoSQL, MongoDB, DocumentDB, Couchbase | No cross-document transactions at scale; duplicated data must be kept in sync |
| **Key-value** | Hash partitioning, value opaque | Extremely fast point access, sessions, caches | Redis, DynamoDB, Azure Table Storage, etcd | No queries beyond the key |
| **Wide-column** | Rows partitioned, columns grouped in families, LSM storage | Very high write throughput, time-ordered per-partition reads | Cassandra, ScyllaDB, HBase, Bigtable | Query patterns must be designed before data is written |
| **Graph** | Nodes/edges with adjacency as a first-class structure | Multi-hop traversal (friends-of-friends, fraud rings, lineage) | Neo4j, Cosmos DB for Gremlin, Neptune, Postgres/SQL Server graph extensions | Weak at aggregate scans; specialized skill |
| **Time-series** | Time-partitioned, heavily compressed, columnar | Append-only measurements, downsampling, retention | TimescaleDB, InfluxDB, Azure Data Explorer/Kusto | Not a general-purpose store |
| **Search** | Inverted index + scoring | Relevance, faceting, fuzzy matching | Elasticsearch/OpenSearch, Azure AI Search | Near-real-time, not transactional; a projection, never a source of truth |
| **Vector** | ANN index over embeddings | Semantic similarity | pgvector, Azure AI Search, SQL Server 2025 `VECTOR`, Cosmos DB vector search | Approximate by construction; index build/refresh cost |

**The judgment layer:** the first four are *storage* decisions; search and vector are almost always **projections of a source of truth** (Module 11's event-carried state transfer). Treating a search index as a system of record is a classic, expensive mistake — say so unprompted if the design puts one there.

---

## Concept 32 — Normalization, denormalization, and the aggregate boundary

**Normalize for write correctness.** Third normal form means each fact is stored once, so an update can't leave two copies disagreeing. That's not aesthetics — it's the elimination of an entire class of bug.

**Denormalize for read cost**, deliberately, and name the price: **you now own the synchronization.** Every denormalized copy is a cache (Module 10) with the same three questions: what's the staleness bound, what invalidates it, and what happens if the update fails halfway?

**The aggregate is the bridge** — and this is the concept that connects data storage to Module 22's DDD work, which is why it's worth stating carefully.

> An **aggregate** is a cluster of objects treated as a unit for data changes, with one root. The rule that matters here: **the aggregate is the unit of atomicity and the unit of storage.** Invariants inside an aggregate are enforced transactionally; invariants across aggregates are enforced eventually, by events and compensations.

Once you have that, three things follow at once:

1. **Document stores make aggregates cheap.** One document = one aggregate = one atomic write = one read. That's the real argument for a document model, and it's a much better answer than "flexible schema."
2. **Aggregate boundaries predict service boundaries.** If two things must be transactionally consistent, they belong in the same service and the same store. Deciding services first and discovering the transaction spans them is the single most common cause of accidental distributed transactions (Part E).
3. **Big aggregates are a concurrency problem.** A `Customer` aggregate containing all their orders means every order write contends on the customer. Small aggregates, referenced by ID, are the rule.

**The sentence:** *"I'd draw the aggregate boundaries first — what has to be consistent at every instant — and let the storage model and the service boundaries follow from that, rather than the other way round."*

---

## Concept 33 — Access-pattern-driven design and single-table design

In a relational database you model entities and derive queries. In a partitioned NoSQL store you do the reverse: **enumerate the access patterns, then design storage so each one is a single-partition operation.**

The method, which you can narrate in an interview:
1. List every read and write, with its frequency and latency target.
2. For each, identify the key you have at query time.
3. Design item keys and partition keys so that key answers the query without a cross-partition scan.
4. Where two patterns need different keys for the same data, **duplicate it** — a second item, a materialized view, or a global secondary index — and decide how it's kept in sync and how stale it may be.
5. Check every logical partition against the size and throughput ceilings (Concept 34).

**Single-table design** (the DynamoDB idiom, applicable to Cosmos DB) puts multiple entity types in one container with generic key attributes, so one query can return an aggregate and its children together:

```
PK = "TENANT#42"   SK = "ORDER#1001"            → order header
PK = "TENANT#42"   SK = "ORDER#1001#ITEM#1"     → line item
PK = "TENANT#42"   SK = "ORDER#1001#ITEM#2"     → line item
```
A range query on `SK begins_with "ORDER#1001"` returns the whole order in one partition-local read, and `TransactionalBatch` can write them atomically because they share a partition key.

**Say the trade-off, because it's the part candidates skip:** single-table design buys single-digit-millisecond aggregate reads and atomic multi-item writes, and costs readability, ad-hoc queryability, and — most importantly — **flexibility to change access patterns later**. It suits a system whose queries are known and stable. Proposing it for a product still discovering its domain is a bad call, and knowing that is the signal.

---

## Concept 34 — Cosmos DB modeling, in the depth an interview reaches

This is the concrete calibration artifact for the whole of Part D in an Azure interview.

**The hierarchy:** account → database → **container** (the unit of throughput and partitioning) → **logical partition** (all items sharing a partition key value) → **physical partition** (a replica set holding one or more logical partitions).

**The limits, which you should have memorized:**
- A **logical partition** is capped at **20 GB** of storage and **10,000 RU/s**. A partition key value that grows unboundedly is a design bug, not a scaling problem.
- A **physical partition** holds up to **50 GB** and serves up to **10,000 RU/s**; Cosmos splits them automatically as data or throughput grows.
- Item size limit: **2 MB**. `TransactionalBatch`: up to **100 operations / 2 MB**, and **only within one logical partition**.
- Minimum **1 RU/s per 1 GB** stored — worth knowing because it makes cold archival data in Cosmos surprisingly expensive.

**Choosing a partition key — the four criteria:**
1. **High cardinality**, so data spreads.
2. **Even access distribution**, so RU/s spreads (cardinality alone is not enough — a random GUID spreads writes perfectly and makes every read a cross-partition fan-out).
3. **Present in your most frequent queries**, so reads are single-partition.
4. **Bounded growth per value**, so you never approach 20 GB.

**Hierarchical partition keys (subpartitioning)** are the answer to the classic multi-tenant conflict — `/TenantId` is queryable but unbounded, `/UserId` is bounded but makes every tenant query a fan-out. With up to a **three-level** key such as `/TenantId` → `/UserId` → `/id`, the *prefix* can exceed 20 GB and 10,000 RU/s while prefix queries are still routed to just the partitions holding that prefix. Microsoft's guidance is explicit: ending the key path with a unique value like `/id` gives effectively unlimited storage per first-level key. Being able to say "hierarchical partition key with `/id` as the last level" is a precise, current answer to a very common question.

**The other surface to know by name:** the **change feed** (an ordered, per-partition, persistent log of changes — Module 11's CDC, and the basis for outbox-style integration in Cosmos), **TTL** for automatic expiry, **indexing policy** (everything is indexed by default; excluding paths is a real RU and write-cost lever), **autoscale** (scales between 10% and 100% of a max, billed on the peak within the hour), **serverless** (per-operation billing, for spiky/dev workloads), **analytical store + Fabric mirroring** for HTAP, and **global secondary indexes** in preview (Concept 18).

**The RU reframing worth having:** *"RU/s is a normalized currency for CPU, IOPS and memory. The useful consequence is that cost and throughput are the same conversation — a bad partition key doesn't just cause a hot partition, it shows up on the invoice, which makes it one of the few architectural mistakes with a directly observable price."*

---

## Concept 35 — Schema evolution without DDL

"Schemaless" means there is still a schema — it has just moved from the database into **every piece of code that reads the data, forever**. In a relational database a column rename is one migration; in a document store it's a permanent branch in every reader until you've rewritten every document.

**The disciplines that make document stores survivable at year three:**

1. **A version field on every document.** `"schemaVersion": 3`. Non-negotiable, and cheap.
2. **Tolerant readers.** Ignore unknown fields; treat missing fields as defaults. (Identical to Module 11's schema-evolution rules — because it's the same problem.)
3. **Additive-only changes** as the default. Renames and type changes are breaking changes and need the same treatment as a breaking event-contract change.
4. **Lazy migration (upcasting on read)** — read v2, convert to v3 in the mapping layer, write back v3 when the document is next written. Combine with a background backfill so the long tail eventually converges, and *set a deadline* after which the v2 branch is deleted; otherwise you accumulate readers forever.
5. **Validation at the edge.** JSON Schema validation in the application or the store, so the "flexible" schema is at least an *intentional* one.

**The senior sentence:** *"Schemaless doesn't remove the migration, it removes the migration's deadline — which means it also removes the forcing function that gets it finished. I'd take the flexibility deliberately and add the version field and the backfill on day one."*

**Worth knowing for the relational side:** modern relational engines let you have both. **SQL Server 2025 and Azure SQL have a native `json` data type** with JSON indexing and array/object functions in T-SQL, and PostgreSQL's `jsonb` with GIN indexes has been production-grade for a decade. **EF Core maps owned entities and complex types to JSON columns** and can query and (in EF Core 10) `ExecuteUpdate` into them. So "we need schema flexibility for part of the model" is frequently *not* a reason to add a document database — it's a reason to use a JSON column in the database you already run. That's a high-value thing to say, because it demonstrates you consider the cheaper option first.

---

## Concept 36 — Distributed SQL (NewSQL)

The category that refuses the original NoSQL trade: keep SQL, keep serializable transactions, *and* scale horizontally. The mechanism is always the same shape — **the data is range-partitioned into shards, each shard is a consensus group (Raft/Paxos, Module 9), and a transaction spanning shards runs two-phase commit *over* those consensus groups** (Concept 43).

| System | Wire protocol | Distinctive property |
|---|---|---|
| **Google Spanner** | Custom / PostgreSQL interface | **TrueTime**: GPS+atomic-clock bounded uncertainty, enabling *external consistency* via commit wait |
| **CockroachDB** | PostgreSQL | Raft over a Pebble (LSM) key-value store; serializable by default; multi-region survival goals |
| **YugabyteDB** | PostgreSQL (reuses actual Postgres query layer) + Cassandra | High Postgres compatibility because it runs Postgres's own query code |
| **TiDB** | MySQL | HTAP: TiKV (row) + TiFlash (columnar) replicated in real time |
| **Amazon Aurora DSQL** | PostgreSQL | Serverless, active-active multi-region, disaggregated architecture |
| **Azure SQL Hyperscale** | T-SQL | *Not* distributed SQL — a single-writer engine with disaggregated storage. Know the difference. |

**The cost, stated plainly, which is the part that matters in an interview:** a distributed transaction commits at the speed of a consensus round trip. Within a region that's a few milliseconds; across regions it's tens to hundreds. A workload made of many small sequential transactions — the classic ORM pattern — feels dramatically slower than on single-node Postgres even though every individual operation is "fast." Distributed SQL removes the *sharding* work, not the *physics*.

**The senior position:** *"Distributed SQL is the right answer when multi-region active-active with strong consistency is a genuine requirement — data residency, regulatory, or a real availability target that a failover can't meet. It's the wrong answer when the actual requirement is 'we might get big someday,' because you'd be paying a permanent latency and complexity tax for a scenario a few minutes of failover downtime would have covered."*

---

## Concept 37 — Polyglot persistence and its real costs

"Use the right tool for the job" is true and incomplete. Each additional data store adds:

- **An operational surface**: backup, restore, patching, monitoring, capacity, security review, DR runbook.
- **A consistency boundary**: keeping two stores in sync is the dual-write problem (Module 11, Concept 17), so you now need an outbox or CDC — per store.
- **A skills dependency**: who debugs it at 3 a.m., and what happens when they leave?
- **A failure mode**: your availability is now the product of more terms.
- **A cost line**, usually with a different pricing model that makes comparison hard.

**The bar to apply:** a new store must serve a **first-class access pattern that the existing store genuinely cannot serve at acceptable cost**, and the team must be able to operate it. "It's better at this" is not sufficient; "this pattern is central to the product and the current store forces a full scan for it" is.

**Where it genuinely pays off:** a search index alongside a relational source of truth; Redis for sessions and hot reads (Module 10); a time-series store for high-cardinality metrics; a lake/warehouse for analytics (Concept 39); a graph store when multi-hop traversal is the core feature rather than a report.

**The architect framing:** *"Each store is a permanent operational commitment, so I'd rather add an index, a JSON column, or a materialized view to the database we already run. When I do add one, I want it to be a projection with a clear owner and a defined rebuild path, so losing it is an inconvenience and not an incident."*

---

## Concept 38 — "Just use Postgres" (or SQL Server): the arithmetic

Have this argument ready, because volunteering it is one of the strongest signals available in a data design round — it shows you optimize for total cost rather than for sounding sophisticated.

**The numbers:** a single modern managed relational instance handles low-TB data volumes, thousands to tens of thousands of simple OLTP transactions per second, with sub-millisecond cached reads. Azure SQL Hyperscale extends that to **128 TB** with rapid scaling and snapshot-based backup/restore, up to 192 vCores, and read-scale replicas. That envelope covers the overwhelming majority of business systems, including most that describe themselves as "high scale."

**What one Postgres can now absorb that used to require a second store:** JSON documents (`jsonb` + GIN), full-text search (`tsvector`), geospatial (PostGIS), time-series (TimescaleDB), vectors (`pgvector`), queues (`SKIP LOCKED`), and columnar analytics (Citus/columnar extensions). The SQL Server equivalent list now includes a native `json` type with indexing, `VECTOR` with DiskANN indexes, columnstore, spatial, and full-text.

**How to say it without sounding like you're ducking the question:**
> *"Let me size it first. 500M rows at 300 bytes is ~200 GB of data, maybe 400 GB with indexes, with a working set well under 64 GB — that fits comfortably on a single instance with read replicas, and the write rate is two orders of magnitude below what the hardware does. So I'd start there, and the thing I'd design for now is the seam where we'd shard later: a tenant-scoped key on every table, no cross-tenant queries in the hot path, and the migration path written down."*

That answer demonstrates estimation, restraint, and forward planning at once — and it invites the interviewer to raise the scale, which is exactly the conversation you want.

---

## Concept 39 — OLTP/OLAP separation, CDC, and the lakehouse

**The rule:** don't run analytics on the transactional store. Not for purity — for a mechanical reason you should state (Concept 8): analytic queries are scan-heavy, evict the OLTP working set from the buffer pool, take long-lived read locks or snapshots (blocking version cleanup, Concept 29), and have unbounded resource profiles.

**The escalation ladder, cheapest first:**
1. **A read replica** for reporting. Cheapest, and it isolates resources but not the data model. Watch replication lag and the fact that long analytic queries on a replica can conflict with replay.
2. **A columnstore index** on the OLTP table (Concept 8) — real-time analytics, no second system, some write cost.
3. **CDC / change feed into a lake or warehouse.** SQL Server CDC or change tracking, Postgres logical decoding (Debezium), Cosmos DB change feed. This is Module 11's Concept 20, now as the standard data-platform integration.
4. **A full analytical platform** — Microsoft Fabric (OneLake, Lakehouse, Warehouse), Synapse, Databricks — with a medallion architecture (bronze raw → silver conformed → gold aggregated).
5. **Zero-ETL mirroring.** Cosmos DB's analytical store and Fabric **mirroring** for Azure SQL and Cosmos replicate into OneLake continuously without you writing a pipeline. Worth naming because it's the current Azure answer and it removes a whole class of pipeline maintenance.

**The thing to volunteer:** the **staleness contract**. "The warehouse is 15 minutes behind; the dashboard says so; finance reports run against the 00:00 UTC snapshot." Analytics consumers make decisions on stale data whether or not you tell them — the difference between a mature and an immature platform is whether the staleness is stated and monitored.

---

# Part E — Distributed transactions: 2PC vs Saga, completed

## Concept 40 — The atomic commit problem, stated precisely

The setup: a transaction has touched data on several nodes. Every node must reach the *same* decision — commit or abort — and must reach it durably.

**Why this is harder than consensus, which is the insight that scores.** Consensus (Module 9) needs a **majority** to agree on a value; a minority can be outvoted and the system still makes progress. Atomic commit needs **unanimity**: a single participant voting "no" must abort the transaction, so you cannot outvote a participant, and you cannot proceed without hearing from one. The result:

> **Consensus is available under partial failure; atomic commit is not.** That is why two-phase commit blocks and Raft does not.

Add the FLP result (Module 9): in an asynchronous system with even one crash failure, no deterministic protocol can guarantee both safety and termination. For atomic commit this shows up concretely as the **in-doubt** state (Concept 41).

**The second framing, which is the practical one.** Pat Helland's argument in *Life Beyond Distributed Transactions*: beyond a certain scale you should design so that each transaction touches exactly one entity, and everything crossing entities is at-least-once messaging with idempotent, retryable handlers. That is not a workaround — it is the design. Almost everything in Part E and in Module 11 is a corollary.

---

## Concept 41 — Two-phase commit: the protocol and the blocking problem

**Phase 1 — Prepare.** The coordinator asks every participant "can you commit?" Each participant does all the work, writes it durably to its log, **acquires and keeps all locks**, and answers *yes* or *no*. A *yes* is an irrevocable promise: the participant can no longer unilaterally abort.

**Phase 2 — Commit/Abort.** If all voted yes, the coordinator durably logs `commit` — this write is the point of no return for the whole transaction — and tells everyone to commit. Any *no* means abort everywhere. Participants acknowledge; the coordinator forgets the transaction.

**The failure that defines the protocol.** A participant votes yes, then the coordinator crashes before sending the decision. The participant is **in doubt**: it cannot commit (the decision might have been abort) and cannot abort (it promised). It **holds its locks and waits** — potentially for hours, until the coordinator returns. Other transactions touching those rows block behind it.

Three consequences to name:

1. **The coordinator is a single point of failure** for liveness, and its log is critical state that must be durable and recoverable.
2. **Locks are held across a network round trip *and* the decision window.** Throughput on contended rows collapses.
3. **Recovery is operational.** In-doubt transactions must be resolved — automatically when the coordinator recovers, or manually by a DBA (`KILL`, `ROLLBACK PREPARED`, `xa recover`). Anyone who has run XA in production has a story about this, so knowing the term "in-doubt transaction" signals you have been near one.

**Why 3PC is not the answer.** Three-phase commit adds a pre-commit phase to make the protocol non-blocking — but only under a synchronous model with reliable failure detection and no network partitions. Under real partitions it can produce inconsistent decisions. It is a textbook item, not a production one. The real fix is Concept 43.

**Costs to quantify:** two round trips plus several durable log writes on the critical path, locks held throughout, and availability that is the *product* of all participants' availability. Four participants at 99.9% each give roughly 99.6% for the composite operation — a number worth doing out loud, because it reframes 2PC as an availability decision rather than purely a correctness one.

---

## Concept 42 — XA, MSDTC, and the .NET reality

**XA** is the X/Open standard interface for 2PC between a transaction manager and resource managers (databases, message brokers). It is the reason 2PC persists in enterprise systems: it is standardized, it works, and a lot of working software depends on it.

In the Microsoft world the transaction manager is **MSDTC** (Microsoft Distributed Transaction Coordinator) and the API is `System.Transactions.TransactionScope`:

```csharp
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
using (var c1 = new SqlConnection(connA)) { /* ... */ }
using (var c2 = new SqlConnection(connB)) { /* ... */ }   // promotes to a distributed transaction
scope.Complete();
```

**Four things you must know before proposing this in 2026** — exactly the kind of currency check interviewers use:

1. **It is Windows-only.** Distributed transaction support in modern .NET is tightly coupled to MSDTC and the OleTx protocol; on Linux you get a `PlatformNotSupportedException`. Cross-platform support is still an open request on `dotnet/runtime`. If your service runs in a Linux container — as most ASP.NET Core services now do — `TransactionScope` across two connections simply will not work.
2. **It is opt-in.** Since .NET 7 you must set `TransactionManager.ImplicitDistributedTransactions = true` at startup, or you get an explicit exception telling you so. That was a deliberate choice to stop accidental promotion.
3. **Promotion is accidental by nature.** A `TransactionScope` starts as a lightweight local transaction and **silently promotes** the moment a second connection — even to the same server, with a different connection string — enlists. The classic production surprise is a routine refactor that adds a second `DbContext` and converts a fast local transaction into an MSDTC transaction.
4. **`TransactionScope` and `async` need `TransactionScopeAsyncFlowOption.Enabled`**, or the ambient transaction does not flow across `await` and the behaviour becomes bewildering. Knowing that specific flag is a small, reliable .NET credibility marker.

**Where XA/2PC is still legitimate:** inside one trust and one operations boundary, with a small number of participants, on low-throughput paths, with a legacy resource manager you cannot change. Nightly financial postings between two on-premises databases is the archetype. It is *not* the answer for microservices communicating over a network you do not control.

---

## Concept 43 — 2PC over consensus: what Spanner and Percolator actually do

This is the nuance that makes you sound like you have read the papers rather than the blog posts — because "2PC is bad" and "Spanner does distributed transactions at global scale" are both true, and the reconciliation is the interesting part.

**The move:** in Spanner — and in CockroachDB, YugabyteDB, TiDB — each *participant* is not a single node, it is a **Paxos/Raft group**. So:

- The coordinator's decision log is itself replicated by consensus, so **the coordinator can fail over** rather than leaving participants in doubt.
- Each participant's prepared state is replicated, so **a participant crash does not lose the promise** or block indefinitely.
- 2PC's blocking problem is therefore **removed at the infrastructure level** rather than merely tolerated. What remains is latency, not indefinite unavailability.

**Spanner's other piece is TrueTime.** Commit timestamps come from a clock API that returns an *interval* with bounded uncertainty, backed by GPS and atomic clocks, and Spanner performs a **commit wait** — it waits out that uncertainty before making the transaction visible. That is what buys **external consistency** (strict serializability: serializable *and* respecting real-time order). The price is a few milliseconds of deliberate waiting on every commit. That trade — *spend latency to buy a global order* — is one of the most elegant ideas in distributed systems and a great thing to be able to explain in two sentences.

**Percolator** (Google, for incremental index updates over Bigtable) is the other model worth naming: snapshot isolation implemented with client-driven 2PC, a primary-lock row acting as the commit point, and a centralized timestamp oracle. TiDB's transaction model descends from it.

**Calvin** is the third shape: avoid 2PC entirely by **deterministically ordering transactions up front** through a sequencing layer, so every replica executes the same order and no agreement about committing is needed. FaunaDB used this approach. Naming it shows you know 2PC is not the only option.

**The sentence to have ready:** *"When people say 'never use 2PC' they mean 2PC with a single non-replicated coordinator over independently-owned resource managers, which blocks. Spanner-class systems run 2PC over Paxos groups, which removes the blocking failure — the cost moves from availability to latency, and that is a much better trade."*

---

## Concept 44 — Sagas, completed

Module 11, Concept 21 introduced sagas as a messaging pattern. Here is the database-side completion.

**Definition** (Garcia-Molina & Salem, 1987): a long-lived transaction decomposed into a sequence of local transactions `T1…Tn`, each with a **compensating** transaction `C1…Cn-1`. If `Tk` fails, run `Ck-1…C1` to semantically undo the completed work.

**The acronym that captures it is ACD, not ACID:**

- **A**tomicity — yes, but *semantic*: eventually either all steps complete, or all completed steps are compensated.
- **C**onsistency — yes, eventually.
- **I**solation — **no**. This is the whole cost.
- **D**urability — yes, per local transaction.

**"No isolation" concretely.** Between `T1` and `Tn`, other transactions see intermediate state: an order that exists but is not paid, inventory reserved for an order that will be cancelled, money debited but not yet credited. The system passes through states that would be impossible under a single transaction. Two named hazards:

- **Dirty reads** — another saga reads the intermediate state and makes a decision on it.
- **Lost updates** — another saga overwrites a value the first saga will later need in order to compensate.

**The four countermeasures** — naming these is a strong depth signal:

1. **Semantic lock** — mark the record with a pending state (`Status = PendingPayment`) so other actors know it is in flight and can refuse or wait.
2. **Commutative updates** — design operations so order does not matter (`credit(+50)` / `debit(-50)` rather than `setBalance(x)`), which eliminates lost updates.
3. **Pessimistic view** — reorder steps so the riskiest state change happens last, shrinking the window in which a dirty read matters.
4. **Re-read value / version file** — re-read and verify before acting (optimistic concurrency, Concept 26), or record operations so they can be reordered during compensation.

**Compensation is not rollback, and this is the part most candidates miss.** A compensation is a **new business fact**, not an erasure. You do not un-send an email; you send an apology. You do not un-charge a card; you issue a refund, which appears on the customer's statement. Some steps are **not compensatable at all** — the email is sent, the parcel has shipped. That is exactly why you order saga steps so irreversible actions come last, and why the **pivot transaction** — the step after which the saga can only go forward — is worth naming.

**Orchestration vs choreography** gets its full treatment in Module 11, Concept 29. The short form: orchestrate anything with compensations and timeouts, because "what happens when an order is placed" should be one readable artifact; choreograph peripheral reactions.

**In .NET:** Durable Functions and the Durable Task Scheduler, MassTransit or NServiceBus sagas, Temporal, or a hand-rolled state machine persisted in your own database with an outbox. The state machine *must* be persistent and idempotent — a saga is a long-lived process and will span deployments, restarts, and duplicate messages.

---

## Concept 45 — TCC and the reservation pattern

**Try-Confirm/Cancel** is a saga with a specific, very useful shape: make the first step a **reservation with a TTL** rather than a commitment.

1. **Try** — reserve the resource (hold the seat, authorize the card without capturing, move stock into a `Reserved` bucket) with an expiry.
2. **Confirm** — turn reservations into commitments once every participant has said yes.
3. **Cancel** — release reservations explicitly; and crucially, **the TTL releases them anyway if nobody calls**.

**Why it is worth knowing by name:** it converts compensation from "undo something the customer already saw" into "let a hold expire," which is far safer, and it makes the failure path self-healing rather than dependent on a compensation message arriving. Card authorizations, seat holds, and inventory reservations all work this way in the real world, so you can justify the pattern from the domain rather than from architecture fashion.

**The design requirements:** reservations need a TTL and a sweeper; `Confirm` and `Cancel` must be idempotent; available quantity is computed as `total − reserved − sold`; and you must decide what happens when a confirm arrives *after* the reservation expired — usually try to re-reserve, otherwise fail the saga.

Say this in an interview about order or payment systems and you will be well ahead of the standard saga answer, because it shows you have thought about what the compensation actually does to a customer.

---

## Concept 46 — The outbox and idempotency as the default substrate

The practical default for "write to my database and tell another service," derived fully in Module 11, Concepts 17–19, and restated here in database terms:

1. **One local ACID transaction** writes the business change *and* a row in an `Outbox` table. Atomic, because it is one database.
2. **A relay** reads unpublished outbox rows and publishes them, at least once. Competing relays use `FOR UPDATE SKIP LOCKED` (Postgres) or `WITH (UPDLOCK, READPAST)` (SQL Server).
3. **Consumers are idempotent** — an `IdempotencyKeys` table, or a unique index on the natural key, written **in the same transaction as the effect**.

**The database-side details worth adding here:**

- The unique index *is* the idempotency mechanism. A duplicate insert raises a unique-constraint violation, which you catch and treat as success. It is the cheapest correct deduplication available, and it works precisely because the index is transactional.
- **Cleanup matters.** An outbox table that is never pruned becomes the largest and most fragmented table in the database. Delete published rows in batches, or partition by date and drop partitions.
- **Ordering.** Outbox rows publish in insertion order per key if the relay honours it; in general treat ordering as per-key, never global (Module 11, Concept 11).
- **CDC as the low-latency relay** — Debezium over Postgres logical decoding, SQL Server CDC, or the **Cosmos DB change feed**, which makes the outbox pattern especially natural in Cosmos because the change feed *is* the relay.

**The sentence that ties Part E together:** *"For cross-service writes my default is one local transaction plus at-least-once delivery plus an idempotent consumer. That gives me atomicity where it is cheap — inside one database — and converts the distributed part of the problem from 'agreement' into 'retry', which is a problem I know how to operate."*

---

## Concept 47 — The decision framework, and the aggregate as the transaction boundary

When a design needs to change state in two places, work down this list **in order**. Narrating the order is the answer; jumping straight to saga or 2PC without considering the earlier options is the mid-level tell.

1. **Move the boundary.** Do these two things really belong to different aggregates or services? If they must be consistent *at every instant*, they are one aggregate — put them in one service and one database, and the problem disappears. **This resolves more cases than any other option, and it is almost always the right first question.**
2. **One local transaction.** Same database, one commit. Free.
3. **Make the second step unnecessary.** Can the reader derive the state instead of being told? Can the second effect be computed on read, or maintained by an index or materialized view the engine keeps for you?
4. **Reservation / TCC** (Concept 45). Bound the inconsistency with a TTL so the failure path is self-healing.
5. **Saga with compensation** (Concept 44). Accept the lost isolation, add semantic locks, order irreversible steps last.
6. **2PC.** Only inside one operational boundary, with few participants, low throughput, latency tolerance, and where a blocked in-doubt transaction is an acceptable worst case — or where the engine gives you 2PC-over-consensus (Concept 43) and the trade is latency rather than availability.

**The framing to close on:** *"Distributed transactions are usually a symptom of a boundary drawn in the wrong place. Before I reach for a saga I want to check whether these two things are actually one aggregate — because if they are, I have turned a hard distributed-systems problem into a `SaveChanges` call."*

**The bridge to Module 22 (DDD):** aggregate → transaction boundary → service boundary, in that order. Vaughn Vernon's rule of thumb — one aggregate per transaction, reference other aggregates by identity, update them eventually via domain events — is the same statement seen from the domain side, and quoting it connects Phase 3 to Phase 5 in a way interviewers notice.

---

# Part F — Scaling and operating data

## Concept 48 — The scaling ladder

When someone says "the database is the bottleneck," work down this ladder and say why you are skipping each rung. Skipping straight to sharding is the most common mid-level failure in a design round.

1. **Fix the query and the index.** The cheapest order-of-magnitude available. Most "we need to scale the database" conversations end here, and saying so first is a strength, not a dodge.
2. **Fix the access pattern.** N+1 queries, unbounded result sets, chatty ORMs, missing pagination, `SELECT *` over wide rows. This is application work with database symptoms.
3. **Cache** (Module 10). Effective only where reads are skewed and staleness is acceptable. Note explicitly that a cache does nothing for write pressure.
4. **Scale up.** Boring, immediate, and often the right answer. Modern instances go to hundreds of cores and terabytes of RAM. The ceiling is real but distant, and the cost of buying a bigger instance is almost always less than the cost of an engineer-quarter spent sharding.
5. **Read replicas** (Concept 49). Reads scale out; writes do not.
6. **Functional partitioning / vertical split.** Move a bounded context to its own database. Preserves single-node simplicity within each piece, and aligns with service boundaries. **This is the most under-used rung and often the best one.**
7. **Table partitioning** (Concept 50). Same engine, split table — for manageability and partition elimination, not for throughput.
8. **Shard** (Module 8). Horizontal split across engines. Real scale-out, and a permanent increase in complexity: cross-shard queries, cross-shard transactions, rebalancing, per-shard schema migrations, and a routing layer to own.
9. **Change the storage technology.** A different engine class because the access pattern demands it (Concept 31), not because the current one is unfamiliar.

**The sentence:** *"I would want to know which resource is actually saturated — CPU, I/O, memory, locks, or connections — because each points at a different rung. Sharding fixes a data-volume and write-throughput problem, and it is the wrong answer to a lock-contention problem or a missing-index problem."*

---

## Concept 49 — Read replicas and read-your-writes

Replication (Module 8) gives you read scale-out. The consequence you must handle is **lag**, and how you handle it *is* a consistency decision (Module 7).

**Lag is not constant.** It is typically single-digit milliseconds and then spikes — during bulk writes, index rebuilds, long transactions on the primary, or replica-side contention. Monitor it in **time** (seconds behind primary, or LSN/log-sequence gap), the same way Module 11 monitors message age rather than queue depth.

**The read-your-writes problem**, which is the interview question hidden inside "let's add read replicas": a user updates their profile, is redirected, the read goes to a replica that has not caught up, and they see their old data. They conclude the save failed and do it again.

**The five remedies, in order of preference:**

1. **Route reads that follow a write to the primary** — for a short, per-user window (a cookie or session flag for N seconds). Simple, effective, and the usual answer.
2. **Read your own writes from your own write.** Return the updated entity in the write response and use it, instead of re-reading.
3. **LSN / token-based routing.** Capture the log position at write time, pass it with the read, and either wait for the replica to reach it or fall back to the primary. Precise and more work — it is Module 7's session consistency, hand-built.
4. **Consistent-prefix / session guarantees from the platform.** Cosmos DB's Session consistency does exactly this with a session token — a great concrete example to cite, because it shows the platform solving a problem you can otherwise only solve by hand.
5. **Accept it and show it in the UI** — optimistic rendering with a "saving…" state. Legitimate, and worth saying, because some designs do not need the machinery.

**The rule to state:** *"A read replica is an asynchronous copy, so putting a read on it is choosing eventual consistency for that read. I would classify reads explicitly: anything that follows a write in the same user flow, or drives a decision with money attached, goes to the primary; reporting, search and browse go to replicas."*

---

## Concept 50 — Partitioning vs sharding

The words are used interchangeably in conversation and mean different things in practice. Distinguishing them quickly is a small, clean signal.

**Partitioning (table partitioning):** one logical table split into physical partitions inside *one* database engine, usually by range on a date or by a tenant/hash key.

What it buys:
- **Partition elimination** — queries with a predicate on the partition key touch only relevant partitions.
- **Sliding-window maintenance** — `SWITCH`/`DETACH` a whole month's partition out in near-constant time instead of a `DELETE` that generates hundreds of gigabytes of log. **This is the real reason to partition**, and saying so is more precise than "for performance."
- Partition-level index maintenance, compression, and statistics.

What it does not buy: additional CPU, memory, or IOPS. It is one engine.

**Sharding:** data split across *independent* database instances, each holding a subset. Buys genuine write and storage scale-out, at the cost of everything Module 8 catalogued: routing, cross-shard queries and joins, cross-shard transactions (Part E), rebalancing, hotspots, and N× the operational surface — including N× the schema migrations.

**Shard key selection is the whole game**, and it is the same list as Cosmos DB's partition key criteria (Concept 34): high cardinality, even distribution, present in the common query, and bounded growth per value. In a business system a **tenant ID** is the usual right answer, because it aligns the shard boundary with a natural isolation boundary — queries rarely cross tenants, so cross-shard queries rarely arise.

**Resharding is the part people forget.** Prefer **consistent hashing** or, better, **many logical shards mapped onto few physical shards** (the "virtual bucket" approach) so rebalancing means moving buckets rather than rehashing everything. Design the routing indirection on day one even if you never use it, because retrofitting it into a live system is a project.

**Azure specifics worth naming:** Azure SQL **elastic database tools** and the split-merge service, sharded **elastic pools**, Azure Database for PostgreSQL **Elastic Clusters** (Citus-based distributed Postgres), and — for the NoSQL side — Cosmos DB, where partitioning is automatic and the design work is entirely in the partition key.

---

## Concept 51 — Connection pooling and the connection wall

An under-appreciated scaling limit, and a favourite of interviewers who have operated real systems.

**PostgreSQL:** every connection is an OS **process**, costing roughly 5–10 MB. A few hundred connections is normal; a few thousand is a crisis. Beyond the memory cost, high connection counts cause contention on shared structures and degrade throughput even when the connections are idle.

**SQL Server:** connections are threads/fibres and cheaper, but worker threads, memory grants and `tempdb` contention still form a ceiling. Azure SQL publishes explicit max concurrent sessions and workers **per service tier** — a real, documented limit to design against.

**The .NET side.** `SqlConnection` pools per *exact* connection string; the default `Max Pool Size` is **100**. Pool exhaustion presents as `InvalidOperationException: Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool` — you should recognize that message on sight, because it is nearly always a symptom of something else: connections not disposed, a long transaction holding a connection (Concept 29), or a slow downstream making every request hold its connection longer. **Raising `Max Pool Size` is almost never the fix**; it usually just moves the queue from your process into the database.

**The modern squeeze:** serverless and autoscaling compute multiply the problem. 200 Azure Function instances × a pool of 100 = 20,000 potential connections against a database sized for 500. This is a genuine architectural constraint of serverless + relational, and naming it unprompted is a strong signal.

**The remedies:**
- **An external pooler** — PgBouncer (transaction-mode pooling is the one that matters), Azure Database for PostgreSQL's built-in **PgBouncer**, or **Azure SQL Database's built-in connection-pooling proxy** behaviour. Note the catch: transaction-mode pooling breaks session state, prepared statements, advisory locks and `SET` commands, so it is not free.
- **Cap concurrency upstream** — smaller pools per instance, a concurrency limiter, or queue-based load levelling (Module 11).
- **Shorten the time a connection is held** — no network calls inside a transaction; open late, close early; do not hold a `DbContext` across an `await` on something slow.
- **Use HTTP-based data APIs** where the platform offers them (Cosmos DB is HTTP/TCP with its own multiplexed connection management, which is part of why it scales well against serverless compute).

---

## Concept 52 — Zero-downtime schema migration: expand/contract

This is the single most practically useful operational pattern in the module, and a very common architect-round question because it exposes whether you have shipped changes against live data.

**The problem:** you cannot deploy a schema change and the code that depends on it at the same instant. During a rolling deployment, **old code and new code run simultaneously against one schema**. Therefore the schema must be compatible with both.

**Expand / contract (also called parallel change), in six steps.** Renaming `Orders.Total` to `Orders.TotalAmount`:

1. **Expand** — add the new nullable column `TotalAmount`. Additive, non-blocking, backward compatible.
2. **Dual-write** — deploy code that writes both columns and reads the old one. Both code versions now work.
3. **Backfill** — copy existing rows in **batches** with commits between (Concept 24: avoid lock escalation and log growth). Throttle it; monitor replication lag while it runs.
4. **Switch reads** — deploy code that reads the new column. Verify with metrics and comparison checks before continuing.
5. **Stop writing the old column** — deploy code that writes only the new one. Wait long enough that rollback is no longer plausible.
6. **Contract** — drop the old column. A separate, later deployment.

Six steps and typically four deployments for one rename. That is the honest cost, and **saying the honest cost is the point of the answer** — it demonstrates you know why teams do risky big-bang migrations and why they should not.

**The rules that generalize:**
- **Additive changes are safe; destructive changes are a separate, later deployment.** Never in the same release as the code change.
- **Every migration must be backward compatible for at least one deployment cycle**, because rollback is a normal event.
- **Never take a long blocking lock at peak.** Know your engine's behaviour: adding a nullable column with no default is metadata-only in both SQL Server and Postgres; adding `NOT NULL` with a default is metadata-only in modern versions but was a full table rewrite in older ones; `CREATE INDEX CONCURRENTLY` / `WITH (ONLINE = ON)` for indexes; a type change is usually a rewrite. **Postgres 18 added the ability to add a `NOT NULL` constraint without a full table scan** — a concrete example of why you check your engine's current behaviour rather than trusting folklore.
- **Migrations must be idempotent and forward-only** in the deployed pipeline. EF Core migrations, DbUp, Flyway, Liquibase — pick one and make it part of deployment, not a manual step.
- **Separate schema migration from data migration.** Schema changes are fast and transactional; backfills are long, throttled background jobs with progress tracking and resumability.
- **EF Core specifics worth knowing:** generate SQL scripts (`dotnet ef migrations script --idempotent`) for review rather than running `Database.Migrate()` at application startup in production — startup migration races across instances and gives you no review gate. That single opinion, offered unprompted, reads as production experience.

---

## Concept 53 — Multi-tenancy data models

A recurring architect-round question, with three answers and no universally right one.

| Model | Shape | Isolation | Noisy neighbour | Per-tenant restore | Migration cost | Cost per tenant |
|---|---|---|---|---|---|---|
| **Silo** — database per tenant | N databases | Strongest | None | Easy | **N migrations** | Highest |
| **Bridge** — schema per tenant | One DB, N schemas | Medium | Shared resources | Medium | N schemas to migrate | Medium |
| **Pool** — shared tables with `TenantId` | One set of tables | Weakest (code-enforced) | **Real** | **Hard** | One migration | Lowest |

**The decision drivers**, which matter more than the table:
- **Regulatory / contractual isolation** requirements push toward silo.
- **Tenant count and size distribution.** Thousands of small tenants make silo operationally absurd; a handful of large enterprise tenants make it natural. **A hybrid is extremely common and worth proposing**: pooled by default, siloed for the largest or most regulated tenants, with the code path identical because tenant→connection resolution is a lookup.
- **Per-tenant restore** is the requirement that most often forces silo, and it is the one teams discover late. Ask about it early.
- **Noisy neighbours** in the pool model need per-tenant rate limiting or resource governance — Azure SQL elastic pools, Cosmos DB hierarchical partition keys, or an application-level limiter.

**Enforcement in the pool model** is where systems fail. A missing `WHERE TenantId = @t` is a cross-tenant data leak — the worst class of bug in a SaaS product. Defend in depth:
- **EF Core global query filters** (`HasQueryFilter`) applied to every tenant-scoped entity. **EF Core 10 added named query filters**, so an entity can carry both a tenant filter and a soft-delete filter and you can disable one selectively — which removes the old, dangerous workaround of `IgnoreQueryFilters()` turning off *everything*.
- **Row-Level Security** in SQL Server / Postgres as a second layer that does not depend on application code being correct.
- **`TenantId` as the leading column of every index and the partition/shard key**, so isolation and performance align.
- **Tests that assert cross-tenant access fails**, not just that same-tenant access succeeds.

---

## Concept 54 — Backup, PITR, and the restore you have not tested

Start every answer here with two numbers, because they convert a vague question into an engineering one:

- **RPO (Recovery Point Objective)** — how much data may we lose? Determined by backup/log-shipping frequency and replication mode.
- **RTO (Recovery Time Objective)** — how long may recovery take? Determined by restore mechanism and data volume.

**The mechanisms, and what each actually protects against:**

| Mechanism | Protects against | Does *not* protect against |
|---|---|---|
| Full + differential + log backups | Data loss, corruption, deletion | Slow restore for very large databases |
| **Point-in-time restore (PITR)** | "We ran the wrong UPDATE at 14:32" | Anything outside the retention window |
| Snapshot-based backup (Hyperscale, Cosmos continuous) | Slow restores — near-constant time regardless of size | Logical corruption propagated instantly |
| Synchronous replica / zone redundancy | Hardware and zone failure | **Logical errors — they replicate perfectly** |
| Async geo-replication / failover group | Regional failure | Logical errors; and it has a nonzero RPO |
| Long-term retention / immutable storage | Ransomware, compliance retention | Nothing operational day to day |

**The two statements worth making unprompted, because they are the ones that distinguish a considered answer:**

1. **"Replication is not a backup."** A `DELETE FROM Orders` replicates to every replica in milliseconds. Only a point-in-time restore recovers from logical error. This is the most common and most expensive misconception in this area.
2. **"A backup you have not restored is a hypothesis."** Restore testing must be scheduled, automated, and timed — because the *measured restore time* is your actual RTO, and teams routinely discover it is 6 hours when the SLA says 1.

**Azure specifics:** Azure SQL takes automated backups with PITR retention of 1–35 days (configurable, with long-term retention up to 10 years); Hyperscale uses **file-snapshot backups**, so backup and restore are near-constant-time regardless of database size — a genuinely strong argument for the tier at large data volumes. Cosmos DB offers periodic or **continuous backup** with PITR (7 or 30 days), and note that **continuous backup is a prerequisite for enabling global secondary indexes** (Concept 18). **Active geo-replication and auto-failover groups** handle regional failure and are a DR feature, not a backup feature.

**The operational half:** document the runbook, name who can execute it, script it, and rehearse it. An architect answer to "what is your DR strategy?" ends with *"and we test it quarterly, with the last measured restore time"* — not with a list of features.

---

## Concept 55 — Data lifecycle and cost

Data volume grows monotonically unless someone decides otherwise, and that decision is an architectural one with a direct cost consequence.

**Retention as a first-class design input.** "How long must we keep this, and who says so?" has three different answers — legal/regulatory, business value, and operational cost — and they rarely agree. Get the number early; it drives partitioning strategy, storage tiering, and cost.

**The mechanisms:**
- **TTL** — Cosmos DB per-item or per-container TTL, Redis expiry, blob lifecycle policies. Free deletion, effectively.
- **Partition rotation** — partition by month, `SWITCH`/`DETACH` and drop old partitions. Near-instant, minimal log. The archetypal reason to partition (Concept 50).
- **Tiered storage** — hot → cool → cold → archive in Blob Storage, with retrieval-latency and cost trade-offs at each step.
- **Archive-then-delete** — move to Parquet in a lake (queryable cheaply via Fabric/Synapse serverless) and delete from the OLTP store. Usually the right answer for "we might need it for analysis someday."

**Soft delete deserves a warning**, because it is the default reflex in .NET codebases and it is quietly expensive. `IsDeleted = 1` means: deleted rows are still in every index; every query needs the predicate (and one missing predicate is a data-leak bug); the table never shrinks; and GDPR erasure requests are *not* satisfied by it. If you do use it, use **filtered/partial indexes** (`WHERE IsDeleted = 0`), a **named** global query filter in EF Core so it composes with tenancy, and a **hard-delete job** that actually removes rows after a retention window.

**Cost models to have roughly calibrated**, because architect rounds ask:
- **Azure SQL vCore**: compute (dominant), storage, backup storage beyond the included allowance, plus licensing unless you have Hybrid Benefit. Serverless bills per-second and auto-pauses, which is excellent for dev/test and spiky workloads and poor for steady ones.
- **Cosmos DB**: provisioned RU/s (billed on the *peak* RU/s in the hour for autoscale, between 10% and 100% of max), storage per GB, plus a minimum of 1 RU/s per GB. This is why cold data in Cosmos is expensive and why TTL plus archival is a *cost* decision, not just hygiene.
- **Storage**: pennies per GB, but egress and transaction counts are the line items that surprise people.
- **The unpriced cost**: engineer time. A second database technology can easily be the most expensive line item on the page, and it never appears on the invoice.

---

# Part G — The Azure and .NET surface

## Concept 56 — Azure SQL: tiers, Hyperscale, and HA/DR

**The product family**, which you should be able to distinguish in one line each:
- **Azure SQL Database** — single databases and elastic pools, fully PaaS, newest features first.
- **Azure SQL Managed Instance** — near-full SQL Server surface (SQL Agent, cross-database queries, CLR, Service Broker) for lift-and-shift.
- **SQL Server on Azure VMs** — IaaS, full control, full responsibility.
- **SQL database in Microsoft Fabric** — the same engine inside the Fabric analytics platform, with automatic mirroring into OneLake.

**Purchasing models:** DTU (a blended, legacy unit — recognize it, do not choose it) vs **vCore** (compute, storage and IOPS priced separately; supports Azure Hybrid Benefit and reserved capacity). vCore is the default answer.

**The service tiers, by architecture rather than by marketing:**

| Tier | Storage architecture | HA mechanism | Best for |
|---|---|---|---|
| **General Purpose** | Remote premium storage, compute and storage separated | Replace the compute node, reattach storage | Most workloads; cheapest of the three |
| **Business Critical** | Local SSD, data on the compute node | **Always On availability group** of 4 replicas, synchronous commit; includes a free read replica | Low latency, high IOPS, fastest failover, in-memory OLTP |
| **Hyperscale** | **Disaggregated**: page servers + log service + Azure Storage | Log service is the durability boundary; up to 4 HA secondaries; named replicas | Large databases, fast scaling, fast backup/restore |

**Hyperscale is worth knowing properly**, because it is the architecturally interesting one and a favourite interview topic:
- Storage grows automatically from **10 GB up to 128 TB**, and you pay for what is allocated — you never set a max size.
- Components: **compute nodes**, **page servers** (each responsible for a slice of the database, caching hot pages on local SSD), the **log service** (the durability and propagation point), and Azure Storage for the durable data files.
- **Backups are file-snapshot based**, so backup and restore are near-constant-time regardless of size — the headline operational benefit.
- Compute scales in **single-digit minutes** (sub-second for serverless) because no data moves.
- **Named replicas** enable read scale-out and HTAP scenarios with independent compute sizes.
- Compute from **2 to 192 vCores**; elastic pools up to 100 TB.

**HA and DR, distinguished** (interviewers probe this because people conflate them):
- **Zone redundancy** — replicas in different availability zones within a region. Protects against a zone failure; transparent.
- **Active geo-replication** — asynchronous readable secondaries in other regions, up to four; manual failover; nonzero RPO.
- **Auto-failover groups** — geo-replication plus a listener endpoint and automatic failover policy, so connection strings do not change. This is what you propose for regional DR.
- **None of these is a backup** (Concept 54).

---

## Concept 57 — SQL Server 2025: what is actually new and worth naming

SQL Server 2025 (17.x) reached **general availability in November 2025**. In a Microsoft-stack interview, being current here is cheap credibility. The items that actually matter architecturally:

- **Native `JSON` data type**, with JSON indexing and array/object functions in T-SQL. Changes the "we need a document database for the flexible part" conversation (Concept 35).
- **Native `VECTOR` data type** with `VECTOR_DISTANCE()` for cosine, Euclidean and dot-product, plus **approximate vector indexes based on DiskANN**, and **half-precision (`float16`)** vectors to halve storage. Vectors are stored in an optimized binary format but surfaced as JSON arrays, and updated drivers transmit them in binary over TDS rather than as JSON text.
- **Built-in AI functions** — `AI_GENERATE_EMBEDDINGS`, `AI_GENERATE_CHUNKS`, and `CREATE EXTERNAL MODEL` to define model endpoints inside the database — so embedding generation can happen where the data lives.
- **Optimized locking** (Concept 24): TID locking plus lock-after-qualification, off by default in SQL Server 2025 and enabled per database, always on in Azure SQL Database. Requires Accelerated Database Recovery; benefits most with RCSI.
- **Regular expression support** in T-SQL — a small thing that removes a common reason to pull data into application code.
- **Change event streaming** — engine-level change publication toward event brokers, relevant to Module 11's CDC discussion.

**Also worth knowing for the landscape question:** Microsoft's **DocumentDB** — an open-source, MongoDB-compatible document database built on PostgreSQL, released under MIT and now governed by the **Linux Foundation** — is the engine behind what used to be "Cosmos DB for MongoDB (vCore)" and is now surfaced as its own Azure service. It is a good example of a trend to be able to comment on: **Postgres becoming the substrate for other data models** rather than those models needing their own engines.

---

## Concept 58 — Cosmos DB: the product surface

The modelling is Concept 34. This is the surface you should be able to talk through in an Azure interview.

**Throughput modes:**
- **Provisioned RU/s** (manual) — you set it; cheapest for steady, predictable load.
- **Autoscale** — scales between 10% and 100% of a maximum; billed on the highest RU/s reached in each hour. Roughly 1.5× the per-RU rate, so it pays off when your peak-to-average ratio exceeds about 1.5.
- **Serverless** — billed per operation; for spiky, low-volume, or development workloads, with lower throughput ceilings.

**Features to have ready by name:**
- **Consistency levels** — the five-level ladder from Module 7, set at account level and relaxable per request. Session is the default and the right default.
- **Change feed** — persistent, ordered per partition; the basis for CDC, projections, outbox relays, and materialized views. Consumed via the change-feed processor in the .NET SDK or an Azure Function trigger.
- **TTL** — per container or per item; deletion costs RU but no code.
- **Indexing policy** — everything indexed by default, which costs write RU. Excluding unused paths is one of the most effective cost optimizations available, and a good specific answer to "how would you reduce Cosmos spend?"
- **`TransactionalBatch`** — atomic multi-item writes, up to 100 operations / 2 MB, **within one logical partition only**. This is Cosmos's entire transaction story, and it is why partition key choice *is* the transaction boundary.
- **Stored procedures / triggers / UDFs** in JavaScript — scoped to a single partition; generally avoid.
- **Global distribution** with multi-region writes and configurable conflict resolution (last-write-wins by default, or a custom resolver) — Module 8's multi-leader replication as a product feature.
- **Analytical store and Fabric mirroring** for HTAP without ETL.
- **Vector search**, integrated and supported in EF Core 10 as a non-experimental feature.
- **Global secondary indexes** — preview, evolved from materialized views, eventually consistent, with a catchup-gap metric (Concept 18).

**The .NET SDK details that separate users from readers:** prefer **Direct mode over TCP** for latency; the `CosmosClient` is thread-safe and expensive to construct, so register it as a **singleton**; always pass the partition key on point reads; inspect **`RequestCharge`** on every response and log it (RU is your real latency and cost metric); handle **429 Too Many Requests** with the SDK's built-in retry but also alert on the rate, because sustained 429s mean under-provisioning or a hot partition.

---

## Concept 59 — The rest of the Azure data lineup

| Service | Shape | Use when |
|---|---|---|
| **Azure Database for PostgreSQL — Flexible Server** | Managed Postgres, zone-redundant HA, read replicas, built-in PgBouncer | Postgres-first teams; extensions (PostGIS, pgvector, TimescaleDB) matter |
| **Azure PostgreSQL Elastic Clusters** | Citus-based sharded Postgres | Postgres beyond one node without changing engines |
| **Azure DocumentDB** | Managed MongoDB-compatible, built on the open-source DocumentDB (Linux Foundation) | MongoDB workloads; vertical scale and high wire-protocol compatibility |
| **Azure Database for MySQL — Flexible Server** | Managed MySQL | Existing MySQL estates |
| **Azure Managed Instance for Apache Cassandra** | Managed wide-column | Existing Cassandra; very high write throughput |
| **Azure Table Storage** | Cheap key-value/wide-column | Simple, enormous, cheap; almost no query capability |
| **Azure Blob Storage / ADLS Gen2** | Object store, hot/cool/cold/archive tiers | Files, images, documents, Parquet for the lake — and the **claim-check** pattern from Module 11 |
| **Azure Cache for Redis / Azure Managed Redis** | In-memory | Module 10 |
| **Azure Data Explorer (Kusto)** | Time-series/log analytics | High-volume telemetry with KQL |
| **Microsoft Fabric** | Unified analytics: OneLake, Lakehouse, Warehouse, mirroring | The current default answer for the analytical side (Concept 39) |
| **Azure AI Search** | Inverted index + vector + semantic ranking | Search and RAG retrieval as a projection |

**The one architectural rule to state about blobs:** do not store large binaries in the relational database. They inflate backups, evict the buffer pool, and bloat replication. Store them in Blob Storage and keep a reference (plus a SAS-token or managed-identity access path) in the database — the claim-check pattern, applied to storage.

---

## Concept 60 — ADO.NET and EF Core at architecture level

Module 19 goes deep on EF Core. What matters *here* is the set of decisions with architectural consequences.

**Connection management.** Pooling is on by default and keyed by the exact connection string — a per-tenant connection string means a pool per tenant (Concept 51). Open late, close early, and never hold a connection across a slow `await`.

**Transient-fault handling.** Cloud databases fail transiently by design: failovers, throttling, reconfiguration. Use `EnableRetryOnFailure()` (the SQL Server execution strategy) or a Polly policy (Module 25), and know the caveat from Concept 25: **with a user-initiated transaction, EF Core cannot retry automatically** and you must wrap the whole unit of work in `strategy.ExecuteAsync(...)`.

**`DbContext` lifetime.** Scoped per request/unit-of-work. It is a unit of work *and* an identity map, not a repository and not a singleton. A `DbContext` shared across threads throws; a long-lived one accumulates tracked entities and leaks memory. `DbContextFactory` is the right tool for background services and Blazor.

**Tracking.** `AsNoTracking()` for read-only queries removes the change-tracking overhead and the identity map — real gains on large result sets. `AsNoTrackingWithIdentityResolution()` when you still need object identity.

**The query traps with architectural consequences** (details in Module 19):
- **N+1** from lazy loading — the most common EF performance bug in production.
- **Cartesian explosion** from multiple `Include`s on collections; `AsSplitQuery()` is the fix.
- **Client-side evaluation** — EF Core throws on unsupported translations rather than silently materializing, which is a good thing; know how to read the exception.
- **`ExecuteUpdate` / `ExecuteDelete`** for set-based operations instead of loading entities to modify them. EF Core 10 allows `ExecuteUpdateAsync` to take a regular non-expression lambda, which makes conditional bulk updates far more readable.

**EF Core 10 (November 2025, LTS, supported to November 2028)** — the items worth naming: `LeftJoin`/`RightJoin` LINQ operators, **named query filters** (Concept 53), improved JSON-column mapping including `ExecuteUpdate` into JSON, non-experimental vector search, full support for the SQL Server/Azure SQL `VECTOR` type and `VECTOR_DISTANCE()`, Cosmos DB improvements, and redaction of literal values in SQL logs when sensitive logging is off.

**Where the ORM stops, which is the architect-level judgment:** EF Core is excellent for aggregate-shaped reads and writes — load an aggregate, change it, save it. It is the wrong tool for set-based batch operations, complex reporting queries, and anything where you need to control the plan. The mature position is **EF Core for the write model, Dapper or raw SQL for demanding reads** — which is CQRS at the persistence level, without the ceremony, and a natural bridge to Module 23.

---

## Concept 61 — Data security

Enough to answer confidently; Module 29 goes deeper.

- **Encryption at rest** — TDE, on by default in Azure SQL, with service-managed or customer-managed keys (BYOK in Key Vault). Protects stolen media and satisfies auditors; protects nothing against a compromised application.
- **Encryption in transit** — TLS enforced; `Encrypt=True` in connection strings; do not disable certificate validation.
- **Always Encrypted** — encryption/decryption in the **client driver**, so the database engine never sees plaintext. Protects against a compromised DBA or cloud operator. The cost: equality-only queries (with deterministic encryption), no ranges or sorting; the secure-enclave variant relaxes this. Know the trade, because it is the interesting part.
- **Row-Level Security** — a predicate function applied automatically; the defence-in-depth layer under multi-tenancy (Concept 53).
- **Dynamic data masking** — presentation-layer obfuscation. A convenience, not a security boundary; say so, because candidates often overstate it.
- **Managed identity instead of passwords.** Microsoft Entra ID authentication to Azure SQL and Cosmos DB, no secrets in configuration. This should be your default answer to any "how do you store the connection string?" question.
- **Network isolation** — Private Link/private endpoints, firewall rules, VNet integration; never a public endpoint with a broad IP allowlist.
- **Auditing and classification** — SQL Audit, Microsoft Purview data classification, and Defender for SQL for anomaly detection.

**GDPR and the right to erasure** is the one that has architectural consequences, and it is worth raising unprompted in a design that stores personal data. Deletion must reach backups, replicas, caches, search indexes, event logs and analytics copies — which is precisely why immutable event stores and event sourcing (Module 24) are hard here. The standard answer is **crypto-shredding**: encrypt each subject's personal data with a per-subject key and delete the key, which renders every copy unreadable without rewriting immutable history. Knowing that term and why it exists is a strong architect signal.

---

## Concept 62 — Observability for data

The four SLIs that tell you whether the data layer is healthy — the data-tier equivalent of Module 11's messaging SLIs:

1. **Latency per query class** (p50/p95/p99), not a single database-wide average.
2. **Throughput and saturation** — transactions/s, and the *utilization* of the binding resource: CPU, IOPS, worker threads, connections, or RU/s.
3. **Errors** — deadlocks, timeouts, 429s, constraint violations, connection-pool exhaustion.
4. **Replication lag and transaction age** — the two leading indicators (Concepts 29, 49).

**The tools by platform:**
- **SQL Server / Azure SQL** — **Query Store** (the single most valuable feature for query performance work: historical plans, regressions, forced plans), wait statistics (`sys.dm_os_wait_stats` — waits tell you *what the server is waiting on*, which is the fastest route to a diagnosis), `sys.dm_exec_query_stats`, index usage DMVs, the deadlock extended-events session, and Azure SQL Insights.
- **PostgreSQL** — `pg_stat_statements`, `pg_stat_activity` (watch for `idle in transaction`), `pg_stat_user_indexes`, `auto_explain`, and `EXPLAIN (ANALYZE, BUFFERS)`.
- **Cosmos DB** — `RequestCharge` per operation, normalized RU consumption per partition (the hot-partition detector), 429 rate, and the per-partition metrics split.

**Tracing.** OpenTelemetry has database semantic conventions; `db.system`, `db.statement` (parameterized, never with literal values), and duration as a child span of the request. The payoff is being able to answer "which upstream endpoint caused this table's load?" from a trace rather than a guess. Module 28 covers this properly.

**The architect-level point:** instrument the *query*, not just the *server*. A healthy-looking server with one pathological query class is the normal failure mode, and dashboards showing only CPU and memory will show you a calm system while users time out.

---

# Part H — Judgment

## Concept 63 — Anti-patterns worth naming on sight

| Anti-pattern | Why it is wrong | What to do instead |
|---|---|---|
| **Shared database across services** | The schema becomes an undocumented integration contract nobody can change; you have a distributed monolith with extra steps | One owner per schema; integrate via APIs or events |
| **EAV (entity-attribute-value)** | Reinvents a schemaless store inside a relational one, with none of the benefits and no query plan the optimizer can reason about | A JSON column with a real type where flexibility is genuinely needed |
| **Random GUID clustered key** | Page splits, fragmentation, cache misses, and 8 extra bytes in every nonclustered index | `bigint` identity, UUIDv7 (`Guid.CreateVersion7()`), or `NEWSEQUENTIALID()` |
| **`SELECT *` in application code** | Reads columns you do not need, breaks covering indexes, and breaks when the schema changes | Project exactly the columns needed |
| **An index per WHERE column** | Index intersection instead of one seek; multiplied write cost | One well-ordered composite index (Concept 12) |
| **`WITH (NOLOCK)` everywhere** | Dirty reads *and* missing or duplicated rows | Enable RCSI |
| **Soft delete on everything, forever** | Indexes carry dead rows; one missing predicate is a data leak; GDPR is not satisfied | Filtered indexes, named query filters, a hard-delete job with a retention window |
| **The database as a message queue** | Polling load, lock contention, no dead-lettering — though `SKIP LOCKED` makes it *viable* at small scale | A broker (Module 11); or accept it deliberately at low volume and say why |
| **Network calls inside a transaction** | Locks held at the mercy of someone else's latency (Concept 29) | Commit, then act; outbox for the cross-system part |
| **`Database.Migrate()` at application startup** | Racing instances, no review gate, no rollback plan | Idempotent scripts in the deployment pipeline |
| **ORM-generated schema as the design** | The physical model is an accident of the class model | Design the schema; map the classes to it |
| **Single tenant-less pooled table** | Cross-tenant leakage one missing predicate away | `TenantId` leading every index, global query filters, RLS |
| **Analytics on the OLTP primary** | Evicts the working set; long snapshots block version cleanup | Replica, columnstore index, or CDC to a lake (Concept 39) |
| **Storing files in the database** | Backup bloat, buffer-pool eviction, replication cost | Blob Storage + a reference (claim check) |
| **Unbounded tables** | Every query slows as history accumulates; maintenance windows grow | Retention policy, partition rotation, TTL, archival |
| **A distributed transaction because the boundary is wrong** | All of Part E | Move the boundary first (Concept 47) |

---

## Concept 64 — Choosing a database: the decision table

Two questions resolve most of it:

1. **What must be transactionally consistent together?** That defines the aggregate, and therefore the minimum unit of storage.
2. **Which queries must be fast, and do I know all of them?** If yes, a partition-key-shaped store is viable. If no, you need the relational model's ability to answer questions you have not thought of.

| If the requirement is… | Reach for… | Because |
|---|---|---|
| Cross-entity invariants, ad-hoc queries, reporting | **Relational** (Azure SQL, Postgres) | Joins, constraints, transactions, and a query optimizer |
| A well-known set of aggregate reads/writes at very high scale | **Document** (Cosmos DB) | Single-partition operations with predictable RU cost |
| Sub-millisecond point reads on hot data, ephemeral | **Key-value / cache** (Redis) | Module 10 |
| Very high write throughput, time-ordered, per-key reads | **Wide-column / time-series** (Cassandra, ADX, Timescale) | LSM storage and time partitioning |
| Relevance, fuzzy matching, faceting | **Search index** (Azure AI Search) as a projection | Inverted index and scoring; never a source of truth |
| Multi-hop relationship traversal as the core feature | **Graph** | Adjacency as a first-class structure |
| Semantic similarity over embeddings | **Vector index** — in your existing database if the vectors are attributes of stored records | Avoids a second store and keeps filters/joins/transactions |
| Multi-region active-active with strong consistency | **Distributed SQL** (Spanner, CockroachDB) or Cosmos DB multi-region | Consensus-replicated shards; pay in commit latency |
| Scans and aggregates over history | **Columnar / lakehouse** (Fabric, Synapse, Parquet) | 5–10× compression and column pruning |

**The meta-answer to give in an interview:** *"I would start with the transaction boundaries and the access patterns, size the data, and check whether one relational instance covers it — because it usually does, and everything else is a specific pattern the relational store cannot serve at acceptable cost. When I add a store, I want it to be a projection with an owner and a documented rebuild path."*

---

## Concept 65 — Migrating a live database without downtime

The data version of the strangler fig (Module 32). Six phases, and the discipline is that **every phase is independently reversible**:

1. **Replicate.** Start continuous replication from old to new — CDC, change feed, logical replication, or an ETL job. Let it run until lag is steady and small.
2. **Backfill.** Copy history in throttled batches, with progress tracking and resumability. Monitor the source's load while it runs.
3. **Verify continuously.** Row counts, checksums, and spot-comparison of samples. Build this before you need it, because it is what gives you the confidence to cut over.
4. **Shadow read.** Route a percentage of reads to the new store, compare results against the old, log mismatches, serve the old result. **This is the phase teams skip and the phase that finds the bugs.**
5. **Cut over writes.** Either a brief write freeze (seconds — usually acceptable, and vastly simpler) or dual-write with a designated source of truth. Dual-write is the dual-write problem (Module 11), so if you use it, one store must be authoritative and the other reconciled.
6. **Decommission** — after a deliberate soak period, with the old store kept readable and restorable for a defined window.

**What to say about rollback:** *"At every phase before the write cutover, rollback is 'stop the new path'. After the cutover it is 'reverse the replication direction', which is why I set that up before cutting over rather than after."* Having a rollback story for the irreversible step is the mark of someone who has done this.

---

## Concept 66 — When to say no

The highest-value architect sentences in a data design review:

- **"That should be one aggregate."** Most proposed distributed transactions are a boundary error (Concept 47).
- **"This fits on one instance for the next four years — here is the arithmetic."** Backed by Concept 9 and Concept 38.
- **"We do not need a second database for that."** A JSON column, a filtered index, a materialized view, or `pgvector`/`VECTOR` usually covers it (Concept 37).
- **"Sharding is a permanent complexity increase; let us exhaust rungs 1–6 first."** (Concept 48.)
- **"That is a cache, so tell me the staleness bound."** Applies to read replicas, denormalized copies, global secondary indexes, and search indexes alike.
- **"Event sourcing solves a problem we do not have."** Keep publishing events; do not make the event log the system of record without a reason (Module 24).
- **"We cannot delete this user's data under that design."** Raise GDPR erasure before the immutable log is built, not after.
- **"What is the restore time, and when did we last measure it?"** (Concept 54.)

---

# Putting it together

## Worked example 1 — "Design the data layer for a multi-tenant B2B SaaS (10k tenants, 5M users)"

Narrate it in this order:

1. **Aggregates and boundaries first.** Tenant, User, Project, Document, AuditEvent. Which must be atomic together? Project + its members, yes. Project + AuditEvent, no — audit is append-only and can be eventual (Concept 32).
2. **Size it.** 5M users × 1 KB ≈ 5 GB. 10k tenants × 50k documents × 2 KB ≈ 1 TB of metadata, with the actual files in Blob Storage. Working set: the active tenants in a given hour — call it 10% — so ~100 GB against an instance with 128–256 GB RAM. **This fits on one Azure SQL Hyperscale database**, so the design is "one database, sharding-ready," not "shard now" (Concepts 9, 38).
3. **Multi-tenancy: pooled, with a silo escape hatch** (Concept 53). `TenantId` as the leading column of every index and every clustered key; EF Core **named** global query filters plus Row-Level Security as defence in depth; tenant→connection-string resolution behind an interface so the biggest tenants can be moved to their own database without an application change.
4. **Keys.** `bigint` identity clustered key for storage efficiency; a UUIDv7 public identifier with a unique nonclustered index for external contracts (Concept 11).
5. **Indexing.** Composite indexes led by `TenantId`, then the equality predicate, then the range/sort column; cover the two or three hottest list queries with `INCLUDE` (Concepts 12–13).
6. **Files to Blob Storage**, references in SQL — no binaries in the database (Concept 59).
7. **Reads.** Session-consistent by default: primary for anything following a write in the same flow; a read replica for reporting and exports; staleness stated in the UI (Concept 49).
8. **Audit and integration.** An append-only `AuditEvent` table partitioned by month with partition rotation for retention; outbound integration via the outbox pattern, not direct calls inside transactions (Concepts 46, 50, 55).
9. **Migrations.** Expand/contract, scripted in the pipeline, with backfills as throttled background jobs. Be explicit that this is the operational cost of the pooled model (Concept 52).
10. **DR.** Auto-failover group for regional failure; PITR at 35 days for logical error; a **measured** restore time; and per-tenant restore explicitly called out as a limitation of pooling — with the answer being "silo the tenants who contractually require it" (Concept 54).

## Worked example 2 — "Orders and payments: what is the transaction boundary?"

The question is really Part E, and the narration should go:

1. **Name the aggregates.** `Order` (header + lines) is one aggregate. `Payment` is another. `Inventory` is a third. They belong to different services.
2. **What must be atomic?** Order header + lines + an outbox row: one local transaction. Everything else is cross-aggregate (Concept 47, step 1 — and say that you checked whether Order and Payment should be one aggregate, and concluded no because they have different lifecycles and owners).
3. **Do not reach for 2PC.** State why in one line: unanimity, blocking in-doubt transactions, availability as a product, and — decisively — MSDTC is Windows-only while these services run in Linux containers (Concepts 41, 42).
4. **Design the saga**, orchestrated because there are compensations and timeouts: `ReserveInventory` → `AuthorizePayment` → `ConfirmOrder` → `CapturePayment` → `CreateShipment` (Concept 44).
5. **Use reservations, not commitments, for the first two steps** (Concept 45). Inventory reserved with a TTL; the card **authorized**, not captured. Compensation becomes "let the hold expire," which is safe and self-healing.
6. **Order irreversible steps last.** Capture and shipment after the pivot; the confirmation email after everything (Concept 44).
7. **Semantic lock:** `Order.Status = PendingPayment` so the rest of the system knows the state is in flight and no other process acts on it.
8. **Idempotency everywhere.** Idempotency key written in the same transaction as the effect, protected by a unique index; the payment provider's own idempotency key for the external charge (Concept 46).
9. **State the isolation cost out loud:** "between reservation and capture, other readers can see an order that exists and is not paid. That is visible in the UI as a pending state, and the window is bounded by the 15-minute authorization TTL." Volunteering the anomaly is the senior move.

## Worked example 3 — "This query got slow. Walk me through it."

Use the Concept 19 narrative, and make the *ordering* the visible part of the answer:

1. **"Slow for whom, since when, and p50 or p99?"** Establish whether this query is the cause or a victim.
2. **Pull the actual plan and the waits.** If the top wait is `PAGEIOLATCH`, it is I/O and possibly a working-set problem; `LCK_M_X` is blocking, so the problem is another transaction; `SOS_SCHEDULER_YIELD` is CPU; `RESOURCE_SEMAPHORE` is memory-grant pressure. **Naming waits by class is a strong, specific signal.**
3. **Estimated vs actual rows.** Find the first operator where they diverge (Concept 14).
4. **Name the physical cause.** "A nonclustered index seek returning 400k rows, then 400k key lookups — past the tipping point, so the optimizer should have scanned, and the reason it did not is a stale statistic on an ascending date column."
5. **Fix at the cheapest level.** Update statistics; if the query is hot, make the index covering; if the plan flips under different parameters, it is parameter sniffing and Query Store forcing is the operational fix (Concept 15).
6. **Check the systemic version.** "Was this the only query, or did the table just cross the point where its working set stopped fitting in the buffer pool? If the latter, the index fix buys time and the real decision is memory, archival, or partitioning" (Concepts 7, 55).
7. **Guard it.** A Query Store baseline and an alert on regression, so the next occurrence is detected rather than reported by a user.

---

## Common questions and what a strong answer contains

**"SQL or NoSQL for this?"** Refuse the binary. Answer on the axes (Concept 30): transaction scope, query predictability, scale shape, schema ownership, operational cost. Then size it (Concept 9) and say what would change your mind.

**"How do indexes work?"** A sorted copy answering three questions — seek, cover, order (Concept 10). Then B+ tree fanout and the "3–4 logical reads" number. Then the cost side, unprompted: write amplification, lock surface, and space.

**"Why is this query slow?"** The Concept 19 narrative, and name a wait type. The ordering of your diagnosis *is* the answer.

**"How would you index this query?"** Equality columns, then range, then order-by; `INCLUDE` the projected columns; then check what existing index it makes redundant (Concepts 12–13).

**"What are the isolation levels?"** Do not recite the ladder — derive it from the anomalies (Concepts 21–22). Then the part that scores: snapshot isolation does not prevent write skew, here is the on-call example, and here are four ways to fix it.

**"What is MVCC?"** Versions instead of overwrites, readers do not block writers; then the cost — Postgres bloat and vacuum, SQL Server version store and tempdb — and the connection to long transactions (Concepts 23, 29).

**"How do you handle concurrent updates to the same record?"** Optimistic concurrency with `rowversion`/`xmin` and the three resolution strategies; the atomic conditional `UPDATE` for counters; and the distinction that optimistic concurrency does not solve write skew (Concept 26).

**"Explain two-phase commit and why we avoid it."** The protocol in four sentences, then the in-doubt participant holding locks while the coordinator is down, then availability as a product of participants. Then the nuance that distinguishes you: Spanner-class systems run 2PC over consensus groups, which removes the blocking failure (Concepts 41, 43).

**"2PC or saga?"** The decision ladder (Concept 47) — move the boundary, one transaction, make it unnecessary, reservation, saga, 2PC last. Then sagas as ACD with no isolation, the countermeasures, and compensation as a new fact rather than an erasure.

**"How do you keep two services' data consistent?"** Outbox + at-least-once + idempotent consumer, with the unique index as the dedup mechanism (Concept 46). Then name the staleness window.

**"How would you shard this?"** First say what you would do before sharding (Concept 48). Then the shard key criteria, then cross-shard queries, rebalancing, per-shard migrations, and the routing indirection you would build on day one (Concept 50).

**"How do you migrate a schema with zero downtime?"** Expand/contract with the six steps, four deployments, and the "old and new code run simultaneously" constraint that explains why (Concept 52).

**"How do you handle multi-tenancy?"** Silo/bridge/pool with the trade table, then the hybrid, then per-tenant restore as the requirement that most often forces the decision, then the defence-in-depth enforcement story (Concept 53).

**"What is your backup strategy?"** RPO and RTO first. Then "replication is not a backup," then PITR, then the **measured** restore time and when you last tested it (Concept 54).

**"How does Cosmos DB partitioning work?"** Logical vs physical partition, 20 GB / 50 GB / 10k RU/s, the four partition-key criteria, hierarchical partition keys for the multi-tenant case, and `TransactionalBatch` being partition-scoped — which makes the partition key the transaction boundary (Concept 34).

**"What is the difference between serializable and linearizable?"** Transaction ordering vs single-object recency; orthogonal; strict serializability is both; Spanner buys it with commit wait (Concept 28). This is the highest-leverage vocabulary answer in the module.

**"When would you use a cache versus a read replica versus an index?"** All three are denormalized copies. An index is transactional and free of staleness; a replica is asynchronous with a lag you must measure; a cache is asynchronous with a TTL you choose. Pick by whether the write path can afford the cost and whether the read path can tolerate the staleness (Concepts 10, 18, 49).

**"When would you *not* add another database?"** Concept 37, with the JSON-column and `pgvector`/`VECTOR` counterexamples that make the point concrete.

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "NoSQL scales better" | Names the actual trade: giving up joins and cross-entity transactions to make every operation single-partition |
| Choosing a store by familiarity or fashion | Sizes the data, lists access patterns, checks whether one relational instance covers it, and says what would change the answer |
| Reciting ACID | Says atomicity is about failure, isolation is about concurrency, consistency is the domain's job, durability is about replicas |
| Reciting isolation levels | Derives them from anomalies, and identifies write skew as the one snapshot isolation misses |
| "We'll use serializable to be safe" | Scopes serializable to the specific paths with cross-row invariants; offers materialized conflicts and constraints as alternatives |
| Treating `NOLOCK` as a performance tip | Knows it can return duplicate and missing rows, and proposes RCSI instead |
| Adding an index per WHERE column | One composite in equality/range/order sequence; checks what becomes redundant; states the write cost |
| "Add an index" as the universal fix | Asks whether it can seek, cover, or order — and whether the real problem is the working set, the plan, or a lock |
| Ignoring statistics | Compares estimated vs actual rows and diagnoses skew, staleness, correlation, or non-sargable predicates |
| GUID primary keys with no comment | Explains page splits and nonclustered-index bloat; proposes UUIDv7, `NEWSEQUENTIALID()`, or a sequential clustered key |
| Proposing `TransactionScope` across services | Knows MSDTC is Windows-only, opt-in since .NET 7, and promotes silently — and does not propose it for containers |
| "Just use a distributed transaction" | Works the ladder: move the boundary → one transaction → make it unnecessary → reservation → saga → 2PC |
| Knowing "saga" as a word | ACD not ACID; the four countermeasures; compensation as a new business fact; irreversible steps ordered last |
| Saga with compensations everywhere | Reaches for TCC/reservations with a TTL so the failure path is self-healing |
| `SaveChanges` then publish | Outbox, relay with `SKIP LOCKED`/`READPAST`, idempotent consumer with a unique index |
| "We'll shard it" | Names six cheaper rungs first, then the shard key criteria and the resharding plan |
| Confusing partitioning and sharding | One engine vs many; partition elimination and sliding windows vs write scale-out |
| Ignoring connection limits | Knows Postgres connections are processes, `Max Pool Size` defaults to 100, and serverless multiplies both |
| "We'll add a read replica" | Classifies reads, states the lag in seconds, and solves read-your-writes explicitly |
| Big-bang schema migrations | Expand/contract, four deployments, backward compatibility for one cycle, throttled backfills |
| `Database.Migrate()` at startup | Idempotent scripts in the pipeline with a review gate |
| Network calls inside a transaction | Treats transaction duration as an SLI; commits first, then acts via the outbox |
| Soft delete everywhere | Filtered indexes, named query filters, a hard-delete job, and GDPR raised before the immutable log is built |
| "We have geo-replication, so we're covered" | Distinguishes HA from DR from backup; logical errors replicate; PITR is the only answer |
| "We have backups" | Quotes the **measured** restore time and when it was last tested |
| Analytics on the OLTP primary | Explains buffer-pool eviction and long snapshots; proposes replica, columnstore, or CDC with a stated staleness contract |
| Vague about Cosmos limits | 20 GB logical, 50 GB / 10k RU/s physical, 2 MB item, 100-op batch, hierarchical keys for multi-tenancy |
| Unaware of current platform state | Knows SQL Server 2025 GA'd with native JSON and `VECTOR`+DiskANN, optimized locking, EF Core 10 LTS, Postgres 18's async I/O / UUIDv7 / skip scan |
| Never pushing back | Volunteers "this is one aggregate," "this fits on one instance," and "we do not need a second store" before being asked |

---

## Practice exercises

**Exercise 1 — Feel the B-tree.** In SQL Server or Postgres, create two tables with 5M rows: one with a `bigint` identity clustered key, one with a `NEWID()`/`gen_random_uuid()` clustered key. Insert in both, measure insert throughput, index size, page density, and fragmentation. Then repeat with `NEWSEQUENTIALID()` / `uuidv7()`. **This is the highest-value exercise in Part A** — it converts Concept 11 from a rule you repeat into a number you have measured.

**Exercise 2 — Index archaeology.** Take a real table with several indexes. Use `sys.dm_db_index_usage_stats` (or `pg_stat_user_indexes`) to find indexes with zero seeks and heavy update counts. Identify redundant leftmost-prefix duplicates. Drop them in a copy, re-run your workload, and measure both write throughput and total size. Write the one-paragraph justification you would put in a PR.

**Exercise 3 — Cause every anomaly deliberately.** Two connections, one table. Reproduce, in order: dirty read (Read Uncommitted), non-repeatable read, phantom, lost update, read skew, and **write skew under snapshot isolation**. Then fix the write skew four ways — serializable, a materialized conflict row, `SELECT ... FOR UPDATE`/`UPDLOCK`, and a constraint — and note which ones cost throughput. **Do this one properly; it permanently fixes Concepts 21–22 in your head.**

**Exercise 4 — Break the dual write, then fix it.** (Carried over from Module 11 — if you did it there, extend it.) Now add the database-side details: three competing relay instances, `SKIP LOCKED`/`READPAST`, an `IdempotencyKeys` unique index, and a pruning job. Prove duplicate suppression with a test that calls the handler twice and asserts one effect.

**Exercise 5 — Build a TCC reservation.** Implement inventory reservation with a TTL, a sweeper, idempotent confirm/cancel, and `available = total − reserved − sold`. Then deliberately kill the process between reserve and confirm, and show the reservation expiring and stock returning without any compensation message being delivered. This makes Concept 45 something you have built.

**Exercise 6 — Expand/contract for real.** Rename a column on a table with 10M rows using all six steps, with a rolling deployment of two app instances running old and new code simultaneously. Time the backfill, watch the log growth, and measure replication lag while it runs. Then do it the naive way on a copy and observe the blocking. **This is the closest exercise in the module to an actual architect-round question.**

**Exercise 7 — Find the tipping point.** Write a query that uses a nonclustered index seek with key lookups. Vary the selectivity of the predicate and find the row count at which the optimizer switches to a scan. Then make the index covering and find the new behaviour. Record the numbers — being able to say "on my data it tipped around 1.8%" is a memorable, credible detail.

**Exercise 8 — Cosmos partitioning under pressure.** Create a container partitioned by a low-cardinality key and load data until you see 429s and uneven normalized RU consumption per partition. Then redesign with a hierarchical partition key ending in `/id` and compare RU charge for the same queries. Log `RequestCharge` throughout. This makes Concept 34 concrete rather than memorized.

**Exercise 9 — Connection-pool exhaustion, on purpose.** Build an endpoint that opens a `SqlConnection`, starts a transaction, and awaits a 5-second HTTP call before committing. Drive 200 concurrent requests. Observe pool-timeout exceptions, then watch blocked sessions on the server. Fix it by moving the HTTP call outside the transaction, and re-measure. This teaches Concepts 29 and 51 in a way no reading will.

**Exercise 10 — The architect write-up (one page).** For a system you know: list every table over 10M rows with its growth rate and retention policy; every index with its justification; every transaction that spans more than one aggregate; every denormalized copy with its staleness bound and rebuild path; the RPO/RTO and the last measured restore time; and the point at which the current design stops working, with the number that defines it. Mark what you would change first. This is very close to a real architect take-home, and it will find something broken.

---

## Free resources

### Papers and primary sources

| Resource | What it covers | Why read it |
|---|---|---|
| [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) — Berenson, Bernstein, Gray, Melton, O'Neil & O'Neil, 1995 | Why the ANSI phenomena are ambiguous, and where snapshot isolation actually sits | **The source for Concept 22.** Short, readable, and citing it by name is a strong signal |
| [Serializable Isolation for Snapshot Databases](https://www.cs.nyu.edu/courses/fall22/CSCI-GA.2434-001/p729-cahill.pdf) — Cahill, Röhm & Fekete, 2008 | The SSI algorithm PostgreSQL implements | How you get true serializability without 2PL (Concept 27) |
| [Life Beyond Distributed Transactions: An Apostate's Opinion](https://queue.acm.org/detail.cfm?id=3025012) — Pat Helland | Entity-scoped transactions, at-least-once messaging, "almost-infinite scaling" | **The clearest argument for designing around distributed transactions** (Concept 40) |
| [Idempotence Is Not a Medical Condition](https://queue.acm.org/detail.cfm?id=2187821) — Pat Helland | Why at-least-once plus idempotence is the practical contract | The substrate under Concept 46 |
| [Sagas](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) — Garcia-Molina & Salem, SIGMOD 1987 | Long-lived transactions with compensations | The origin of Concept 44; short and readable |
| [Spanner: Google's Globally-Distributed Database](https://research.google/pubs/pub39966/) — Corbett et al., OSDI 2012 | TrueTime, commit wait, external consistency, 2PC over Paxos | **The paper behind Concept 43**; the commit-wait idea is worth the read alone |
| [Large-scale Incremental Processing Using Distributed Transactions and Notifications](https://research.google/pubs/pub36726/) — Peng & Dabek (Percolator), OSDI 2010 | Client-driven 2PC with snapshot isolation over Bigtable | The other model of distributed transactions; TiDB descends from it |
| [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — DeCandia et al., SOSP 2007 | Consistent hashing, quorums, vector clocks, eventual consistency | One of the two papers that created the NoSQL category (with Bigtable) |
| [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/pub27898/) — Chang et al., OSDI 2006 | SSTables, LSM-style storage, wide-column model | The origin of Concept 4's shape and the wide-column family |
| [Designing Access Methods: The RUM Conjecture](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf) — Athanassoulis et al., EDBT 2016 | Read/Update/Memory overheads as a three-way trade | The formal statement of Concept 5 |
| [ARIES: A Transaction Recovery Method...](https://cs.stanford.edu/people/chrismre/cs345/rl/aries.pdf) — Mohan et al., 1992 | WAL, logging, checkpointing, redo/undo recovery | The basis of Concept 6; skim the first sections for the model |
| [Calvin: Fast Distributed Transactions for Partitioned Database Systems](https://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) — Thomson et al., SIGMOD 2012 | Deterministic transaction ordering as an alternative to 2PC | The third shape in Concept 43 |
| [Jepsen analyses](https://jepsen.io/analyses) | Empirical testing of what databases actually guarantee under partition | The evidence behind "your database's default isolation is weaker than you think" |

### Books, courses, and explainers (free online)

| Resource | What it covers |
|---|---|
| [Use The Index, Luke!](https://use-the-index-luke.com/) — Markus Winand | **The best free indexing resource that exists.** Column order, covering, sargability, joins, sorting, paging — with SQL Server, Postgres, Oracle and MySQL specifics side by side. If you read one thing from this module, read this |
| [Modern SQL](https://modern-sql.com/) — Markus Winand | Window functions, `MERGE`, `WITH`, `FILTER`, temporal features — modern SQL most codebases never use |
| [Database Internals — companion material](https://www.databass.dev/) — Alex Petrov | Chapter-level summaries of B-trees, LSM trees, WAL, distributed transactions. The book itself is the deepest treatment of Part A |
| [CMU 15-445 Database Systems](https://15445.courses.cs.cmu.edu/) — Andy Pavlo (full lectures free on [YouTube](https://www.youtube.com/@CMUDatabaseGroup)) | **The single best free course on this material.** Storage, indexes, concurrency control, recovery, query execution |
| [CMU 15-721 Advanced Database Systems](https://15721.courses.cs.cmu.edu/) | In-memory engines, MVCC implementations, vectorized execution, HTAP — the graduate follow-on |
| [The Internals of PostgreSQL](https://www.interdb.jp/pg/) — Hironobu Suzuki | Free online book: heap tuples, MVCC, vacuum, buffer manager, WAL, the planner. Exceptional depth for Concept 23 |
| [PostgreSQL 18 release notes](https://www.postgresql.org/docs/18/release-18.html) | Async I/O, `uuidv7()`, B-tree skip scan, virtual generated columns, `NOT NULL` without a full scan |
| [Brent Ozar's free SQL Server training](https://www.brentozar.com/training/) and [first responder kit](https://www.brentozar.com/first-aid/) | `sp_Blitz`, `sp_BlitzIndex`, `sp_BlitzCache` — the practical diagnostic toolkit, plus a lot of free video |
| [SQLPerformance.com](https://sqlperformance.com/) — Paul White, Aaron Bertrand et al. | Deep, correct articles on plans, cardinality estimation, and the optimizer's internals |
| [Microsoft Research: SQL Server Query Optimizer / Query Processing Architecture Guide](https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide) | How plans are produced, cached, and reused; the basis of Concepts 14–15 |
| [microservices.io — data patterns](https://microservices.io/patterns/data/database-per-service.html) — Chris Richardson | Database-per-service, saga, CQRS, API composition, transactional outbox — the pattern names interviewers use |
| [Martin Fowler: Polyglot Persistence](https://martinfowler.com/bliki/PolyglotPersistence.html) and [DDD Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html) | Concepts 32 and 37, from the source of the vocabulary |
| [Kleppmann's DDIA chapter notes and talks](https://martin.kleppmann.com/talks.html) | Free talks covering transactions, isolation, and distributed consistency — a good companion to the book |
| [AWS Builders' Library: Challenges with distributed systems](https://aws.amazon.com/builders-library/challenges-with-distributed-systems/) | Why partial failure makes cross-node atomicity expensive; good framing for Part E |
| [PlanetScale blog: how B-trees and LSM trees work](https://planetscale.com/blog/btrees-and-database-indexes) | Clear, visual explanations of Concepts 3–5 |
| [pgvector](https://github.com/pgvector/pgvector) | HNSW and IVFFlat indexes, distance operators, and the trade between recall and speed |

### Azure and .NET documentation

| Resource | What it covers |
|---|---|
| [Azure Architecture Center — Choose a data store](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/data-store-overview) | **The decision guide behind Concept 64**, in the vocabulary an Azure design review uses |
| [Data partitioning guidance](https://learn.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning) · [Data partitioning strategies](https://learn.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning-strategies) | Horizontal/vertical/functional partitioning per service, with Azure specifics |
| Azure patterns: [Saga](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga) · [CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) · [Materialized View](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view) · [Index Table](https://learn.microsoft.com/en-us/azure/architecture/patterns/index-table) · [Sharding](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding) · [Claim-Check](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check) | The named patterns of this module as Microsoft names them |
| [Transactional Outbox with Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-outbox-cosmos) | A concrete outbox implementation using the change feed as the relay |
| [Azure SQL Hyperscale architecture](https://learn.microsoft.com/en-us/azure/azure-sql/database/hyperscale-architecture) · [Hyperscale service tier](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale) | Page servers, the log service, 10 GB–128 TB, snapshot backups — the source for Concept 56 |
| [Azure SQL business continuity & DR](https://learn.microsoft.com/en-us/azure/azure-sql/database/business-continuity-high-availability-disaster-recover-hadr-overview) · [Automated backups](https://learn.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview) · [Failover groups](https://learn.microsoft.com/en-us/azure/azure-sql/database/failover-group-sql-db) | RPO/RTO per feature — read this before any DR question (Concept 54) |
| [Transaction locking and row versioning guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide) | **The definitive reference for Concepts 22–24**: isolation levels, lock modes, escalation, RCSI vs SNAPSHOT |
| [Optimized locking](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optimized-locking) | TID locking and lock-after-qualification, prerequisites, and per-platform availability |
| [Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store) · [Intelligent Query Processing](https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing) | Plan history, forcing, automatic correction, and parameter-sensitive plan optimization (Concept 15) |
| [SQL Server 2025 what's new](https://learn.microsoft.com/en-us/sql/sql-server/what-s-new-in-sql-server-2025) · [`vector` data type](https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type) · [`json` data type](https://learn.microsoft.com/en-us/sql/t-sql/data-types/json-data-type) | The Concept 57 list, from the primary source |
| [Cosmos DB partitioning](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview) · [Hierarchical partition keys](https://learn.microsoft.com/en-us/azure/cosmos-db/hierarchical-partition-keys) · [Service quotas](https://learn.microsoft.com/en-us/azure/cosmos-db/concepts-limits) | The limits of Concept 34, authoritative and current |
| [Cosmos DB request units](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units) · [Change feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed) · [Indexing policy](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy) · [Global secondary indexes (preview)](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/global-secondary-indexes) | The Concept 58 surface |
| [Cosmos DB modeling & partitioning (real-world example)](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/model-partition-example) | A worked end-to-end NoSQL data-modelling exercise — the best free practice for Concept 33 |
| [EF Core: What's new in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew) · [Concurrency conflicts](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) · [Connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) · [Migrations in production](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying) | Concepts 26, 52 and 60, with code |
| [EF Core performance guidance](https://learn.microsoft.com/en-us/ef/core/performance/) | Tracking, split queries, compiled queries, `ExecuteUpdate`/`ExecuteDelete` — preview of Module 19 |
| [.NET data access architecture in the microservices e-book](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/) (free) | Database-per-service, integration events, outbox, idempotency, with real .NET code |
| [Architecting Cloud Native .NET Applications for Azure](https://learn.microsoft.com/en-us/dotnet/architecture/cloud-native/) (free e-book) | Data-store chapter: relational vs NoSQL on Azure, Cosmos, caching, consistency |
| [System.Transactions / TransactionScope](https://learn.microsoft.com/en-us/dotnet/api/system.transactions.transactionscope) and [dotnet/runtime #71769 (cross-platform distributed transactions)](https://github.com/dotnet/runtime/issues/71769) | The Concept 42 reality, including the open cross-platform request |
| [Azure Database for PostgreSQL Flexible Server docs](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/) · [DocumentDB project](https://github.com/documentdb/documentdb) | The Concept 59 lineup |
| [Dapper](https://github.com/DapperLib/Dapper) · [DbUp](https://dbup.readthedocs.io/) · [Testcontainers for .NET](https://dotnet.testcontainers.org/) | The micro-ORM, the migration runner, and the tool that makes Exercises 1–9 easy to run locally |

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What a database gives you | A bundle of guarantees paid for by a physical layout and a concurrency-control mechanism |
| B-tree vs LSM | In-place, read-optimized, 3–4 logical reads vs sequential-append, write-optimized, compaction-bound; RUM conjecture |
| Why a query is slow | Measure → plan → estimated vs actual → physical cause → cheapest fix; name a wait type |
| Indexing a query | Equality, then range, then order-by; `INCLUDE` the projection; check what becomes redundant |
| Why an index isn't used | Non-sargable predicate, implicit conversion, past the key-lookup tipping point, or a bad estimate |
| The cost of an index | Write amplification, lock surface, space (often 1–3× the table across all indexes), maintenance log volume |
| Durability | The log is fsynced on enough independent failure domains — not "the page was written" |
| Buffer pool | Working set vs RAM; the miss cliff is the usual cause of "it got slow and nothing changed" |
| ACID | Atomicity = failure, Isolation = concurrency, Consistency = your invariants, Durability = replicas |
| Isolation levels | Derive from the anomalies; snapshot ≠ serializable; write skew is the gap |
| Write skew | Read a set → check an invariant → write a different row in it; four fixes: serializable, materialize the conflict, lock the read set, constraint |
| MVCC | Versions not overwrites; readers don't block writers; the cost is bloat/version store and it's blocked by long transactions |
| Optimistic concurrency | `rowversion`/`xmin` token, `DbUpdateConcurrencyException`, three resolution strategies; doesn't solve write skew |
| Deadlocks | Access order, lock upgrades, missing indexes, long transactions — and always retry |
| Serializable vs linearizable | Transaction ordering vs single-object recency; strict serializability is both; Spanner pays commit wait for it |
| Long transactions | Locks + bloat + WAL retention + pool exhaustion; treat duration as an SLI |
| SQL vs NoSQL | The axes: transaction scope, query predictability, scale shape, schema ownership, ops cost — then size it |
| Aggregates | The unit of atomicity and of storage; aggregate → transaction boundary → service boundary |
| Cosmos partitioning | 20 GB logical / 50 GB + 10k RU/s physical; four key criteria; hierarchical keys ending in `/id`; batch is partition-scoped |
| Secondary indexes at scale | Local (scatter-gather reads) vs global (async, eventually consistent) — a GSI is a replica with a different key |
| Distributed SQL | Shards as consensus groups + 2PC over them; the cost is commit latency, not availability |
| 2PC | Unanimity → blocking; in-doubt participants hold locks; availability is the product of participants |
| MSDTC in .NET | Windows-only, opt-in since .NET 7, promotes silently, needs `TransactionScopeAsyncFlowOption.Enabled` |
| Spanner | 2PC over Paxos groups + TrueTime commit wait = external consistency |
| Saga | ACD, no isolation; semantic lock, commutative updates, pessimistic view, re-read; compensation is a new fact |
| TCC | Reserve with a TTL, confirm, cancel — the failure path self-heals |
| Cross-service writes | Outbox + at-least-once + idempotent consumer behind a unique index |
| Distributed-transaction decision | Move the boundary → one transaction → make it unnecessary → reservation → saga → 2PC |
| Scaling a database | Query/index → access pattern → cache → scale up → replicas → functional split → partition → shard |
| Partition vs shard | One engine (elimination, sliding windows) vs many engines (write scale-out, and all of Module 8's costs) |
| Read replicas | Lag is a consistency decision; classify reads; solve read-your-writes explicitly |
| Connections | Postgres connections are processes; `Max Pool Size` = 100; serverless multiplies instances × pool |
| Schema migration | Expand/contract: add → dual-write → backfill → switch reads → stop old writes → drop; four deployments |
| Multi-tenancy | Silo/bridge/pool; hybrid is normal; per-tenant restore usually forces the decision; RLS + query filters |
| Backup/DR | RPO and RTO first; replication is not a backup; quote the measured restore time |
| Azure SQL tiers | GP (remote storage) / BC (local SSD + AG) / Hyperscale (page servers + log service, 10 GB–128 TB, snapshot backups) |
| SQL Server 2025 | GA Nov 2025: native `json`, `VECTOR` + DiskANN, AI functions, optimized locking, regex |
| EF Core 10 | LTS to Nov 2028; `LeftJoin`/`RightJoin`, named query filters, JSON improvements, vector support |
| Postgres 18 | Async I/O, native `uuidv7()`, B-tree skip scan, virtual generated columns, `NOT NULL` without a full scan |
| Numbers | B-tree seek 3–4 reads; 8 KB page; Cosmos 20 GB/50 GB/10k RU/s/2 MB item; Hyperscale 128 TB; pool default 100; cross-region RTT 80–150 ms |
| When *not* to add a store | It must serve a first-class pattern the current store can't serve at acceptable cost, and someone must operate it |

---

## Progress

Module 12 complete. Phase 3 now covers **guarantees** (7), **mechanisms** (8), **agreement** (9), **weakening for latency** (10), **weakening for availability and decoupling** (11), and **where the data actually lives, and what it costs to keep it correct** (12).

This module closes three loops deliberately:
- **2PC vs saga** is now complete — Module 11 gave you the messaging half, Concepts 40–47 give you the protocol, the .NET reality, the Spanner nuance, and the decision ladder.
- **Isolation vs consistency** is now disambiguated. Module 7's consistency models were the distributed half; Part C is the local half; Concept 28 is the bridge.
- **Every denormalized copy is the same object.** Index, cache, read replica, materialized view, global secondary index, search index, projection — all of them trade write cost and staleness for read cost, and all of them get the same three questions.

Two threads left open on purpose:
- **EF Core internals** — change tracking, query translation, N+1, compiled queries, and migration strategy at scale get their own treatment in **Module 19**. Concept 60 is the architect-level summary only.
- **Aggregates as a domain-modelling discipline** — Concept 32 and Concept 47 set up **Module 22** (DDD tactical patterns) and **Module 23** (CQRS), where the transaction boundary becomes a design method rather than a storage constraint.

Next in the curriculum: **Module 13 — Reliability patterns** (circuit breakers, retries and backoff, bulkheads, active-active vs active-passive redundancy), which closes Phase 3 by formalizing the failure-handling tactics that Modules 10, 11 and 12 have each introduced piecemeal.
