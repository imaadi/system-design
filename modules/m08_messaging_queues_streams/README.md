# Module 8 — Messaging, Queues & Stream Processing ⭐⭐ 🏭

> So far, services called each other **synchronously** (m03): the caller waits for the reply. But the
> most resilient, scalable systems lean on **asynchronous** communication — drop a message and move on;
> someone processes it later. This module is the async backbone: **queues vs logs**, **delivery
> guarantees** (the at-least-once / exactly-once reality), **ordering vs parallelism**, **backpressure**,
> **Kafka** deeply, and the **Outbox + CDC** pattern that *finally* solves the dual-write problem from
> m04/m06. This is your home turf — DCE's Kafka claim stream and Questimate's Celery pipeline are §-by-§
> examples.

Read in passes: **§1–§6** (why async, queue vs log, patterns, delivery, ordering, backpressure) and
**§7–§11** (Kafka, RabbitMQ vs Kafka, Outbox/CDC, stream processing, event-driven).

---

## 1. Synchronous vs asynchronous — the fundamental ⭐

- **Synchronous (request/response, m03):** caller sends a request and **blocks** until the reply.
  Simple, immediate, easy to reason about — but the caller is **coupled** to the callee's availability
  and speed (if the callee is slow/down, the caller is too), and a traffic spike hits the callee
  directly.
- **Asynchronous (messaging):** producer **hands a message to a broker** and moves on; a consumer
  processes it **later**, independently. The producer and consumer are **decoupled in time and space.**

What async buys you (memorize these four):
1. **Decoupling** — producer doesn't know/care who consumes, when, or how many consumers there are.
2. **Buffering / burst absorption** — the broker is a **shock absorber**: a spike queues up instead of
   overwhelming the consumer (the consumer drains at its own pace).
3. **Resilience** — if a consumer is down, messages **wait** in the broker; it catches up on recovery
   (no lost work, no cascading failure).
4. **Independent scaling** — scale producers and consumers separately; add consumers to drain faster.

The costs (always state them): **eventual** (not immediate) processing → higher end-to-end latency;
**complexity** (ordering, idempotency, redelivery, monitoring); and **harder debugging/tracing** (no
single call stack — you need correlation IDs across hops).

> **The senior framing:** *"I make a path async when the work is slow, bursty, or can be deferred, and
> the caller doesn't need the result immediately — I trade immediacy and simplicity for decoupling,
> burst absorption, and resilience. I keep it sync when the caller needs the answer now."* DCE ingests
> claims via Kafka precisely because scoring is heavy and bursty — you don't block the producer for
> seconds.

---

## 2. The core distinction: message queue vs event log ⭐⭐ (get this right)

People say "queue" for both, but there are **two different abstractions**, and confusing them is a
classic mistake.

### Message queue (RabbitMQ, AWS SQS, Celery's broker)
A queue holds messages until a consumer takes one. The defining behaviors:
- A message is **delivered to one consumer**, processed, **acknowledged, and removed** from the queue.
- **Competing consumers:** add N workers and they **share the load** (each message goes to exactly one
  of them) → great for **distributing work / task queues**.
- Once consumed and acked, the message is **gone** — no history, no replay.
- Mental model: a **to-do list** — take a task, do it, cross it off.

### Event log / stream (Kafka, AWS Kinesis, Apache Pulsar)
A log is an **append-only, ordered, durable** sequence of events that consumers **read by position**.
- Events are **appended** and **retained** (by time or size), **not deleted on read.**
- **Multiple independent consumers** each read the whole log at their own pace, tracking their own
  **offset** (position). Consumer A reading doesn't affect consumer B.
- You can **replay** — rewind the offset and reprocess history (reprocessing, new consumers,
  backfills, recovering from a bug).
- Mental model: a **ledger / commit log** you can read and re-read from any point.

```
  QUEUE (RabbitMQ):  [m1][m2][m3] → worker takes m1, acks, m1 DELETED.  Work distribution, no replay.
  LOG   (Kafka):     offset:  0   1   2   3   4 ...   (append-only, retained)
                     consumer-A reads at offset 2 ─┐
                     consumer-B reads at offset 4 ─┘ each independent; can rewind/replay
```

| | **Message queue** (RabbitMQ/SQS) | **Event log** (Kafka/Kinesis) |
|---|---|---|
| On consume | message **removed** | offset advances, event **retained** |
| Consumers | competing (share work) | independent groups (each reads all) |
| Replay / history | no | **yes** (rewind offset) |
| Ordering | per-queue (limited) | **strict within a partition** |
| Best for | task distribution, RPC, work | event streaming, multiple subscribers, audit, replay |
| Throughput | high | **very high** (sequential disk, m04 LSM motif) |

> **Say this:** *"A queue is for distributing work — consume, ack, delete. A log (Kafka) is a durable,
> ordered, replayable record that many consumers read independently. Kafka isn't 'a faster RabbitMQ' —
> it's a different abstraction; I pick a queue for work distribution and a log when I need replay,
> multiple consumers, ordering, or event history."* DCE uses Kafka (a log) for claims; Questimate's
> Celery uses a broker (a queue) for background tasks — and that difference is intentional.

---

## 3. Messaging patterns
- **Point-to-point (queue):** one message → **exactly one** consumer (competing consumers share the
  load). For task/work distribution. *(Celery workers.)*
- **Publish/subscribe (fan-out):** one event → **all** subscribers receive it (each independent). For
  broadcasting events to many interested services. *(Kafka consumer groups; RabbitMQ fanout exchange.)*
- **Request/reply (async RPC):** producer sends a request with a `reply-to` queue + correlation ID; the
  consumer processes and replies on that queue. Async RPC when you *do* eventually need an answer but
  don't want to block a thread.

> **A subtle Kafka point:** Kafka does **both** patterns with one mechanism — within a **consumer
> group** it's point-to-point (each partition → one consumer in the group, work shared); across
> **different consumer groups** it's pub/sub (every group independently reads all events). That duality
> is why Kafka is so flexible.

---

## 4. Delivery guarantees — at-most / at-least / exactly-once ⭐⭐ (the big one)

Networks fail and processes crash, so "did this message get delivered and processed?" is genuinely
hard. Three guarantee levels:

- **At-most-once:** deliver, don't retry on failure. **No duplicates, but messages can be lost.** Fire-
  and-forget (e.g. non-critical metrics where a lost sample is fine). Cheapest.
- **At-least-once:** retry until acknowledged. **No loss, but duplicates are possible** (a message can
  be redelivered if the ack is lost or the consumer crashes after processing but before acking). **The
  practical default** for important work.
- **Exactly-once:** every message processed once, no loss, no duplicates. **The holy grail.**

> **Think differently — "exactly-once delivery is essentially impossible; exactly-once *processing* is
> achievable."** ⭐⭐ Over an unreliable network you *cannot* guarantee a message is delivered exactly
> once (the classic two-generals problem: if an ack is lost, the sender can't know whether to resend).
> So real systems use **at-least-once delivery + an idempotent (or de-duplicating) consumer** → the
> *effect* happens exactly once even though the message may arrive twice. This is **"effectively-once"**,
> and it's the **same idempotency-key idea from m03** and the lost-update fix from m04 — they're all the
> same insight: **make redelivery/retry harmless.** (Kafka's "exactly-once semantics" works within
> Kafka via idempotent producers + transactions, but the moment you touch an external system, you're
> back to "idempotent consumer.")

**How an idempotent consumer works:** before acting, check "have I already processed message-id X?"
(a dedup table / seen-set / a natural idempotency key). If yes, skip (just ack). If no, process + record
the id **atomically**. Then a duplicate delivery is a no-op.

```
  at-least-once delivery + idempotent consumer = effectively-once processing
  (retry freely; duplicates are harmless because the consumer dedups by message id / key)
```

---

## 5. Acks, redelivery, dead-letter queues & poison messages

The machinery that makes at-least-once work:
- **Acknowledgment (ack):** the consumer tells the broker "I've successfully processed this" → broker
  marks it done. **Ack *after* processing, not before** — if you ack-then-crash, the message is lost
  (that turns it into at-most-once). Ack-after means a crash-before-ack → **redelivery** (at-least-once).
- **Visibility timeout (SQS) / unacked redelivery:** when a consumer takes a message, it's hidden for a
  timeout; if not acked in time (consumer crashed/slow), it's **redelivered** to another consumer.
- **Retries with backoff + jitter:** transient failures → retry with exponential backoff (m10), not a
  tight loop.
- **Poison message:** a message that **always fails** (malformed, triggers a bug). Without handling, it
  redelivers forever and **blocks the queue / wastes resources.**
- **Dead-letter queue (DLQ):** after N failed attempts, move the message to a separate **DLQ** for
  later inspection/repair, so it stops blocking healthy processing. A must-have in production.

> **The senior detail:** *"I ack after processing for at-least-once, make the consumer idempotent so
> redelivery is safe, retry transient failures with backoff, and route repeated failures to a DLQ so a
> poison message can't wedge the pipeline."* That sentence shows you've operated a real queue.

---

## 6. Ordering vs parallelism ⭐

A fundamental tension: **strict ordering and parallel consumption fight each other.**
- If you need messages processed in **exact global order**, you can have **only one consumer** (any
  parallelism could reorder) → no horizontal scaling.
- If you want **parallelism** (many consumers), messages can be processed **out of order** across
  consumers.

**The resolution (Kafka's model):** order is guaranteed **only within a partition**, and you
**parallelize across partitions**. You route messages that must stay ordered to the **same partition**
(by partition key — m05), and unrelated messages spread across partitions for throughput.
- **Same key → same partition → ordered.** Different keys → different partitions → parallel.
- Example: order all events **for a given claim** (key = claim id) so they're processed in order, while
  *different* claims process in parallel. (Exactly the DCE choice.)

> **Carry this:** *"Global ordering doesn't scale — it forces a single consumer. So I scope ordering to
> what actually needs it (a per-entity key) and parallelize across keys/partitions. Ordering is a
> per-key guarantee, bought by partitioning."* This directly reuses the shard-key reasoning from m05.

---

## 7. Backpressure & consumer lag ⭐

What happens when producers outpace consumers? The queue **grows**. This is where m02's **Little's Law**
returns:
- A queue is a **buffer** for *temporary* bursts (producer briefly faster than consumer → queue
  absorbs it → drains when the burst passes). ✅
- But if **arrival rate > processing rate *persistently*** (λ_in > λ_out), the queue grows **without
  bound** → memory/disk fills, latency climbs, eventually failure. **A queue does NOT fix an
  under-provisioned consumer — it only delays the collapse and hides it.** ⚠️
- **Consumer lag** (how far behind the latest message the consumers are) is **the** metric to monitor —
  growing lag = consumers falling behind = trouble coming.

**Handling overload (backpressure):**
- **Scale consumers** (more workers / partitions) so λ_out ≥ λ_in.
- **Backpressure / flow control:** signal producers to slow down (Kafka's **pull** model gives this for
  free — consumers fetch at their own rate; a push model can overwhelm consumers).
- **Load shedding / sampling:** drop or sample low-priority messages when overwhelmed (m10).
- **Bounded queues:** cap queue size and reject/shed beyond it — fail fast rather than melt slowly.

> **Think differently — "just add a queue" is not a capacity fix.** A queue smooths bursts; it doesn't
> add throughput. If the consumer can't keep up *on average*, the queue just grows until something
> breaks. The real fixes are more consumer capacity, backpressure, or shedding. Monitor **lag**.

---

## 8. Kafka deep dive ⭐ 🏭 (the dominant event log)

Kafka is the de-facto event-streaming platform; you've run it. The model:
- **Topic:** a named stream of events ("claims", "clicks").
- **Partition:** a topic is split into **partitions** — each an **ordered, append-only log** on a broker
  (this is **sharding**, m05). Partitions give Kafka its scale and parallelism. **Ordering is guaranteed
  within a partition**, not across.
- **Partition key:** decides which partition an event lands in (`hash(key) % partitions`). **Same key →
  same partition → ordered** (§6). Choosing it = the shard-key decision (m05).
- **Offset:** each event's position in its partition. Consumers track their offset → they control where
  they read (and can **rewind to replay**).
- **Producer:** appends events (optionally keyed). Writes are **sequential appends** → very high
  throughput (the m04 LSM/sequential-I/O motif).
- **Consumer group:** a set of consumers that **share** a topic's partitions — each partition is read by
  **exactly one** consumer in the group (point-to-point within the group), and **different groups** each
  read everything (pub/sub across groups). **Max useful parallelism = number of partitions.**
- **Replication:** each partition has replicas across brokers (leader + followers, m05) → durability +
  availability; a partition leader handles its reads/writes, followers take over on failure.
- **Retention:** events are kept for a configured time/size (e.g. 7 days) regardless of consumption →
  enables replay. **Log compaction** (optional): keep only the **latest** value per key (turns the log
  into a changelog / materialized current-state — used for CDC and stateful streams).
- **Pull-based:** consumers **pull** at their own rate → natural backpressure (§7).

```
  TOPIC "claims"
   partition 0: [e0][e1][e2][e3]...   (ordered)   ← consumer C1 (group G)
   partition 1: [e0][e1][e2]...       (ordered)   ← consumer C2 (group G)
   partition 2: [e0][e1]...           (ordered)   ← consumer C3 (group G)
   key=claim123 → always partition 1 → its events stay ordered; claims spread across partitions
```

> Why Kafka scales: **append-only sequential writes** (fast disk), **partitioning** (parallelism),
> **pull + offsets** (consumer-controlled, replayable), **retention** (history). It's a distributed
> commit log — basically the WAL idea (m04) turned into infrastructure.

---

## 9. RabbitMQ vs Kafka ⭐ (the classic compare)

| | **RabbitMQ** (message broker / queue) | **Kafka** (event log / stream) |
|---|---|---|
| Abstraction | smart broker, **queues**, flexible routing (exchanges) | dumb-broker/smart-consumer, **append-only log** |
| On consume | message **removed** after ack | **retained**; consumers track offsets |
| Replay | no (gone once acked) | **yes** (rewind offset) |
| Consumers | competing consumers share a queue | consumer groups + independent groups |
| Ordering | per-queue, weak under competing consumers | **strict per partition** |
| Routing | rich (direct/topic/fanout/headers exchanges) | by partition/topic; routing logic in consumers |
| Throughput | high | **very high** (millions/sec) |
| Best for | **task queues, RPC, complex routing, per-message workflows** | **high-throughput event streaming, replay, multiple consumers, log/audit, analytics pipelines** |

> **Decision rule:** *RabbitMQ (or SQS/Celery) when you need **work distribution, flexible routing, or
> request/reply** and don't need history. Kafka (or Kinesis/Pulsar) when you need **high-throughput
> event streaming, replay, multiple independent consumers, ordering, or an event log/audit trail.**
> Many systems use both: Kafka as the event backbone, a queue for specific task workflows.* This is
> literally your stack — **Kafka for the DCE claim stream, Celery/RabbitMQ for Questimate's background
> tasks.**

---

## 10. The Outbox pattern + CDC — solving dual-write at last ⭐⭐ 🏭

Recall the **dual-write problem** (m04/m06): you must update the **database** *and* **publish an event**
(or update a cache/search index) — but they're two systems, so you can't wrap both in one transaction.
If the DB commit succeeds and the event publish fails (or a crash lands between), they **diverge**:
either an event with no DB change, or a DB change with no event. Naively "write DB, then publish to
Kafka" is **not atomic.**

**The Transactional Outbox pattern (the canonical fix):**
1. In the **same database transaction** as your business write, also insert a row into an **`outbox`**
   table describing the event. Because it's one local DB transaction, it's **atomic** — either both the
   business change and the outbox row commit, or neither.
2. A separate **relay/publisher** process reads unpublished outbox rows and **publishes them to Kafka**
   (or the queue), marking them sent. If publishing fails, it **retries** (at-least-once) — safe because
   downstream consumers are idempotent (§4).

```
  ┌─────────── one DB transaction (atomic) ───────────┐
  │  INSERT order ...                                  │
  │  INSERT outbox(event='OrderCreated', payload=...)  │   ← both commit or neither
  └────────────────────────────────────────────────────┘
         │ separate relay process
         ▼
  read outbox rows → publish to Kafka → mark sent (retry on failure; consumers idempotent)
```

**Change-Data-Capture (CDC):** instead of an explicit outbox table + relay, tail the database's **WAL/
binlog** (its replication log, m04/m05) and turn every committed change into an event stream. Tools:
**Debezium** → Kafka. CDC is **log-based** (efficient, no app changes, captures everything) vs the
outbox's **polling/explicit** approach (more control over event shape). Both achieve the same thing:
**the event is derived from the same atomic commit as the data**, so they can never diverge.

> **Why this is the punchline of the building-blocks half:** it ties together the **WAL** (m04),
> **replication log** (m05), **cache/index sync** (m06), and **idempotent consumers** (§4) into the
> robust answer for "keep two systems consistent." When an interviewer asks *"how do you update the DB
> and publish an event atomically?"*, the senior answer is **"transactional outbox or CDC — never a
> naive dual write."** *(This is exactly how you'd reliably emit a DCE claim-decision event alongside
> writing the decision to Mongo.)*

---

## 11. Stream processing & event-driven architecture (brief)
Beyond moving messages, you can **process streams** continuously:
- **Stateless processing:** transform/filter/route each event independently (map). Easy to scale.
- **Stateful processing:** aggregations, joins, **windowing** (tumbling/sliding/session windows — e.g.
  "count events per 1-min window"). Needs managed state + checkpointing for fault tolerance. Tools:
  **Kafka Streams, Apache Flink, Spark Streaming.**
- **Event-driven architecture:** services communicate by **emitting/reacting to events** rather than
  direct calls → loose coupling, but **eventual consistency** and harder end-to-end reasoning.
- **Event sourcing:** store the **events themselves** as the source of truth (an append-only log) and
  **derive current state** by replaying them — the log *is* the database (the WAL idea, m04, as your
  primary model). Gives a full audit trail + time-travel, at the cost of complexity and rebuild time.
- **CQRS (Command Query Responsibility Segregation):** separate the **write model** (commands) from the
  **read model** (queries/materialized views), often kept in sync via events. Lets you optimize reads
  and writes independently (pairs naturally with event sourcing).

> You don't need to architect a Flink job in most interviews, but knowing **windowing**, **event
> sourcing**, and **CQRS** as concepts (and that they're advanced tools with real complexity costs) is
> the right altitude.

---

## 12. From your systems 🏭
- **Kafka as the event backbone (a log):** DCE streams claims through **Kafka** — ingestion decoupled
  from heavy/bursty scoring (§1), partitioned for throughput with **ordering per claim** via the
  partition key (§6/§8), consumed by scoring workers (consumer group), at-least-once with **idempotent
  processing** so a redelivery/reprocess doesn't double-adjudicate (§4). Your **reprocessing scripts**
  (`aws_batch_reprocessing`) are literally **replay** from the log (§2) — only possible because it's a
  log, not a queue.
- **Celery/RabbitMQ as a task queue:** Questimate's **Celery + Celery Beat** run scheduled/background
  jobs (the daily pipeline, heavy compute off the request path) — point-to-point **work distribution**
  across workers (§2/§3), the right tool because you need task execution, not event replay.
- **Idempotency everywhere:** placing the same trade twice, or reprocessing a claim, must be safe —
  that's §4 idempotent/effectively-once, the through-line from m03 idempotency keys and m04 lost-update.
- **Outbox/CDC opportunity:** reliably emitting a "claim decided" event alongside the Mongo write is the
  §10 dual-write problem — the correct answer is outbox/CDC, not "write Mongo then publish."
- **Backpressure:** a claims surge around payer cycles is absorbed by the broker (§7) and drained by
  scaling consumers — and **consumer lag** is the metric that tells you if scoring is keeping up.

---

## 13. Key concepts (interview-ready)
- **Async messaging** decouples producer/consumer → **decoupling, burst absorption, resilience,
  independent scaling**; costs eventual processing + complexity + harder debugging. Use it for slow/
  bursty/deferrable work.
- **Queue vs log:** a **queue** (RabbitMQ/SQS) distributes work and **deletes on consume**; a **log**
  (Kafka) is **append-only, retained, replayable**, read by **independent consumers via offsets**.
  Kafka ≠ "fast RabbitMQ."
- **Patterns:** point-to-point (one consumer), pub/sub (fan-out), request/reply. Kafka does both via
  consumer groups.
- **Delivery:** at-most-once (lossy), **at-least-once (default, duplicates possible)**, exactly-once
  (delivery is ~impossible). **At-least-once + idempotent consumer = effectively-once** — the same
  idempotency idea as m03/m04. **Ack after processing.**
- **Reliability machinery:** acks, visibility timeout/redelivery, retries with backoff, **DLQ** for
  **poison messages**.
- **Ordering vs parallelism:** global ordering forces one consumer; scope ordering to a **per-key
  partition** (same key → same partition → ordered) and parallelize across partitions.
- **Backpressure:** a queue absorbs **bursts**, not sustained overload (Little's Law) — if λ_in > λ_out
  persistently it grows unbounded. Monitor **consumer lag**; fix with more consumers / backpressure
  (pull model) / load shedding / bounded queues. "Add a queue" ≠ more capacity.
- **Kafka:** topics → **partitions** (ordered logs = sharding), **offsets**, **consumer groups**,
  **replication**, **retention + compaction**, **pull-based**. A distributed commit log.
- **Outbox / CDC** solves the **dual-write problem**: write the event in the **same DB transaction**
  (outbox) or tail the **WAL/binlog** (CDC/Debezium) → the event derives from the same atomic commit, so
  data and event never diverge. The answer to "update DB + publish event atomically."
- **Advanced:** stream processing (windowing, Flink/Kafka Streams), **event sourcing** (events as source
  of truth), **CQRS** (separate read/write models).

---

## 14. Go deeper (the well-researched reading list)
- **Kleppmann, DDIA** — Ch. 11 (Stream Processing: logs vs queues, delivery, CDC, event sourcing) and
  Ch. 4/11 on the unifying "everything is a log" idea. The definitive source for this module.
- **Jay Kreps, "The Log: What every software engineer should know about real-time data's unifying
  abstraction"** — the foundational essay behind Kafka and the log abstraction (§2, §8).
- **Kafka documentation: Design + Consumer Groups + Exactly-Once Semantics** — authoritative §8.
- **microservices.io: "Transactional Outbox" & "Saga"** (Chris Richardson) — the canonical §10 pattern.
- **Debezium docs** — log-based CDC in practice (§10).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m08_messaging_queues_streams/`](../../solutions/m08_messaging_queues_streams/README.md).

When you've done the exercises, say **"Module 9"** to build *Distributed Systems Theory* (consensus —
Raft/Paxos, leader election, logical/vector clocks, 2PC vs Saga, CRDTs, the 8 fallacies) — the formal
underpinnings behind the failover, quorums, and ordering you've been using.
