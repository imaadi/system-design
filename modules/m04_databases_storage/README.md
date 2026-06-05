# Module 4 — Databases & Storage ⭐⭐

> The database is where almost every system's hardest decisions live: it's the **stateful** part, the
> part that's expensive to scale, the part where "correct" and "fast" fight. You already use four
> stores daily (Postgres, MongoDB, Redis, MySQL) — this module turns that working knowledge into
> *architect-level* reasoning: **why** each exists, the transaction/isolation guarantees you're
> actually getting (often weaker than you think), how indexes and storage engines really work, and how
> to choose and model. **This module is single-node storage + data modeling; distributing it
> (replication, sharding) is m05.**

Read in passes: **§1–§4** (the landscape, relational model, ACID, isolation) and **§5–§10**
(indexing, storage engines, NoSQL, choosing, modeling, OLTP/OLAP).

---

## 1. The storage landscape — pick the right tool ⭐

There is no "best database." There are **data models**, each optimized for an **access pattern**. The
senior skill is matching them. The major families:

| Family | Shape | Optimized for | Examples | Use when |
|---|---|---|---|---|
| **Relational (SQL)** | tables (rows/cols), schema, joins | **transactions, complex queries, integrity** | PostgreSQL, MySQL | structured data + relationships + you need correctness (the **safe default**) |
| **Key-Value** | `key → value` (opaque blob) | **O(1) lookups, caching, speed** | Redis, DynamoDB, Memcached | sessions, cache, simple fast lookups by key |
| **Document** | JSON-like docs, flexible schema | **nested/variable data, dev velocity** | MongoDB, Couchbase | semi-structured data that's read/written as a unit; evolving schema |
| **Wide-column** | rows with dynamic columns, partitioned | **massive write throughput, scale** | Cassandra, HBase, ScyllaDB | huge write volume, time-series, known query patterns, AP needs |
| **Graph** | nodes + edges | **relationship traversal** | Neo4j, Neptune | social graphs, fraud rings, recommendations (many-hop joins) |
| **Time-series** | timestamped points | **append + time-range queries** | InfluxDB, TimescaleDB, Prometheus | metrics, IoT, prices (your Questimate NAV series!) |
| **Search** | inverted index | **full-text, relevance ranking** | Elasticsearch, OpenSearch | text search, log analytics, autocomplete (m16) |
| **Object/Blob** | files by key + metadata | **cheap, durable, huge binaries** | S3, GCS, MinIO | images, video, model artifacts, backups (m18) |
| **Columnar / OLAP** | columns stored together | **analytics, aggregations** | Snowflake, BigQuery, Redshift, ClickHouse | dashboards, reporting, data warehouse (your DCE Snowflake!) |

> **Polyglot persistence** = using several of these in one system, each for what it's best at. *This
> is literally your DCE/Questimate architecture* (§11) — Postgres + Mongo + Redis + Snowflake/MinIO,
> not one DB doing everything. Naming this term and justifying each choice is a strong senior signal.

---

## 2. The relational model (SQL): structure, normalization, joins

**Relational** = data in **tables** (relations) of **rows** and typed **columns**, with a **schema**
defined up front (schema-on-write: the DB rejects data that doesn't fit). Tables relate via **keys**:
- **Primary key (PK):** unique row identifier.
- **Foreign key (FK):** a column referencing another table's PK — enforces **referential integrity**
  (you can't reference a user that doesn't exist).
- **Join:** combine rows across tables on a key (the relational superpower — and its scaling cost).

### Normalization (organize to avoid redundancy)
**Normalization** = structuring tables so each fact is stored **once**, eliminating redundancy and
update anomalies. The practical levels:
- **1NF:** atomic columns (no arrays/repeating groups in a cell).
- **2NF:** no partial dependency on part of a composite key.
- **3NF:** no transitive dependencies (non-key columns depend only on the key). *3NF is the usual
  practical target.*

Why normalize? **Integrity + cheap writes.** If a customer's address lives in one place, updating it
is one write and can never be inconsistent. The cost: reads need **joins** to reassemble data.

### Denormalization (the read-side trade)
**Denormalization** = deliberately **duplicating data** to avoid joins and speed up reads. You copy
the customer's name into the orders table so the order list needs no join.
> **Think differently — denormalization IS caching.** It buys read speed by storing a redundant copy,
> and it inherits caching's exact problem: **the copies can go stale and must be kept in sync on
> write.** So the rule mirrors §1 of m06: normalize by default (correctness, cheap writes);
> denormalize *deliberately* for a proven read hotspot, and own the sync cost. *Questimate's
> precomputed risk **snapshot tables** are exactly this — denormalized/materialized reads of an
> expensive join+compute, refreshed by the nightly pipeline.*

---

## 3. ACID — the transaction guarantees ⭐

A **transaction** is a group of operations treated as **one unit**. **ACID** is what a transactional
database promises:

- **A — Atomicity:** all-or-nothing. Either every operation in the transaction commits, or none does
  (on failure it **rolls back**). The classic example: a money transfer (debit A, credit B) must not
  leave the debit applied without the credit.
- **C — Consistency:** a transaction moves the DB from one **valid state to another**, preserving all
  constraints/invariants (FKs, uniqueness, checks). *(This is ACID's C — different from CAP's C; see
  m02 §6.)*
- **I — Isolation:** concurrent transactions don't corrupt each other; the result is **as if** they
  ran in some order. *How much* isolation is the subtle part — see §4.
- **D — Durability:** once committed, it **survives crashes** (written to non-volatile storage, via a
  **write-ahead log / WAL** so a crash mid-commit can be recovered).

> **How A & D actually work — the Write-Ahead Log (WAL).** Before changing the data pages, the DB
> appends the change to a sequential **log** and flushes *that* to disk first. The log is the source
> of truth: on crash, replay it (redo committed, undo uncommitted) → atomicity + durability. *This
> same log is also what powers replication (m05) and Change-Data-Capture (m08).* Sequential log
> writes are also fast (m02 ladder: sequential ≫ random) — a recurring theme: **append to a log, apply
> later.**

**BASE** (the NoSQL counterpoint): **B**asically **A**vailable, **S**oft state, **E**ventually
consistent. Instead of strong ACID guarantees, prioritize availability and accept eventual
consistency (m02). It's the AP-leaning philosophy behind many wide-column/KV stores.

---

## 4. Isolation levels & the anomalies they allow ⭐⭐ (the part most engineers get wrong)

The "I" in ACID is a **dial**, not a switch — and the default is **weaker than you think.** Full
isolation (**Serializable**) is expensive, so databases offer weaker levels that allow specific
**anomalies** in exchange for concurrency. You must know what your level lets through.

**The anomalies (read these slowly — they're real bugs):**
- **Dirty read:** you read another transaction's **uncommitted** change (which may roll back).
- **Non-repeatable read:** you read a row twice in one transaction and get **different values**
  (someone committed an update between your reads).
- **Phantom read:** you run the same `WHERE` query twice and get **different rows** (someone inserted/
  deleted matching rows).
- **Lost update:** two transactions read a value, both modify it, and one overwrites the other's
  change (read-modify-write race — e.g. two `+1`s net only +1).
- **Write skew:** two transactions read an overlapping set, each makes a decision based on it, and
  both commit — together violating an invariant **neither violated alone**. (Classic: two on-call
  doctors each check "is *someone else* on call? yes" and both go off-call → nobody is on call.)

**The standard isolation levels (SQL standard), weakest → strongest:**

| Level | Dirty read | Non-repeatable | Phantom | Notes |
|---|---|---|---|---|
| **Read Uncommitted** | ❌ allowed | ❌ | ❌ | rarely used; can read garbage |
| **Read Committed** | ✅ prevented | ❌ allowed | ❌ allowed | **PostgreSQL & Oracle default** — only sees committed data, but values can change between reads |
| **Repeatable Read** | ✅ | ✅ prevented | ❌ allowed* | **MySQL/InnoDB default**; rows you read won't change |
| **Serializable** | ✅ | ✅ | ✅ prevented | as if transactions ran one at a time — strongest, slowest |

> **Think differently #1 — your default allows real bugs.** Postgres's default **Read Committed**
> permits **non-repeatable reads, lost updates, and write skew.** That means the naive
> `balance = read(); write(balance - 100)` pattern is a **lost-update bug** at the default level. The
> fixes: an **atomic** `UPDATE ... SET balance = balance - 100` (let the DB do the read-modify-write),
> **`SELECT ... FOR UPDATE`** (pessimistic row lock), an **optimistic** version/CAS check, or bump the
> isolation level. Knowing this — and that "I used a transaction" does **not** automatically mean
> "I'm safe" — is a major senior signal.

> **Think differently #2 — MVCC: readers don't block writers.** Most modern DBs (Postgres, MySQL/
> InnoDB, Oracle) implement isolation with **MVCC (Multi-Version Concurrency Control)**: each write
> creates a **new version** of a row; each transaction reads a consistent **snapshot** as of its start.
> So **reads never block writes and writes never block reads** — only write-write conflicts contend.
> This is why "Repeatable Read" in these DBs is really **snapshot isolation** (and why it can still
> allow *write skew* — snapshot isolation isn't fully serializable). The cost is keeping old versions
> around → **garbage to vacuum/compact** (Postgres `VACUUM`). Saying "MVCC, so my readers don't lock"
> is exactly the depth interviewers want.

*Note: Postgres's Repeatable Read (snapshot isolation) actually prevents many phantoms too; it's
stricter than the SQL-standard minimum. Implementations vary — always know *your* engine's behavior.*

---

## 5. Indexing — the #1 performance lever ⭐⭐

An **index** is an **auxiliary, sorted data structure** that lets the DB find rows **without scanning
the whole table** — turning an O(N) full-table scan into an O(log N) lookup. It's the single most
common fix for a slow query. (Think: the index at the back of a book vs reading every page.)

**The dominant index type: the B-tree (really B+tree).**
- A balanced tree kept **sorted by the indexed column(s)**, with data in the leaves and high fan-out
  (each node holds many keys) → very **shallow** (3–4 levels indexes billions of rows), so a lookup
  is a handful of disk reads.
- Because it's **sorted**, a B-tree serves **equality** (`=`), **range** (`<`, `>`, `BETWEEN`),
  **prefix** (`LIKE 'abc%'`), **and ORDER BY** efficiently. This generality is why it's the default.
- **Hash indexes** (O(1) equality) exist but can't do ranges/ordering → niche.

**The properties that decide whether an index *actually helps*:**
- **Selectivity / cardinality:** an index helps when it narrows to few rows. Indexing a `gender`
  column (2 values) is useless — the DB still reads ~half the table; it may ignore the index.
- **Composite (multi-column) indexes & the leftmost-prefix rule:** an index on `(a, b, c)` can serve
  queries filtering on `a`, `a+b`, or `a+b+c` — but **not** `b` alone or `c` alone (it's sorted by
  `a` first). Column **order matters**; design it for your query patterns.
- **Covering index:** if the index contains *all* columns a query needs, the DB answers from the index
  alone — an **index-only scan**, never touching the table (fastest).
- **Clustered vs non-clustered:** a **clustered** index *is* the table, physically ordered by the key
  (one per table; e.g. InnoDB orders rows by PK). A **non-clustered/secondary** index is a separate
  structure pointing back to rows. (This is why a random-UUID PK can be worse than a sequential one in
  InnoDB — random inserts fragment the clustered order.)

> **Think differently — an index is a write tax you pay for read speed.** Every index is a **derived,
> redundant structure the DB must update on every INSERT/UPDATE/DELETE.** So indexes **speed up reads
> but slow down writes and consume storage.** The senior framing: *"index the columns you filter/
> sort/join on, in the right order — but don't over-index; on a write-heavy table each extra index is
> a real write-amplification cost."* This is the same read-vs-write tension you'll see in storage
> engines next. Diagnose with **`EXPLAIN`/`EXPLAIN ANALYZE`** — it shows whether a query uses an index
> or does a seq-scan (the single most useful DB-perf tool).

> **Common "index doesn't work" traps** (great cross-questions): a **function on the column**
> (`WHERE lower(email)=...` defeats a plain index on `email` → need a functional/expression index); a
> **leading wildcard** (`LIKE '%abc'` can't use a B-tree); **low selectivity**; or **filtering on a
> non-leading column** of a composite index.

---

## 6. Storage engines: B-tree vs LSM-tree ⭐⭐ (think differently)

*How* a DB physically stores and updates data on disk is a fundamental fork that explains why
Postgres/MySQL feel different from Cassandra/RocksDB. It's the **read-optimized vs write-optimized**
trade.

**B-tree storage (Postgres, MySQL/InnoDB, most relational):**
- Data lives in a B-tree **updated in place**. A write seeks to the right page and modifies it.
- **Reads are fast** (go straight to the page). **Writes can be slower** (random I/O to find+update
  the page; plus the WAL write).
- Great for **read-heavy, transactional** workloads with point + range queries.

**LSM-tree (Log-Structured Merge-tree) storage (Cassandra, RocksDB, LevelDB, ScyllaDB, HBase):**
- Writes go to an in-memory **memtable** (sorted) + an append-only commit log → **all writes are
  sequential and fast** (no seek). When the memtable fills, it's flushed to an immutable sorted file
  (**SSTable**) on disk.
- Background **compaction** merges SSTables, discarding overwritten/deleted keys.
- **Writes are very fast** (sequential appends — m02 ladder: sequential ≫ random). **Reads can be
  slower**: a key might be in the memtable or any SSTable, so reads check several places (mitigated by
  **Bloom filters** that cheaply say "key definitely not in this SSTable").

```
  B-TREE:  write → seek to page → update in place   (read: 1 lookup, fast; write: random I/O)
  LSM:     write → append to memtable/log (fast!) → flush to SSTable → compact later
           read: check memtable + SSTables (+ Bloom filter)   (write: fast; read: more work)
```

| | **B-tree** | **LSM-tree** |
|---|---|---|
| Write path | update-in-place (random) | append (sequential) → compact |
| **Optimized for** | **reads** | **writes** |
| Read amplification | low | higher (multiple SSTables) |
| Write amplification | WAL + page write | compaction rewrites data repeatedly |
| Space | can fragment | compresses well; needs compaction headroom |
| Used by | Postgres, MySQL/InnoDB | Cassandra, RocksDB, LevelDB, HBase, ScyllaDB |

> **The senior line:** *"If the workload is write-heavy (logs, metrics, time-series, high-ingest
> event data), an LSM-based store is built for sequential write throughput; if it's read-heavy and
> transactional, a B-tree relational store is the better fit. The trade is read-amplification (LSM)
> vs write-amplification/random-I/O (B-tree)."* This single comparison shows you understand databases
> at the storage layer, not just the API.

---

## 7. NoSQL families in depth (and what "NoSQL" really buys)

"NoSQL" ("Not Only SQL") is an umbrella for non-relational stores. They generally trade **joins +
strong multi-row transactions + rigid schema** for **horizontal scale, flexible schema, and a data
model that matches a specific access pattern.** The four you'll discuss:

- **Key-Value (Redis, DynamoDB, Memcached):** `key → value`. Dead-simple, O(1), blazing fast. Perfect
  for **caching, sessions, rate-limit counters, feature flags**. Limited query power (you fetch by
  key). *(Redis is more: data-structure server — lists, sets, sorted sets, pub/sub, TTLs — which is why
  you use it for leaderboards, queues, and JWT sessions.)*
- **Document (MongoDB, Couchbase):** stores **JSON-like documents**; the whole entity (with nested
  arrays/objects) lives in one document you read/write atomically. Flexible/evolving schema; great
  **dev velocity**. Best when data is **read/written as a unit** and is naturally hierarchical (a
  claim, a product, a user profile). *(Modern MongoDB has multi-document transactions, but the model
  shines when you* don't *need cross-document joins.)*
- **Wide-column (Cassandra, HBase, ScyllaDB):** rows partitioned across nodes; each row can have
  different columns. Built for **massive write throughput + horizontal scale + AP availability**. You
  must **design tables around your queries** (often one table per query pattern). Great for
  time-series, event logs, messaging at scale.
- **Graph (Neo4j, Neptune):** **nodes + edges**, optimized for **traversing relationships** ("friends
  of friends who like X", "fraud rings"). Queries that would be many self-joins in SQL become natural
  traversals.

> **Two myths to kill (think differently):**
> 1. **"NoSQL = schemaless."** It's **schema-on-read**, not no-schema — the structure moves from the
>    DB into your *application code*, which now must handle every shape your data has ever had. The
>    schema didn't vanish; you just took responsibility for it.
> 2. **"NoSQL because it scales / SQL doesn't."** Outdated. Modern Postgres/MySQL scale enormously
>    (replicas, partitioning, and managed offerings like Aurora/Spanner/CockroachDB give horizontal
>    scale *with* transactions). You choose NoSQL for a **data-model/access-pattern** fit (or extreme
>    write scale with relaxed consistency), **not** reflexively "to scale."

---

## 8. SQL vs NoSQL — the decision (the senior take) ⭐

Don't answer "NoSQL, it scales." Reason from **data shape, access pattern, consistency, and scale**:

**Lean SQL (relational) when:**
- Data is **structured with clear relationships** and you benefit from **joins**.
- You need **multi-row ACID transactions** and strong integrity (money, inventory, bookings).
- Query patterns are **varied/ad-hoc** (analytics, flexible filtering) — SQL is a powerful query language.
- *(It's the right **default**; deviate only with a reason.)*

**Lean NoSQL when:**
- The **access pattern is simple and known** (lookup by key, fetch a document) → KV/document.
- Data is **semi-structured / rapidly evolving** → document.
- You need **extreme write throughput / horizontal scale** with relaxed consistency → wide-column.
- The problem is **relationship traversal** → graph; **full-text** → search; **metrics/time** →
  time-series; **huge binaries** → object store.

> **The honest senior answer:** *"I'd default to a relational database — it gives me transactions,
> integrity, and flexible queries, and modern Postgres scales further than people assume. I'd move a
> specific workload to a NoSQL store when its access pattern clearly fits one better — e.g. a
> high-write event stream to a wide-column store, a cache/session layer to Redis, or huge blobs to
> object storage. Most real systems are **polyglot**: the right store per concern."* That's exactly
> how DCE and Questimate are built — and you can say so.

---

## 9. Data modeling / schema design ⭐

Modeling differs fundamentally between the paradigms:

**Relational modeling:** model the **entities and relationships** (normalize to ~3NF), then **index
for your queries**, then **denormalize selectively** for proven read hotspots. The schema is designed
around the *data*; the flexible query language adapts to new questions.

**NoSQL modeling (the mindset flip): model your *queries*, not your data.** ⭐
Because there are no joins (or they're expensive), you **shape the data to match how you'll read it**:
- **Embed vs reference:** embed related data in one document when it's read together and bounded
  (an order + its line items); reference when it's large, shared, or unbounded (don't embed 10M
  followers in a user doc).
- **One table per access pattern / single-table design (DynamoDB):** you often **duplicate** data
  across items so each query is a single fast lookup — denormalization is the *norm*, not the
  exception. You design the **partition key + sort key** around your queries up front.
- The cost: adding a *new* query pattern later may require a new table/index or a backfill — NoSQL
  rewards known patterns and punishes ad-hoc ones.

> **Think differently — the dual-write problem (you WILL hit this).** The moment your data lives in
> two places that must agree — DB + cache, DB + search index, DB + a denormalized copy — keeping them
> in sync is a **distributed transaction in disguise.** If you write to the DB then the cache and the
> second write fails (or a crash lands between them), they diverge. You can't wrap two systems in one
> ACID transaction. The robust fix (m08): the **Transactional Outbox + Change-Data-Capture (CDC)** —
> write the change *and* an event to the same DB transaction (atomic), then a separate process tails
> the DB log and updates the cache/index/copy. Recognizing that "update the DB and the cache" is a
> consistency hazard, not a one-liner, is a senior insight. *This is exactly the risk in Questimate's
> snapshot tables and any cache.*

---

## 10. OLTP vs OLAP (transactional vs analytical) — and row vs columnar

Two opposite workloads that want **opposite storage**:

| | **OLTP** (Online Transaction Processing) | **OLAP** (Online Analytical Processing) |
|---|---|---|
| Workload | many small reads/writes of **few rows** (place order, fetch profile) | few huge queries scanning **millions of rows**, aggregating |
| Pattern | point lookups, updates, short transactions | `GROUP BY`, `SUM`, joins over history |
| Storage | **row-oriented** (a row's columns stored together → fetch a whole row fast) | **column-oriented** (each column stored together) |
| Examples | Postgres, MySQL, your app DB | Snowflake, BigQuery, Redshift, ClickHouse |
| Freshness | live, current | often batch-loaded (minutes–hours behind) |

> **Think differently — why columnar wins at analytics.** An aggregate like `SELECT AVG(amount) FROM
> claims` needs **one column** across millions of rows. **Row storage** must read every full row
> (all columns) off disk to get that one field — huge waste. **Columnar storage** keeps each column
> contiguous, so it reads *only* the `amount` column → far less I/O, plus excellent **compression**
> (similar values adjacent) and vectorized CPU processing. That's why you don't run analytics on your
> OLTP database: you **ETL/ELT** it into a **data warehouse** (columnar). *DCE does exactly this —
> claims live in MongoDB (OLTP-ish, per-claim access) and are moved to **Snowflake** (columnar OLAP)
> for reporting/analysis.*

**Where data lives at scale:** OLTP **database** (live transactions) → **ETL/ELT** → **data
warehouse** (structured, columnar, for BI) and/or **data lake** (raw files in object storage,
schema-on-read, for ML/exploration). The "lakehouse" blends both. (You don't need to design a
warehouse in most interviews, but knowing *why* analytics is a separate system is the point.)

---

## 11. From your systems 🏭
- **Polyglot persistence, lived:** DCE = **MongoDB** (claims — nested, variable JSON, read/written as
  a unit → document model), **Redis** (hot reference/eligibility/auth data → KV, sub-ms), **Snowflake**
  (ETL/analytics → columnar OLAP), **MinIO** (model artifacts → object/blob). Questimate = **Postgres**
  (relational core with `JSONField`, `CheckConstraint`, composite indexes), **Redis** (JWT sessions +
  cache), **MongoDB** (NAV/historical time-series). You can justify **every** store by its access
  pattern — that's §1 and §8 in your own words.
- **Denormalization / materialized reads:** Questimate **precomputes risk decomposition into snapshot
  tables** so the API is an O(1) read instead of a heavy join+compute — textbook §2/§9
  denormalization, with the nightly Celery pipeline owning the sync (and the §9 dual-write risk).
- **Constraints = ACID's C:** Questimate's `CheckConstraint`/`UniqueConstraint` on the polymorphic
  holdings table are integrity invariants the DB enforces — ACID Consistency in practice.
- **Schema-on-write vs schema-on-read:** DCE's **claim-preprocessing layer** turns raw, variable
  claim JSON (schema-on-read, in Mongo) into the **structured 100+-field inputs** the models need —
  you literally bridge the two paradigms.
- **OLTP→OLAP split:** claims in Mongo for per-claim processing, **Snowflake** for analysis/reporting
  — §10 in production (and you helped move ETL to Snowpark for cost).
- **Lost-update awareness:** placing/adjusting an order or decrementing a balance is exactly the
  read-modify-write hazard of §4 — a great place to mention atomic `UPDATE` / `FOR UPDATE` / optimistic
  versioning.

---

## 12. Key concepts (interview-ready)
- **No best DB — match the data model to the access pattern.** Know the families (§1) and use
  **polyglot persistence**. SQL is the **safe default**; pick NoSQL for a specific fit, not "to scale."
- **ACID** = Atomicity, Consistency, Isolation, Durability; durability/atomicity via the **WAL**
  (append to a log, apply later — the recurring motif).
- **Isolation is a dial.** The default (**Read Committed** in Postgres) **allows lost updates &
  write skew** — "I used a transaction" ≠ "I'm safe." Fix with atomic `UPDATE`, `SELECT FOR UPDATE`,
  optimistic versioning, or a higher level. **MVCC** = versioned rows → readers don't block writers
  (Repeatable Read ≈ snapshot isolation; still allows write skew).
- **Indexes** turn O(N) scans into O(log N) via a sorted **B-tree**; mind **selectivity**, **composite
  leftmost-prefix**, **covering** indexes, clustered vs secondary. An index is a **write tax for read
  speed** — don't over-index. Diagnose with **`EXPLAIN`**.
- **Storage engines:** **B-tree = read-optimized** (update-in-place; Postgres/MySQL); **LSM-tree =
  write-optimized** (append + compact; Cassandra/RocksDB). Read-amplification vs write-amplification.
- **NoSQL myths:** it's **schema-on-read** (not schemaless) and chosen for **fit**, not scale.
- **Modeling:** SQL → normalize then index then denormalize selectively; NoSQL → **model your
  queries**, embed vs reference, single-table design.
- **The dual-write problem:** keeping DB + cache/index in sync is a hidden distributed transaction →
  use **Outbox + CDC** (m08).
- **OLTP (row, live) vs OLAP (columnar, analytics)** — separate systems; ETL/ELT between them; columnar
  wins analytics by reading only needed columns + compression.

---

## 13. Go deeper (the well-researched reading list)
- **Kleppmann, *Designing Data-Intensive Applications* (DDIA)** — Ch. 3 (storage engines: B-tree vs
  LSM, column stores) and Ch. 7 (transactions, isolation levels, the anomalies, MVCC) are the
  definitive treatment of §3–§6 and §10. *If you read one thing, read these two chapters.*
- **Use The Index, Luke! (use-the-index-luke.com)** — the best practical guide to how indexes really
  work (§5), free.
- **PostgreSQL docs: "Transaction Isolation"** and **`EXPLAIN`** — see your actual engine's behavior;
  run `EXPLAIN ANALYZE` on a real query.
- **The Google *Bigtable* paper** (wide-column origin) and the **Amazon *Dynamo* paper** (KV / BASE) —
  the source ideas behind §7.
- **Pat Helland, "Life Beyond Distributed Transactions"** — the deep "why the dual-write problem is
  fundamental" (§9).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m04_databases_storage/`](../../solutions/m04_databases_storage/README.md).

When you've done the exercises, say **"Module 5"** to build *Replication, Partitioning & Sharding*
(leader-follower, quorums, consistent hashing, hot shards) — i.e. how we take everything here and
**distribute** it across machines.
