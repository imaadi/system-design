# Module 9 — Distributed Systems Theory ⭐⭐

> You've been *using* the results all along — failover (m05), quorums (m05), ordering (m08), idempotency
> (m03/m08). This module gives you the **why** underneath: the fallacies that cause every distributed
> bug, the one ambiguity that makes it all hard (**you can't tell slow from dead**), how **time** lies,
> how nodes **agree** (consensus/Raft), and how to do **transactions across services** (2PC vs Saga).
> It's the most theoretical module — so every section ends in a *practical* implication and a system you
> already know.

Read in passes: **§1–§4** (partial failure, the fallacies, slow-vs-dead, time/clocks) and **§5–§9**
(consensus/Raft, coordination services, 2PC vs Saga, distributed locks, CRDTs).

---

## 1. The defining property: partial failure ⭐

A single program either works or crashes — **all or nothing**. A distributed system has a third state:
**partial failure** — *some* nodes/links fail while others keep running, and **no node has a complete,
current view** of the whole system. Worse, failures are **non-deterministic** (the same operation might
work, time out, or half-succeed) and you often **can't tell what happened** (did the request fail, or
succeed but the response was lost?).

> **This is the root of everything in this module.** Every hard problem — consensus, ordering, exactly-
> once, failover — exists because a node must make decisions with **incomplete, possibly-stale, possibly-
> wrong information about the rest of the system**, while parts of it may be silently broken. Keep this
> lens; it explains *why* the rest is hard.

---

## 2. The 8 fallacies of distributed computing ⭐ (the bugs everyone ships)

A classic list (Deutsch & Gosling, Sun) of false assumptions developers make. Each fallacy = a class of
real outage:

1. **The network is reliable.** → It drops/delays/reorders packets. *Design for failure: retries,
   timeouts, idempotency.*
2. **Latency is zero.** → Remote calls are *orders of magnitude* slower than local (m02 ladder); chatty
   designs die. *Batch, cache, colocate; mind round trips (m03).*
3. **Bandwidth is infinite.** → It's finite and costs money. *Compress, paginate, don't ship huge
   payloads.*
4. **The network is secure.** → It isn't. *Encrypt (TLS/mTLS), authenticate, zero-trust (m03).*
5. **Topology doesn't change.** → Nodes come and go, IPs change, things move. *Service discovery, don't
   hard-code hosts.*
6. **There is one administrator.** → Many teams/clouds/configs. *Expect inconsistent config/versions.*
7. **Transport cost is zero.** → Serialization, marshalling, and network egress all cost CPU/$. *Mind
   payload formats (gRPC/protobuf, m03).*
8. **The network is homogeneous.** → Mixed hardware, protocols, versions. *Design for heterogeneity;
   version your APIs.*

> **Use it as a checklist.** When you design (or critique) a system, walk the fallacies: "what if the
> network drops this? what's the latency of this hop? what if this node moves?" Naming a fallacy when
> spotting a flaw ("that assumes the network is reliable — I'd add retries + idempotency") is a senior
> reflex.

---

## 3. The fundamental ambiguity: you can't tell *slow* from *dead* ⭐⭐ (the deepest insight)

You send node B a request and hear nothing back. Is B **crashed**? Or just **slow** (overloaded, GC
pause)? Or **alive but the network is partitioned**? Or did B **process it but the reply was lost**? In
an **asynchronous network** (no bound on message delay), **you cannot reliably distinguish these.** Your
only tool is a **timeout** — and a timeout is a **guess**:
- **Too short** → you declare a healthy-but-slow node dead → unnecessary failovers, split-brain risk
  (m05), duplicate work.
- **Too long** → you keep waiting on a truly-dead node → slow failure detection, downtime.

> **Why this matters everywhere:** this single ambiguity is *why* failover is dangerous (m05 split-
> brain), *why* consensus is hard (§5), *why* you need idempotency (a "failed" request may have actually
> succeeded — m08), and *why* exactly-once delivery is impossible. **There is no perfect failure
> detector in an async system.** Every "is it alive?" decision is a probabilistic bet via heartbeats +
> timeouts. Say this and you've shown the deepest understanding in the module.

**Failure models** (what kinds of failure you design against):
- **Crash-stop:** a node halts and stays halted. (Simplest; what most systems assume.)
- **Crash-recovery:** a node crashes and later restarts (may have lost in-memory state). (Common — needs
  durable logs.)
- **Omission:** messages are lost (send/receive omissions).
- **Byzantine:** a node behaves **arbitrarily/maliciously** (lies, sends conflicting info). Hardest;
  needs **Byzantine fault tolerance (BFT)** — relevant to blockchains and adversarial settings, rarely
  to internal systems (you usually trust your own nodes → crash-fault tolerance is enough).

---

## 4. Time & clocks — why you can't trust timestamps ⭐⭐

Intuition says "order events by their timestamps." **In a distributed system this is a bug.**

**Physical clocks lie.** Each machine has its own clock; they drift and are corrected (NTP) — so two
machines' wall-clocks differ by **milliseconds or more (clock skew)**, and a clock can even **jump
backwards** when corrected. Consequences:
- Ordering two events on different machines by wall-clock timestamp can be **wrong** (event that
  happened *later* may have an *earlier* timestamp).
- **Last-Write-Wins** (m05) based on wall-clocks can **silently drop the actually-newer write.**
- *Use a **monotonic clock** for measuring durations (it never goes backwards); use the wall-clock only
  for "what time is it for humans," never for ordering across machines.*

**Logical clocks** capture **causality** without trusting physical time, via the **happens-before**
relation (→): if event A could have caused B (same process in order, or A is a message-send and B its
receive), then A → B. Events with no happens-before path are **concurrent**.
- **Lamport timestamps:** each process keeps a counter, increments on every event, and on receiving a
  message sets `counter = max(local, received) + 1`. Gives a **total order consistent with causality**
  (if A → B then L(A) < L(B)) — but **L(A) < L(B) doesn't prove A → B** (they might be concurrent). Good
  for a consistent tie-broken ordering.
- **Vector clocks:** each process tracks a **vector** of all processes' counters → can **detect
  concurrency** (tell whether A → B, B → A, or they're concurrent). Used to detect **conflicting
  concurrent writes** in Dynamo-style stores (m05) so you can surface/merge them instead of silently
  losing one.

> **Think differently — buying your way out: Google TrueTime (Spanner).** Spanner gives strict
> serializability across the globe partly by making **time trustworthy**: special **atomic clocks + GPS**
> in every datacenter bound clock uncertainty to a small interval `[earliest, latest]`, and Spanner
> **waits out the uncertainty** (a few ms) before committing so timestamp order is guaranteed correct.
> It literally **spends money on hardware to make physical time reliable** — the exception that proves
> the rule that, normally, you must not trust clocks. (This is the m02 "PC/EC" corner made real.)

---

## 5. Consensus — how nodes agree ⭐⭐

**Consensus** = getting a group of nodes to **agree on a single value** (e.g. "who is the leader?",
"what's the next entry in the log?", "is this committed?") **despite failures**. It's the heart of
fault-tolerant coordination. A correct consensus protocol guarantees **agreement** (all decide the same
value), **validity** (the value was proposed), and **termination** (it eventually decides).

**FLP impossibility (the famous limit):** in a fully **asynchronous** network, **no deterministic
consensus protocol can guarantee termination if even one node may fail** (because of §3 — you can't tell
a slow node from a dead one, so you can't safely decide). Practical systems sidestep FLP by using
**timeouts** (a *partial* synchrony assumption) and/or **randomness** — they give up the theoretical
guarantee of always terminating in exchange for terminating *in practice* almost always.

**Raft (the one to know — designed for understandability):** elects a **leader** that handles all writes,
and replicates a **log** to followers via **majority quorum**.
- **Terms:** logical time divided into numbered terms; each term has at most one leader.
- **Leader election:** if a follower hears no heartbeat within an election timeout, it becomes a
  **candidate**, increments the term, and requests votes; a node wins with a **majority** of votes →
  becomes leader. **Randomized election timeouts** prevent endless split votes.
- **Log replication:** the leader appends a command, replicates it to followers; once a **majority** have
  it, it's **committed** and applied. A new leader must have all committed entries (it can only win with
  an up-to-date log).
- **Why majority (quorum):** any two majorities **overlap** in ≥1 node, so a new leader always sees the
  latest committed data, and a **minority partition can't elect its own leader** → **no split-brain**
  (the formal fix for m05's failover danger). With 2f+1 nodes you tolerate **f** failures.

```
  Raft: [leader] ──append──▶ [follower]  ──┐
                 ──append──▶ [follower]  ──┤ majority ack → COMMIT (tolerate f of 2f+1 down)
                 ──append──▶ [follower]  ──┘   minority partition can't elect → no split-brain
```

**Paxos** is the original (Lamport) consensus algorithm — correct and influential but notoriously hard
to understand and implement; **Raft** was created as an understandable equivalent and now dominates
(etcd, Consul, CockroachDB, TiKV). **Multi-Paxos/ZAB** (ZooKeeper) are in the same family.

> **The cost framing (ties to m02 "coordination is the tax on scale"):** consensus needs **multiple
> round trips + a majority** for every decision → it's **expensive and limits throughput/latency**. So
> you use it **sparingly** — for **metadata, leader election, configuration, distributed locks** — and
> **not** on the high-volume data hot path. "Run consensus on the control plane, keep the data plane
> coordination-free" is the senior instinct.

---

## 6. Coordination services — ZooKeeper, etcd, Consul

Rather than every system implementing consensus, you delegate to a **coordination service**: a small,
strongly-consistent (consensus-backed) key-value store that provides the primitives distributed systems
need:
- **Leader election**, **service discovery / membership** (who's alive), **configuration** (shared, with
  watch/notify), **distributed locks**, **metadata** (the partition map / shard assignments from m05).
- **ZooKeeper** (ZAB consensus) — the classic; used by older Kafka, HBase, etc. **etcd** (Raft) — used by
  **Kubernetes** for all cluster state. **Consul** — service discovery + config + health.

> **The pattern:** these are **CP** systems (m02) — they choose consistency over availability (a
> minority partition stops serving rather than risk disagreement), because the *whole point* is to be
> the **single source of truth** for coordination. They're deliberately **small and not on the data hot
> path** (you don't store app data in etcd) — exactly the §5 "consensus is expensive, use sparingly"
> rule. *You've run a system on top of one: **Kubernetes stores everything in etcd** (Raft), and Kafka
> historically used ZooKeeper.*

---

## 7. Distributed transactions: 2PC vs Saga ⭐⭐

You need a transaction **spanning multiple services/databases** (debit account-service, credit
ledger-service; or write Mongo + update a downstream system). A local ACID transaction (m04) can't span
systems. Two approaches:

### Two-Phase Commit (2PC) — strong, but blocking
A **coordinator** drives an atomic commit across participants:
1. **Prepare phase:** coordinator asks all participants "can you commit?" Each does the work, locks
   resources, and votes **yes/no** (a promise it *can* commit).
2. **Commit phase:** if **all** voted yes, coordinator says "commit"; if any said no, "abort." Everyone
   obeys.
It gives **atomicity across systems** (all commit or all abort) — but the fatal flaw: **it's blocking.**
If the **coordinator crashes** after participants voted yes but before sending the decision, participants
are stuck holding **locks** indefinitely, unable to safely commit or abort (they don't know the
decision). This kills availability and throughput → **2PC is avoided in high-scale systems.** (3PC adds a
phase to reduce blocking but adds latency and still fails under partitions.)

### Saga — availability over atomicity (the modern default)
A **Saga** breaks the distributed transaction into a **sequence of local transactions**, each in one
service, with a **compensating transaction** to undo each step if a later one fails. No global locks; no
2PC. You trade **atomicity** for **availability + eventual consistency**.
- **Example:** book trip = reserve flight → reserve hotel → charge card. If "charge card" fails, run
  **compensations**: cancel hotel, cancel flight. The system is briefly inconsistent but **converges**.
- **Two coordination styles:**
  - **Choreography:** each service emits **events**; others react (no central brain). Decoupled, but the
    flow is **implicit/hard to follow** and can tangle as steps grow.
  - **Orchestration:** a central **orchestrator** explicitly drives the steps and compensations.
    Clearer, easier to monitor/change, but the orchestrator is a component to build/run.
- **Requirements:** each step must be **idempotent** (retries — m08) and have a **compensating action**
  (and compensations must also be idempotent + ideally commutative). Some effects can't be perfectly
  undone (an email sent) → you compensate semantically (send a correction).

```
  2PC:  prepare(all) → vote → commit(all)/abort   atomic but BLOCKS if coordinator dies (locks held)
  SAGA: T1 → T2 → T3 ... on failure → C2 → C1 (compensate)   no locks; eventually consistent
```

> **The senior call:** *"For cross-service consistency I avoid 2PC — its coordinator is a blocking SPOF
> that holds locks. I use a **Saga** (usually orchestration for clarity), with **idempotent steps** and
> **compensating transactions**, accepting eventual consistency. If I truly need cross-shard atomicity I
> reach for a database that provides it (Spanner/CockroachDB) rather than rolling 2PC myself."* This
> connects the outbox/CDC (m08) — sagas are typically implemented over reliable messaging.

---

## 8. Distributed locks — harder than they look ⭐

Sometimes you need mutual exclusion across nodes ("only one worker processes this job"). A **distributed
lock** (via Redis, ZooKeeper, etcd) grants it — but it's subtle:
- **Locks need a TTL** (or a crashed lock-holder holds it forever) → but then a **slow** holder's lock
  can **expire while it still thinks it holds it** (the §3 slow-vs-dead problem again!), and a second
  holder acquires it → **two holders** → corruption.
- **The fix: fencing tokens.** The lock service hands out a **monotonically increasing token** with each
  grant; the protected **resource rejects any write with a stale (lower) token**. So even if an old
  holder wakes up and tries to act, its token is outdated and the write is refused. (Same fencing idea as
  m05 split-brain.)
- **The Redlock debate:** Redis's "Redlock" multi-node locking algorithm is **controversial** — Martin
  Kleppmann argued it's unsafe under clock skew/GC pauses without fencing tokens; the lesson is **a lock
  alone isn't enough for correctness; you need fencing on the resource.** A great nuance to cite.

> **The honest framing:** *"Distributed locks are a correctness hazard because of the slow-vs-dead
> problem — a lease can expire while the holder thinks it's valid. I'd use a proper lock service
> (etcd/ZooKeeper) **and** fencing tokens checked by the resource, or, better, design to **avoid** the
> lock entirely (idempotency, single-writer-per-partition, optimistic concurrency)."*

---

## 9. CRDTs (recap, in theory terms)
**Conflict-free Replicated Data Types** (introduced in m05) are the **coordination-free** answer for
multi-writer/offline scenarios: data types (G-counters, OR-sets, sequences) whose merge is
**commutative, associative, idempotent**, so replicas applying updates in any order **converge without
consensus or locks.** They're the practical face of m02's **CALM theorem** (monotonic computations need
no coordination). Use them where conflicts are otherwise painful (collaborative editing, carts,
counters) — trading data-model constraints for **zero coordination** (the opposite end of the spectrum
from consensus).

> The spectrum to carry: **strong agreement = consensus (expensive, coordinated)** ⟷ **no coordination =
> CRDTs/eventual (cheap, but only for monotonic/mergeable data).** You pick the *least* coordination that
> meets correctness — the recurring "coordination is the tax on scale" theme (m02).

---

## 10. From your systems 🏭
- **Slow-vs-dead + leader election, lived:** every primary failover you've run (MongoDB **replica-set
  elections**, Redis Sentinel, Postgres failover) is §3 + §5 in action — a heartbeat times out, a
  **majority** elects a new primary (Raft-like), and a minority partition can't elect (no split-brain).
  You can now explain *why* it needs a majority.
- **etcd/ZooKeeper under your platforms:** **Kubernetes** (DCE on k8s/OpenShift→EKS) stores all cluster
  state in **etcd (Raft consensus)**; **Kafka** historically used **ZooKeeper** for coordination/leader
  election. You operated systems whose control planes *are* consensus services (§6).
- **Saga over your pipeline:** DCE's multi-stage claim flow (exclusions → models/rules → suppressions →
  emit decision) is saga-shaped — each stage a local step, with **suppressions acting like compensating/
  guard logic** that prevents incorrect downstream effects. Emitting the decision reliably is the
  outbox/saga-over-messaging story (m08).
- **Idempotency = living with §3:** because a "failed" claim send may have actually succeeded, and
  reprocessing replays events, your **idempotent processing** is the practical answer to "you can't tell
  what happened" (§3) — exactly-once *effect* despite at-least-once delivery.
- **No 2PC across stores:** you don't run 2PC between Mongo and downstream systems — you use messaging +
  idempotency (saga-style), which is the correct §7 choice.

---

## 11. Key concepts (interview-ready)
- **Partial failure** is the defining property: some parts fail while others run, with no node having a
  complete/current view → the root of every hard problem here.
- **The 8 fallacies** (network reliable, latency zero, bandwidth infinite, network secure, topology
  static, one admin, transport free, homogeneous) — use as a design checklist.
- **You can't distinguish slow from dead** in an async network → failure detection is a **timeout guess**
  (too short = false failover/split-brain; too long = slow detection). The deepest insight; underlies
  failover, consensus, idempotency, exactly-once.
- **Failure models:** crash-stop / crash-recovery / omission / **Byzantine** (malicious — needs BFT,
  rarely needed internally).
- **Time lies:** physical clocks skew/jump → don't order cross-machine events by wall-clock or do
  clock-based LWW. Use **monotonic** clocks for durations; **logical clocks** (Lamport = consistent
  total order; **vector clocks** = detect concurrency) for causality. **TrueTime** = buy reliable time
  with hardware (Spanner).
- **Consensus** = agree on one value despite failures (agreement/validity/termination). **FLP**: no
  guaranteed async consensus → use timeouts/randomness. **Raft** (terms, majority leader election +
  log replication; majority quorums overlap → up-to-date leader, no split-brain; 2f+1 tolerates f).
  **Consensus is expensive → use sparingly (control plane), not the data hot path.**
- **Coordination services** (etcd/ZooKeeper/Consul, **CP**) provide leader election, discovery, config,
  locks, metadata — the consensus you rent (k8s→etcd, Kafka→ZooKeeper).
- **Distributed transactions:** **2PC** = atomic but **blocking** (coordinator crash → held locks) →
  avoid at scale; **Saga** = local steps + **compensating transactions** (choreography vs orchestration),
  trading atomicity for availability + eventual consistency; needs **idempotent** steps.
- **Distributed locks** are hazardous (lease expiry vs slow holder) → need **fencing tokens** on the
  resource (the Redlock debate); better to design the lock away.
- **CRDTs** = coordination-free convergence for monotonic/mergeable data (CALM). Spectrum: **consensus
  (max coordination)** ⟷ **CRDTs/eventual (none)**; pick the least that's correct.

---

## 12. Go deeper (the well-researched reading list)
- **Kleppmann, DDIA** — Ch. 8 (The Trouble with Distributed Systems: faults, unreliable clocks, slow-vs-
  dead) and Ch. 9 (Consistency & Consensus: linearizability, ordering, 2PC, consensus). The definitive
  source for this whole module.
- **"In Search of an Understandable Consensus Algorithm (Raft)" — Ongaro & Ousterhout**, and the
  interactive **thesecretlivesofdata.com/raft** visualization. The best way to *get* §5.
- **Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System"** — the origin of
  happens-before / logical clocks (§4).
- **Kleppmann, "How to do distributed locking"** — the Redlock critique + fencing tokens (§8).
- **The "Fallacies of Distributed Computing" (Deutsch/Gosling)** and **Spanner/TrueTime paper** — §2/§4.
- **microservices.io: Saga pattern** (Chris Richardson) — choreography vs orchestration + compensations
  (§7).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m09_distributed_systems_theory/`](../../solutions/m09_distributed_systems_theory/README.md).

When you've done the exercises, say **"Module 10"** to build *Reliability, Observability & Operations*
(SLA/SLO/SLI & error budgets, retries/backoff/jitter, circuit breakers, bulkheads, graceful degradation,
metrics/logs/traces) — how you **run** all this in production and keep it up.
