# m08 — Solutions (Messaging, Queues & Stream Processing)

> Check the *reasoning*. Recurring lessons: queue vs log are different abstractions; at-least-once +
> idempotent = effectively-once; queues absorb bursts not sustained load; outbox/CDC for dual-write.

---

## Exercise 1 — Sync or async?
1. **Sync** — the user must see the payment result before proceeding (caller needs the answer now).
2. **Async** — fan out side effects (email, analytics, warehouse) after the order; decouple and don't
   block the order response on them (pub/sub).
3. **Async** — heavy, bursty scoring; decouple ingestion from processing, absorb surges (DCE Kafka).
4. **Sync** — a simple, fast read the page needs immediately; async adds latency/complexity for nothing.
5. **Async** — image resizing is slow and deferrable; queue the job and return; workers process later.

---

## Exercise 2 — Queue or log?
1. **Queue** — distribute resize jobs across competing workers; consume-ack-delete, no replay needed.
2. **Log (Kafka)** — multiple independent teams/services each read all events via their own consumer
   groups.
3. **Log (Kafka)** — replay requires retained events + offsets; a queue deletes on consume.
4. **Queue (RabbitMQ)** — request/reply with flexible routing (exchanges) is RabbitMQ's strength.
5. **Log (Kafka)** — one retained stream feeding both a real-time consumer group and a batch consumer
   group independently.

---

## Exercise 3 — Delivery guarantees
1. **At-most-once:** deliver without retry → no dups, possible **loss**. **At-least-once:** retry until
   acked → no loss, possible **duplicates**. **Exactly-once:** once, no loss, no dups.
2. **Impossible delivery** because of the unreliable network / two-generals: if the **ack is lost**, the
   sender can't know whether the message was processed, so any choice risks loss or a duplicate.
3. **At-least-once delivery + idempotent (de-duplicating) consumer** → exactly-once **effect**, called
   **"effectively-once."**
4. You turned it into **at-most-once** → the in-flight message is **lost** (broker thinks it's done, but
   processing never completed). Ack *after* processing instead.

---

## Exercise 4 — Make it idempotent
1. **Redelivery double-charges** the customer. It happens when the ack is lost or the consumer crashes
   after charging but before acking (at-least-once redelivers).
2. **Idempotent consumer:** key = a stable **charge/idempotency id** on the message; **store** = a dedup
   table (or a unique constraint) of processed ids; **atomic step** = within one transaction, check "id
   seen?" → if not, perform the charge **and** insert the id; if seen, skip and ack. A duplicate then
   finds the id present and does nothing.
3. **Same idea everywhere:** HTTP idempotency keys (m03) make a retried POST safe; the m04 atomic-update/
   versioning fixes the lost-update race; all three **make retry/redelivery harmless** by deduping on a
   key. At-least-once + idempotency = effectively-once.

---

## Exercise 5 — Poison message & DLQ
1. With naive at-least-once, the malformed message **redelivers forever** (never acked), blocking the
   queue / wasting workers and potentially stalling healthy messages behind it.
2. **Handling:** retry transient failures with **exponential backoff**; after **N attempts**, move the
   message to a **dead-letter queue (DLQ)** and ack/skip it in the main flow; alert on DLQ growth and
   inspect/repair/replay later.
3. **Backoff before DLQ** because many failures are **transient** (a dependency blip, a timeout) and
   would succeed on retry — immediate dead-lettering would discard recoverable work. Backoff gives
   transient issues time to clear; only *persistent* failures get DLQ'd.

---

## Exercise 6 — Ordering vs parallelism
1. **Can't have both** because global ordering requires processing one-at-a-time (a single consumer);
   any parallel consumers can finish in a different order than messages arrived → reordering.
2. **Kafka:** use **account_id as the partition key** so all of one account's events land in the **same
   partition** (ordered), and run **multiple partitions** with a consumer per partition → different
   accounts process in **parallel**. Same key → same partition → ordered; cross-key → parallel.
3. **Increasing partitions changes the key→partition mapping**, so a given account's *new* events may go
   to a different partition than its in-flight/old ones → **ordering can break** for keys during/after
   the change. So you provision partitions for target parallelism up front and repartition carefully.

---

## Exercise 7 — Backpressure
1. λ_in (8,000) > λ_out (5,000) **persistently**, so the queue **grows unbounded** (Little's Law) until
   memory/disk fills and it fails. **"Bigger queue" is not a fix** — it only delays the collapse; a queue
   absorbs *bursts*, not sustained over-capacity.
2. **Three fixes:** scale consumers (more workers/partitions so λ_out ≥ 8,000); apply **backpressure**
   (Kafka pull lets consumers fetch at their rate / signal producers to slow); **shed or sample**
   low-priority messages (m10). (Bounded queue + reject is a fail-fast variant.)
3. Alarm on **consumer lag** (how far behind the latest offset the consumers are) — growing lag is the
   earliest signal that consumers are losing the race.

---

## Exercise 8 — The outbox pattern
1. **Failure:** Postgres commit succeeds, then the **Kafka publish fails** (or crash in between) → an
   order exists with **no event** (downstream never hears about it); or, if you publish first, an event
   with no order. The two systems **diverge** — not atomic.
2. **Outbox:** in the **same Postgres transaction** as the `order` insert, also insert an **outbox row**
   (`OrderCreated`, payload). One local transaction → **atomic** (both or neither). A separate **relay**
   reads unpublished outbox rows, publishes them to Kafka, marks them sent, retrying on failure. Atomic
   because the only thing tying data and event together is a **single DB commit**; publishing is a
   separate, retryable step.
3. **CDC** tails the database's **WAL/binlog** (the replication log) and emits committed changes as
   events (e.g. Debezium → Kafka) — no explicit outbox table; events derive from the same commits.
4. **Idempotent consumers still required** because the relay/CDC publishes **at-least-once** (it may
   retry and double-publish after a crash) — so consumers must dedupe to avoid double-processing.

---

## Exercise 9 — Kafka mechanics
1. **6 consumers** max do useful work in one group (one per partition); additional consumers sit **idle**
   (a partition is owned by exactly one consumer in a group). Parallelism is capped by partition count.
2. Use **`user:123` (or user_id) as the partition key** → all its events hash to the **same partition**,
   which is consumed by one consumer in order.
3. Each team uses a **separate consumer group** → both read **all** events independently (pub/sub across
   groups), with their own offsets, not affecting each other.
4. **Reset the consumer group's offset** back 3 days and reprocess (with the fixed, idempotent consumer).
   Possible because Kafka is a **retained, replayable log** (offsets + retention) — a queue would have
   deleted the messages on consume.

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **DCE claim stream = a log** because: heavy/bursty scoring (decouple ingestion), **multiple consumers/
   stages** read the stream, and you need **replay/reprocessing** — all log properties a queue lacks.
   **Partition key:** key by claim/member id so a given claim's events stay **ordered** on one partition
   while claims spread for throughput (avoid a low-cardinality key like LOB that hot-spots). **Delivery:**
   **at-least-once**; **idempotent processing** is required so a redelivery or a reprocess doesn't
   **double-adjudicate** a claim.
2. **Reprocessing scripts work because Kafka *retains* events (offset + retention) → you can rewind and
   replay.** With **RabbitMQ** the messages are **deleted on consume**, so there's nothing to replay —
   you'd have to re-source the data from elsewhere. (This is the queue-vs-log distinction made concrete.)
3. **Celery for the daily pipeline** because it's **work distribution / scheduled task execution** (run
   these jobs across workers, on a schedule via Beat) — you need tasks **done**, not an event log you
   replay or fan out to many independent consumers. Right tool = a task queue, not Kafka.
4. **Naive "write Mongo then publish to Kafka" = the dual-write problem** (they can diverge on partial
   failure). **Correct pattern: transactional outbox or CDC** — emit the "claim decided" event derived
   from the same atomic commit as the Mongo write, with idempotent consumers downstream.
