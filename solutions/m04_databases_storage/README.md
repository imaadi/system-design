# m04 — Solutions (Databases & Storage)

> Check the *reasoning*. The recurring lessons: match the store to the access pattern; "I used a
> transaction" ≠ "I'm safe"; indexes trade write cost for read speed; keep two stores in sync only
> with care.

---

## Exercise 1 — Pick the store
1. **Relational** — money needs multi-row ACID transactions + integrity.
2. **Key-Value** (Redis) — O(1) lookup by token, sub-ms, TTL for expiry.
3. **Document** (MongoDB) — variable nested attributes per product; read as a unit.
4. **Time-series / wide-column** (Timescale/Influx or Cassandra) — massive sequential writes (LSM),
   partitioned by device+time, time-range queries.
5. **Graph** (Neo4j) — friends-of-friends is multi-hop relationship traversal (a nightmare of self-
   joins in SQL).
6. **Search** (Elasticsearch) — inverted index + relevance ranking for full text.
7. **Object/Blob** (S3/MinIO) — cheap durable storage for huge binaries; metadata elsewhere.
8. **Time-series** (or relational with a time index / Mongo) — append-mostly daily points charted over
   years; this is literally Questimate's NAV series (you used Mongo for it).

---

## Exercise 2 — The lost update
1. Final = **70**; should be **40** (100 − 30 − 30). Anomaly = **lost update** (one tx's write
   overwrote the other's). Allowed by **Read Committed** (Postgres default) — neither read a dirty
   value, but the read-modify-write raced.
2. Four fixes:
   - **Atomic update:** `UPDATE wallets SET balance = balance - 30 WHERE id=42` (and `AND balance>=30`).
     The DB does read-modify-write in one statement. *Best/simplest; minimal locking.*
   - **Pessimistic lock:** `SELECT ... FOR UPDATE` then update. Correct, but **locks the row** →
     contention/serialization under load.
   - **Optimistic concurrency:** read `version`; `UPDATE ... WHERE version = old`; if 0 rows changed,
     retry. *Great for low contention; wasteful retries under high contention.*
   - **Serializable isolation:** correct by construction, but **lowest throughput** (more aborts/retries).
3. **High-contention hot wallet:** atomic update or pessimistic lock (optimistic would thrash on
   retries). **Low-contention:** optimistic versioning (no locking, rare conflicts).

---

## Exercise 3 — Isolation anomalies
1. **Non-repeatable read** → prevented by **Repeatable Read**.
2. **Dirty read** → prevented by **Read Committed**.
3. **Write skew** → prevented by **Serializable** (snapshot isolation/Repeatable Read does *not* stop
   it).
4. **Phantom read** → prevented by **Serializable** (SQL-standard; note Postgres's snapshot RR also
   blocks many phantoms).
5. **Lost update** → prevented by **Repeatable Read** in many engines (or atomic update / `FOR UPDATE`
   at lower levels).

---

## Exercise 4 — Index design
1. **(a)** Composite index **`(user_id, created_at DESC)`**. `user_id` leads because the query always
   filters by it (equality), then `created_at` supports the `ORDER BY ... LIMIT` directly (no sort
   step). Order matters: leading with `created_at` wouldn't serve the per-user filter efficiently.
2. **(b)** Yes — even though `status` has low *cardinality* overall, **'pending' is only ~0.5% of
   rows** (high selectivity for *that value*). A **partial index** `WHERE status='pending'` is ideal:
   tiny, only indexes the rows you query. (What changed: selectivity is about the *queried value's*
   rarity, not the column's total distinct count.)
3. **(c)** Index on **`(created_at)`** for the date-range scan.
4. **Covering index for (a):** `(user_id, created_at DESC) INCLUDE (total, status)` → an index-only
   scan that returns the list without touching the table.
5. **Not used because:** a function on the column (`WHERE date(created_at)=`), a leading wildcard, low
   selectivity making a seq-scan cheaper, filtering a non-leading composite column, or stale planner
   stats — confirm with `EXPLAIN ANALYZE`.

---

## Exercise 5 — B-tree vs LSM
1. **LSM** — writes append to an in-memory memtable + commit log (all **sequential** I/O), flushed to
   immutable SSTables, compacted later → built for huge sequential write throughput. Perfect for an
   append-heavy log.
2. **B-tree** (Postgres/MySQL InnoDB) — updates in place for **fast reads**, supports point + range +
   ORDER BY and **multi-row ACID transactions**, which the app needs.
3. **Read amplification (LSM):** a read may check the memtable + multiple SSTables to find the latest
   value. **Write amplification:** one logical write → multiple physical writes (B-tree: WAL + page +
   index updates; LSM: compaction rewrites data across levels). LSM mitigates read amplification with
   **Bloom filters** (cheaply rule out SSTables that can't contain the key).

---

## Exercise 6 — SQL or NoSQL
1. **SQL** — ledger needs ACID transactions + integrity; correctness dominates.
2. **NoSQL KV** (or KV-style) — pure key→value lookup at massive read scale; no joins needed (often
   **both**: KV/DB for the mapping + a cache like Redis in front).
3. **NoSQL document** (or SQL with JSON) — evolving schema favors schema-on-read for MVP velocity;
   honestly Postgres + JSONB also works, so either, with a bias to whatever ships fastest.
4. **OLAP/columnar** (Snowflake/BigQuery) — aggregations over 2 years of events; **both** in practice
   (OLTP source → ETL → warehouse).
5. **NoSQL wide-column** (Cassandra) — billions of messages, partition by conversation_id, sequential
   writes, known query pattern (fetch a conversation's messages); SQL would struggle at that write
   scale.

---

## Exercise 7 — Model your queries (NoSQL)
1. **Embed** the detail lines in the claim document: a claim is processed/read **as a unit**, the lines
   are **bounded** and belong to the claim's lifecycle, and one read fetches everything atomically.
   (This is exactly DCE's Mongo model.)
2. "Find all claims with edit code `597`" cuts **across** documents — the embed model isn't organized
   for it, so a naive query scans many documents. Support it by **indexing the edit-code field** (a
   multikey/secondary index on the embedded array) or maintaining a **separate query-optimized
   collection / search index** keyed by edit code (model-your-queries again).
3. **Lesson:** NoSQL is fast for the access patterns you **designed for** (fetch claim by id) and
   awkward for **new, cross-cutting** patterns — relational + a flexible query language would handle
   the ad-hoc query more naturally. Choose based on whether your patterns are known and stable.

---

## Exercise 8 — The dual-write trap
1. **Inconsistency sequence:** write order to Postgres ✅ → update Redis ✅ → **update Elasticsearch
   fails / the process crashes.** Now ES is missing the order (or stale) while Postgres+Redis have it.
   Any partial-failure or crash between the writes diverges them.
2. **Can't wrap in one transaction:** the three are **separate systems** with no shared transaction
   coordinator; an ACID transaction is per-database. (Distributed 2PC across them is fragile/blocking
   and usually unavailable.)
3. **Outbox + CDC:** in the **same Postgres transaction**, write the order **and** an `outbox` event
   row — this is atomic (both commit or neither). A separate **relay/CDC** process tails the DB log /
   outbox table and **reliably** propagates the event to Redis and ES (with retries + idempotency). It's
   atomic where the naive approach isn't because the only multi-system step is reading the durable log
   and re-applying — which can be retried safely until it succeeds (m08).

---

## Exercise 9 — OLTP vs OLAP
1. `AVG(total)` needs **one column** across 500M rows. **Row store** reads every full row (all columns)
   off disk to get `total` → huge wasted I/O. **Columnar** reads only the `total` column, contiguous
   and compressed → far less I/O + vectorized aggregation → fast.
2. Don't run it on OLTP because: **(a)** the big scan competes for I/O/CPU/locks with live transactions
   → user latency suffers (no performance isolation); **(b)** row storage is the wrong shape for
   aggregation.
3. **Flow:** OLTP DB → **ETL/ELT** (extract, transform, load — e.g. into Snowpark/Snowflake) → **data
   warehouse** (columnar) → BI/dashboards. (DCE: Mongo → Snowflake.)

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **DCE stores:** **MongoDB** = claims (nested, variable JSON, read/written per-claim → document
   model); **Redis** = hot reference/eligibility/auth data (sub-ms KV lookups during scoring);
   **Snowflake** = ETL/analytics/reporting (columnar OLAP over history); **MinIO** = model artifacts
   (pkl/sav blobs → object storage). Each store is chosen by its access pattern = polyglot persistence.
2. **Snapshot tables = denormalization/materialized read.** **Buys:** O(1) API reads instead of a
   heavy join+compute (latency win). **Costs:** extra storage + the nightly pipeline must recompute
   them (write/compute cost). **Hidden risk:** the snapshot is a **stale copy** until refreshed — a
   dual-write/consistency hazard; if the pipeline fails, reads serve stale risk numbers.
3. **Schema-on-read → schema-on-write bridge.** Raw claim JSON sits in **Mongo as schema-on-read**
   (flexible, variable shapes). The **preprocessing layer** imposes structure, producing the **100+
   fixed fields** the models require (effectively schema-on-write for the model inputs). You convert
   flexible storage into a rigid, validated contract right before the ML/rules that demand it — the
   classic "flexibility at ingestion, structure at the point of computation" pattern.
