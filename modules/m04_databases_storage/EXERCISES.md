# m04 — Exercises (Databases & Storage)

> Do these on paper / out loud before checking `solutions/m04_databases_storage/`. Goal: pick the
> right store, reason about transactions/isolation, and design indexes/schemas.

---

## Exercise 1 — Pick the store (and justify)
For each, name the **data-model family** (relational / KV / document / wide-column / graph /
time-series / search / object) and one line of why:
1. User accounts with balances, where money transfers must be correct.
2. Session tokens looked up by token id on every request, sub-ms.
3. A product catalog where each product has wildly different, nested attributes.
4. 5 million IoT sensor readings per second, queried by device + time range.
5. "People you may know" (friends-of-friends) on a social network.
6. Full-text search over support tickets with relevance ranking.
7. Original uploaded videos (multi-GB each).
8. Daily NAV history per mutual-fund scheme, charted over years.

---

## Exercise 2 — The lost update
A wallet service runs this on two concurrent requests (Postgres default isolation):
```
bal = SELECT balance FROM wallets WHERE id = 42;     -- both read 100
UPDATE wallets SET balance = bal - 30 WHERE id = 42;  -- both write 70
```
1. What's the final balance, and what *should* it be? Name the anomaly and the isolation level that
   allowed it.
2. Give **four** different fixes (one atomic, one pessimistic-lock, one optimistic, one isolation-level)
   and the trade-off of each.
3. Which would you choose for a high-contention hot wallet vs a low-contention one?

---

## Exercise 3 — Isolation anomalies
Match each scenario to the anomaly (dirty read / non-repeatable read / phantom / write skew / lost
update), and name the **weakest isolation level** that prevents it:
1. Tx reads a row, Tx2 commits an update, Tx re-reads and sees a different value.
2. Tx reads data another tx hasn't committed yet (and that tx later rolls back).
3. Two doctors each verify "another doctor is still on call" and both go off-call; now none are.
4. Tx runs `SELECT count(*) WHERE status='open'` twice and gets 5 then 6 (someone inserted).
5. Two transactions read X=10, both set X=11 (a `+1` each); X ends at 11, not 12.

---

## Exercise 4 — Index design
A table `orders(id, user_id, status, created_at, total)` with 500M rows. The hot queries are:
- (a) `WHERE user_id = ? ORDER BY created_at DESC LIMIT 20` (a user's recent orders)
- (b) `WHERE status = 'pending'` (pending count is ~0.5% of rows)
- (c) `WHERE created_at BETWEEN ? AND ?` (all orders in a date range)
1. Design the indexes. For (a), what's the right **column order** and why?
2. For (b), is an index worth it? What changed your answer vs indexing `status` generally?
3. Could a **covering index** help (a)? What would it include?
4. Name two reasons an index you created might *not* get used.

---

## Exercise 5 — B-tree vs LSM
1. You're choosing a store for an **append-heavy event log** (10M writes/sec, occasional time-range
   reads). B-tree or LSM-based? Why, in terms of write path?
2. You're choosing for a **read-heavy transactional app** (user profiles, orders, lots of point +
   range reads, multi-row transactions). Which engine family? Why?
3. Explain **read amplification** (LSM) and **write amplification** (both) in one sentence each, and
   name the mechanism LSM uses to mitigate read amplification.

---

## Exercise 6 — SQL or NoSQL, with reasons
For each, choose and justify in 2 lines (and note if you'd actually use **both**):
1. A banking core ledger.
2. A URL shortener's code→long-URL mapping at massive read scale.
3. A startup MVP whose data model is changing weekly.
4. An analytics dashboard aggregating 2 years of events.
5. A chat app storing billions of messages, queried by conversation.

---

## Exercise 7 — Model your queries (NoSQL)
You're storing **claims** in a document store (DCE-style). Each claim has a header + many detail lines
+ edit codes, and is processed as a unit. Common access: "fetch a claim by id with all its lines."
1. Would you **embed** the detail lines in the claim document or **reference** them in a separate
   collection? Why?
2. Now a new requirement: "find all claims across all members that contain edit code `597`." What does
   this cost in your document model, and how would you support it?
3. What's the general lesson about NoSQL modeling vs relational here?

---

## Exercise 8 — The dual-write trap
Your service writes an order to Postgres and then updates a Redis cache and an Elasticsearch index so
reads are fast.
1. Describe a sequence of events that leaves the three stores **inconsistent**.
2. Why can't you just "wrap all three in a transaction"?
3. Sketch the **Outbox + CDC** solution and why it's atomic where the naive approach isn't.

---

## Exercise 9 — OLTP vs OLAP
1. Why would `SELECT AVG(total) FROM orders` (500M rows) be slow on your row-store OLTP DB and fast on
   a columnar warehouse?
2. Why shouldn't you just run it on the OLTP DB anyway (two reasons)?
3. Sketch the data flow from OLTP DB → analytics, naming the step in between.

---

## Exercise 10 — Apply to your own systems 🏭
1. List every datastore in **DCE** and justify each by its **access pattern** (the polyglot argument).
2. Questimate precomputes risk into **snapshot tables**. Frame this as normalize-vs-denormalize: what
   does it buy, what does it cost, and what's the hidden consistency risk?
3. DCE's **claim-preprocessing** turns raw claim JSON into 100+ structured fields. Frame this as
   schema-on-read vs schema-on-write — which store holds which form, and why?

---

When done, check `solutions/m04_databases_storage/README.md`, then say **"Module 5"**.
