# m01 — Solutions (the framework)

> Worked answers. Yours don't need to match word-for-word — check that you hit the **structure**
> (requirements-first, numbers with implications, trade-offs with costs).

---

## Exercise 1 — Scope "Design a URL shortener"

**Functional (core):**
1. Create a short URL from a long URL.
2. Redirect a short URL → original (the hot path).
3. (optional) Custom alias.
**Cut to stretch:** 4) click analytics, 5) expiry/TTL, 6) user accounts/auth. *State the cut:*
"I'll focus on create + redirect; analytics and expiry are stretch."

**Non-functional (with targets):**
- **Scale:** ~100M new URLs/day, read-heavy.
- **Read:write ratio:** ~100:1 (people click far more than they create).
- **Latency:** redirect < 100 ms p99 (it's in the user's critical path).
- **Availability:** high — 99.9%+; a dead redirect breaks every link ever shared.
- **Consistency:** eventual is fine for reads (a new short link being visible a second late is OK);
  creation should be durable.
- **Durability:** never lose a mapping — links live "forever".

**Clarifying questions:** "Global or single region?" / "Do links expire?" / "Custom aliases?"
**Assumption if shrugged:** "Global, links don't expire, no custom alias for v1, managed cloud OK."

---

## Exercise 2 — Estimation drills

1. **500M reads/day** ÷ 10⁵ s ≈ **5,000 reads/sec** avg. **5M writes/day** ÷ 10⁵ ≈ **50 writes/sec**
   avg. **Ratio ≈ 100:1** (read-heavy). Peak at 5× → **~25,000 reads/sec**. *Implication:* reads
   dominate → cache + read replicas; writes are trivial for one primary.
2. 5M writes/day × 1 KB = **~5 GB/day**. ×365 ≈ **~1.8 TB/year**. ×3 replicas ≈ **~5.5 TB/year**.
   *Implication:* fits comfortably; no urgent sharding from size alone for years.
3. 10M × 1.5 MB = **~15 TB/day** (!). ×365 ×5 ≈ **~27 PB** over 5 years. *Implication:* this is a
   **blob storage + CDN** problem, not a DB problem; metadata is tiny vs the bytes.
4. 2B/day ÷ 10⁵ ≈ **20,000 req/sec** avg; peak (×5) ≈ **100,000 req/sec**. *Implication:* needs
   horizontal scaling + sharding + CDN/caching; no single box.
5. (implications given inline above — the habit is what's graded.)

---

## Exercise 3 — Property → product
1. **In-memory cache** (sub-ms key reads, high throughput) → **Redis / Memcached**.
2. **Message queue** (decouple producer/consumer, buffer, retry) → **RabbitMQ / SQS**.
3. **Distributed log** (ordered, durable, replayable, multi-consumer) → **Kafka** (or Pulsar/Kinesis).
4. **Load balancer** (spread traffic, health-check, route around failures) → **L7 LB: nginx /
   Envoy / ALB**.
5. **CDN** (edge-cached static content close to users) → **CloudFront / Cloudflare / Akamai**.

*The pattern:* state the generic component + the property, **then** the product. Never product-only.

---

## Exercise 4 — Trade-off framing
1. "A wide-column store **buys** horizontal write scale + simple key access, **costs** joins,
   multi-row transactions, and ad-hoc queries; I'd pick it **because** my access is point-reads by
   key at huge scale — but only if I confirm I don't need transactions."
2. "Caching **buys** low latency + DB offload, **costs** staleness + invalidation complexity +
   memory; I'd cache the **hot read path** with a short TTL **because** reads dominate 100:1."
3. "Strong consistency **buys** every reader sees the latest write, **costs** higher write latency
   and reduced availability under partitions (CAP); I'd use it **only** where correctness demands it
   (e.g. payments), not for a social feed."
4. "A queue **buys** decoupling + burst absorption + retry/durability, **costs** added latency,
   eventual processing, and ordering/idempotency concerns; I'd add it **because** the producer is
   bursty and the consumer is slow."
5. "Sharding **buys** scale beyond one node, **costs** cross-shard queries/joins, rebalancing, and
   hot-shard risk; I'd shard **when** one primary can't take the write volume — and pick the shard
   key to spread load evenly."

---

## Exercise 5 — Short-code generation deep-dive (model answer)
> "Three options. **(A) Hash the long URL** (e.g. MD5) and take 7 base-62 chars — simple and
> stateless, but **collisions** are possible, so I need a check-and-retry against the store, which
> adds a read on every write. **(B) A global auto-increment counter**, base-62 encoded — guarantees
> uniqueness and gives short codes, but the counter is a **single point of failure and a write
> bottleneck**, and sequential IDs are guessable/enumerable. **(C) A key-generation service that
> hands out pre-allocated ID *ranges*** to each app server (or Snowflake-style IDs from timestamp +
> machine + sequence) — servers mint codes locally with **no coordination on the hot path**, so it
> scales horizontally. I'd pick **C**: it removes the write-path bottleneck and the SPOF if I make
> the KGS highly available (replicated, and servers pre-fetch the next range so a brief KGS outage
> doesn't block writes). The **residual cost** is operational complexity — another service to run and
> monitor — and you can 'lose' a range of IDs on a crash, which is fine since the space is huge."

(Options + trade-offs + pick + justification + residual cost = full marks.)

---

## Exercise 6 — Stress the URL-shortener design
- **SPOFs:** (1) the **DB primary** for writes → use a replicated primary with automatic failover;
  (2) the **KGS** → replicate it and pre-fetch ID ranges so writes survive a brief outage.
- **Hot key:** a viral short link → it's already cache-served, but to protect one Redis node I'd
  put it behind a **CDN/edge cache** for the redirect, and/or replicate the hot key across cache
  nodes.
- **Consistency window:** right after creation, a read on a different replica/cache might 404 →
  bound it by writing to the primary + populating cache on create (write-through), or accept a
  sub-second eventual window.
- **Cache tier dies entirely:** system **survives** — reads fall through to the DB — but at the cost
  of a latency spike and a load surge on the DB (a potential **thundering herd**). Mitigate by
  sizing DB read replicas for fallback, using request coalescing, and warming the cache on recovery.

---

## Exercise 7 — DCE through the framework 🏭 (model)
1. **Requirements:** FR — for each incoming claim, recommend DENY/BYPASS/ROUTE/UPDATE. Dominant
   **NFR = correctness + auditability** (a wrong auto-deny has real consequences) and throughput at
   ~1M/day. 
2. **Scale:** ~1M claims/day ≈ **~12/sec average**, bursty around payer/cycle windows → size for peak.
3. **Interface:** claims arrive as messages (Kafka) / batch; output = a recommendation + evidence
   written to the recommendations store; downstream adjudication (CIW/WGS) consumes it.
4. **Data model:** **claims in MongoDB** (deeply nested, variable claim JSON — flexible schema fits);
   **reference/eligibility/auth data in Redis** (hot, low-latency lookups during scoring); model
   artifacts in MinIO. *Right store per access pattern.*
5. **High-level flow:** ingest → **exclusion rules** (keep certain claims out of the pipeline) →
   **ML models + rule-based models** produce recommendations → **suppression rules** at the exit
   (block incorrect recommendations) → write decision + audit trail.
6. **Deep-dive:** the **suppression layer** as a correctness safety net (rules over ERR_CDS/LOB/
   claim type, vectorized in pandas, logged to Mongo for audit) — or Questimate's **Redis-backed JWT**
   (stateless verify + instant revocation) as an options/trade-off story.
7. **Bottlenecks/evolution:** real migrations — OpenShift→EKS ($75K→$20K/mo), ENSO→K8s, Lambda/Glue
   →Snowpark (60% ETL cut) — with parity-validated via comparison scripts. Step 7, lived.

---

## Exercise 8 — Red flags
1. **Solution-first / buzzword soup.** → Scope requirements first, then choose tech with a *why*.
2. **Going silent.** → Narrate even uncertainty: "I'm weighing two options…".
3. **Over-engineering.** → Design for the *stated* 10K-user scale; *mention* how it'd evolve.
4. **Defensiveness under pushback.** → "Good point, that changes things — let me adjust."
5. **Stopping early / no failure analysis.** → Use the time: deep-dive + bottlenecks + "next steps".
