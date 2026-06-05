# Module 2 — Core Fundamentals (the entire vocabulary) ⭐

> Every later module — and every interview sentence — is built from these words: *scalability,
> latency, throughput, availability, consistency, CAP, durability.* This module defines each one
> **precisely**, shows the trade-off it hides, and gives you the **numbers** to quantify with. If
> you can teach this module back, you can hold your own in any design conversation.

This is a long module on purpose. Read it in two sittings: **§1–§5** (scaling, latency, availability)
and **§6–§9** (consistency, CAP/PACELC, the numbers). Then drill the cross-questions.

---

## 1. Scalability — vertical vs horizontal

**Scalability** = the ability to handle growing load by adding resources, *without* a redesign.
"Load" can be more requests/sec, more data, more concurrent users. A system is scalable if you can
keep up by adding hardware (ideally **linearly**: 2× hardware → ~2× capacity).

Two ways to add resources:

| | **Vertical scaling (scale up)** | **Horizontal scaling (scale out)** |
|---|---|---|
| What | Bigger machine (more CPU/RAM/disk) | More machines |
| Pros | Simple — no code/architecture change; no distributed-systems problems | (Near-)unlimited; fault-tolerant (lose one of many); cheap commodity boxes |
| Cons | **Hard ceiling** (biggest box on Earth); expensive at the top; **single point of failure** | Requires statelessness, load balancing, data partitioning; **introduces distributed-systems complexity** (consistency, coordination, partial failure) |
| When | Early stage; stateful systems hard to distribute (some SQL DBs); quick wins | Web/app tiers; anything that must survive a machine loss or grow past one box |

```
   VERTICAL (scale up)                 HORIZONTAL (scale out)
                                          ┌────┐ ┌────┐ ┌────┐ ┌────┐
   ┌──────┐   ┌──────────┐               │ S1 │ │ S2 │ │ S3 │ │ S4 │   ← add more boxes
   │ small│ → │  HUGE box │   vs          └────┘ └────┘ └────┘ └────┘
   └──────┘   └──────────┘                  ▲ behind a load balancer ▲
   one bigger machine                     many machines, share the load
```

**The senior framing:** *"I scale the stateless tiers horizontally (add app servers behind a load
balancer) and scale the data tier with replication + partitioning. I avoid vertical scaling as the
long-term answer because it has a hard ceiling and is a single point of failure."*

> Real systems do **both**: scale up the database a bit *and* out the app tier. The art is knowing
> which dimension is your bottleneck.

### The limits of scaling out: Amdahl's Law & the Universal Scalability Law ⭐ (think differently)
Here's the thing almost nobody says out loud: **adding machines does *not* give you linear speedup,
and past a point it can make things *worse*.** Two laws explain why — internalize them and you'll
reason about scale like an architect, not a hopeful optimist.

- **Amdahl's Law** — if a fraction **s** of the work is **inherently serial** (can't be
  parallelized), then no matter how many processors **N** you throw at it, the max speedup is
  capped at **1 / (s + (1−s)/N)**, which approaches **1/s** as N→∞. *Translation:* if just **5%**
  of your work is serial, your **maximum possible speedup is 20×** — even with infinite machines.
  The serial part (a global lock, a single coordinator, one shared counter) is your real ceiling.
- **Universal Scalability Law (USL, Neil Gunther)** — Amdahl + a second, nastier term:
  **coherency/coordination cost**. Nodes don't just fail to help linearly; they must *talk to each
  other* (cache coherence, locks, consensus, cross-shard chatter), and that crosstalk grows roughly
  **O(N²)**. So real throughput **rises, plateaus, then actively *declines*** as you add nodes —
  the coordination overhead eventually costs more than the node adds. This is the **retrograde
  scaling** you see when "we added more servers and it got slower."

```
  throughput
     │           ___________  ← USL: plateaus, then DROPS (coordination O(N²) dominates)
     │        ／              ＼____
     │      ／  ← Amdahl: plateaus at 1/s (serial fraction caps it)
     │    ／________________________  linear (the myth nobody achieves)
     │  ／
     └──────────────────────────────▶  number of nodes N
```

> **The one insight to carry forever:** *coordination is the tax on scale.* Every shared resource,
> every lock, every "all nodes must agree" is a serial bottleneck (Amdahl) and a crosstalk cost
> (USL). The architect's whole job in scaling is to **minimize coordination** — shared-nothing
> services, partitioning, async messaging, eventual consistency. This single idea links
> statelessness (§1), sharding (m05), and the CAP/PACELC trade-offs (§7–8): they're all about
> *buying scale by giving up coordination.*

### The prerequisite for horizontal scaling: **statelessness**
A **stateless** service keeps **no client/session state in its own memory** between requests —
everything needed is in the request, or in a shared store (DB/cache). Why it matters: if any server
can handle any request, you can add/remove servers freely and a load balancer can route anywhere.
**Stateful** servers (session stored locally) force "sticky sessions" and break when a node dies.

> **Move state out:** put sessions in **Redis**, files in **object storage (S3)**, data in the DB.
> Then the app tier is disposable and infinitely cloneable. *(This is exactly why Questimate stores
> JWT/session state in Redis, not in the Django process — the app servers stay stateless and
> horizontally scalable.)* 🏭

---

## 2. Latency vs throughput (don't confuse them) ⭐

- **Latency** = time for **one** operation (a single request), e.g. "p99 redirect latency 40 ms."
  *How long you wait.*
- **Throughput** = **rate** of operations the system handles, e.g. "50,000 requests/sec." *How much
  flows per unit time.*

They are **different and often in tension**. A highway analogy:
- **Latency** = how long *your* car takes to drive end-to-end.
- **Throughput** = how many cars cross the bridge per minute.

You can raise throughput (more lanes / batching) while latency stays the same or even *gets worse*
(e.g. **batching** requests increases throughput but each request waits for the batch → higher
latency). You optimize **for the one the requirement demands** — and they pull against each other.

```
 batching:  ↑ throughput (process 100 at once)   but ↑ latency (wait to fill the batch)
 caching:   ↓ latency (no DB hit)                and ↑ throughput (DB offloaded)
 more replicas: ↑ throughput (parallelism)        latency ~unchanged per request
```

> **Interview reflex:** when someone says "make it faster", ask **"faster latency or higher
> throughput?"** — they're not the same fix.

### Little's Law — the bridge between latency, throughput & concurrency ⭐ (think differently)
One tiny equation connects the two and is *the* most useful capacity tool you're not using:

> **L = λ × W**  →  *concurrency = arrival rate × time-in-system*
> (items in the system) = (requests/sec) × (avg seconds each spends inside)

It's astonishingly general — it holds for any stable system (a queue, a thread pool, a whole
service), regardless of distribution. Three ways it makes you think differently:

1. **Sizing pools (the everyday use).** Requests arrive at **λ = 2,000/s**, each takes **W = 50 ms
   = 0.05 s** to serve → you have **L = 100 requests in flight at any instant** → you need ~**100
   concurrent workers/threads/connections** to keep up (plus headroom). This is how you size thread
   pools, DB connection pools, and worker counts *from first principles* instead of guessing.
2. **Why latency spikes cause throughput *collapse* (the cascading-failure insight).** If
   concurrency is **capped** at N (a fixed pool of, say, 100 connections), then rearranging:
   **λ = N / W**. So if latency **W** doubles (a slow dependency, GC pause), your **max throughput
   λ halves** — the pool fills with slow requests, new ones queue, latency climbs further, and the
   service spirals. This is *the* mechanism behind most outages: a small latency bump silently
   throttles throughput until everything times out.
3. **It explains queues.** A growing queue means λ_in > λ_out (arrival outpaces service). Little's
   Law says the queue length (and thus the wait W) grows without bound unless you **shed load,
   add capacity, or speed up service.** "Just add a queue" doesn't fix an under-provisioned
   consumer — it only buffers the burst.

> **Carry this:** latency, throughput, and concurrency are **not three independent dials** — they're
> bound by L = λW. You can't "just lower latency" without it affecting how much concurrency you need,
> and a fixed concurrency limit turns any latency regression into a throughput cliff.

### Percentiles & tail latency (the senior way to talk about latency) ⭐
Never say "average latency." **Averages lie** (one 10-second request hides behind a thousand fast
ones). Use **percentiles**:
- **p50 (median):** half of requests are faster than this — the "typical" experience.
- **p99:** 99% are faster; the slowest 1%. **p999** = slowest 0.1%.
- **Tail latency** = the p99/p999. It matters enormously: a page that makes 100 backend calls will,
  on average, hit the p99 of *one* of them — so **your p99 becomes the user's p50**. At scale, the
  tail *is* the experience.

Causes of tail latency: GC pauses, cache misses, queueing, a slow shard, network blips, retries.
Mitigations (we'll meet them): **hedged requests**, **timeouts**, **load shedding**, replicas, and
keeping work off the hot path.

---

## 3. Availability, redundancy & failover

**Availability** = the fraction of time the system is up and serving correctly. Measured in
**"nines"**:

| Availability | Downtime / year | Downtime / day | Casual name |
|---|---|---|---|
| 99% | ~3.65 days | ~14.4 min | "two nines" |
| 99.9% | ~8.76 hours | ~1.44 min | "three nines" |
| 99.99% | ~52.6 min | ~8.6 s | "four nines" |
| 99.999% | ~5.26 min | ~0.86 s | "five nines" |

Each extra nine is **~10× harder and costlier**. So you don't chase nines blindly — you match
availability to the **business need** (a bank's ledger vs an internal dashboard).

**How you *get* availability: eliminate single points of failure (SPOF) via redundancy.**
- **Redundancy** = having spare copies of every critical component (servers, DBs, network paths, even
  datacenters/regions) so one failing doesn't take you down.
- **Failover** = automatically switching to a healthy replica when one fails.
  - **Active-active:** all replicas serve traffic; lose one, the rest absorb it. Best utilization,
    needs careful state handling.
  - **Active-passive (standby):** a hot/warm standby takes over on failure. Simpler, wastes the
    standby's capacity, has a failover delay.
- **Health checks** (the LB pings each server) detect failure; the LB then **routes around** dead nodes.

```
   SPOF (bad):  client → [single server] → ☠  = whole system down
   Redundant:   client → LB → [S1] [S2] [S3]   = lose S2, S1+S3 carry on (route around it)
```

> **Availability math (series vs parallel):** components **in series** multiply (a request that must
> pass A *and* B has availability A×B — so chains of dependencies *reduce* availability). Components
> **in parallel/redundant** improve it (two 99% replicas in parallel ≈ 99.99%). This is *why* you
> add redundancy and *why* you minimize hard dependencies on the critical path.

### Availability vs Reliability vs Durability (commonly confused) ⭐
- **Availability:** is it **up and responding** right now? (uptime)
- **Reliability:** does it behave **correctly** over time (right answers, no data corruption, handles
  failures gracefully)? A system can be available but unreliable (up, but returning wrong data).
- **Durability:** once data is **committed, is it never lost** — even through crashes/disk failures?
  (Usually achieved by replication + persistent storage; "eleven nines of durability" like S3.)

> A payment ledger needs **durability** above all (never lose a committed transaction). A social
> feed needs **availability** (better to show a slightly stale feed than an error). Knowing *which
> one* a system prioritizes is half of system design.

---

## 4. The single-machine limits (why we distribute at all)
You distribute because **one machine has ceilings**. Know the rough ones so you can argue "this fits
on one box" or "this needs many":
- **CPU:** tens of cores; a single process is bounded by them.
- **RAM:** up to ~TBs on big boxes, but expensive; your working set may not fit.
- **Disk:** TBs–PBs, but **throughput/IOPS** is the limit, not just size.
- **Network:** a NIC is ~10–100 Gbps; that caps bandwidth in/out of one box.
- **Connections:** an OS can hold ~tens of thousands of concurrent connections per box (tunable) —
  relevant for chat/WebSocket systems (m14).

**The reasoning move:** estimate (m01) → compare to these ceilings → *"~5 GB/day and 5K QPS fits one
beefy box + cache; ~15 TB/day and 100K QPS does not → distribute."*

---

## 5. The toolbox at a glance (each is a later module)
A mental map so the vocabulary has homes:

```
            ┌─────────────────────────── EDGE ───────────────────────────┐
  clients → │ DNS (m03) · CDN (m06) · Load Balancer (m07) · API GW (m03)  │
            └───────────────────────────────┬─────────────────────────────┘
                                             ▼
            ┌──────────────────────── APP TIER (stateless) ──────────────┐
            │  many replicas · horizontally scaled · behind the LB (m07) │
            └───────┬───────────────────┬───────────────────┬────────────┘
                    ▼                   ▼                   ▼
              Cache (m06)        Message queue /      Other services
              Redis/Memcached    log: Kafka (m08)     (microservices)
                    │                   │
                    ▼                   ▼
            ┌──────────────────── DATA TIER ──────────────────────────────┐
            │ SQL / NoSQL (m04) · replication + sharding (m05) ·           │
            │ object store / blob (m18) · search index (m16) · DW/lake     │
            └──────────────────────────────────────────────────────────────┘
```

Each box answers a need: **CDN/cache** = latency, **LB** = scale+availability, **queue** =
decoupling+bursts, **replication** = availability+read-scale, **sharding** = write-scale+size.

---

## 6. Consistency models (the subtle, high-value part) ⭐⭐

**Consistency** here = *what guarantee a reader gets about how fresh/ordered the data is*, when data
is **replicated** across nodes. (Note: this is a *different* "consistency" from the C in ACID — see
the box below.) Models, from strongest to weakest:

- **Strong (linearizable) consistency:** once a write completes, **every** subsequent read (anywhere)
  sees it. The system behaves as if there's a single copy and a single global order. *Easiest to
  reason about; costs latency and availability (you must coordinate across replicas).* Needed for
  bank balances, inventory, locks.
- **Sequential / causal consistency:** weaker; preserves *some* ordering (e.g. causally related
  operations are seen in order) but not a single global "now". Causal is popular because it matches
  intuition (you never see a reply before its comment) without full coordination cost.
- **Eventual consistency:** if writes stop, **all replicas eventually converge** to the same value —
  but for a window, different readers may see different (stale) values. *Cheap, highly available,
  low latency.* Fine for likes, view counts, DNS, social feeds, shopping-cart-ish data.

**Useful client-centric guarantees** (often what you actually need):
- **Read-your-own-writes:** *you* always see *your* latest write (even if others see it later). E.g.
  you post a comment and immediately see it. Often implemented by routing your reads to the primary
  or a session-pinned replica.
- **Monotonic reads:** you never see time go backwards (a value you saw won't "un-happen" on a later
  read from a laggier replica).

```
   STRONG:    write X=5 ──▶ every read everywhere returns 5 immediately   (coordinate → slower)
   EVENTUAL:  write X=5 ──▶ some replicas still return old value for a bit ─▶ all converge to 5
```

> ⚠️ **Two different "C"s — don't conflate:**
> - **ACID's "Consistency"** (databases) = a transaction moves the DB from one **valid state to
>   another**, respecting constraints/invariants. It's about correctness rules within one DB.
> - **Distributed "Consistency"** (CAP, replication) = **how replicas agree** on the latest value.
>   This is the one we mean in §6–§7.
> Interviewers love catching this — say which one you mean.

### Linearizability vs Serializability (the distinction that signals depth) ⭐ (think differently)
"Strong consistency" actually splits into **two orthogonal guarantees** that get sloppily merged.
Knowing the difference is a senior tell:

- **Linearizability** = a **recency** guarantee about a **single object/operation**: once a write
  completes, every later read (anywhere) sees it, and operations appear to happen **instantaneously,
  in real-time order**. It's about *"is my read fresh?"* — there is one global timeline per object.
  **This is the "C" in CAP.**
- **Serializability** = an **isolation** guarantee about **transactions over multiple objects**: the
  outcome of concurrent transactions equals *some* serial (one-at-a-time) order. **This is the "I"
  (top level) in ACID.** Crucially, it says **nothing about real time** — a serializable system may
  execute transactions in an order that ignores when they actually arrived.
- **Strict serializability** = **both** at once (serializable *and* respecting real-time order). This
  is the gold standard — what **Google Spanner** provides (using synchronized TrueTime clocks).

```
                 single object          │   multi-object transactions
   recency-aware │ LINEARIZABILITY       │   STRICT SERIALIZABILITY
   recency-blind │ (sequential consist.) │   SERIALIZABILITY
```

> **Why it matters in design:** they fail differently. A store can be **linearizable but not
> serializable** (fresh single-key reads, but no multi-row transaction isolation — many KV stores).
> It can be **serializable but not linearizable** (transactions are correctly isolated, but a read
> replica can still hand you a *stale* committed snapshot — common with snapshot isolation / read
> replicas). When an interviewer says "I need strong consistency," the depth move is to ask: *"Do you
> need single-key recency (linearizability), transaction isolation (serializability), or both (strict
> serializability)?"* — because each has a very different cost and a different default in real DBs.

**The trade-off in one line:** *stronger consistency → more coordination → higher latency and lower
availability under failure.* You pick the **weakest model that still meets the requirement**, because
weaker is cheaper and more available. (Likes? eventual. Account balance? strong.)

---

## 7. The CAP theorem (and why most people state it wrong) ⭐⭐

**CAP** (Brewer): in a distributed data store, when a **network Partition (P)** happens, you can
guarantee **at most one** of **Consistency (C)** and **Availability (A)** — you must choose.

Definitions (CAP's own, precise):
- **C (Consistency):** every read sees the most recent write (i.e. **linearizable**).
- **A (Availability):** every request to a non-failing node gets a (non-error) response.
- **P (Partition tolerance):** the system keeps working despite the network dropping/delaying
  messages between nodes.

**The crucial, widely-misunderstood point:** in any real distributed system, **partitions WILL
happen** (networks fail), so **P is not optional**. CAP is therefore really a choice *during a
partition* between **C and A**:

- **CP system:** during a partition, **refuse/err on operations** that can't guarantee consistency
  (sacrifice availability to stay correct). E.g. a system that won't accept a write it can't
  replicate to a quorum. *Use when correctness > uptime* (banking, locks, inventory).
- **AP system:** during a partition, **keep serving** (possibly stale) data and reconcile later
  (sacrifice consistency to stay up). E.g. Dynamo-style stores, DNS, shopping carts. *Use when uptime
  > perfect freshness.*

```
        Network partition occurs ✂  (nodes can't talk)
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   CHOOSE C (CP)           CHOOSE A (AP)
   reject/err to stay      answer with maybe-stale
   correct  → less         data → stay up → reconcile
   available               when partition heals
```

**Common mistakes (don't make them):**
1. "I'll pick CA." ❌ You don't get to drop P in a real network — CA is only for single-node systems.
2. Treating it as a permanent label. The choice is **per-operation, during a partition** — many
   systems are tunable (e.g. Cassandra/Dynamo let you choose consistency per query).
3. Thinking "AP = no consistency ever." No — AP systems are usually **eventually consistent**; they
   just don't guarantee linearizability *during* a partition.

> **What to actually say:** *"It's not a free choice in normal operation — both C and A are fine when
> the network is healthy. CAP only forces a choice **during a partition**: do I return possibly-stale
> data (AP) or refuse to answer to stay correct (CP)? For [this feature] I'd choose [X] because [the
> cost of staleness vs the cost of downtime]."*

### The grown-up take: stop labeling databases "CP" or "AP" ⭐ (think differently)
CAP is a great teaching device but its definitions are **narrower than people use them**, and the
sophisticated answer acknowledges this (it's the well-known critique by **Martin Kleppmann**,
*Designing Data-Intensive Applications*):
- CAP's **C is specifically linearizability** (§6) — not "consistency" in general. A system can give
  you read-your-writes, causal, or serializable-but-stale guarantees and CAP says nothing about them.
- CAP's **A is *total* availability** — *every* request to *every* non-failing node must succeed.
  Most real systems don't promise that even when healthy (they have leaders, leases, timeouts).
- CAP's **P** assumes a specific fault model (arbitrary message loss) and ignores the things that
  cause most *real* incidents: slow networks, node pauses (GC), and clock skew.

So **most production databases are honestly neither purely "CP" nor "AP"** — the labels
oversimplify. The mature framing: *"I don't pick a CAP letter; I pick the **consistency model per
operation** (§6) and reason about **availability under partition vs latency under no partition**
(PACELC). CAP just tells me the hard limit: I can't have linearizability **and** total availability
**and** survive partitions all at once."* Use CAP to show you know the boundary; use **consistency
models + PACELC** to actually describe the system.

---

## 8. PACELC (the more honest version of CAP)
CAP only talks about behavior **during a partition**. But you make consistency/latency trade-offs
**all the time, even when the network is fine.** **PACELC** captures both:

> **if Partition (P): trade Availability (A) vs Consistency (C);
> Else (E): trade Latency (L) vs Consistency (C).**

In plain terms: *during a partition* it's the CAP choice (A vs C); *the rest of the time* you still
choose between **lower latency** (don't wait to coordinate replicas) and **stronger consistency**
(do wait). Examples:
- **Dynamo / Cassandra:** **PA/EL** — favor availability under partition, and latency otherwise.
- **Spanner / a strongly-consistent SQL setup:** **PC/EC** — favor consistency in both cases (and
  pay the latency, which Spanner mitigates with synchronized clocks).

> PACELC is the sophisticated answer: "Even with no partition, every replica read forces a
> latency-vs-consistency call. I'd default to low-latency eventual reads and reserve strong reads
> for the operations that truly need them."

---

## 9. Back-of-the-envelope numbers (memorize these) ⭐⭐

You quoted the *method* in m01; here's the **data**. Knowing these cold lets you reason about
"memory or disk? one box or many? cache or not?" instantly.

### 9a. Latency numbers every engineer should know (orders of magnitude)
> These descend from **Jeff Dean / Peter Norvig's "Latency Numbers Every Programmer Should Know"**,
> updated for modern hardware (cf. Colin Scott's interactive version). **Don't memorize digits —
> memorize the orders of magnitude and the *ladder* below.** Real values vary by hardware (NVMe vs
> SATA, DDR4 vs DDR5); what's stable is the *ratios*.

| Operation | Time | Intuition |
|---|---|---|
| L1 cache reference | ~0.5–1 ns | the CPU's fastest memory |
| Branch mispredict | ~3–5 ns | — |
| L2 cache reference | ~4–7 ns | ~10× L1 |
| Mutex lock/unlock | ~15–25 ns | uncontended |
| **Main memory (RAM) reference** | **~100 ns** | RAM is ~100× slower than L1 |
| Compress 1 KB (Snappy/Zstd) | ~1–3 µs | — |
| Send 1 KB over 10 Gbps network | ~1 µs | transmission time (not RTT) |
| **Read 1 MB sequentially from RAM** | **~3–25 µs** | memory-bandwidth bound (~30–100 GB/s) |
| **SSD random read (NVMe)** | **~16–100 µs** | NVMe ~16 µs, SATA SSD ~100 µs · ~100–1000× RAM |
| **Round trip within same datacenter** | **~0.5 ms (500 µs)** | network in a DC |
| **Read 1 MB sequentially from SSD** | **~0.3–1 ms** | NVMe sequential is fast |
| **Disk (HDD) seek** | **~10 ms** | ~100,000× slower than RAM — *why caches exist* |
| Read 1 MB sequentially from HDD | ~20–30 ms | spinning rust |
| **Round trip CA ↔ Netherlands** | **~150 ms** | speed of light is real & unbeatable |

**The mental ladder (this is what actually matters):**
> **L1 (1 ns) ≪ RAM (100 ns) ≪ SSD random (10–100 µs) ≈ DC round trip (0.5 ms) ≪ disk seek
> (10 ms) ≪ cross-continent (150 ms).** Memory ≫ flash/network ≫ disk ≫ another continent.
> *Every optimization is moving hot data up this ladder and doing slow things off the critical path.*
> Two ratios to burn in: **RAM→disk-seek ≈ 100,000×** (the reason for caching) and
> **same-DC→cross-continent ≈ 300×** (the reason for CDNs and regional replicas — you can't beat the
> speed of light, so you move the data closer).

### 9b. Powers of two & data-size units
| Power | ≈ | Unit |
|---|---|---|
| 2¹⁰ | ~1 thousand | 1 KB |
| 2²⁰ | ~1 million | 1 MB |
| 2³⁰ | ~1 billion | 1 GB |
| 2⁴⁰ | ~1 trillion | 1 TB |
| 2⁵⁰ | ~1 quadrillion | 1 PB |

Typical sizes: a char/ASCII ≈ 1 byte · an int ≈ 4 bytes · a UUID ≈ 16 bytes · a short text record
(tweet/URL row) ≈ a few hundred bytes · a web page ≈ 100s of KB · a photo ≈ 100s of KB–few MB · a
minute of video ≈ ~10–50 MB.

### 9c. Time & rate conversions
- **1 day ≈ 86,400 s ≈ 10⁵ s** (the single most useful number — memorize it).
- **1 year ≈ 31.5M s ≈ 3×10⁷ s.**
- **"1 million/day" ≈ 12/sec** average. **"1 billion/day" ≈ ~11,500/sec** average.
- **Peak ≈ 2–10× average** (state your factor).
- QPS from per-day: divide by 10⁵. Storage/year: ×365 (≈ ×400 to round up), then × replication.

### 9d. Worked example (tie it together)
> "Design for **500M reads/day, 5M writes/day, 1 KB/record, keep 5 years.**
> Reads: 500M/10⁵ = **5K rps avg**, ~25K peak. Writes: 5M/10⁵ = **50 wps avg**.
> → read-heavy 100:1 → **cache + read replicas**, writes trivial for one primary.
> Storage: 5M×1KB = 5 GB/day → ×400×5 ≈ **~10 TB over 5 yrs** → ×3 replicas ≈ 30 TB → fits a small
> sharded cluster; *not* a single-box problem at 5 yrs but no emergency either.
> Bandwidth: 5K reads/s × 1 KB ≈ **5 MB/s** egress — trivial."

That whole paragraph takes ~90 seconds and decides the architecture. That's the skill.

---

## 10. From your systems 🏭
- **Statelessness for horizontal scale:** Questimate keeps **JWT/session in Redis**, not in the
  Django worker → app servers are stateless and clone freely behind nginx/uWSGI. Textbook §1.
- **Latency via precompute (throughput vs latency):** Questimate **precomputes risk decomposition
  in parallel into snapshot tables** so the API is an O(1) read — trading background throughput/work
  for hot-path latency. §2.
- **Consistency choice:** market/analytics data is **eventually consistent** (a snapshot a few
  minutes old is fine), while **auth/session revocation is strongly consistent** (delete the Redis
  key → instantly revoked). You already pick the *weakest model that meets the need* per feature. §6.
- **Availability & the data ladder:** DCE keeps **hot reference data in Redis** (RAM, ~100 ns–sub-ms)
  vs **claims in MongoDB** (disk) — you've literally placed data on the §9 latency ladder by access
  frequency. And the **suppression layer = reliability** (correct output) distinct from availability.
- **Durability:** a committed claim decision and its audit trail **must not be lost** (durability),
  even though a few seconds of processing latency is acceptable (availability/latency relaxed). You
  prioritized the right "-ility" for the domain.

---

## 11. Key concepts (interview-ready)
- **Scale out, not up**, for the long term; **statelessness** is the enabler. Vertical scaling has a
  ceiling and is a SPOF.
- **Scaling isn't linear:** **Amdahl** (serial fraction *s* caps speedup at 1/s) + **USL**
  (coordination cost makes throughput *plateau then drop*). ⇒ **coordination is the tax on scale** —
  the architect minimizes it (shared-nothing, partition, async, eventual consistency).
- **Latency ≠ throughput**, and **Little's Law (L = λW)** binds them via concurrency — it sizes your
  pools and explains why a latency spike collapses throughput. Talk in **percentiles (p99)**, not
  averages; **tail latency** dominates at scale.
- **Availability = nines**, achieved by **redundancy + failover + health checks**, by removing
  **SPOFs**. Series multiply (worse), parallel/redundant improves.
- **Availability vs reliability vs durability** are distinct — know which the system prioritizes.
- **Consistency models:** strong → causal → eventual; plus read-your-writes / monotonic reads. Pick
  the **weakest that meets the requirement** (stronger = more coordination = slower/less available).
- **Linearizability (single-object recency, CAP's C) ≠ serializability (transaction isolation,
  ACID's I).** Both = **strict serializability** (Spanner). Ask which one "strong" means.
- **CAP:** P is mandatory in real networks, so it's a **C-vs-A choice during a partition** (CP =
  correct-but-may-error; AP = up-but-may-be-stale). "CA" isn't a real distributed option — and
  honestly, **most real DBs are neither cleanly CP nor AP** (CAP's C=linearizability, A=*total*
  availability are narrow). Reason with **consistency-model-per-operation + PACELC** instead.
- **PACELC:** even with no partition, it's **latency vs consistency** (Else-Latency-Consistency).
- **The numbers:** RAM 100 ns ≪ SSD random 10–100 µs ≈ DC RTT 0.5 ms ≪ disk seek 10 ms ≪
  cross-continent 150 ms (RAM→disk ≈ 100,000×; same-DC→cross-continent ≈ 300×); 1 day ≈ 10⁵ s;
  "1M/day" ≈ 12/sec; peak ≈ 2–10× avg.

---

## 12. How to go deeper (the well-researched reading list)
- Re-derive the §9d worked example for 3 made-up systems until it's reflexive (~90 s each).
- For any system you use, name: its dominant NFR, its consistency model per feature, its SPOFs.
- **The canon (read these — they're the *source* of this module):**
  - **Kleppmann, *Designing Data-Intensive Applications* (DDIA)** — the single best book here;
    Ch. 5 (replication), 7 (transactions/isolation), 9 (consistency & consensus, linearizability).
  - **Brewer, "CAP Twelve Years Later"** — the per-operation, during-partition reframing of CAP.
  - **Kleppmann, "Please stop calling databases CP or AP"** — the §7 grown-up critique.
  - **DeCandia et al., the Amazon *Dynamo* paper** — eventual consistency & AP design in practice.
  - **Corbett et al., *Spanner*** — strict serializability via TrueTime (the PC/EC, "both" corner).
  - **Dean & Barroso, "The Tail at Scale"** — why p99 dominates and how to fight it.
  - **Gunther, the Universal Scalability Law** — the math behind retrograde scaling (§1).
  - *(Advanced, optional)* **Hellerstein & Alvaro, the CALM theorem** — a program has a
    coordination-free consistent implementation **iff it is monotonic**; the deep "why coordination
    is sometimes avoidable entirely" result behind §1's "coordination is the tax on scale."

---

## Practice
➡️ Test yourself with [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do
[`EXERCISES.md`](EXERCISES.md) before checking
[`solutions/m02_fundamentals/`](../../solutions/m02_fundamentals/README.md).

When you've done the exercises, say **"Module 3"** to build *Networking & Communication* (DNS, HTTP,
REST vs gRPC vs GraphQL, WebSockets/SSE/polling, API gateway).
