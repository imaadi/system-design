# m05 — Cross-Questions ("if-and-buts") on Replication, Partitioning & Sharding

> Answer out loud in 2–3 sentences before reading the model answer. This module's follow-ups are where
> distributed-systems depth shows.

---

### Q1. "How do you scale a database?" — what's the first thing you say?
**A.** I split the question: **reads & availability → replication** (add follower copies); **writes &
data size beyond one node → partitioning/sharding** (split the data). They solve different problems and
are usually **combined** — shard the data, then replicate each shard. So my answer depends on whether
the bottleneck is read throughput, write throughput, or storage size — which I'd get from the
read/write ratio and the estimate.

---

### Q2. Replication vs sharding — what does each NOT solve?
**A.** **Replication doesn't solve write throughput or total size** — every replica holds the full
dataset and every write still goes through the leader (or to all replicas), so copies don't add write
capacity or shrink the dataset. **Sharding doesn't solve availability** — each shard is a single point
of failure until you *also* replicate it. That's why production systems do both: shard for write-scale/
size, replicate each shard for availability/read-scale.

---

### Q3. In leader-follower replication, does adding followers help write throughput?
**A.** No. All writes go to the **single leader**, so followers add **read** capacity and availability,
not write capacity. Leader-follower is ideal for **read-heavy** workloads (most web apps, ~100:1). If
the bottleneck is **writes**, you must **shard** (multiple leaders, each owning a partition) — adding
read replicas won't help. Knowing this distinction is the core of the module.

---

### Q4. Synchronous vs asynchronous replication — trade-offs?
**A.** **Sync:** the leader waits for follower acks before confirming → the write is safely on ≥2 nodes
(durable, no loss on leader death), but **higher latency** and writes **block** if a sync follower is
down (availability hit). **Async:** leader confirms immediately, streams in the background → fast, but
a leader crash **loses** un-propagated writes. **Semi-sync** (one sync follower, rest async) is the
usual compromise. It's PACELC: even with no partition you trade latency vs durability/consistency on
every write.

---

### Q5. What is replication lag and what bugs does it cause?
**A.** With async replication, followers trail the leader by some **lag** (ms–seconds). That window
breaks: **read-your-writes** (you write to the leader, read a stale follower → "where's my post?"),
**monotonic reads** (successive reads hit different-lag replicas → values appear to go backwards), and
**consistent prefix** (effects seen out of causal order). Fixes: route a user's reads to the leader (or
a caught-up replica) for a window after they write; **pin** a user to one replica for monotonic reads;
causal tracking / same-partition writes for prefix.

---

### Q6. You scale reads by adding replicas. A user complains their just-saved change "disappeared." What happened and how do you fix it?
**A.** Classic **read-your-writes** violation: the write went to the leader, but the user's next read hit
a **lagging follower** that hadn't received it. Fixes: for a short window after a user writes, **route
their reads to the leader** (or to a replica you've confirmed is caught up by tracking the write's log
position / LSN); alternatively pin that user's session to the leader. The general principle: replicas
are **eventually consistent**, so you must engineer read-your-writes where the UX needs it.

---

### Q7. What is split-brain, and how do you prevent it?
**A.** **Split-brain** happens during failover when the old leader isn't actually dead (just slow or
network-partitioned), so a follower gets promoted and now **two nodes accept writes** → divergent,
conflicting data. Prevention: **fencing** — make sure the old leader is truly stopped before promoting
(STONITH), or use **fencing tokens** (a monotonically increasing epoch number that the storage layer
checks, rejecting writes from a stale-epoch leader), and prefer **quorum/consensus-based** election so
only a majority-backed node can become leader (a partitioned minority can't elect itself).

---

### Q8. What can go wrong with automatic failover besides split-brain?
**A.** **Data loss:** with async replication, writes the old leader hadn't shipped yet are gone when a
less-current follower is promoted. **Flapping:** too-aggressive failure detection promotes on a
transient blip, causing unnecessary (and risky) failovers. **Cascading load:** the new leader suddenly
takes all traffic and may be cold/overwhelmed. So failover timeouts are a real tuning trade (too short →
false failovers; too long → more downtime), and you accept some possible loss with async or pay latency
with sync.

---

### Q9. When would you use multi-leader replication, and what's the catch?
**A.** When you need **writes in multiple regions** (low write latency per region, survive a DC outage)
or **offline-first** apps (each device is a leader that syncs later). The catch is **write conflicts**:
two leaders concurrently change the same record. You resolve with **Last-Write-Wins** (simple but
**loses data** and relies on clock sync — risky), **version vectors** (detect/merge concurrent
writes), or **CRDTs** (data types that merge automatically). Conflict handling is hard, so I avoid
multi-leader unless cross-region writes or offline are real requirements.

---

### Q10. Explain quorums: what does W + R > N give you?
**A.** With **N** replicas per key, requiring **W** nodes to ack a write and **R** to respond to a read,
if **W + R > N** the write set and read set **overlap by at least one node** → a read is guaranteed to
include a node with the latest write. It's a **tunable consistency dial**: W=2,R=2,N=3 is the balanced
default; W=3,R=1 gives fast reads but fragile writes; W=1,R=3 the reverse. **W+R ≤ N** is faster but may
return stale data (eventual). It's the leaderless (Dynamo/Cassandra) way to trade consistency vs
availability per operation.

---

### Q11. Does W+R>N guarantee strong (linearizable) consistency?
**A.** No — it's a strong *tendency*, not a guarantee. Edge cases still allow stale or non-linearizable
reads: **sloppy quorums** (writes accepted on non-home nodes during failures), **concurrent writes**
(which node "wins" is ambiguous without versioning), and **partially-failed writes** (W not reached but
some replicas applied it). Quorums give **tunable** consistency in an AP system, not guaranteed
linearizability — that needs consensus (Raft/Paxos, m09). I'd say "tunable/strong-ish," never
"linearizable," for a quorum store.

---

### Q12. What are read repair, anti-entropy, and hinted handoff?
**A.** They keep leaderless replicas converging. **Read repair:** during a read, if the coordinator
notices a replica returned a stale value, it writes the fresh value back to it (repairs on the read
path). **Anti-entropy:** a background process compares replicas (often via **Merkle trees** to find
differences cheaply) and syncs divergences. **Hinted handoff:** if a target replica is down at write
time, another node accepts the write and holds a "hint" to forward it when the replica returns — keeps
writes available during failures (a "sloppy quorum"). Together they make eventual consistency actually
converge.

---

### Q13. Range vs hash partitioning — trade-offs?
**A.** **Range** assigns contiguous key ranges to partitions → great for **range scans** (data stays
ordered) but prone to **hot spots** (uneven ranges, and especially **timestamp keys** where all new
writes hit the latest partition). **Hash** assigns by `hash(key)` → **even load distribution** (kills
sequential hot spots) but **destroys ordering**, so range queries become scatter-gather across all
partitions. Choose range when range queries dominate and you can avoid skew; hash when even write
distribution matters more than ordered scans.

---

### Q14. Even with hash partitioning, you can still get a hot partition. How?
**A.** A **single hot key** — a celebrity user, a viral product, a flash-sale item. Hashing distributes
*different* keys evenly, but **one key always maps to one partition**, so all of that key's traffic
concentrates there; you can't hash your way out of a single hot key. Mitigations: **append a random
suffix** to split the key into N sub-keys (and fan-in on read), **cache** the hot item at the app/CDN
tier (m06), or **special-case** it. This is the "celebrity problem" and it recurs in the news feed
(m13).

---

### Q15. What's wrong with `hash(key) % N` for partitioning, and what's the fix?
**A.** When you add/remove a node, **N changes**, so the modulus changes for **almost every key** →
nearly all data must move (a catastrophic reshuffle that overwhelms the cluster). The fix is
**consistent hashing**: place nodes and keys on a hash ring; a key belongs to the next node clockwise.
Now a membership change only moves the keys **between the changed node and its neighbor** — about
**K/N** keys, not all of them — enabling cheap, incremental rebalancing.

---

### Q16. What problem do virtual nodes solve in consistent hashing?
**A.** Two problems with one-point-per-node: **uneven distribution** (random placement gives some nodes
much bigger arcs) and **unbalanced failure** (when a node dies, *all* its keys dump onto a single
neighbor). **Virtual nodes** map each physical node to **many** points on the ring, so load is **even**
and a departed node's load **spreads across many** neighbors. They also let you weight by capacity (a
bigger machine gets more vnodes). It's what makes consistent hashing practical in Dynamo/Cassandra and
distributed caches.

---

### Q17. How do you choose a shard key, and why is it such a big deal?
**A.** Choose for: **even distribution** (high-cardinality, uniformly accessed → no hot spots), **query
locality** (things queried together share a shard → avoid scatter-gather), and **minimal cross-shard
transactions** (keep atomic operations on one shard). It's a big deal because the shard key is
**extremely hard to change later** — resharding means migrating huge volumes of live data — and a bad
key (low cardinality, hot-spotting, or mismatched to queries) can force a painful migration. So you
optimize the **dominant** access pattern up front and accept the rest are costlier.

---

### Q18. You shard chat messages by conversation_id. What's good and bad about that?
**A.** **Good:** "fetch a conversation's messages" is a **single-shard** read, and all messages in a
conversation are co-located and ordered — matches the dominant access pattern. **Bad:** "all messages
**this user** sent across conversations" becomes a **scatter-gather** across shards (the user's
messages are spread by conversation), and a very busy conversation could become a **hot shard**. You
accept the secondary pattern being expensive because the primary one (open a conversation) is what
users do constantly.

---

### Q19. How do secondary indexes work when data is partitioned?
**A.** Two approaches, a read-vs-write trade. **Local (document-partitioned) index:** each shard indexes
only its own rows → **writes are cheap** (update one shard) but a query on the secondary field must
**scatter-gather** across all shards. **Global (term-partitioned) index:** the index itself is
partitioned by the index term → **reads hit one shard** (fast) but a **write** must update the index on
a *different* shard (a cross-shard, usually eventually-consistent write). You pick based on whether the
secondary-field reads or the writes dominate.

---

### Q20. How does a client find which shard holds a given key (request routing)?
**A.** Three patterns: **(1) a routing tier/coordinator** that knows the partition map and forwards
(e.g. MongoDB's mongos, or a proxy); **(2) client-aware routing** where the client library knows the
map and contacts the right node directly (fewer hops, but the client must track topology changes);
**(3) any-node/gossip** where you hit any node and it forwards/redirects (Cassandra-style). The
partition map lives in a **coordination service** (ZooKeeper/etcd) or is gossiped between nodes — and
keeping it consistent during rebalancing is its own challenge (m09).

---

### Q21. How do replication and sharding combine in a real system?
**A.** You **shard** the dataset into partitions (for write-scale and size), and **replicate each
shard** (a leader + followers per shard, for availability and read-scale). So a key maps to a shard,
that shard has its own leader-follower group, writes for that key go to its leader, reads can use its
followers, and failover happens per-shard. You get both properties at the cost of both complexities —
this is how MongoDB (sharded replica sets), Cassandra, and most large datastores are structured.

---

### Q22. Why does a Kafka topic have partitions, and what does the partition key control?
**A.** Partitions are how Kafka **shards a topic for parallelism and scale** — each partition is an
independent ordered log on (potentially) different brokers, so more partitions = more throughput and
more parallel consumers. The **partition key** (e.g. user/claim id) decides **which partition** a
message lands in, and Kafka guarantees **ordering only within a partition**, not across them. So you
choose the key to (a) balance load across partitions and (b) keep messages that must stay ordered (same
entity) on the same partition — exactly the shard-key trade-offs of §9, which I dealt with on the DCE
claim stream.

---

### Q23. A node in your sharded cluster is overloaded (hot shard). How do you rebalance without downtime?
**A.** First diagnose: is it a **hot key** (one key) or a **hot range/shard** (too much data/traffic on
one partition)? For a hot **shard**, **rebalance** by moving some partitions to new/underutilized nodes
— done well by pre-creating **many fixed partitions** so you reassign whole partitions without
re-hashing keys, and by **moving as little data as possible** while serving reads from the old location
until the move completes. For a hot **key**, rebalancing won't help — you split the key (random suffix)
or cache it. Throughout, you throttle the migration so it doesn't starve live traffic.

---

### Q24. CRDTs — what are they and when would you reach for one?
**A.** **Conflict-free Replicated Data Types** are data structures (grow-only counters, sets, sequences,
maps) whose merge operation is **commutative, associative, and idempotent**, so replicas that received
updates in any order **converge to the same state without coordination or conflicts**. I reach for them
in **multi-leader/offline** scenarios where conflicts are otherwise painful: collaborative editors
(text sequences), shopping carts, presence, distributed counters. They're the practical embodiment of
m02's **CALM theorem** (monotonic ⇒ coordination-free), trading some data-model constraints for
automatic, conflict-free merging.
