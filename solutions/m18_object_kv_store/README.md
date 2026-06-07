# m18 — Solutions (KV Store + Object Store)

> Check the *reasoning*. Recurring lessons: a Dynamo KV store = m05+m09 assembled; conflicts resolved at
> read time via vector clocks; object store = metadata/data split + erasure coding (40% vs 200%).

---

## Exercise 1 — Assemble Dynamo
1. **Consistent hashing + virtual nodes** → partition keys evenly, cheap rebalancing (m05).
2. **N-way replication** (next N nodes on ring) → durability + availability (m05).
3. **Quorums W+R>N** → tunable consistency (m05).
4. **Hinted handoff + sloppy quorum** → stay writable when replicas are down (m05).
5. **Read repair + anti-entropy (Merkle trees)** → converge divergent replicas (m05).
6. **Vector clocks** → detect concurrent/conflicting writes (m09).
7. **Gossip** → decentralized membership + failure detection, no SPOF (m09).

---

## Exercise 2 — put/get walkthrough
1. **put(N=3,W=2):** hash key → next 3 ring nodes are replicas → coordinator writes to them, waits for
   **2 acks**. If one replica is **down**, **hinted handoff** writes to a healthy node (with a hint to
   forward later) → W=2 still met → ack client (sloppy quorum keeps it writable).
2. **get(R=2), versions differ:** use **vector clocks** — if one version **descends** from the other, keep
   the newer and **read-repair** the stale replica; if **concurrent** (a real conflict), return **both**
   for the app to reconcile.
3. (a) **W=1,R=1** (W+R=2 ≤3 → fast/available, may read stale); (b) **W=1,R=3** or **W=3,R=1** (W+R=4>3 →
   strong-ish); (c) **W=2,R=2** (W+R=4>3 → balanced default).

---

## Exercise 3 — Conflicts
1. Because the store **accepts writes during a partition** (for availability), two clients can write the
   **same key on different sides** → conflicting versions when it heals.
2. **Vector clocks** — they reveal causality: a conflict is when neither version **happened-before** the
   other (they're **concurrent**); a simple newer-version is when one **descends from** the other.
3. **LWW** (keep latest timestamp — simple but **loses data**, trusts clocks); **app-merge** (return both,
   app reconciles — **Amazon cart unions** items so no add-to-cart is lost — semantic, no loss); **CRDTs**
   (auto-merge — no conflict, but constrains the data model).

---

## Exercise 4 — Vector clocks vs timestamps
1. A vector clock tells you whether two versions are **causally ordered** (one happened-before the other)
   or **concurrent** (a true conflict) — a timestamp only gives a (skew-prone) "which is newer."
2. **LWW** picks the larger timestamp, but with **clock skew** (m09) the "larger" timestamp can be the
   **actually-older** write → it **silently drops the newer write** (data loss).
3. LWW is acceptable when **conflicts are rare/cosmetic** or the data is **last-value-wins by nature**
   (e.g. a status field, a cache), and you accept occasional loss for simplicity (Cassandra's default).

---

## Exercise 5 — Gossip, hinted handoff, sloppy quorum
1. **Gossip** provides **decentralized membership + failure detection** — nodes exchange cluster state
   with random peers, so everyone learns who's alive/the ring layout **without a central coordinator** (no
   SPOF).
2. **Hinted handoff + sloppy quorum** — the write goes to a healthy substitute node (with a hint) and W is
   met by the first N **healthy** nodes, so the write **succeeds despite the outage**.
3. When the intended replica **recovers**, the node holding the **hint forwards** the missed write to it;
   **read repair / anti-entropy** also reconcile any remaining divergence.

---

## Exercise 6 — Object store architecture
1. Separate because metadata is **small/queryable/hot** (→ KV/DB) and data is **huge/bulk/write-once** (→
   storage nodes). **Metadata** = key → {location, size, checksum, version}; **data** = the actual bytes.
2. **Multipart upload** solves **large-file reliability/throughput**: upload parts in parallel,
   independently retryable (resume a failed part, don't restart), then assemble — vs one huge PUT that
   fails at 99%.
3. **Immutability** = write-once → no in-place updates → **no consistency-during-update problem**, no
   read-modify-write races; a change is a **new version**, so caching/CDN with versioned URLs never goes
   stale.

---

## Exercise 7 — Erasure coding
1. **3× replication = 200% overhead** (store 3× the data).
2. **(10+4):** survive **any 4** shard losses; **~40% overhead** (4 parity / 10 data).
3. **Replicate** hot/small/latency-sensitive data (simple, fast reads); **erasure-code** cold/large/bulk
   data (huge cost saving). Cost of erasure coding: **CPU to encode/decode** + **extra network/latency to
   reconstruct** a lost shard on read.
4. **100 PB cold data:** 3× → store ~**300 PB**; (10+4) → ~**140 PB** (40% overhead) → roughly **~2.1×
   cheaper** storage for the same-or-better durability.

---

## Exercise 8 — Durability & integrity
1. **(a) Redundancy across independent failure domains** (replicas/parity spread over racks/AZs/regions so
   no correlated failure loses all) + **(b) erasure coding/replication** for efficient redundancy.
2. **Checksum per object/shard**, **verify on read**, and a **background scrubber** that re-checks stored
   data and **re-creates** corrupted shards from surviving replicas/parity.
3. Because copies can suffer **correlated failures** (same rack/AZ) and **silent bit-rot** — "3 copies in
   one rack" dies with the rack, and undetected corruption rots all copies. Durability = redundancy across
   **independent** domains **plus continuous verification/repair**.

---

## Exercise 9 — Choosing the store
1. **KV store** (Dynamo/DynamoDB) — simple key access at scale, must always accept writes (AP, the cart
   use case).
2. **Object store** (S3/MinIO) — large immutable blobs fetched by name; durability + cost matter.
3. **Relational** — multi-row ACID transactions + integrity (money).
4. **Object store** — large user-uploaded video blobs (+ CDN delivery).
5. **KV store** (Redis) — sub-ms lookup by id; ephemeral.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **DCE:** **Mongo = metadata/data store for claims** (queryable records), **MinIO = object store for
   model artifacts (pkl/sav blobs)** — Part B's split: small/queryable in a DB, large write-once blobs in
   object storage. Model files go to **object storage** because they're **large, immutable, fetched by
   name** (not queried/joined) and want cheap durable storage — a DB would be the wrong tool (bloated,
   expensive, no benefit).
2. **Operated Dynamo primitives:** **replication** (Mongo replica sets), **consistent-hashing-style
   partitioning** (Redis Cluster hash slots), and **tunable consistency / quorums** (Mongo read/write
   concerns — choosing W/R per operation). (Also failover/election ≈ gossip-backed membership.)
3. **The payoff:** since a Dynamo KV store is just **m05 (consistent hashing/replication/quorums/hinted
   handoff/read-repair) + m09 (vector clocks/gossip)** composed — and an object store adds metadata/data
   split + erasure coding + immutability — you can **derive** these designs from requirements instead of
   memorizing them. That derive-from-primitives ability is the architect skill the course builds.
