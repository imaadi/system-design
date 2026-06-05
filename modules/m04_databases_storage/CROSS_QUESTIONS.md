# m04 — Cross-Questions ("if-and-buts") on Databases & Storage

> Answer out loud in 2–3 sentences before reading the model answer. These are the follow-ups that
> reveal whether you understand databases at the *guarantee + storage* level, not just the API.

---

### Q1. "Should I use SQL or NoSQL?" — what's your actual decision process?
**A.** I reason from **data shape, access pattern, consistency needs, and scale** — not "NoSQL
scales." I **default to relational** (transactions, integrity, flexible queries; modern Postgres
scales far) and move a specific workload to NoSQL only when its access pattern fits better: simple
key lookups → KV, semi-structured/evolving docs → document, extreme write throughput with relaxed
consistency → wide-column, relationship traversal → graph. Most real systems are **polyglot** — the
right store per concern, which is exactly how DCE/Questimate are built.

---

### Q2. Is "NoSQL is schemaless" true?
**A.** No — it's **schema-on-read**, not no-schema. The structure didn't disappear; it **moved from
the database into your application code**, which now must handle every shape the data has ever taken.
You trade the DB enforcing a schema-on-write for flexibility, but you inherit the responsibility for
consistency and migrations in app logic. So "schemaless" really means "you own the schema now."

---

### Q3. Explain ACID. Which letter is most commonly misunderstood?
**A.** **Atomicity** (all-or-nothing), **Consistency** (transactions preserve invariants/constraints),
**Isolation** (concurrent transactions don't corrupt each other), **Durability** (committed data
survives crashes). The most misunderstood is **Consistency** — it's about preserving *database
invariants/constraints*, which is **different from CAP's consistency** (replica agreement/
linearizability). I always say which "C" I mean.

---

### Q4. How does a database actually guarantee Durability and Atomicity?
**A.** Via the **Write-Ahead Log (WAL)**: before modifying the data pages, the DB **appends the change
to a sequential log and flushes that to disk first.** On a crash it replays the log (redo committed
work, undo uncommitted) → durability + atomicity. A bonus: sequential log writes are fast (m02
ladder), and the same log feeds **replication** (m05) and **CDC** (m08). "Append to a log, apply the
change later" is a motif across the whole stack.

---

### Q5. What's the default isolation level in Postgres, and what bug does it allow?
**A.** **Read Committed.** It prevents dirty reads (you only see committed data) but **allows
non-repeatable reads, lost updates, and write skew.** So the common `bal = SELECT balance; UPDATE SET
balance = bal - 100` pattern is a **lost-update race** at the default level — two concurrent runs can
net only one decrement. "I wrapped it in a transaction" does **not** make it safe; you need an atomic
`UPDATE`, a row lock, optimistic versioning, or a higher isolation level.

---

### Q6. Two requests both decrement the same inventory count concurrently. How do you prevent a lost update?
**A.** Four options: **(1) Atomic update** — `UPDATE items SET qty = qty - 1 WHERE id=? AND qty > 0`,
let the DB do the read-modify-write in one statement (simplest, best). **(2) Pessimistic lock** —
`SELECT ... FOR UPDATE` to lock the row, then update (blocks others). **(3) Optimistic concurrency** —
read a `version`, then `UPDATE ... WHERE version = old_version`; if 0 rows changed, someone beat you →
retry (great for low contention). **(4) Raise isolation** to Serializable (correct but costs
throughput). I'd pick the atomic update or optimistic versioning by default.

---

### Q7. What is MVCC and why does it matter?
**A.** **Multi-Version Concurrency Control:** each write creates a **new version** of the row, and each
transaction reads a consistent **snapshot** as of its start. The payoff: **readers don't block
writers and writers don't block readers** — only write-write conflicts contend — so you get high
concurrency without read locks. The cost is keeping old row versions around, which must be garbage-
collected (Postgres `VACUUM`). It's how Postgres/InnoDB implement snapshot isolation.

---

### Q8. Snapshot isolation prevents non-repeatable reads — so is it serializable?
**A.** No. Snapshot isolation (what many DBs call "Repeatable Read") still allows **write skew**: two
transactions read an overlapping snapshot, each makes a decision valid in isolation, and both commit,
together breaking an invariant (the two-doctors-going-off-call example). To prevent write skew you
need true **Serializable** isolation (e.g. Postgres's Serializable Snapshot Isolation) or explicit
locking (`SELECT FOR UPDATE`) on the rows the decision depends on.

---

### Q9. What is an index, and what does it cost?
**A.** An index is an **auxiliary sorted data structure** (usually a **B-tree**) that lets the DB find
rows without a full-table scan — O(N) → O(log N). It **speeds up reads** on the indexed columns. The
cost: it's a **redundant structure the DB must update on every write**, so it **slows writes and uses
storage** (write amplification). So I index the columns I filter/sort/join on, but don't over-index a
write-heavy table — each extra index is a real per-write cost.

---

### Q10. Why is a B-tree the default index, vs a hash index?
**A.** A B-tree is kept **sorted**, so it serves **equality, range (`<`/`>`/BETWEEN), prefix
(`LIKE 'abc%'`), and ORDER BY** — one structure for almost everything, and it's shallow (high fan-out)
so lookups are a few I/Os even for billions of rows. A **hash index** gives O(1) equality but **can't
do ranges or ordering** (no sort order), so it's niche. B-tree's generality wins as the default.

---

### Q11. You have an index on `(user_id, created_at)`. Which queries can use it?
**A.** Because it's sorted by `user_id` first, it serves: filter on `user_id`, filter on `user_id` +
`created_at`, and `WHERE user_id=? ORDER BY created_at`. It does **not** help a query filtering on
`created_at` **alone** (the **leftmost-prefix rule** — `created_at` isn't the leading column). To
serve `created_at`-only queries I'd need a separate index. Column **order in a composite index is a
design decision driven by query patterns.**

---

### Q12. A query is slow even though there's an index on the column. Why might the index not be used?
**A.** Several classic reasons: **(1)** a **function/expression on the column** (`WHERE lower(email)=`)
defeats a plain index → need a functional index; **(2)** a **leading wildcard** (`LIKE '%x'`) can't
use a B-tree; **(3)** **low selectivity** (the value matches a large fraction of rows → the planner
chooses a seq-scan as cheaper); **(4)** filtering on a **non-leading column** of a composite index;
**(5)** type mismatch/implicit casts. I'd run **`EXPLAIN ANALYZE`** to see what the planner actually
does.

---

### Q13. Explain B-tree vs LSM-tree storage and when you'd pick each.
**A.** **B-tree** updates data **in place** (random I/O), giving **fast reads** — ideal for read-heavy
transactional workloads (Postgres, MySQL/InnoDB). **LSM-tree** writes to an in-memory memtable + an
**append-only** log, flushes to immutable **SSTables**, and **compacts** in the background — all writes
are **sequential**, giving **very fast writes** (Cassandra, RocksDB), at the cost of **read
amplification** (a key may be in several SSTables; Bloom filters mitigate). I pick LSM for write-heavy
(logs, metrics, time-series, high ingest) and B-tree for read-heavy transactional. The trade is
**read-amplification (LSM) vs write-amplification/random-I/O (B-tree).**

---

### Q14. What's write amplification and where does it appear?
**A.** Write amplification = **one logical write causing multiple physical writes.** In B-trees: the
WAL write + the page write + index updates. In LSM-trees: **compaction** repeatedly rewrites the same
data as SSTables merge across levels. It matters because it consumes disk I/O and **wears out SSDs**,
and it's why a write-heavy table with many indexes, or an LSM store with aggressive compaction, can be
I/O-bound even at modest logical write rates.

---

### Q15. When does denormalization make sense, and what's the catch?
**A.** When a **read hotspot** is dominated by an expensive join/aggregation and reads vastly outnumber
writes — you duplicate data to serve reads without the join (e.g. Questimate's precomputed snapshot
tables). **The catch: denormalization is caching** — the duplicated copy can go **stale** and must be
**kept in sync on every write**, which adds write cost and a consistency hazard. So normalize by
default; denormalize deliberately for a proven hotspot and own the sync.

---

### Q16. What's the "dual-write problem" and how do you solve it?
**A.** When the same fact must live in two systems (DB + cache, DB + search index, DB + a denormalized
copy), updating both is a **distributed transaction in disguise**: if the first write succeeds and the
second fails (or a crash lands between them), they diverge — and you can't wrap two systems in one
ACID transaction. The robust fix is the **Transactional Outbox + Change-Data-Capture (CDC)**: write
the data change **and** an event row in the *same* DB transaction (atomic), then a separate process
tails the DB log and propagates it to the cache/index/copy (m08). Recognizing "update DB + cache" as a
hazard, not a one-liner, is the senior point.

---

### Q17. How do you model data in a document/wide-column store differently from SQL?
**A.** In SQL you model **entities + relationships** (normalize), and the flexible query language
adapts to new questions. In NoSQL you **model your *queries*, not your data**: shape documents/tables
so each access pattern is a single fast lookup — **embed** related data that's read together and
bounded, **reference** what's large/shared/unbounded, and accept **duplication** (single-table /
one-table-per-query design in DynamoDB). The trade-off: it's blazing fast for **known** patterns but
adding a *new* query later can require a new table/index or a backfill.

---

### Q18. When do you embed vs reference in a document database?
**A.** **Embed** when the related data is **read/written together**, is **bounded** in size, and
belongs to the parent's lifecycle (order + its line items; a claim + its detail lines) — one read gets
everything, atomically. **Reference** (store an id and look up separately) when the data is **large**,
**shared across parents**, **unbounded** (don't embed a user's 10M followers), or updated
independently. The deciding questions: "is it read together?" and "could it grow without bound?"

---

### Q19. Why don't you run analytics on your production (OLTP) database?
**A.** Two reasons. **Performance isolation:** big analytical scans (`GROUP BY` over millions of rows)
compete for I/O/CPU/locks with live transactions and can tank user-facing latency. **Storage fit:**
OLTP uses **row storage** (fast to fetch whole rows), but analytics wants **columnar** (reads only the
needed columns + compresses well). So you **ETL/ELT** into a **data warehouse** (Snowflake/BigQuery/
Redshift, columnar, OLAP) and analyze there — which is exactly what DCE does (Mongo → Snowflake).

---

### Q20. Why is columnar storage faster for analytics than row storage?
**A.** An aggregate like `AVG(amount)` needs **one column** across millions of rows. **Row storage**
must read each full row (all columns) off disk to extract that one field — massive wasted I/O.
**Columnar** stores each column contiguously, so it reads **only** `amount`, dramatically less I/O,
plus better **compression** (adjacent values are similar) and **vectorized** CPU processing. The
flip side: columnar is bad at fetching/updating a single whole row, which is why OLTP stays row-based.

---

### Q21. Redis is "just a cache," right?
**A.** No — Redis is an **in-memory data-structure server**: strings, hashes, **lists, sets, sorted
sets**, streams, pub/sub, with **TTLs** and optional persistence (RDB snapshots / AOF log). That's why
it's used far beyond caching — **rate-limit counters, leaderboards (sorted sets), queues/streams,
session/JWT stores with instant revocation, distributed locks, pub/sub backplanes** (e.g. for the
WebSocket fan-out in m03/m14). Calling it "just a cache" undersells it; it's a fast, versatile
primitive — but it's memory-bound and you must plan persistence/eviction.

---

### Q22. Your write-heavy service (millions of events/sec, append-mostly, time-ordered) needs a store. What and why?
**A.** This screams **LSM-tree / wide-column or time-series**: Cassandra/ScyllaDB or a TSDB
(TimescaleDB/InfluxDB), because **all writes are sequential appends** (LSM) → huge ingest throughput,
data is naturally **partitioned by time/key**, and queries are time-range scans the model is built
for. A relational B-tree store would bottleneck on random-I/O writes + index maintenance. I'd also
push the raw events through a **log/queue (Kafka)** first to absorb bursts (m08), then tier old data to
**object storage** for cost. I would *not* reach for a normalized relational DB here.

---

### Q23. PUT vs PATCH vs POST for updates — does the storage/idempotency story matter?
**A.** Yes. **PUT** replaces a resource at a known URI and is **idempotent** (repeat → same final
state) — safe to retry. **PATCH** is a partial update and usually **not** idempotent (e.g. "increment
by 1"). **POST** creates/does-something and isn't idempotent by default. This ties to m03: for
retry-safety on non-idempotent writes you add an **idempotency key**, and at the storage layer you back
it with a uniqueness constraint or a dedup table so a retried write doesn't create a duplicate row.

---

### Q24. How would you migrate a large table's schema (e.g. add/alter a column) with zero downtime?
**A.** With an **expand-contract (parallel-change)** pattern: **(1) Expand** — add the new column/table
as nullable/additive (a cheap metadata change; avoid a blocking rewrite/long lock), backfill in
**batches** to avoid a giant transaction/lock. **(2) Migrate** — deploy code that **writes both** old
and new and reads new (dual-write during transition). **(3) Contract** — once backfilled and verified,
switch reads fully to new and drop the old. Keep each step backward-compatible so you can roll back,
and watch for locks (some `ALTER`s rewrite the whole table). This mirrors the kind of careful parity-
validated migration you did on DCE.
