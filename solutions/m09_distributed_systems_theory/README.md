# m09 — Solutions (Distributed Systems Theory)

> Check the *reasoning* and that you landed each on a practical implication. Recurring lessons: you can't
> tell slow from dead; don't trust clocks; consensus is expensive (control plane only); Saga over 2PC;
> fence your locks.

---

## Exercise 1 — Spot the fallacy
1. **"Latency is zero"** (and "network is reliable") — remote calls are slow and can fail; a sync
   dependency couples you to its latency/availability. Fix: timeouts, retries, consider async (m08).
2. **"The network is reliable"** — messages drop. Fix: retries with backoff + **idempotent** consumer.
3. **"Topology doesn't change"** — nodes/IPs move. Fix: **service discovery**, no hard-coded hosts.
4. **"The network is secure"** — internal traffic can be sniffed/spoofed. Fix: **mTLS**, zero-trust (m03).
5. **"Bandwidth is infinite"** (and "transport cost is zero") — large payloads cost bandwidth/CPU/$.
   Fix: paginate, compress, send deltas/ids not whole docs.

---

## Exercise 2 — Slow vs dead
1. B could be: **crashed**, **slow** (GC pause/overload), **network-partitioned** from A, or **alive and
   processed it but the heartbeat/reply was lost**.
2. In an **asynchronous network** there's no upper bound on delay, so silence is indistinguishable across
   those cases — A has no way to tell "no response yet" from "never coming."
3. **Too short:** A declares a healthy-but-slow B dead → unnecessary failover, possible **split-brain**,
   duplicate work. **Too long:** A keeps waiting on a truly-dead B → slow detection, prolonged downtime.
4. Because a request that *looks* failed (timeout) may have **actually succeeded**, A must retry — and
   safe retry requires **idempotency** so a maybe-completed operation isn't double-applied. The ambiguity
   forces idempotency.

---

## Exercise 3 — Clocks
1. **Clock skew**: the two servers' wall clocks differ (drift + NTP corrections), so an event that
   happened later can carry an earlier timestamp → wrong order.
2. Wall-clock **LWW** can pick the write with the larger timestamp even though it's actually the **older**
   one (due to skew) → it **silently drops the newer write** / loses data.
3. A **monotonic clock** — it never goes backwards and isn't affected by NTP jumps, so duration =
   `mono_end - mono_start` is correct (wall clocks can jump and give negative/garbage durations).
4. A **vector clock** can detect **concurrency** — distinguish A→B, B→A, or "concurrent" — whereas a
   Lamport clock only gives a total order consistent with causality but can't tell concurrent events
   apart.

---

## Exercise 4 — Consensus basics
1. **Agreement** (all nodes decide the same value), **validity** (the decided value was proposed by
   someone), **termination** (it eventually decides).
2. **FLP:** in a fully asynchronous network, no deterministic consensus protocol can guarantee
   termination if even one node may fail (you can't tell slow from dead). Practical systems assume
   **partial synchrony** (use **timeouts**) and/or **randomness** (Raft's randomized election timeouts) —
   terminating in practice almost always, giving up the absolute guarantee.
3. A leader needs a **majority** because **any two majorities overlap** → the new leader has the latest
   committed data, and a **minority can't form a majority**, so it can't elect a competing leader →
   **no split-brain**.
4. **5 nodes tolerate 2 failures** (majority = 3). **4 nodes** also need a majority of 3, so they tolerate
   only **1** failure — same as 3 nodes but with more cost/latency — which is why **odd sizes** (3,5,7)
   are preferred (you get the most fault tolerance per node).

---

## Exercise 5 — Where does consensus belong?
1. **(a) Elect a primary → consensus** (a single agreed decision, must avoid split-brain). **(b) Shard
   map → consensus/coordination service** (shared metadata everyone must agree on; low write rate).
   **(c) 500k writes/sec of user events → absolutely NOT consensus** — that's the high-volume **data
   plane**; route it through partitioned, coordination-free writes (sharding + replication, m05/m08).
2. **Principle:** use **consensus only on the control plane** (leader election, config, metadata) where
   agreement is essential and volume is low; keep the **data plane coordination-free** for scale. This is
   "**coordination is the tax on scale**" (m02) — every coordinated decision costs round trips + a
   majority, so you minimize them.

---

## Exercise 6 — Coordination services
1. **Leader election**, **service discovery/membership**, **shared configuration (with watches)**,
   **distributed locks**, **metadata** (shard map) — any three.
2. **CP** — they choose **consistency over availability** (a minority partition stops serving rather than
   risk disagreement). That's correct because their *job* is to be the **single source of truth** for
   coordination; serving stale/conflicting coordination data would be worse than being briefly
   unavailable.
3. Because they're **consensus-backed (every write needs a majority → expensive/low-throughput)** and
   meant to be small/critical — app data volume would overwhelm them; keep them off the data hot path.
4. **Kubernetes uses etcd** (Raft) to store **all cluster state** (the desired/actual state of every
   object).

---

## Exercise 7 — 2PC vs Saga
1. **2PC:** a coordinator asks A, B, C "can you commit?" (each reserves/locks, votes yes); if all yes →
   "commit." **Danger:** if the **coordinator crashes** after votes but before the decision, A/B/C are
   **stuck holding locks** indefinitely (don't know whether to commit/abort) → blocked resources, lost
   availability.
2. **Saga:** reserve flight (A) → reserve hotel (B) → charge card (C), each a local transaction. If
   **charge card fails**, run **compensations**: cancel hotel (B), cancel flight (A). Briefly
   inconsistent, then converges. No global locks.
3. **Orchestration** — a "trip orchestrator" explicitly drives reserve→reserve→charge and triggers
   compensations on failure. For a money-touching, multi-step flow, the explicit, auditable, easy-to-
   monitor flow beats choreography's implicit event web.
4. Each step must be **idempotent** (safe to retry) and have a **compensating transaction** (to undo it).

---

## Exercise 8 — Distributed lock gone wrong
1. **Danger:** W1's lock expired during the GC pause, W2 acquired it and started J, **but W1 resumes
   thinking it still holds the lock** → **two workers run J concurrently** → double processing/corruption.
2. The **slow-vs-dead** problem (a slow holder is indistinguishable from a dead one; its lease expired
   while it was just paused).
3. **Fencing tokens:** the lock service issues a **monotonically increasing token** with each grant (W1
   got token 33, W2 gets 34). The **resource** (DB/storage) records the highest token it's seen and
   **rejects any write with a lower token** — so W1's post-pause write with token 33 is **refused**.
4. **Avoid relying on the lock for correctness entirely** — make the operation **idempotent**, or design
   **single-writer-per-partition** (e.g. key J to one partition/consumer so only one worker ever owns it,
   m05/m08), or use optimistic concurrency. The lock then becomes at most a performance optimization.

---

## Exercise 9 — Pick the coordination level
1. **Consensus** — electing the primary is a single agreed decision that must avoid split-brain.
2. **CRDT / eventual** — offline collaborative editing needs coordination-free, conflict-free merging
   (a sequence CRDT); consensus would block offline clients.
3. **CRDT / eventual** — an approximate likes counter is a perfect **G-counter** (grow-only, merges by
   max/sum); no need for expensive coordination.
4. **Consensus** — committing the next replicated-log entry is *the* canonical consensus use (Raft).

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **Mongo/Redis failover:** a secondary stops hearing the primary's heartbeat — but it **can't tell slow
   from dead** (§3), so it waits a timeout, then triggers an **election**; a new primary needs a
   **majority** of the replica set to win, so a **minority partition can't elect** one → **no
   split-brain**. (You now know *why* it needs a majority.)
2. **Kubernetes → etcd (Raft)** holds all cluster state; **Kafka → historically ZooKeeper** (ZAB) for
   controller election/metadata (newer Kafka uses KRaft, its own Raft). Both control planes *are*
   consensus services.
3. **DCE pipeline as a Saga:** steps = exclusions → ML/rule models → **suppressions** → emit decision;
   each is a local stage. **Suppressions act like guard/compensation logic** — they stop incorrect
   recommendations from producing downstream effects (a forward-guard rather than an after-the-fact
   undo). You **don't use 2PC** across Mongo + downstream because 2PC's coordinator would block holding
   locks; instead it's messaging + **idempotency** (saga-style), which stays available.
4. **Idempotent reprocessing** is the practical answer to §3: because a "failed" claim send may have
   actually succeeded, and reprocessing **replays** events (m08), processing must be idempotent so
   re-running a claim doesn't **double-adjudicate** — exactly-once *effect* despite the unavoidable
   ambiguity of distributed delivery.
