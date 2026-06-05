# m05 — Solutions (Replication, Partitioning & Sharding)

> Check the *reasoning*. Recurring lessons: replication ≠ sharding; replicas are eventually consistent;
> the shard key is destiny; consistent hashing + vnodes for cheap rebalancing.

---

## Exercise 1 — Replication or sharding?
1. **Replication** — read-bound + read-heavy → add followers, spread reads. (Add caching too, m06.)
2. **Sharding** — size beyond one node → split the data across machines.
3. **Replication** — availability/SPOF → redundant copies + failover.
4. **Sharding** — write throughput beyond one primary → multiple leaders, each owning a partition
   (replication wouldn't add write capacity).
5. **Both** — shard for write volume, replicate each shard for availability.

---

## Exercise 2 — Sync vs async vs semi-sync
1. **Synchronous** (at least one sync follower must ack before commit) — guarantees the committed txn is
   on ≥2 nodes, so a leader crash loses nothing. **Cost:** higher write latency, and writes block if the
   sync follower is down (availability hit) — acceptable for a ledger where correctness dominates.
2. **Asynchronous** — leader acks immediately; fast, unaffected by slow followers; accepts losing the
   most-recent un-propagated events on a crash. Right when throughput > perfect durability.
3. **Semi-sync:** exactly **one** follower is synchronous (guaranteeing a second durable copy), the rest
   async. Buys near-async latency with a guaranteed second copy — avoids both async's data-loss tail and
   full-sync's "block if any follower is slow."

---

## Exercise 3 — Replication lag bugs
1. **Read-your-writes violated** — write hit the leader, read hit a lagging follower. Fix: route the
   user's reads to the leader (or a confirmed-caught-up replica via tracking the write's log position)
   for a short window after they write.
2. **Monotonic reads violated** — two reads hit replicas with different lag, so the count went backwards.
   Fix: **pin** the user to one replica (e.g. hash user_id → replica) so they always read from the same
   (consistent-with-itself) source.
3. **Consistent-prefix violated** — causally-ordered writes seen out of order. Fix: causal tracking /
   ensure causally-related writes go to (and are read from) the same partition so order is preserved.

---

## Exercise 4 — Failover gone wrong
1. **Split-brain** (two leaders accepting writes → divergent data). Prevent with **fencing** (ensure the
   old leader is truly stopped — STONITH — and/or **fencing tokens** the storage layer checks to reject
   a stale-epoch leader) and **quorum/consensus election** (a partitioned minority can't elect itself).
2. With **async** replication, those **3 seconds of writes are lost** (the promoted follower never
   received them). With sync, they'd have been safe but writes would've been slower.
3. A **1-second timeout** causes **false failovers** on transient blips (GC pause, brief network
   hiccup) — and failovers are themselves risky (possible data loss, split-brain, cold new leader). So
   there's a real trade: too short → flapping; too long → more downtime. No free setting.

---

## Exercise 5 — Quorum math (N=5)
1. **W=3, R=3:** W+R = 6 > 5 → **yes**, read and write sets overlap by ≥1 node, so a read sees the
   latest committed write (modulo the usual edge cases).
2. **Fast/available writes tolerating 3 down:** e.g. **W=1** (write succeeds on one node) with **R**
   small → fast and available, but W+R may be ≤ N → **reads can be stale** (eventual consistency).
   Consequence: you've moved the dial toward availability/latency and away from read freshness.
3. **Strong-ish reads, cheap writes:** **W=1, R=5** gives W+R=6>5 (read all → always see latest) but
   reads are slow/fragile (need all 5 up). More practical: **W=4, R=2** (6>5) — durable writes, cheap
   reads. (Any W+R>5 works; you're choosing where the cost lands.)
4. **No** — even W+R>N doesn't guarantee **linearizability** (sloppy quorums, concurrent writes,
   partially-applied writes). Quorums give **tunable** consistency in an AP system; true linearizability
   needs consensus (Raft/Paxos, m09).

---

## Exercise 6 — Partitioning strategy (time-series)
1. Partition by **timestamp** → **all current writes hit the newest partition** (everything "now" lands
   in one range) while older partitions sit idle → a write hot spot.
2. **hash(metric_id)** → writes spread evenly across partitions (no time hot spot), but "metric X over
   24h" is fine (single metric → one partition); the expensive query becomes **"all metrics in a time
   range"** (scatter-gather across all partitions, since time is no longer the partition dimension).
3. **Compound key: hash(metric_id) as the partition key + timestamp as the sort/clustering key within
   the partition.** Writes spread by metric (even distribution), and a metric's recent points are
   co-located and time-ordered within its partition → "metric X over 24h" is a single-partition range
   scan. (This is exactly the Cassandra time-series pattern.)

---

## Exercise 7 — Consistent hashing
1. Going `% 4` → `% 5` changes the modulus for **nearly every key (~80%)** → almost all data must move
   at once → the cluster melts under the migration while still serving traffic. Catastrophic.
2. With **consistent hashing**, adding the 5th node moves only the keys between it and its ring neighbor
   — roughly **K/N ≈ 1/5 (~20%)** of keys, not 80%.
3. **Virtual nodes:** map each physical node to **many** ring points. This evens out the distribution
   (no node gets a giant arc by chance) and spreads a removed node's load across **many** neighbors
   instead of dumping it on one. (Also enables capacity weighting.)

---

## Exercise 8 — Choose the shard key
1. **Chat:** `conversation_id` — optimizes "fetch a conversation" (single shard, ordered); makes
   "all messages by a user across conversations" a scatter-gather (and a huge conversation risks a hot
   shard).
2. **Orders:** `customer_id` if "a customer's orders" is the dominant read (co-locates them); makes
   `order_id` lookups need routing or a secondary index. (Or shard by `order_id` for even distribution
   if single-order lookups dominate — state the trade.)
3. **Multi-tenant SaaS:** `tenant_id` — each tenant's data co-located, queries stay single-shard;
   risk = a giant "whale" tenant becomes a hot shard (mitigate by sub-sharding big tenants).
4. **Social posts:** `user_id` co-locates a user's posts, but **celebrities create a hot shard** — so
   special-case big accounts (cache, sub-key with a suffix, or a separate path), the "celebrity problem."

---

## Exercise 9 — Secondary index, sharded by user_id, querying by email
1. **Local index:** each shard indexes its own users' emails → a query `WHERE email=?` doesn't know which
   shard → **scatter-gather across all shards** (slow, scales with shard count). Writes are cheap (one
   shard).
2. **Global index:** an email→user_id index partitioned by **email** → an email lookup hits **one** shard
   (fast). But a write (new/changed email) must update the index on a **different** shard than the user
   row → a **cross-shard write**, usually asynchronous/eventually consistent.
3. **Global index** — email lookups are frequent (every login) and writes (email changes) are rare, so
   pay the occasional cross-shard write cost to make the hot read a single-shard lookup. Optimize for the
   dominant pattern.

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **Kafka:** a topic is split into **partitions** (independent ordered logs across brokers) = sharding
   for parallelism/throughput. The **partition key** decides which partition a message goes to; Kafka
   guarantees ordering **within** a partition only. For the **DCE claim stream**, I'd choose the key to
   (a) spread load evenly across partitions and (b) keep events that must stay ordered together — e.g.
   key by claim/member id so a given claim's events are ordered on one partition, while overall volume
   spreads. Avoid a low-cardinality key (e.g. LOB) that would hot-spot a partition.
2. **MongoDB:** a **replica set** = one primary + secondaries with automatic election/failover
   (replication → availability + read scaling via secondary reads). Add a **shard key** to partition the
   claims across shards (write scale + size). Combined = **sharded replica sets**: each shard is its own
   replica set. Failover: if a primary dies, the set **elects** a new primary from up-to-date
   secondaries (majority vote → avoids split-brain).
3. **Redis Cluster / 16,384 hash slots:** keys are hashed to one of 16,384 fixed **slots**, and slots are
   assigned to nodes — a **fixed-partition** scheme (cousin of consistent hashing). Adding/removing a
   node **reassigns whole slots** (and their keys) rather than re-hashing everything → cheap, incremental
   rebalancing with minimal data movement, same benefit as consistent hashing.
4. **Questimate:** single Postgres primary + read-heavy + precomputed snapshots = scaling **reads**
   (the leader handles writes; snapshots/caching absorb read load). If reads kept growing, I'd add
   **read replicas** first (route analytics reads to followers, mind read-your-writes), then caching
   (m06); I'd only **shard** if write volume or dataset size outgrew one primary — which for an
   analytics product it likely wouldn't soon.
