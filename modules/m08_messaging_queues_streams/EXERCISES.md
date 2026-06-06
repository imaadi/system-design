# m08 — Exercises (Messaging, Queues & Stream Processing)

> Do these on paper / out loud before checking `solutions/m08_messaging_queues_streams/`. Goal: choose
> queue vs log, reason about delivery/ordering, and design idempotency/outbox.

---

## Exercise 1 — Sync or async?
For each, choose synchronous call or async messaging, and justify in one line:
1. A user clicks "pay" and must see success/failure before the page proceeds.
2. After an order is placed, send a confirmation email, update analytics, and notify the warehouse.
3. Ingesting 1M healthcare claims/day for heavy ML scoring.
4. Reading a user's profile to render their settings page.
5. Resizing an uploaded image into 5 thumbnail sizes.

---

## Exercise 2 — Queue or log?
For each, choose **message queue (RabbitMQ/SQS/Celery)** or **event log (Kafka)** and justify:
1. Distributing background image-resize jobs across a worker pool.
2. An event stream that 4 different teams' services each consume independently.
3. You need to **replay** the last 7 days of events after fixing a consumer bug.
4. Request/reply RPC with flexible routing rules.
5. A clickstream feeding both real-time dashboards and a nightly batch job.

---

## Exercise 3 — Delivery guarantees
1. Define at-most-once, at-least-once, exactly-once in one line each.
2. Why is exactly-once *delivery* essentially impossible? Name the failure that causes it.
3. What combination do real systems use to get exactly-once *effect*, and what's that called?
4. You ack a message *before* processing and the consumer crashes mid-work. Which guarantee did you just
   turn it into, and what's the consequence?

---

## Exercise 4 — Make it idempotent
A consumer reads "charge customer $50" messages from an at-least-once queue.
1. What goes wrong on a redelivery, and when does a redelivery happen?
2. Design an idempotent consumer: what key, what store, what's the atomic step?
3. How is this the same idea as HTTP idempotency keys (m03) and the lost-update fix (m04)?

---

## Exercise 5 — Poison message & DLQ
1. A malformed message fails every time it's processed. Without special handling, what does at-least-
   once do, and what's the impact?
2. Design the handling: retries, threshold, and where the message ends up.
3. Why do you want retries with backoff *before* dead-lettering, not immediately?

---

## Exercise 6 — Ordering vs parallelism
You process "account events" and must apply them **per account in order**, but want to parallelize
across accounts for throughput.
1. Why can't you have strict global ordering AND many parallel consumers?
2. Using Kafka, how do you get per-account ordering with cross-account parallelism? (Name the key and
   the mechanism.)
3. What happens to ordering for a given account if you later increase the partition count?

---

## Exercise 7 — Backpressure
Your consumers process 5,000 msg/s; producers suddenly sustain 8,000 msg/s.
1. Using Little's Law intuition, what happens to the queue over time, and is "buy a bigger queue" a
   fix?
2. List three real fixes.
3. Which single metric would you alarm on to catch this early?

---

## Exercise 8 — The outbox pattern
A service must write an `order` row to Postgres AND publish an `OrderCreated` event to Kafka.
1. Describe the failure if you "write to Postgres, then publish to Kafka."
2. Sketch the **transactional outbox** solution and explain precisely why it's atomic.
3. What is the alternative **CDC** approach, and what does it tail?
4. Why are idempotent consumers still required even with the outbox?

---

## Exercise 9 — Kafka mechanics
1. A topic has 6 partitions. What's the max number of consumers in one group that can do useful work,
   and why?
2. How do you guarantee all events for `user:123` are processed in order?
3. Two separate teams need to consume the same topic without affecting each other. How?
4. After a bug, how do you reprocess the last 3 days? What property of the log makes this possible?

---

## Exercise 10 — Apply to your own systems 🏭
1. **DCE Kafka claim stream:** explain why a *log* (not a queue) is the right choice, how you'd pick the
   partition key, what delivery guarantee you rely on, and why processing must be idempotent.
2. **`aws_batch_reprocessing` / reprocessing scripts:** which property of Kafka makes this possible, and
   why would it be impossible with RabbitMQ?
3. **Questimate Celery + Celery Beat:** why is a task queue (not Kafka) the right tool for the daily
   pipeline / background jobs?
4. **Emitting a "claim decided" event alongside the Mongo write:** name the problem with the naive
   approach and the correct pattern.

---

When done, check `solutions/m08_messaging_queues_streams/README.md`, then say **"Module 9"**.
