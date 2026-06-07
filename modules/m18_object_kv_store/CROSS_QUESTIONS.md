# m18 — Cross-Questions ("if-and-buts") on KV Store + Object Store

> Answer out loud in 2–3 sentences before reading the model answer. This module rewards *naming the
> primitive and why* — it's m05/m09 assembled.

---

### Q1. Design a distributed key-value store. What primitives do you assemble?
**A.** The **Dynamo** toolkit: **consistent hashing + virtual nodes** to partition keys across nodes;
**N-way replication** (a key lives on the next N nodes on the ring) for durability/availability;
**quorums (W+R>N)** for tunable consistency; **vector clocks** to detect concurrent/conflicting writes;
**gossip** for decentralized membership/failure detection; **hinted handoff + sloppy quorum** to stay
writable during failures; and **read repair + anti-entropy (Merkle trees)** to converge replicas. It's
**AP, always-writable** — m05 + m09 assembled.

---

### Q2. How does a `put` flow through a Dynamo-style store?
**A.** Hash the key → locate it on the **consistent-hash ring** → the **N** nodes clockwise are its
replicas → a coordinator writes to them and waits for **W** acknowledgments before acking the client. If
some replicas are **down**, **hinted handoff** writes to a temporary node (a "hint" to forward later) so
the write still succeeds (**sloppy quorum**) — preserving the "always writable" guarantee. Virtual nodes
keep the load even across physical machines.

---

### Q3. How does a `get` work, and what if replicas disagree?
**A.** Read from **R** replicas. If they return **different versions**, use **vector clocks** to decide:
if one version **descends from** another, keep the newer (and **read-repair** the stale replica by
writing the latest back); if the versions are **concurrent** (a genuine conflict from two simultaneous
writes), return **all conflicting versions** to the application to **reconcile**. Background **anti-
entropy** (Merkle-tree comparison) also converges replicas that read repair didn't touch.

---

### Q4. Why does "always writable" create conflicts, and how do you resolve them?
**A.** Because the store **accepts writes even during a partition** (for availability), two clients can
write the **same key on different sides** of the partition → **conflicting versions** when it heals.
**Vector clocks** detect them; resolution options: **Last-Write-Wins** (keep latest timestamp — simple
but **loses data** and trusts clocks, m09), **application merge** (return both, app reconciles — Amazon's
cart **unions** items so no add-to-cart is lost), or **CRDTs** (data types that merge automatically).
Conflict resolution is the **cost** of choosing availability over consistency.

---

### Q5. What do vector clocks do that timestamps can't?
**A.** **Vector clocks** capture **causality** — they can tell whether version A **happened-before** B
(B descends from A → keep B), B before A, or A and B are **concurrent** (a real conflict). A plain
**timestamp** can't distinguish "newer" from "concurrent" and, due to clock skew (m09), can pick the
**wrong** version as "latest" — silently dropping the actually-newer write (the LWW hazard). Vector
clocks let you **detect** conflicts instead of blindly (and sometimes wrongly) overwriting.

---

### Q6. What's the role of gossip in a Dynamo-style store?
**A.** **Gossip** is a decentralized protocol where nodes periodically exchange membership/state info with
random peers, so the cluster learns **who's alive, who joined/left, and the ring layout** **without a
central coordinator** (no SPOF). It's how a fully peer-to-peer store does **failure detection** and
**membership** — every node converges on a consistent view of the cluster over time. It trades instant
consistency of membership for **decentralization and resilience** (m09).

---

### Q7. What are hinted handoff and sloppy quorum?
**A.** When a target replica is **down** at write time, **hinted handoff** writes the data to a different,
healthy node along with a **"hint"** to forward it to the intended replica once it recovers. A **sloppy
quorum** means W/R are satisfied by the **first N healthy nodes** (not strictly the "home" replicas) — so
writes/reads succeed during failures. Together they keep the store **available** during node outages, at
the cost of temporary inconsistency until the hints are delivered (then read-repair/anti-entropy
converge).

---

### Q8. Is the Dynamo *paper* the same as Amazon *DynamoDB*?
**A.** No. The **Dynamo paper** (2007) describes an **AP, eventually-consistent, vector-clock** store.
**DynamoDB** (the product) evolved differently: it's a **managed** service with a **partition-key + sort-
key** model that offers **both** eventually-consistent (cheap) **and strongly-consistent reads**, and
hides partitioning/replication from you. **Cassandra** is the closest open-source descendant of the
*paper*. Don't conflate "the paper's design" with "the product."

---

### Q9. How do you design an object/blob store like S3?
**A.** **Separate metadata from data**: a **metadata service** maps `bucket/key → {location, size,
checksum, version}` in a fast **KV/DB** (small, queryable), while the **actual bytes** live on **storage
nodes** (chunked for large objects, replicated or erasure-coded). PUT writes bytes to storage nodes +
records metadata; GET looks up metadata then fetches bytes (often via CDN). Objects are **immutable**
(write-once + versioning), durability is the #1 NFR (11 nines via replication/erasure coding across racks/
AZs), and large uploads use **multipart**.

---

### Q10. Why separate metadata from the object data?
**A.** Because they have **opposite characteristics**: metadata is **tiny, queryable, and frequently
accessed** (you list buckets, look up objects) → a fast indexed KV/DB; the data is **huge (TBs), bulk,
write-once** → cheap durable storage nodes. Mixing them would either bloat the queryable store or make the
bulk store do indexing it's bad at. It's the same **"split metadata from blobs"** pattern as m04/m11/m17 —
each in the storage best suited to its access pattern. (DCE: claims in Mongo, model blobs in MinIO.)

---

### Q11. Replication vs erasure coding for durability — explain the trade-off.
**A.** **3× replication** stores 3 full copies → simple, fast reads, but **200% storage overhead** (you
pay 3× for 1× of data). **Erasure coding** (Reed-Solomon **k+m**, e.g. **10+4**) splits an object into 10
data + 4 parity shards and can **reconstruct from any 10 of the 14** → survives any 4 failures with only
**~40% overhead** — *cheaper* and *more durable*. The cost: **CPU to encode/decode** and **extra network/
latency to reconstruct** a lost shard on read. So: **replicate hot/small data, erasure-code cold/large
data** (the bulk).

---

### Q12. Why do object stores use erasure coding for the bulk of their data?
**A.** Because at exabyte scale, **storage cost dominates**, and erasure coding gives the same (or better)
**11-nines durability at ~40% overhead instead of 200%** — a massive cost saving on the cold, large
objects that make up most of the store. The downside (more CPU + slower reconstruction) is acceptable for
data that's read less often and where cost matters most. Hot, latency-sensitive, or small objects may
still be replicated; it's a per-tier decision driven by cost vs read-latency.

---

### Q13. Why does immutability simplify an object store?
**A.** Because **write-once objects never change in place**, you avoid the hardest consistency problems —
there's no "update being applied to some replicas but not others," no read-modify-write races, no
invalidation of partial updates. A "change" is just a **new version** (a new immutable object), so old
readers keep a consistent view and you can cache/CDN aggressively (versioned URLs never go stale). It's
the same reason the URL-shortener mapping (immutable) made caching easy (m11) — immutability removes the
consistency-during-update problem.

---

### Q14. How do you guarantee 11 nines of durability and detect bit-rot?
**A.** **Redundancy across failure domains** (replicas/parity shards spread across racks, AZs, and often
regions, so no correlated failure loses all copies), plus **erasure coding** for efficient redundancy.
For silent corruption (**bit-rot**), store a **checksum per object/shard**, **verify on read**, and run a
**background scrubber** that periodically re-checks stored data and **re-creates** corrupted shards from
the surviving replicas/parity. Durability isn't just "have copies" — it's "have copies in independent
failure domains *and* continuously verify/repair them."

---

### Q15. How do you upload a 5 GB file reliably?
**A.** **Multipart upload**: split the file into **parts** (e.g. 100 MB each), upload them **in parallel**
(faster) and **independently retryable** (a failed part is re-sent without restarting the whole upload),
then a final "complete" call **assembles** them into one object (verifying part checksums). This handles
large files, flaky networks (resume from the failed part), and parallel throughput — far better than a
single huge PUT that fails at 99% and starts over.

---

### Q16. What consistency does S3 provide, and has it changed?
**A.** Originally S3 was **eventually consistent** for overwrites/listing; since 2020 it provides
**strong read-after-write consistency** for new objects (a GET right after a PUT returns the new object).
This is possible partly because objects are **immutable** (a new object/version is unambiguous) — the
hard case is overwrites/deletes. The lesson: even "AP" object stores can offer strong consistency for the
**simple, immutable-write** case, where coordination is cheap; you don't always have to choose pure AP.

---

### Q17. The metadata store for your object store gets huge. What do you do?
**A.** It's itself a **distributed KV store / DB at scale**, so you **shard it** (by bucket/key, consistent
hashing) and **replicate** it (Part A) — i.e. you apply the same Dynamo techniques to the metadata layer.
Metadata is small per object but there are **billions of objects**, so the metadata store can be a serious
system on its own (Google's Colossus/GFS master, S3's index). You cache hot metadata and keep the lookups
O(1) since they're on the read path before fetching bytes.

---

### Q18. How does a hot object (a viral video/image) get served at scale?
**A.** **CDN** (m06): the object is cached at edge PoPs near users, so reads are served from the edge, not
origin — handling the **hot-key** problem and saving origin bandwidth. Because objects are **immutable**,
you use **versioned URLs** (no invalidation needed). For an object store backing video (m19), the segments
live in the store but are delivered through a CDN; origin sees only cache-fill traffic. Replicating the
hot object across storage nodes also helps reads that do reach origin.

---

### Q19. When would you pick a KV store vs an object store vs a relational DB?
**A.** **KV store (Dynamo/Cassandra/DynamoDB)**: simple key→value access at massive scale with tunable
consistency, no joins — sessions, carts, profiles, high write throughput. **Object store (S3/MinIO)**:
**large immutable blobs** (images, video, backups, model artifacts) where durability + cost matter and you
fetch by key, not query. **Relational**: structured data with relationships, joins, multi-row
transactions (m04). They're complementary — an app often uses all three (the **polyglot** point, m04):
metadata/transactions in SQL, hot KV in Dynamo/Redis, blobs in S3.

---

### Q20. How is this module the payoff of m05 and m09?
**A.** Because a Dynamo-style KV store is **literally those modules assembled**: m05's **consistent
hashing, replication, quorums, hinted handoff, read repair/anti-entropy** + m09's **vector clocks and
gossip** — no new primitive, just composed into a coherent always-available store. The object store then
layers **metadata/data separation, erasure coding, and immutability** on top. So if you understand m05/m09,
you can **derive** this design from requirements rather than memorize it — which is exactly the architect
skill the course is building.
