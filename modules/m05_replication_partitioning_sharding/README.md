# Module 5 — Replication, Partitioning & Sharding ⭐⭐

> m04 was a database on **one machine**. But one machine fails, fills up, and caps out on writes. This
> module is how you spread data across **many** machines — and it's where the deepest distributed-
> systems trade-offs live (the ones m02's CAP/PACELC predicted). Master two ideas and their costs:
> **replication** (keep copies) and **partitioning/sharding** (split the data). They solve *different*
> problems, you usually need *both*, and each buys scale/availability by paying with **consistency,
> complexity, and new failure modes.**

Read in passes: **§1–§5** (replication: leader-follower, sync/async, lag, failover, multi-leader,
leaderless+quorums) and **§6–§10** (partitioning, consistent hashing, rebalancing, routing, shard
keys, combining it all).

---

## 1. Two orthogonal problems — say which one you're solving ⭐

The #1 clarity move in this whole topic: **replication and partitioning are different things.**

| | **Replication** | **Partitioning (Sharding)** |
|---|---|---|
| What | keep **copies** of the *same* data on multiple nodes | **split** the data so each node holds a *different subset* |
| Solves | **availability** (survive node loss) + **read scaling** + lower latency (data near users) | **write scaling** + **data size** beyond one node |
| Doesn't solve | write throughput / total size (every node still holds everything) | availability (each shard is still a SPOF until *also* replicated) |
| Cost | consistency (replica lag), write overhead | cross-partition queries/joins, rebalancing, hot shards, routing |

```
  REPLICATION (copies)              PARTITIONING (split)
  ┌────┐ ┌────┐ ┌────┐              ┌──────┐ ┌──────┐ ┌──────┐
  │ALL │ │ALL │ │ALL │              │ A–H  │ │ I–P  │ │ Q–Z  │
  └────┘ └────┘ └────┘              └──────┘ └──────┘ └──────┘
  same data, N times                different data per node
```

**Real systems combine them:** shard the data, then replicate **each shard**. So you get write-scale
*and* availability — at the sum of both complexities.

```
  Shard 1 (keys A–H)         Shard 2 (keys I–P)        Shard 3 (keys Q–Z)
  ┌─leader─┐                 ┌─leader─┐                ┌─leader─┐
  │ A–H    │  + 2 replicas   │ I–P    │ + 2 replicas   │ Q–Z    │ + 2 replicas
  └────────┘                 └────────┘                └────────┘
```

> **Interview reflex:** when someone says "how do you scale the database?", split your answer:
> *"Reads and availability → **replicate**. Writes and size beyond one node → **shard**. Usually both:
> shard, then replicate each shard."* That single distinction signals you actually understand the space.

---

## 2. Replication — leader-follower (the default model) ⭐

The most common scheme (Postgres, MySQL, MongoDB replica sets, Redis): one **leader** (primary) +
several **followers** (replicas).
- **All writes go to the leader.** The leader applies them and streams its change log (the **WAL**
  from m04!) to followers.
- **Reads can go to any replica** → this is how you **scale reads** (add followers) and serve data
  closer to users.

```
            writes ─────▶ ┌──────────┐  WAL stream  ┌──────────┐
   clients               │  LEADER   │ ───────────▶ │ FOLLOWER │ ◀── reads
            reads ──────▶ └──────────┘ ───────────▶ ┌──────────┐
                                                     │ FOLLOWER │ ◀── reads
                                                     └──────────┘
```

> **Key insight:** leader-follower scales **reads**, not **writes** — every write still goes through the
> single leader. For a **read-heavy** workload (most web apps, ~100:1) this is perfect. For a
> **write-heavy** one, replication alone won't help; you need **sharding** (§6). Identifying the
> read/write ratio (m01) tells you which lever to pull.

### Synchronous vs asynchronous replication (a durability/latency/availability trade) ⭐
*When* does the leader tell the client "committed"?
- **Synchronous:** leader waits for the follower(s) to confirm before acknowledging the write.
  **Pro:** the write is safely on ≥2 nodes (durable; no data loss on leader failure). **Con:** higher
  write latency; if a sync follower is down, **writes block** (availability hit).
- **Asynchronous:** leader acknowledges immediately, streams to followers in the background.
  **Pro:** fast writes, leader unaffected by slow/down followers. **Con:** if the leader crashes
  before a write propagates, that write is **lost** (durability gap).
- **Semi-synchronous (the usual compromise):** **one** follower is synchronous (guaranteed a second
  copy), the rest async. Balances durability and latency.

> This is m02's PACELC made concrete: *"Even with no partition, I trade **latency vs consistency/
> durability** on every write — sync is safe-but-slow, async is fast-but-can-lose-the-tail."*

### Replication lag — the source of most "weird" bugs ⭐
With async replication, followers are **behind** the leader by some **lag** (ms to seconds, more under
load). That window creates exactly the consistency problems from m02 §6:
- **Read-your-own-writes violated:** you write to the leader, then your read hits a lagging follower
  that doesn't have it yet → "I just posted, where's my comment?!" **Fix:** route a user's reads to
  the leader for a short window after they write, or to a replica known to be caught up (track the
  write position).
- **Monotonic reads violated:** successive reads hit different-lag replicas → you see a value, then it
  "disappears" (time goes backwards). **Fix:** pin a user to one replica (e.g. hash user → replica).
- **Consistent-prefix violated:** you see effects out of causal order (a reply before its question).
  **Fix:** causal tracking / write to the same partition.

> **The trap to avoid:** "I'll just read from replicas to scale." Correct, but **replicas are
> eventually consistent** — naming the read-your-writes/monotonic problems *and their fixes* is what
> separates senior from junior here.

---

## 3. Failover — when the leader dies (and how it goes wrong) ⭐

If the leader fails, a follower must be **promoted**. Automatic failover:
1. **Detect** the leader is dead (heartbeat timeout — but is it dead, or just slow/partitioned?).
2. **Choose** a new leader (the most up-to-date follower; often via a consensus algorithm — m09).
3. **Reconfigure** clients/replicas to use the new leader.

**The dangers (great cross-question material):**
- **Data loss:** with async replication, writes the old leader hadn't shipped are **lost** when a
  less-current follower is promoted.
- **Split-brain ⚠️ (the scary one):** the old leader wasn't actually dead (just partitioned), so now
  **two nodes think they're leader** and both accept writes → divergent, conflicting data. **Fix:
  fencing** — ensure the old leader is truly stopped before promoting (STONITH "shoot the other node
  in the head"), or use **fencing tokens** (a monotonically increasing epoch; the storage layer
  rejects writes from an stale-epoch leader). Quorum-based systems avoid split-brain by requiring a
  majority to elect/accept (m09).
- **Timeout tuning:** too short → unnecessary failovers on a transient blip (which are themselves
  risky); too long → longer downtime. There's no free setting.

> **Say this:** *"Automatic failover is necessary but dangerous — the two failure modes are **data
> loss** (async tail) and **split-brain** (two leaders). I'd guard split-brain with fencing/fencing
> tokens and prefer **quorum/consensus-based** leader election so only a majority-backed node can
> lead."*

---

## 4. Multi-leader & leaderless replication

### Multi-leader (multi-master)
Multiple nodes accept **writes** (e.g. one leader per datacenter, or per device for offline apps).
- **Why:** lower write latency across regions, keep working during a DC outage, offline-first apps
  (each device is a "leader" that syncs later).
- **The hard problem — write conflicts:** two leaders concurrently modify the same record differently.
  You must **resolve** them:
  - **Last-Write-Wins (LWW):** keep the one with the latest timestamp. Simple but **loses data** and
    depends on clock sync (dangerous — m09).
  - **Version vectors:** detect concurrent edits and surface/merge them.
  - **CRDTs (Conflict-free Replicated Data Types):** data structures (counters, sets, sequences) that
    **merge automatically without conflict** because their operations are commutative/monotonic —
    this is the **CALM theorem** from m02 in practice (collaborative editors, shopping carts).
- **Cost:** conflict handling is genuinely hard; avoid multi-leader unless you need cross-region writes
  or offline.

### Leaderless replication (Dynamo-style: Cassandra, DynamoDB, Riak)
No leader — the **client (or a coordinator) writes to several replicas directly**, and reads from
several. Consistency is tuned with **quorums**:

> **Quorum rule: W + R > N** (where **N** = replicas per key, **W** = nodes that must ack a write,
> **R** = nodes that must respond to a read). If W+R > N, the read set and write set **overlap by ≥1
> node** → a read is guaranteed to see the latest write. ⭐

Examples (N=3): **W=2, R=2** (2+2>3 ✓, the common balanced choice); **W=3, R=1** (write-all/read-one →
fast reads, slow/fragile writes); **W=1, R=3** (fast writes, slow reads). It's a **tunable consistency
dial**:
- High W → more durable/consistent writes, but lower write availability (need more nodes up).
- High R → fresher reads, but slower/less available reads.
- W+R ≤ N → faster but **may read stale** (eventual consistency).

Supporting mechanics: **read repair** (on a read, if a replica is stale, write the fresh value back),
**anti-entropy** (background process syncs divergent replicas, e.g. Merkle-tree comparison), **hinted
handoff** (if a target replica is down, another node holds the write and forwards it later — keeps
writes available during failures = "sloppy quorum").

> **Think differently — quorum isn't a silver bullet.** Even with W+R>N you can still read stale data
> in edge cases (sloppy quorums, concurrent writes, failed writes that partially applied). Quorums
> give you a **tunable** consistency/availability trade, not guaranteed linearizability. This is the
> AP world of m02's CAP — say "tunable consistency," not "strongly consistent."

---

## 5. Replication — putting it together
| Model | Writes | Conflict risk | Use when |
|---|---|---|---|
| **Leader-follower** | one leader | none (single writer) | most apps; read scaling + HA (default) |
| **Multi-leader** | many leaders | **high** (need resolution) | multi-region writes, offline-first |
| **Leaderless (quorum)** | any replica | handled via versions/repair | high availability, write scale, AP (Dynamo/Cassandra) |

The senior default: **leader-follower** (simple, no conflicts) for read scaling + availability. Reach
for the others only when cross-region writes (multi-leader) or extreme availability/write-scale
(leaderless) genuinely demand it.

---

## 6. Partitioning (sharding) — splitting the data ⭐⭐

When writes or **data size** exceed one node, you **partition**: split rows across nodes so each holds
a subset. The whole game is **choosing how to split** so load spreads **evenly** and queries stay
efficient. Two base strategies:

### Range partitioning
Assign contiguous **ranges** of the key to partitions (e.g. users A–H, I–P, Q–Z; or by date).
- **Pro:** efficient **range scans** (e.g. "all orders in March" hit one/few partitions); keys stay
  ordered.
- **Con:** **hot spots** — uneven ranges or sequential keys overload one partition. Classic disaster:
  partitioning by **timestamp** → *all* of today's writes hammer the newest partition while older ones
  idle. Choosing the boundaries is hard and data is skewed.

### Hash partitioning
Apply a hash to the key and assign by hash (e.g. `partition = hash(key) % N`).
- **Pro:** **spreads load evenly** (a good hash randomizes), killing the sequential-key hot spot.
- **Con:** **destroys ordering** → range queries must hit **all** partitions (scatter-gather). And the
  naive `% N` has a catastrophic flaw on resize (next section).

> **The unavoidable hot-key problem:** even perfect hash partitioning can't save you from a **single
> hot key** — a celebrity user, a viral product. All their traffic lands on one partition (one key
> can't be split by hashing the key). Mitigations: add a random suffix to split the key into sub-keys,
> cache the hot item at the app/CDN tier (m06), or special-case it. Name this; it's the "celebrity
> problem" (and it returns in the news feed, m13).

---

## 7. Consistent hashing — the elegant fix for resizing ⭐⭐ (think differently)

**The problem with `hash(key) % N`:** when you add/remove a node (N changes), the modulus changes for
**almost every key**, so **~all data must move** — a catastrophic reshuffle that melts the cluster.
Example: with `% 4`, key→node mappings; switch to `% 5` and ~80% of keys remap.

**Consistent hashing** fixes this. Map both **nodes** and **keys** onto a circular hash space (a
"ring", say 0…2³²−1). A key belongs to the **first node clockwise** from its hash position.

```
        0/2³²
         ┌───────●NodeA──────┐
         │                    │
    ●key3│                    │●NodeB
         │       (ring)       │
   NodeD●│                    │
         │                    │●key1
         └──────●NodeC────────┘
   key → walk clockwise → first node owns it
```

> **Why it's beautiful:** when a node is **added or removed, only the keys between it and its neighbor
> move** — roughly **K/N keys** (K=total keys, N=nodes), *not all of them.* Adding a 5th node to a
> 4-node ring relocates ~1/5 of keys, not 80%. This is why **Dynamo, Cassandra, and consistent caches
> (m06)** use it: cheap, incremental rebalancing.

**Virtual nodes (vnodes) — the essential refinement:** placing one point per physical node gives
**uneven** distribution (and removing a node dumps all its load on one neighbor). So each physical node
is mapped to **many** points on the ring (virtual nodes). Result: **even load distribution**, and when
a node leaves, its load spreads across **many** neighbors (not one). Vnodes also let you weight by
machine capacity (a bigger box gets more vnodes).

> **Interview gold:** *"Naive `hash % N` reshuffles nearly all keys when N changes. **Consistent
> hashing** moves only ~K/N keys on a membership change, and **virtual nodes** make the distribution
> even and spread a departed node's load across many peers. That's why it's the backbone of
> distributed caches and Dynamo-style stores."*

---

## 8. Rebalancing & request routing

**Rebalancing** = moving partitions across nodes as you add/remove capacity or fix skew. The guiding
principle: **move as little data as possible**, and don't disrupt live traffic. A common, robust
approach: **fix a large number of partitions up front** (say 1,000) and assign many to each node;
adding a node just **reassigns some whole partitions** to it (no re-hashing of keys). (This is how
Elasticsearch, Riak, and others avoid the `% N` problem without a literal ring.)

**Request routing** — how does a client find the partition for a key? Three patterns:
1. **Routing tier / coordinator:** a layer that knows the partition map and forwards (e.g. a proxy, or
   mongos in MongoDB).
2. **Client-aware:** the client library knows the map and contacts the right node directly (fewer
   hops; client must stay in sync with topology).
3. **Any-node (gossip):** contact any node; it forwards or redirects (Cassandra-style, nodes gossip
   the topology).
The partition map itself is usually kept in a **coordination service** (ZooKeeper/etcd — m09) or
gossiped between nodes.

---

## 9. Choosing the shard key — the most consequential decision ⭐

The **shard/partition key** decides everything: load distribution, which queries are cheap, and how
much pain resharding will be. Pick it for:
- **Even distribution** (avoid hot spots) — high-cardinality, uniformly-accessed key.
- **Query locality** — queries you run together should land on the **same** shard (avoid scatter-
  gather). E.g. shard chat messages by `conversation_id` so a conversation is one shard.
- **Avoiding cross-shard transactions** — operations that must be atomic should live on one shard
  (cross-shard transactions need 2PC/saga — m09, and are slow/complex).

**Trade-offs are inherent:** sharding chat by `conversation_id` makes "fetch a conversation" a single-
shard read (great) but "all messages by this user across conversations" a scatter-gather (costly). You
optimize the **dominant** access pattern and accept the others are expensive.

> **Why this is scary:** the shard key is **extremely hard to change later** — resharding means moving
> huge amounts of live data. A bad shard key (e.g. low cardinality, or one that creates hot spots) can
> require a painful migration. So you think hard up front, and you choose a key that distributes evenly
> *and* matches your dominant query. (DCE/Mongo and Kafka both force this choice — see §11.)

### Secondary indexes in a partitioned world
A complication: data is partitioned by the **primary** key, but you often query by **other** fields.
- **Local (document-partitioned) index:** each shard indexes its own data → writes are cheap (one
  shard), but a query on the secondary field must **scatter-gather** across all shards.
- **Global (term-partitioned) index:** the secondary index is itself partitioned by the index term →
  reads hit one shard (fast), but a write must update the index on a *different* shard (cross-shard
  write, eventual consistency).
This is the read-vs-write trade again, at the index level.

---

## 10. From your systems 🏭
- **Kafka partitions = sharding, lived:** a Kafka topic is split into **partitions**, and the
  **partition key decides which partition a message lands in** — that's a shard key. Ordering is
  guaranteed **within** a partition, not across — exactly §6/§9. DCE's claim stream is partitioned;
  choosing the key (e.g. by member/claim id) balances throughput while keeping a given entity's events
  ordered. You can speak to partition-key choice from real experience.
- **MongoDB at ~1M claims/day:** Mongo uses **replica sets** (leader-follower: a primary + secondaries,
  automatic failover/election — §2/§3) and optional **sharding** (by a shard key — §9). Your DCE claims
  store is the textbook "shard, then replicate each shard."
- **Redis as a replicated/partitioned store:** Redis primary-replica replication (and **Redis Cluster**
  uses **16,384 hash slots** — a consistent-hashing-style fixed-partition scheme, §7/§8). Your
  Redis-backed session/reference layer is a replicated KV store.
- **Read scaling via replicas/snapshots:** Questimate's read-heavy analytics + **precomputed snapshot
  tables** are the "scale reads, not writes" pattern (§2) — and a single Postgres primary handling the
  writes is the "leader-follower scales reads" reality.
- **Failover awareness:** any primary-replica setup you ran (Mongo/Postgres/Redis) has the §3 failover
  + split-brain concerns — a great place to mention fencing/quorum election.

---

## 11. Key concepts (interview-ready)
- **Replication ≠ partitioning.** Replication = copies → **availability + read scale**; partitioning =
  split → **write scale + size**. Combine them (shard, then replicate each shard). Say which you need.
- **Leader-follower** scales **reads**, not writes (single leader). **Sync vs async** = durability/
  latency trade (semi-sync = compromise). **Replication lag** breaks read-your-writes / monotonic /
  consistent-prefix — know the fixes (route to leader, pin to replica, causal tracking).
- **Failover is dangerous:** **data loss** (async tail) + **split-brain** (two leaders) → guard with
  **fencing tokens** and **quorum/consensus** election.
- **Multi-leader** → write **conflicts** → LWW (lossy)/version vectors/**CRDTs** (auto-merge,
  CALM). **Leaderless (Dynamo)** → **quorums: W+R>N** for overlap; tunable consistency dial; read
  repair / anti-entropy / hinted handoff. Quorum = *tunable*, not guaranteed linearizable.
- **Partitioning:** **range** (good range scans, hot-spot risk, esp. timestamps) vs **hash** (even
  load, no range scans). The **hot-key/celebrity** problem survives hashing → suffix/cache/special-case.
- **Consistent hashing**: naive `hash%N` reshuffles ~all keys on resize; consistent hashing moves
  ~**K/N**; **virtual nodes** make it even + spread a lost node's load. Backbone of caches + Dynamo.
- **Shard key is the most consequential, hardest-to-change choice** — pick for even distribution +
  query locality + minimal cross-shard transactions. Secondary indexes → local (scatter-gather reads)
  vs global (cross-shard writes).

---

## 12. Go deeper (the well-researched reading list)
- **Kleppmann, DDIA** — Ch. 5 (Replication: leader/multi-leader/leaderless, lag, quorums) and Ch. 6
  (Partitioning: strategies, rebalancing, routing, secondary indexes). The definitive source for this
  whole module.
- **Amazon *Dynamo* paper** — consistent hashing + virtual nodes + quorums + hinted handoff + read
  repair, all in one (§4, §7).
- **Karger et al., the original *Consistent Hashing* paper** (1997) — the §7 idea at its source.
- **"Jepsen" analyses (jepsen.io)** — real distributed databases tested under partition/failover;
  shows how split-brain and lost-update bugs actually happen (§3).
- **Kafka docs: partitions & keys** — your §10 example, authoritative.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m05_replication_partitioning_sharding/`](../../solutions/m05_replication_partitioning_sharding/README.md).

When you've done the exercises, say **"Module 6"** to build *Caching & CDNs* (cache strategies,
eviction, Redis/Memcached, CDNs, invalidation, thundering herd, hot keys) — the other half of scaling
reads.
