# m08 — Cross-Questions ("if-and-buts") on Messaging, Queues & Streams

> Answer out loud in 2–3 sentences before reading the model answer. This is your home turf (Kafka/
> Celery) — these follow-ups make sure you can reason about the guarantees, not just name the tools.

---

### Q1. Why use asynchronous messaging instead of a synchronous call?
**A.** Four wins: **decoupling** (producer doesn't depend on the consumer's availability/speed),
**burst absorption** (the broker buffers spikes so consumers drain at their own pace), **resilience**
(if a consumer is down, messages wait and it catches up — no cascading failure), and **independent
scaling** (scale producers and consumers separately). The cost is **eventual** (not immediate)
processing, more **complexity** (ordering, idempotency, redelivery), and harder **debugging/tracing**. I
go async for slow/bursty/deferrable work where the caller doesn't need the result immediately — which is
why DCE ingests claims via Kafka rather than blocking on heavy scoring.

---

### Q2. What's the difference between a message queue and an event log?
**A.** A **queue** (RabbitMQ/SQS) delivers a message to **one** consumer, which acks it and the message
is **removed** — it's for **work distribution**, with no history or replay. An **event log** (Kafka) is
an **append-only, retained, ordered** sequence that **multiple independent consumers** read by tracking
their own **offset**; consuming doesn't delete anything, so you can **replay** and add new consumers
that read all history. Kafka isn't a faster RabbitMQ — it's a different abstraction (a commit log vs a
to-do list).

---

### Q3. When would you pick Kafka over RabbitMQ, and vice versa?
**A.** **Kafka** when I need high-throughput **event streaming, replay, multiple independent consumers,
strict per-partition ordering, or an event/audit log** (analytics pipelines, event backbones). **RabbitMQ**
(or SQS/Celery) when I need **task/work distribution, flexible routing (exchanges), or request/reply**
and don't need history/replay. Many systems use both — Kafka as the event backbone, a queue for specific
task workflows — which is exactly my stack (Kafka for DCE claims, Celery for Questimate background jobs).

---

### Q4. Explain at-most-once, at-least-once, and exactly-once.
**A.** **At-most-once:** deliver without retry → no duplicates but messages can be **lost** (fire-and-
forget; fine for disposable metrics). **At-least-once:** retry until acked → **no loss but possible
duplicates** (the practical default for important work). **Exactly-once:** once, no loss, no dups — the
ideal, but **exactly-once *delivery* is essentially impossible** over an unreliable network. So real
systems do at-least-once + an idempotent consumer to get **exactly-once *effect* ("effectively-once").**

---

### Q5. Why is exactly-once delivery considered impossible, and what do you do instead?
**A.** Because of the unreliable network / two-generals problem: if the consumer processes a message but
the **ack is lost**, the sender can't tell whether to resend — any choice risks either loss or a
duplicate. So you can't guarantee a message is *delivered* exactly once. Instead you accept
**at-least-once delivery** and make the **consumer idempotent** (dedup by message id / use a natural
idempotency key, and record "processed X" atomically with the action). Then duplicates are harmless and
the **effect** happens exactly once — "effectively-once," the same idea as m03 idempotency keys and the
m04 lost-update fix.

---

### Q6. How do you make a consumer idempotent?
**A.** Give each message a stable **id / idempotency key**, and before acting, check a **dedup store**
("have I processed id X?"). If yes → skip and just ack; if no → perform the action **and** record the id
**in the same atomic operation** (e.g. a unique constraint / a transaction that writes both the result
and the seen-id). Then a redelivery is a no-op. Natural keys help — e.g. keying on claim-id + a content
hash so reprocessing the same claim doesn't double-adjudicate.

---

### Q7. Should you acknowledge a message before or after processing it?
**A.** **After** processing. If you ack **before** and then crash mid-processing, the broker thinks it's
done and the message is **lost** (effectively at-most-once). Ack **after** means a crash before the ack
causes **redelivery** (at-least-once) — no loss, at the cost of possible duplicates, which your
idempotent consumer handles. The order of ack vs work *is* the difference between at-most-once and
at-least-once.

---

### Q8. What's a poison message and how do you handle it?
**A.** A **poison message** always fails processing (malformed payload, triggers a bug), so with naive
at-least-once it **redelivers forever**, blocking the queue and wasting resources. The fix: after **N
failed attempts**, move it to a **dead-letter queue (DLQ)** for later inspection/repair, so healthy
messages keep flowing. I also add **retry with exponential backoff** for transient failures (so I don't
DLQ something that would've succeeded on retry) and alert on DLQ growth.

---

### Q9. How do you guarantee message ordering — and what does it cost?
**A.** Strict **global** ordering requires a **single consumer** (any parallelism can reorder), which
kills horizontal scaling. So you **scope ordering to a key**: route messages that must stay ordered to
the **same partition** (Kafka: `same key → same partition → ordered`) and **parallelize across
partitions**. You get per-key ordering + cross-key parallelism. The cost is that you must choose a good
partition key (m05) and accept that two different keys have no ordering guarantee between them — which is
almost always fine (e.g. order events per claim, process different claims in parallel).

---

### Q10. Your queue keeps growing. What's happening and what do you do?
**A.** Producers are outpacing consumers — **arrival rate > processing rate** persistently — so by
Little's Law the queue grows **unbounded** (a queue absorbs *bursts*, not sustained overload). Fixes:
**scale consumers** (more workers / more partitions so max parallelism rises), apply **backpressure**
(Kafka's pull model lets consumers fetch at their rate; signal producers to slow), **shed/sample**
low-priority load (m10), or use a **bounded queue** that rejects beyond a limit (fail fast). And monitor
**consumer lag** — it's the early-warning metric. "Just add a bigger queue" doesn't add throughput.

---

### Q11. Is adding a queue a fix for an under-provisioned consumer?
**A.** No — that's a common misconception. A queue is a **shock absorber for temporary bursts**: it
smooths a spike so the consumer can catch up afterward. If the consumer is too slow **on average**, the
queue just **grows until it fills** memory/disk and the system fails — the queue **delays and hides** the
problem rather than solving it. The real fix is more consumer capacity (or backpressure/shedding); the
queue only buys you time and burst tolerance.

---

### Q12. Walk me through how Kafka achieves both high throughput and scalability.
**A.** **Append-only sequential writes** (fast disk I/O — the LSM/sequential motif from m04),
**partitioning** (a topic split into partition logs across brokers → parallel produce/consume; sharding
from m05), **pull-based consumers tracking offsets** (consumers control their rate → backpressure, and
can replay), and **replication** of each partition for durability/availability. Parallelism scales with
partition count, and because consumers just track an offset into a retained log, many independent
consumer groups read the same data without extra cost.

---

### Q13. What's a consumer group, and how does it give you both point-to-point and pub/sub?
**A.** A **consumer group** is a set of consumers that **share** a topic's partitions — each partition is
assigned to **exactly one** consumer in the group, so within the group it's **point-to-point** (work is
distributed, no double-processing). **Different consumer groups** each independently read **all** the
partitions, so across groups it's **pub/sub** (every group sees every event). That's how one Kafka
mechanism serves both patterns — and why max useful parallelism in a group equals the number of
partitions.

---

### Q14. You have 3 partitions and add a 4th, 5th... consumer to a group. What happens?
**A.** Max useful parallelism in a consumer group = **number of partitions**, so with 3 partitions only
**3 consumers** can each own a partition; a 4th/5th consumer sits **idle** (no partition to own). To
scale consumers further you must **increase partitions** (which is itself a careful operation — it
changes key→partition mapping and can disrupt ordering for in-flight keys). So you provision partition
count for your target parallelism up front, much like choosing shard count (m05).

---

### Q15. What is the dual-write problem in a messaging context?
**A.** When a service must **update its database AND publish an event** (e.g. write a claim decision to
Mongo *and* emit a "claim decided" event to Kafka), those are **two systems** with no shared
transaction. If the DB commit succeeds but the publish fails (or a crash lands between them), they
**diverge** — a DB change with no event, or an event with no DB change. Naive "write DB, then publish" is
**not atomic**, so downstream systems get an inconsistent view.

---

### Q16. How does the Transactional Outbox pattern solve dual-write?
**A.** Instead of publishing directly, you insert an **outbox row** describing the event **in the same DB
transaction** as the business write — so they commit **atomically** (both or neither). A separate
**relay** process then reads unpublished outbox rows and publishes them to Kafka, marking them sent,
**retrying** on failure (at-least-once, safe because consumers are idempotent). The event is now derived
from the same atomic commit as the data, so they **can never diverge** — the only remaining concern,
duplicate publishes, is handled by idempotent consumers.

---

### Q17. What is CDC, and how does it relate to the outbox pattern?
**A.** **Change-Data-Capture** tails the database's **transaction log (WAL/binlog)** — the same log used
for replication (m04/m05) — and turns every committed change into an **event stream** (e.g. **Debezium →
Kafka**). It achieves the same goal as the outbox (events derived atomically from committed data, no dual
write) but **log-based** rather than via an explicit table + relay: no app changes, captures everything,
very efficient. The outbox gives you more control over **event shape/semantics**; CDC is lower-touch and
captures raw row changes. Both beat a naive dual write.

---

### Q18. What's the difference between event sourcing and just publishing events?
**A.** **Publishing events** is a side-effect of your normal state-mutating writes (you update a row,
then emit an event). **Event sourcing** makes the **events themselves the source of truth**: you store an
append-only log of events and **derive current state by replaying them** (the state is a *projection* of
the log, not the primary store). It gives a complete **audit trail and time-travel**, and pairs with
**CQRS** (separate read models), but costs complexity, eventual consistency, and state-rebuild time. It's
the WAL/"the log is the database" idea (m04) taken as the primary model.

---

### Q19. What's CQRS and when is it useful?
**A.** **Command Query Responsibility Segregation** separates the **write model** (commands that mutate
state) from the **read model** (queries / materialized views), often kept in sync via events. It's
useful when reads and writes have **very different shapes or scale** — e.g. complex writes but
high-volume denormalized reads — so you can optimize each independently (the write side normalized/
transactional, the read side denormalized/cached, m04/m06). The cost is **eventual consistency** between
the two models and added complexity, so I'd use it only when the read/write asymmetry justifies it.

---

### Q20. Push vs pull message delivery — why does Kafka pull?
**A.** In **push**, the broker sends messages to consumers as they arrive — low latency, but it can
**overwhelm** a slow consumer (the broker doesn't know the consumer's capacity). In **pull**, consumers
**fetch** at their own rate — which gives **natural backpressure** (a slow consumer just pulls less, the
log retains the backlog), lets consumers **batch** for throughput, and enables **replay** (pull from any
offset). Kafka pulls because it prioritizes throughput, backpressure, and replay; RabbitMQ pushes
(with prefetch limits to cap in-flight messages) for lower latency.

---

### Q21. A consumer processes a message, performs a side effect (charges a card), then crashes before acking. What happens, and how do you prevent a double charge?
**A.** The broker never got the ack, so it **redelivers** the message to another consumer → the card
could be **charged twice**. Prevention is **idempotency**: the charge carries an **idempotency key**
(m03), and the payment step records "key X already charged" atomically — so the redelivered message
finds the key already processed and **skips the charge** (just acks). At-least-once delivery + idempotent
side effect = the charge happens exactly once in effect. (Never ack before the side effect, or you'd lose
it instead.)

---

### Q22. How do you trace/debug a request across an async, event-driven system?
**A.** You lose the single call stack of synchronous calls, so you propagate a **correlation/trace id**
(and causation id) through every message's metadata, and use **distributed tracing** (OpenTelemetry,
m10) so all the hops for one logical operation are stitched together in a trace. You also monitor
**consumer lag**, DLQ size, and per-stage throughput. The honest point: async buys decoupling/resilience
but **costs observability**, so you invest in correlation ids + tracing + good metrics up front — it's
part of the trade I name when choosing async.

---

### Q23. When should you NOT use a message queue / go async?
**A.** When the caller **needs the result immediately** to proceed (a synchronous read, a UI that must
show the outcome now) — async adds latency and a round-trip of complexity for no benefit. Also when the
operation is **simple, fast, and reliable** (no bursts, no slow dependency) — a direct call is simpler
and easier to debug. And async isn't a fix for **insufficient capacity** (Q11). I add messaging
deliberately for slow/bursty/deferrable/fan-out work, not by default.

---

### Q24. How would you reliably reprocess/replay events after fixing a bug in your consumer?
**A.** This is a key reason to use a **log (Kafka), not a queue** — the events are **retained**, so I
**reset the consumer group's offset** back to before the bad data and reprocess, with the fixed
consumer. Because processing is **idempotent** (§4), replaying already-processed events is safe (no
double effects). I'd often replay into a **new** consumer group / output topic first to validate, then
cut over. (This is exactly what DCE's reprocessing/`aws_batch_reprocessing` scripts do — replay from the
log, only possible because it's a log with retention, not a delete-on-consume queue.)
