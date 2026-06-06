# m09 — Cross-Questions ("if-and-buts") on Distributed Systems Theory

> Answer out loud in 2–3 sentences before reading the model answer. This module is theory, so the
> follow-ups always push you to the **practical implication** — that's what interviewers grade.

---

### Q1. What makes distributed systems fundamentally harder than a single program?
**A.** **Partial failure**: instead of all-or-nothing, *some* nodes/links fail while others keep running,
and **no node has a complete, current view** of the whole system. Failures are non-deterministic and
often **indistinguishable** (did the request fail, or succeed with a lost response?). So every node must
decide with **incomplete, possibly-stale, possibly-wrong** information while parts may be silently
broken — that's the root of consensus, ordering, exactly-once, and failover difficulty.

---

### Q2. Name a few "fallacies of distributed computing" and why they matter.
**A.** "The network is reliable" (it drops/reorders → need retries + idempotency), "latency is zero"
(remote calls are orders of magnitude slower → batch/cache/colocate), "the network is secure" (→ TLS/
mTLS), and "topology doesn't change" (→ service discovery, no hard-coded hosts). They matter because each
false assumption is a **class of real outage** — I use them as a design checklist: "what if the network
drops this? what's this hop's latency? what if this node moves?"

---

### Q3. Why can't you tell whether a node is dead or just slow?
**A.** In an **asynchronous network** there's no upper bound on message delay, so silence from a node is
ambiguous: it could be crashed, slow (GC pause, overload), partitioned, or it processed the request but
the reply was lost. Your only tool is a **timeout, which is a guess** — too short declares healthy-but-
slow nodes dead (false failovers, split-brain), too long delays detecting real death. **There's no
perfect failure detector in an async system** — this single ambiguity underlies failover danger,
consensus difficulty, and the need for idempotency.

---

### Q4. How does the slow-vs-dead problem connect to idempotency?
**A.** Because a request that **appears** to fail (timeout, lost response) may have **actually
succeeded**, you must **retry** — and retrying a non-idempotent operation can double-apply it (double
charge, double create). So the practical answer to "I can't tell what happened" is **make operations
idempotent** (idempotency keys, dedup) so that retrying a maybe-completed request is **harmless**. The
ambiguity (§3) forces idempotency (m03/m08) — they're two sides of one coin.

---

### Q5. What are the failure models, and which do you usually design for?
**A.** **Crash-stop** (node halts permanently), **crash-recovery** (crashes then restarts, maybe losing
in-memory state → needs durable logs), **omission** (messages lost), and **Byzantine** (node behaves
arbitrarily/maliciously — lies, sends conflicting data). Internally you usually trust your own nodes, so
you design for **crash/omission faults (crash-fault tolerance)**; **Byzantine fault tolerance** is for
adversarial settings (blockchains, untrusted participants) and is much more expensive.

---

### Q6. Why shouldn't you order events across machines by their timestamps?
**A.** Because **physical clocks drift and are corrected**, so two machines' wall-clocks differ by ms or
more (**clock skew**) and can even **jump backwards**. An event that truly happened *later* may carry an
*earlier* timestamp → ordering by wall-clock is wrong, and **timestamp-based Last-Write-Wins can silently
drop the actually-newer write**. For ordering you use **logical clocks** (causality); wall-clocks are
only for "human time," and **monotonic** clocks for measuring durations.

---

### Q7. Lamport clocks vs vector clocks?
**A.** **Lamport timestamps** (a per-process counter, `max(local,received)+1` on receive) give a **total
order consistent with causality**: if A → B then L(A) < L(B). But the converse fails — `L(A) < L(B)`
doesn't prove A caused B (they might be concurrent). **Vector clocks** track a **vector of all
processes' counters**, so they can **detect concurrency** — tell whether A → B, B → A, or they're
concurrent. Use Lamport for a cheap consistent ordering; use vector clocks when you must **detect
conflicting concurrent writes** (Dynamo-style stores, m05).

---

### Q8. How does Google Spanner get away with using physical time?
**A.** With **TrueTime**: every datacenter has **atomic clocks + GPS** that bound clock uncertainty to a
small interval `[earliest, latest]`. Spanner **waits out that uncertainty** (a few ms) before committing
a transaction, guaranteeing that timestamp order matches real commit order — so it can offer **strict
serializability** globally. It's the exception that proves the rule: normally you can't trust clocks, so
Spanner **spends money on special hardware** to make time reliable (the m02 "PC/EC" corner in practice).

---

### Q9. What is consensus, and what's it used for?
**A.** **Consensus** is getting a group of nodes to **agree on a single value despite failures**, with
agreement (all decide the same), validity (the value was proposed), and termination (it eventually
decides). It's used for the **control plane**: **leader election**, agreeing the **next log entry**
(replication), **configuration**, **distributed locks**, and **membership/metadata** (the shard map,
m05). It's the foundation of fault-tolerant coordination — but it's expensive, so you use it for
metadata, not the high-volume data path.

---

### Q10. What is FLP impossibility and how do real systems get around it?
**A.** **FLP** proves that in a **fully asynchronous** network, **no deterministic consensus protocol can
guarantee it always terminates if even one node can fail** — precisely because you can't distinguish a
slow node from a dead one (§3), so you can't safely decide. Real systems **sidestep** it by assuming
**partial synchrony** (using **timeouts**) and/or **randomness** (e.g. Raft's randomized election
timeouts): they give up the theoretical guarantee of always terminating in exchange for terminating **in
practice almost always**.

---

### Q11. Explain Raft at a high level.
**A.** Raft elects a single **leader** per **term** that handles all writes and replicates a **log** to
followers. **Election:** a follower that hears no heartbeat becomes a **candidate**, bumps the term, and
requests votes; whoever gets a **majority** becomes leader (randomized timeouts avoid split votes).
**Replication:** the leader appends a command and replicates it; once a **majority** have it, it's
**committed** and applied. A candidate can only win with an **up-to-date log**, so committed entries are
never lost. It's Paxos's guarantees in an understandable form (used by etcd, Consul, CockroachDB).

---

### Q12. Why does Raft (and quorum systems) require a majority?
**A.** Because **any two majorities overlap in at least one node.** That overlap guarantees a newly
elected leader has seen the latest committed entry (the data isn't lost), and — crucially — a
**minority partition can't assemble a majority, so it can't elect its own leader or commit** → **no
split-brain** (the formal fix for m05's failover danger). With **2f+1** nodes you tolerate **f**
failures and still have a majority. It's why consensus clusters are odd-sized (3, 5).

---

### Q13. Why is consensus "expensive," and what's the implication?
**A.** Every decision needs **multiple network round trips and a majority of nodes to agree**, which adds
latency and caps throughput (and availability drops if you can't reach a majority). The implication
(the m02 "coordination is the tax on scale" theme): **use consensus sparingly** — on the **control
plane** (leader election, config, metadata, locks) — and keep the **data plane coordination-free**
(partitioning, single-writer-per-partition, eventual consistency, CRDTs). Don't run consensus on the
high-volume request path.

---

### Q14. What is a coordination service and when do you need one?
**A.** A small, strongly-consistent (consensus-backed) store — **ZooKeeper, etcd, Consul** — that
provides distributed-systems primitives: **leader election, service discovery/membership, shared
configuration with watches, distributed locks, and metadata** (e.g. the shard map). You need one when
multiple nodes must agree on shared state and you don't want to implement consensus yourself. They're
**CP** (choose consistency over availability) and deliberately **off the data hot path** — e.g.
Kubernetes keeps all cluster state in **etcd**, Kafka historically used **ZooKeeper**.

---

### Q15. Walk through Two-Phase Commit and its fatal flaw.
**A.** A **coordinator** runs two phases: **prepare** (ask all participants "can you commit?"; each does
the work, locks resources, votes yes/no) and **commit** (if all voted yes → commit; else abort). It gives
**atomicity across systems**. The **fatal flaw is blocking**: if the **coordinator crashes** after
participants voted yes but before broadcasting the decision, participants are **stuck holding locks**,
unable to safely commit or abort (they don't know the outcome). That stalls throughput and availability,
so **2PC is avoided in high-scale systems.**

---

### Q16. What is a Saga and how is it different from 2PC?
**A.** A **Saga** implements a cross-service transaction as a **sequence of local transactions**, each in
one service, with a **compensating transaction** to undo each step if a later one fails — **no global
locks, no coordinator holding everyone hostage.** It trades **atomicity** for **availability + eventual
consistency** (the system is briefly inconsistent, then converges via compensations). Unlike 2PC's
synchronous all-or-nothing lock-and-vote, a Saga is asynchronous and non-blocking — which is why it's the
modern default for microservice transactions.

---

### Q17. Saga choreography vs orchestration?
**A.** **Choreography:** each service emits **events** and others react — no central coordinator. It's
decoupled but the overall flow is **implicit and hard to follow/debug**, and it tangles as steps grow.
**Orchestration:** a central **orchestrator** explicitly invokes each step and triggers compensations on
failure — clearer, easier to monitor and modify, at the cost of building/running the orchestrator. I
default to **orchestration** for non-trivial sagas because the explicit flow is far easier to reason
about and operate.

---

### Q18. What do Saga steps require to work correctly?
**A.** Each step must be **idempotent** (it may be retried after a failure/timeout — §3/m08), and each
must have a **compensating action** to semantically undo it (and compensations should be idempotent and
ideally order-independent). Some effects can't be perfectly reversed (an email already sent) → you
**compensate semantically** (send a correction/cancellation). You also need reliable messaging
(outbox/CDC, m08) to drive the steps. Without idempotency + compensations, a Saga can't safely recover
from partial failure.

---

### Q19. Why are distributed locks dangerous, and what makes them safe?
**A.** They hit the slow-vs-dead problem: a lock needs a **TTL/lease** (so a crashed holder doesn't hold
forever), but a **slow** holder's lease can **expire while it still thinks it holds the lock** → a second
holder acquires it → **two holders** → corruption. The fix is **fencing tokens**: the lock service issues
a **monotonically increasing token** with each grant, and the **protected resource rejects writes with a
stale (lower) token** — so an old holder's late write is refused. The lesson (the Redlock debate): **a
lock alone isn't enough; you need fencing on the resource** — or design the lock away (idempotency,
single-writer-per-partition).

---

### Q20. The "Redlock" controversy — what's the takeaway?
**A.** Redlock is Redis's multi-node distributed-lock algorithm; Martin Kleppmann argued it's **unsafe
for correctness** under clock skew, GC pauses, and network delays (the slow-vs-dead problem) **without
fencing tokens**. The takeaway isn't "Redis bad" — it's that **no timeout-based lock can guarantee mutual
exclusion by itself**; you must **fence at the resource** (reject stale token holders), or use a lock
only as a performance optimization (avoid duplicate work) rather than for correctness. When correctness
matters, fence — or eliminate the lock.

---

### Q21. When would you use a CRDT instead of consensus?
**A.** When I need **coordination-free** convergence for **multi-writer/offline** data and the data is
**mergeable/monotonic** — collaborative editing, shopping carts, counters, presence. CRDTs (G-counters,
OR-sets, sequences) merge **commutatively/associatively/idempotently**, so replicas converge **without
consensus or locks** (the CALM theorem: monotonic ⇒ no coordination, m02). I'd use consensus instead when
I need a **single agreed decision** (leader, committed log entry) or strong invariants that aren't
mergeable. The spectrum: consensus (max coordination) ⟷ CRDTs/eventual (none) — pick the least
coordination that's still correct.

---

### Q22. You need a cross-service "transfer money" operation. 2PC, Saga, or something else?
**A.** Not raw 2PC (blocking coordinator + held locks). I'd use a **Saga** — debit in the account service
(local txn), credit in the ledger service (local txn), and on failure run **compensations** (reverse the
debit), with **idempotent** steps driven over reliable messaging (outbox/CDC). Money needs care, so I'd
use **orchestration** for a clear, auditable flow and ensure compensations + idempotency are airtight. If
the two datasets could live in **one database that offers cross-shard transactions** (Spanner/
CockroachDB), I'd prefer that over rolling my own distributed transaction.

---

### Q23. How does all this connect to your CAP/PACELC knowledge from m02?
**A.** Directly: **consensus systems are CP** (a minority partition stops serving to avoid disagreement —
etcd/ZooKeeper), which is *why* they guarantee correctness for coordination. **Quorums (m05) and the
need for a majority** come from the same overlap argument as Raft. **Spanner's TrueTime** is the **PC/EC**
corner (consistency in both partition and normal operation, paying latency). And the whole "use the least
coordination that's correct" instinct is PACELC's latency-vs-consistency trade applied as a design
principle. Theory (this module) is the *why* behind the m02 trade-offs.

---

### Q24. How do you actually run consensus-backed leader election in practice without building Raft yourself?
**A.** You **rent it** from a coordination service: register the candidates in **etcd/ZooKeeper/Consul**,
which uses consensus internally to elect/track the leader and notify others on change (watches/leases).
For example, Kubernetes components use **etcd-based leader election**, and MongoDB replica sets / Kafka
controllers run their own Raft-like elections internally. The senior move is to **not hand-roll
consensus** (it's famously easy to get subtly wrong) — use a battle-tested service and focus on **fencing
+ idempotency** so a brief dual-leader window during election can't corrupt data.
