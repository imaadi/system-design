# m02 — Solutions (Core Fundamentals)

> Check the *reasoning*, not exact digits. Rounding differences are fine; the implication is the point.

---

## Exercise 1 — Classify the requirement
1. **Functional** (a feature).
2. **Non-functional → availability.**
3. **Non-functional → consistency** (specifically *read-your-writes*).
4. **Non-functional → scalability.**
5. **Non-functional → durability.**
6. **Non-functional → latency** (p99).
7. **Non-functional → consistency** (strong; protects an invariant — no negative balance).

---

## Exercise 2 — The latency ladder
Fastest → slowest:
- **L1 cache** ~1 ns
- **RAM reference** ~100 ns
- **Same-datacenter round trip** ~0.5 ms (500 µs)
- **SSD random read** ~100 µs (≈0.1 ms) — *note: SSD read is comparable to / a bit faster than a DC
  round trip; both ≪ disk seek* (order: RAM ≪ SSD ≈ DC-RTT ≪ HDD seek)
- **HDD seek** ~10 ms
- **Cross-continent round trip** ~150 ms

(Strict order by typical numbers: L1 1 ns → RAM 100 ns → SSD 100 µs → DC RTT 500 µs → HDD seek
10 ms → cross-continent 150 ms.)

**Disk seek vs RAM:** ~10 ms vs ~100 ns ≈ **100,000× slower** (10⁵). That gap is *why* caching exists.

---

## Exercise 3 — Photo app back-of-envelope
Givens: 300M DAU; 20 views/user/day; 2 uploads/user/day; photo 300 KB; metadata 1 KB; 5 yrs; ×3.

1. **Reads/sec:** 300M × 20 = 6B views/day ÷ 10⁵ = **60,000 reads/sec avg**; peak ×3 ≈ **180K
   reads/sec**. **Uploads/sec:** 300M × 2 = 600M/day ÷ 10⁵ = **6,000 uploads/sec avg**.
   *Implication:* read-heavy and high absolute QPS → **CDN + cache** for reads, horizontally-scaled
   upload path.
2. **Read:write ≈ 60K : 6K = 10:1.** *Implication:* read-optimize, but writes aren't negligible.
3. **New photo storage/day:** 600M uploads × 300 KB = **180 TB/day** (raw). Over 5 yrs: 180 TB ×
   ~400 × 5 ≈ **~360 PB**; ×3 replication ≈ **~1 exabyte**. *Implication:* this is an **object-store
   + CDN + lifecycle/tiering** problem; you'd compress, tier cold photos to cheaper storage, and not
   keep 3× hot replicas of everything forever.
4. **Metadata over 5 yrs:** 600M × 1 KB = 600 GB/day → ×400×5 ≈ **~1.2 PB** → ×3 ≈ **~3.6 PB**.
   *Implication:* metadata (~PB) is **~300× smaller** than photos (~360 PB) — the ratio is just
   photo-size/metadata-size = 300 KB / 1 KB = 300×. So metadata fits a sharded database while
   **bytes belong in blob storage**. Classic "split metadata from blobs."
5. **Read bandwidth:** 60K reads/s × 300 KB ≈ **18 GB/s** (peak ~54 GB/s). *Implication:* must be
   served from a **CDN at the edge**, not from origin — origin egress at 18 GB/s is huge/expensive.
6. **Photos → object storage (S3) + CDN; metadata → a sharded DB.** Because the access patterns and
   sizes differ by ~100×, and blobs want cheap durable storage + edge delivery while metadata wants
   indexed queries.

---

## Exercise 4 — Consistency model picker
1. **Eventual** — a like count off by one for a second is cosmetic.
2. **Strong** — a stale balance enables double-spend; protects a money invariant.
3. **Read-your-writes** — you must see your own tweet immediately; others can see it eventually.
4. **Strong** (for the decrement/oversell check) — stale stock causes overselling. (You can show an
   *approximate* count eventually-consistently, but the *purchase decrement* must be strong.)
5. **Read-your-writes** — the editor must see their new bio; propagation to others can lag.
6. **Eventual** — DNS is the canonical eventually-consistent system; TTL-bounded staleness is fine.

---

## Exercise 5 — CAP / PACELC
1. **CP:** the minority side **refuses the write (errors)** because it can't reach a quorum /
   guarantee consistency — stays correct, loses availability for that partition. **AP:** the minority
   side **accepts the write** locally and serves possibly-stale reads, then **reconciles** (e.g.
   last-write-wins or version vectors) when the partition heals — stays available, loses
   linearizability temporarily.
2. Banking ledger → **CP** (correctness over uptime). Presence/"who's online" → **AP** (a slightly
   wrong online indicator is fine; availability matters). Shopping cart → **AP** (Amazon's Dynamo
   case: never block adds-to-cart; reconcile conflicts). Distributed lock service → **CP** (a lock
   that isn't globally agreed is worse than no lock; e.g. ZooKeeper/etcd are CP).
3. Payment ledger → **PC/EC** (consistency under partition *and* willing to pay latency normally —
   correctness dominates). Recommendation feed → **PA/EL** (stay available under partition; serve
   from nearest replica with low latency normally — a slightly stale rec is fine). The **Else**
   half: payment waits for a quorum/consensus commit even when healthy (slower, correct); the rec
   feed reads from the closest replica without coordinating (faster, maybe stale).

---

## Exercise 6 — Availability math
1. Series → multiply: 0.999 × 0.999 × 0.9995 ≈ **0.9975 ≈ 99.75%**. **Worse** than any single
   component — series dependencies only ever *reduce* availability. (≈ 21+ hours downtime/year.)
2. Redundant pair, succeeds if either up: 1 − (0.001 × 0.001) = **0.999999 ≈ 99.9999% ("six nines")**
   for the pair. (Parallel redundancy of the *failure*, ignoring shared dependencies.)
3. (a) **Long dependency chains hurt** — every hard dependency on the critical path multiplies in,
   dragging availability down; minimize them (and make non-critical ones degrade gracefully).
   (b) **Redundancy dramatically helps** — parallel copies turn a component's failure probability
   into its square. This is the core reason we replicate.

---

## Exercise 7 — Latency vs throughput
1. **Throughput** (snappy individually, fails under aggregate load) → scale out app tier / add
   capacity / autoscale / shed load.
2. **Latency** (low traffic, slow per request) → profile the request path; add caching, fix slow
   queries/N+1, reduce hops.
3. **Latency** caused by a throughput optimization (**batching raises throughput but adds per-item
   wait**) → cap batch size / max wait time, or separate the real-time path from the batch path.
4. **Throughput** → horizontal scale (more replicas), sharding for write-bound parts, caching/CDN to
   offload reads.

---

## Exercise 8 — Little's Law
1. **L = λ × W** = 3,000/s × 0.040 s = **120 in flight**. Provision the pool at ~**150–180** workers/
   connections (120 + headroom for variance/bursts).
2. New W = 120 ms = 0.12 s, pool capped at **N = 150**. Max sustainable **λ = N / W = 150 / 0.12 =
   1,250 req/s** — down from 3,000. So a 3× latency increase **cut throughput to ~40%**. Requests
   beyond 1,250/s **queue up**; the queue grows unbounded (arrivals > departures), wait time climbs,
   clients time out and **retry** (adding *more* load), and the service collapses — a classic
   **latency-induced throughput cliff / retry storm**. Mitigate with timeouts, bounded queues, load
   shedding, and circuit breakers (m10).
3. **Lesson:** latency, throughput, and concurrency aren't independent — **L = λW** ties them. A
   fixed concurrency limit turns any latency regression into a throughput collapse, so you must plan
   for graceful degradation, not just "happy-path" capacity.

---

## Exercise 9 — Amdahl & USL
1. Serial fraction s = 0.10 → max speedup = **1/s = 10×**, no matter how many machines. (The 10%
   serial part is the hard ceiling.)
2. **Universal Scalability Law.** Mechanism: the extra nodes add **coordination/coherency overhead**
   (locks, cache coherence, cross-node/cross-shard chatter) that grows ~**O(N²)**; past a point that
   crosstalk costs more than each node contributes, so throughput **plateaus and then declines**
   (retrograde scaling).
3. Three moves: (a) **remove the serial bottleneck** — replace a global lock/single primary/global
   counter with per-partition or sharded state; (b) **go shared-nothing + partition** so nodes don't
   coordinate (independent shards, consistent hashing); (c) **make work async / eventually consistent**
   (queues, batching, CRDTs/monotonic state) so nodes don't synchronously wait on each other.
   *(All three reduce the serial fraction and the coordination term — the two things that kill linear
   scaling.)*

---

## Exercise 10 — Linearizability vs serializability
1. **Linearizability** — single object (your balance), you must see the most-recent write (recency).
2. **Serializability** — multi-object transaction (debit A + credit B) that must be isolated from
   other concurrent transfers so balances stay correct; it's about transaction *ordering/isolation*.
3. **Neither** (eventual is fine) — a lagging like count is cosmetic; no recency or isolation needed.
4. **Both → strict serializability** — transaction isolation (serializability) **and** real-time-fresh
   reads (linearizability). This is the most expensive guarantee; **Spanner** is the canonical example
   (achieved with TrueTime synchronized clocks).

---

## Exercise 11 — Apply to your own system 🏭 (model, using Questimate)
1. **Stateless tier:** the **Django/DRF app servers** — session/JWT state lives in **Redis**, so any
   uWSGI worker behind nginx can serve any request and you can add workers freely.
2. **Throughput-for-latency trade:** **precomputing risk decomposition in parallel
   (ThreadPoolExecutor) into snapshot tables** via the nightly Celery pipeline — heavy background
   work so the user-facing API is an O(1) read.
3. **Eventual:** market/analytics snapshots (a few minutes stale is fine). **Strong:** **JWT/session
   revocation** — deleting the Redis token key revokes instantly; you can't tolerate a "revoked but
   still works" window for auth.
4. **Dominant "-ility":** for Questimate, **read latency on expensive analytics** → drove the
   precompute-into-snapshots architecture. (For DCE, the dominant ones are **reliability +
   durability** of decisions → drove the suppression layer and Mongo audit trail.)
