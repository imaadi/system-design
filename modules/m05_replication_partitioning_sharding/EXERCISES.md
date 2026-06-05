# m05 — Exercises (Replication, Partitioning & Sharding)

> Do these on paper / out loud before checking `solutions/m05_replication_partitioning_sharding/`.
> Goal: diagnose whether you need replication or sharding, reason about consistency/failover, and
> design partitioning + shard keys.

---

## Exercise 1 — Replication or sharding?
For each symptom, say whether the fix is primarily **replication**, **sharding**, or **both**, and why:
1. "Read traffic is 100× our writes and the DB is CPU-bound on reads."
2. "Our dataset is 8 TB and won't fit on one machine."
3. "If the database server dies, the whole product goes down."
4. "We're taking 200,000 writes/sec and one primary can't keep up."
5. "We need both: survive a node failure AND handle huge write volume."

---

## Exercise 2 — Sync vs async vs semi-sync
1. A payments ledger absolutely cannot lose a committed transaction, even if the leader dies one second
   later. Which replication mode, and what's the cost you accept?
2. A high-traffic analytics event ingester values write throughput and can tolerate losing a few of the
   most recent events on a crash. Which mode?
3. Explain semi-synchronous replication and what it buys over both extremes.

---

## Exercise 3 — Replication lag bugs
You added read replicas and route reads to them. For each user complaint, name the violated guarantee
and give a concrete fix:
1. "I posted a comment, refreshed, and it's gone."
2. "I saw 50 likes, refreshed, and now it says 48."
3. "I see a reply in the thread but not the original message it's replying to."

---

## Exercise 4 — Failover gone wrong
Your leader's network blips for 10 seconds; the system promotes a follower.
1. The old leader comes back and both accept writes for a minute. Name this failure and two ways to
   prevent it.
2. The promoted follower was 3 seconds behind. What happened to those 3 seconds of writes?
3. Why is "just lower the failure-detection timeout to 1 second" not a free fix?

---

## Exercise 5 — Quorum math
A leaderless store has **N = 5** replicas per key.
1. You set **W = 3, R = 3**. Is a read guaranteed to see the latest committed write? Show the
   inequality.
2. You want **fast, highly-available writes** even if 3 nodes are down, accepting possibly-stale reads.
   Pick W and R and state the consequence.
3. You want **strong-ish reads** with cheap writes. Pick W and R.
4. Does any W/R choice here give you guaranteed **linearizability**? Why or why not?

---

## Exercise 6 — Partitioning strategy
You're storing **time-series metrics** (billions of points), queried mostly as "metric X over the last
24h."
1. If you partition by **timestamp** (range), what hot-spot problem appears on writes?
2. If you partition by **hash(metric_id)**, what do you gain and what query becomes expensive?
3. Propose a partition key that balances write distribution *and* keeps a metric's recent points
   queryable together. (Hint: compound keys.)

---

## Exercise 7 — Consistent hashing
1. You have 4 cache nodes using `hash(key) % N`. You add a 5th node. Roughly what fraction of keys
   remap, and why is that catastrophic?
2. With **consistent hashing** instead, roughly what fraction of keys move when you add the 5th node?
3. With one ring point per node, two problems appear: distribution is uneven, and removing a node
   overloads one neighbor. What fixes both, and how?

---

## Exercise 8 — Choose the shard key
For each, propose a shard key and name the access pattern it optimizes + one it makes expensive:
1. A chat app storing billions of messages.
2. An e-commerce orders table (queried by order_id and by customer_id).
3. A multi-tenant SaaS where each tenant's data is queried independently.
4. A social network's posts (some users are celebrities with millions of followers).

---

## Exercise 9 — Secondary index in a sharded DB
Your `users` table is sharded by `user_id`, but you frequently query `WHERE email = ?`.
1. Describe the **local index** approach and its read cost.
2. Describe the **global index** approach and its write cost.
3. Which would you pick if email lookups are frequent (login) and writes are rare? Why?

---

## Exercise 10 — Apply to your own systems 🏭
1. **Kafka:** explain how a topic's partitions are sharding, what the partition key controls, and the
   ordering guarantee. For the DCE claim stream, what would you weigh in choosing the partition key?
2. **MongoDB at ~1M claims/day:** describe how replica sets (replication) and a shard key
   (partitioning) would combine, and the failover behavior of a replica set.
3. **Redis:** Redis Cluster uses 16,384 hash slots. How is that like consistent hashing, and what does
   it buy when you add/remove a node?
4. **Questimate:** it's read-heavy with precomputed snapshots on (likely) a single Postgres primary.
   Which scaling lever does that represent, and what would you add first if reads kept growing?

---

When done, check `solutions/m05_replication_partitioning_sharding/README.md`, then say **"Module 6"**.
