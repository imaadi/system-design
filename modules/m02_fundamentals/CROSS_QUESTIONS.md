# m02 — Cross-Questions ("if-and-buts") on Core Fundamentals

> These are the follow-ups interviewers use to check you *understand* the fundamentals rather than
> reciting them. Answer out loud first. Model answers are senior-level and concise.

---

### Q1. Vertical vs horizontal scaling — which is "better"?
**A.** Neither universally. **Vertical** (bigger box) is simplest and avoids distributed-systems
complexity, but hits a hard ceiling and is a single point of failure. **Horizontal** (more boxes) is
the long-term answer for the web/app tier — near-unlimited and fault-tolerant — but requires
statelessness, load balancing, and data partitioning. In practice you do both: scale the app tier
out, scale/replicate the data tier. I scale up for quick wins early, design to scale out for growth.

---

### Q2. Why can't you just keep buying a bigger database server forever?
**A.** Three reasons: (1) there's a **physical/price ceiling** — the biggest box is finite and gets
exponentially expensive at the top; (2) it's a **single point of failure** — one machine, one
outage; (3) some bottlenecks (write throughput, total dataset size, network in/out of one NIC) can't
be solved by a bigger single node. Past that point you must **replicate** (for reads/availability)
and **shard** (for writes/size) — covered in m05.

---

### Q3. What does "stateless" really mean, and why is it the key to horizontal scaling?
**A.** A stateless service holds **no per-client state in its own memory between requests** — each
request carries (or fetches from a shared store) everything it needs. That means **any** server can
handle **any** request, so a load balancer can route anywhere and you can add/remove servers freely
(and tolerate a node dying). Stateful servers force sticky sessions and lose data when a node dies.
You achieve statelessness by pushing state to **Redis (sessions), the DB (data), object storage
(files)** — which is exactly how Questimate keeps its Django app tier disposable.

---

### Q4. Latency vs throughput — give an example where improving one hurts the other.
**A.** **Batching.** If I batch 100 requests and process them together, **throughput goes up** (more
work per unit time, less per-item overhead) but **latency goes up** for an individual request (it
waits for the batch to fill). Conversely, **caching** improves both (lower latency *and* DB
offload). So when someone says "make it faster," I ask whether they mean per-request latency or
aggregate throughput — the fixes differ.

---

### Q5. Why do we use p99 instead of average latency?
**A.** Averages hide the tail — one 5-second request disappears behind thousands of fast ones, but
that user had a terrible experience. **p99** ("99% of requests are faster than this") exposes the
slow 1%. It matters because at scale the tail compounds: a page that fans out to 100 backend calls
will typically hit the p99 of at least one — so **the backend's p99 becomes the user's median**.
You design and SLO against percentiles, not means.

---

### Q6. What causes tail latency and how do you fight it?
**A.** Causes: garbage-collection pauses, cache misses, queueing/contention, a single slow shard,
network blips, and retries amplifying load. Mitigations: **timeouts** (don't wait forever),
**hedged/parallel requests** (send to two replicas, take the first), **load shedding** (drop excess
to protect the rest), more **replicas**, and keeping slow work **off the hot path** (async/queues).
Also: avoid unbounded fan-out and tune GC.

---

### Q7. Availability vs reliability vs durability — distinguish them with one system each.
**A.** **Availability** = up and responding (a social feed: better stale-but-up than an error).
**Reliability** = correct behavior over time (a system can be "up" but returning wrong data — that's
available but unreliable). **Durability** = committed data is never lost (a payments ledger / S3:
once written, it survives crashes via replication). DCE's **suppression layer protects reliability**
(correct decisions) and its **audit trail protects durability**, both distinct from uptime.

---

### Q8. How do you actually achieve four nines (99.99%) of availability?
**A.** Remove single points of failure with **redundancy at every layer** (multiple app servers,
replicated DB with automatic failover, redundant network paths, often multiple availability
zones/regions), **health checks + automatic failover** so traffic routes around dead nodes,
**graceful degradation** (serve a reduced experience instead of erroring), and operational
discipline (fast rollback, monitoring, runbooks). Note most downtime is from *deploys and config
changes*, not hardware — so safe deploys matter as much as redundancy.

---

### Q9. If two components each have 99% availability, what's the availability of a request that needs both?
**A.** If they're **in series** (the request must pass both), availabilities **multiply**: 0.99 ×
0.99 ≈ **98%** — *worse* than either. That's why long dependency chains hurt and why you minimize
hard dependencies on the critical path. If you have **two redundant** copies of a 99% component
**in parallel**, it fails only if both fail: 1 − (0.01 × 0.01) ≈ **99.99%** — much better. Series
hurts, parallel/redundancy helps.

---

### Q10. The two "consistency" meanings — explain the difference.
**A.** **ACID's C** (single-database) means a transaction takes the DB from one **valid state to
another**, preserving constraints/invariants — it's about correctness rules. **Distributed
consistency** (CAP/replication) is about **how replicas agree on the latest value** — strong vs
eventual. They're unrelated concepts that share a word. In a design discussion I say which one I
mean: "ACID-consistent within the transaction" vs "linearizable across replicas."

---

### Q11. When is eventual consistency acceptable, and when is it dangerous?
**A.** **Acceptable** when a brief window of staleness is harmless: likes/view counts, social feeds,
DNS, product catalogs, analytics. **Dangerous** when stale reads cause incorrect decisions: account
balances (double-spend), inventory (oversell), seat/lock allocation, anything with a uniqueness or
"no negative" invariant. Rule of thumb: if a stale read can let a user do something that violates a
real-world invariant, you need stronger consistency *for that operation*.

---

### Q12. State CAP precisely — and what's the most common mistake people make?
**A.** CAP: during a **network partition**, a distributed store can guarantee **at most one** of
**strong Consistency** and **Availability**. The big mistake is **"I'll choose CA"** — in a real
network, partitions are inevitable, so **P isn't optional**; CA only describes a single-node system.
The real, useful framing: when (not if) a partition happens, do I **refuse to answer to stay correct
(CP)** or **answer with possibly-stale data to stay up (AP)**? And it's often a **per-operation**
choice, not a permanent label.

---

### Q13. Is a CP system "always consistent" and an AP system "never consistent"?
**A.** No. When the network is healthy, **both** can be consistent and available — CAP only bites
*during a partition*. A **CP** system sacrifices availability *only during the partition* (errors/
blocks rather than serve stale). An **AP** system stays up during the partition and is typically
**eventually consistent** — it converges once the partition heals; it just won't promise
linearizability *during* the split.

---

### Q14. What does PACELC add over CAP?
**A.** CAP only covers the partition case. **PACELC**: *if Partition → choose A vs C; Else (normal
operation) → choose Latency vs Consistency.* The insight is that **even with a healthy network**,
every replicated read forces a trade-off: wait to coordinate replicas (stronger, slower) or answer
from the nearest replica now (faster, possibly stale). So Dynamo/Cassandra are **PA/EL** (favor
availability and latency), Spanner is **PC/EC** (favor consistency in both). It's the more honest,
everyday version of the trade-off.

---

### Q15. Roughly, how slow is disk vs RAM vs a network call? Why does it matter?
**A.** Orders of magnitude: **RAM ~100 ns**, **same-datacenter round trip ~0.5 ms** (~5,000× RAM),
**SSD random read ~100 µs–1 ms**, **HDD seek ~10 ms** (~100,000× RAM), **cross-continent round trip
~150 ms**. It matters because it dictates design: keep hot data in **memory/cache** (Redis), avoid
unnecessary **network hops** on the critical path, prefer **sequential** over random disk I/O, and
**replicate near users** to dodge cross-continent latency. The whole game is moving hot data up this
ladder.

---

### Q16. A system does "1 million operations a day." Is that a lot? Reason it out.
**A.** 1M/day ÷ 10⁵ s ≈ **~12 operations/sec average** — that's *tiny*; a single modest server
handles it easily. The catch is **peak**: if it's bursty (e.g. all at 9am), peak could be 10–50×
average → a few hundred/sec, still small. So "a million a day" sounds big but is usually a single-box
workload. (This is exactly DCE's ~1M claims/day ≈ ~12/sec — the scale challenge there is correctness
and the per-claim compute, not raw QPS.)

---

### Q17. Your service must handle 50,000 writes/sec, each 2 KB. Walk the back-of-envelope.
**A.** Throughput: **50K wps** — beyond one DB primary's comfort for durable writes → I'd **shard**
(or use a write-optimized store / LSM-based DB) and likely buffer through a **queue** to absorb
bursts. Bandwidth: 50K × 2 KB = **100 MB/s** ingest — significant but fine across a cluster. Storage:
100 MB/s × 10⁵ s/day ≈ **~8.6 TB/day** (!) → I'd ask about retention/compaction and probably tier
old data to object storage. The numbers immediately say "distributed, write-optimized, with
lifecycle management."

---

### Q18. How do you decide the consistency model for a new feature?
**A.** Ask: *what's the worst thing a stale read can cause?* If it's cosmetic (a like count off by
one for a second) → **eventual**. If the user must see their own action immediately → **read-your-
writes**. If a stale read breaks a real invariant (money, inventory, locks) → **strong** for that
operation. I default to the **weakest model that meets the requirement** because stronger consistency
costs latency and availability — and I can mix models per operation in the same system.

---

### Q19. Can you have strong consistency *and* low latency *and* high availability all at once?
**A.** Not unconditionally. PACELC says you trade them: strong consistency requires coordination
(quorums/consensus), which **adds latency** normally and **costs availability during partitions**.
You can get *close* (e.g. Spanner uses synchronized TrueTime clocks to keep strongly-consistent
commits fast, and multi-region replication to stay available) but you pay in infrastructure
complexity and cost, and still face the partition trade-off. There's no free lunch; you buy
properties with money/complexity.

---

### Q20. "Just add a cache" — when does that *not* help, or even hurt?
**A.** It doesn't help **write-heavy** workloads (low read reuse), **low-locality** access (every
key read once — nothing to reuse), or when you need **strong consistency** (the cache introduces a
staleness window). It can **hurt** via cache **stampede/thundering herd** (mass miss → DB overload),
**stale reads** causing correctness bugs, extra **invalidation complexity**, and memory cost. Caching
is a latency optimization for **read-heavy, high-locality** data — match it to the access pattern,
don't reflex it. (Deep-dived in m06.)

---

### Q21. What's the difference between scaling reads and scaling writes?
**A.** **Reads** scale relatively easily: add **replicas** (and caches/CDNs) and spread reads across
them — reads don't conflict. **Writes** are the hard part: every replica that must stay consistent
has to apply the write, and a single primary caps write throughput. To scale writes you **shard**
(partition data so different keys go to different primaries) — which introduces cross-shard query/
transaction complexity and hot-shard risk. So "read-heavy" systems scale far more cheaply than
"write-heavy" ones; identifying the ratio early (m01) drives the whole design.

---

### Q22. Why is "design for peak but reason from average" the right approach to capacity?
**A.** You **provision for peak** because that's when you'd otherwise fall over (and get paged), but
you **reason from average** because it's the stable, known number from which you derive peak (×2–10).
Over-provisioning for an imaginary 100× peak wastes money; under-provisioning for only the average
fails daily. The mature answer adds **autoscaling** for predictable cycles and **queues/load
shedding** to absorb bursts beyond provisioned peak — so you're not paying for peak 24/7.

---

### Q23. You doubled the number of servers but throughput went up only 40%, not 100%. Why? (Amdahl/USL)
**A.** Two effects. **Amdahl's Law:** some fraction of the work is **inherently serial** (a shared
lock, a single DB primary, a global counter) — that part doesn't speed up, so it caps total speedup
at 1/(serial fraction). **Universal Scalability Law:** the new nodes also add **coordination/crosstalk
cost** (cache coherence, locks, cross-shard chatter) that grows ~O(N²), which eats into the gain and,
past a point, makes throughput *decline* (retrograde scaling). The fix isn't "more nodes" — it's
**removing the serial bottleneck and reducing coordination** (shared-nothing, partition the hot
resource, go async/eventually-consistent). *Coordination is the tax on scale.*

---

### Q24. You have a service where requests arrive at 4,000/sec and each takes 25 ms. How many concurrent in-flight requests, and how big a thread/connection pool do you need?
**A.** **Little's Law: L = λ × W** = 4,000/s × 0.025 s = **100 in flight** at any instant → I'd size
the worker/connection pool to ~**100 plus headroom** (say 130–150) to absorb variance. The deeper
point: if downstream latency W rises (a slow dependency), L rises too — and if my pool is *capped*
at 100, then max throughput λ = pool/W drops, requests queue, and the service can spiral into
collapse. So I'd also add **timeouts, a bounded queue, and load shedding** so a latency bump degrades
gracefully instead of cascading.

---

### Q25. The interviewer says "I need strong consistency." What's your follow-up, and why does it matter?
**A.** I'd ask: *"Strong in what sense — single-key recency (linearizability), multi-row transaction
isolation (serializability), or both (strict serializability)?"* They're different axes with
different costs. **Linearizability** guarantees a read sees the latest write to one object (CAP's C).
**Serializability** guarantees concurrent *transactions* behave as some serial order (ACID's top
isolation level) but can still hand you a *stale* snapshot. Many KV stores are linearizable but not
serializable; snapshot-isolation DBs/read-replicas can be serializable-ish but not linearizable.
Pinning down which one avoids over-paying (strict serializability is the most expensive) or shipping
a subtle bug (assuming recency you don't have).

---

### Q26. Is it accurate to call a database "CP" or "AP"?
**A.** It's a useful shorthand but **usually an oversimplification** (the Kleppmann critique). CAP's
**C is specifically linearizability** and its **A is *total* availability** (every non-failing node
answers) under a narrow **partition** fault model. Real databases rarely sit cleanly in one bucket —
they offer tunable per-operation consistency, have leaders/leases (so not "totally available" even
when healthy), and fail mostly from slow networks/GC pauses/clock skew, which CAP doesn't model. The
senior move: use CAP to state the **hard limit** (can't have linearizability + total availability +
partition tolerance together), then describe the actual system with **consistency-model-per-operation
+ PACELC** rather than a single letter.

---

### Q27. Can you ever scale without paying a coordination cost — or is some coordination always required?
**A.** Sometimes you can avoid it *entirely*, and that's the frontier idea worth knowing: the
**CALM theorem** (Hellerstein/Alvaro) says a computation has a **consistent, coordination-free**
distributed implementation **if and only if it is *monotonic*** (its output only grows as inputs
arrive — e.g. set-union, counters that only increase, "has this ever been true"). Non-monotonic
operations (deletes, "is this the current max", uniqueness checks) inherently need coordination. So
the design lever is: **model state as monotonic where possible** (append-only logs, CRDTs, grow-only
sets) to get scale *for free*, and isolate the genuinely non-monotonic bits (which need consensus/
locks) to as small a surface as you can. That's the deep "why" behind shared-nothing and CRDTs.
