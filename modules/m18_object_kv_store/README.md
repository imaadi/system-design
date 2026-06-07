# Module 18 — Design a Key-Value Store (DynamoDB) + Object Store (S3) ⭐⭐

> This module is where the **building blocks become the whole design.** A **distributed key-value store**
> (the **Dynamo** design) is literally **m05 (consistent hashing, replication, quorums) + m09 (vector
> clocks, gossip) assembled** — the canonical "always-available, AP" store. Then an **object/blob store**
> (S3) adds large-object handling, **metadata/data separation**, and **erasure coding** for cheap extreme
> durability. Two designs, one theme: **distribute, replicate, and tune consistency.**

> **Format (Part 2):** worked walkthrough. **Part A = KV store (Dynamo)** assembles your distributed-
> systems knowledge; **Part B = object store (S3)** adds durability/erasure coding. Your DCE **MinIO**
> (model artifacts) and **Mongo/Redis** are direct experience.

---

# PART A — Distributed Key-Value Store (Dynamo-style) ⭐⭐

## A1 — Requirements
**Functional:** `get(key)` and `put(key, value)` — a simple interface over a huge, distributed dataset.
**Non-functional (the Dynamo philosophy):**
- **Always writable / highly available:** writes must (almost) never be rejected (Amazon's shopping cart:
  never lose an "add to cart") → **AP** (favor A over C under partition, m02).
- **Scalable & decentralized:** scale to thousands of nodes, **no single point of failure** (peer-to-
  peer, no master).
- **Eventually consistent / tunable:** accept temporary divergence; let the app **tune** consistency
  per-operation (quorums).
- **Partition-tolerant + durable:** survive node/network failures without data loss.

> **The insight:** this isn't a new design — it's **assembling the m05/m09 primitives** into the Dynamo
> paper. Naming each technique and *why* is the whole answer.

## A2 — The Dynamo techniques (m05 + m09, assembled) ⭐⭐

| Problem | Technique | Module |
|---|---|---|
| **Partition** the data across nodes | **Consistent hashing + virtual nodes** | m05 |
| **High availability + durability** | **Replication** — each key on **N** nodes (next N on the ring) | m05 |
| **Tunable consistency** | **Quorums: W + R > N** | m05 |
| **Handle node failures on write** | **Hinted handoff + sloppy quorum** (a temp node holds the write) | m05 |
| **Converge divergent replicas** | **Read repair + anti-entropy** (Merkle trees) | m05 |
| **Detect concurrent/conflicting writes** | **Versioning with vector clocks** | m09 |
| **Decentralized membership / failure detect** | **Gossip protocol** | m09 |

**How a `put` works:** hash the key → find its position on the **consistent-hash ring** → the **N**
nodes clockwise are its replicas → the coordinator writes to them and waits for **W** acks (sloppy quorum
+ hinted handoff if some are down) → ack the client. **How a `get` works:** read from **R** replicas;
if they disagree, **read repair** writes back the latest; if there are **concurrent (conflicting)
versions** (detected via **vector clocks**), return them all for the app to **reconcile**.

```
   Consistent-hash ring (with virtual nodes):
        key "cart:42" → hash → lands here → replicated on next N=3 nodes clockwise
        put: write to W of N (sloppy quorum + hinted handoff if a node is down)
        get: read from R of N → read-repair stale; vector clocks flag conflicts → app merges
```

## A3 — Conflict resolution (the cost of "always writable") ⭐
Because Dynamo accepts writes even during partitions, **two clients can concurrently write the same key
on different sides** → **conflicting versions**. **Vector clocks** (m09) detect whether one version
**descends from** the other (keep the newer) or they're **concurrent** (a real conflict):
- **Last-Write-Wins (LWW):** keep the latest timestamp — simple but **loses data** (and trusts clocks,
  m09). Cassandra defaults to LWW.
- **Application reconciliation:** return both versions; the **app merges** (Amazon's cart **unions** the
  items → "never lose an add-to-cart"). Semantic merge, no data loss.
- **CRDTs** (m05/m09): use data types that **merge automatically** (a counter, an OR-set) → no conflict.

> **The senior framing:** *"Dynamo trades consistency for availability — it's always writable, so it
> **pushes conflict resolution to read time**, detected with **vector clocks** and resolved by **LWW
> (lossy), app-merge (semantic), or CRDTs (automatic)**. The whole thing is consistent hashing +
> replication + quorums + gossip + hinted handoff — the m05/m09 toolkit assembled."*

## A4 — DynamoDB vs the Dynamo paper (real-world note)
Amazon's **DynamoDB** (the product) evolved from the Dynamo paper but differs: it's a **managed**
service, uses a **partition-key + sort-key** model, offers **both** eventually-consistent (cheap, fast)
and **strongly-consistent reads** (it added strong consistency), and handles partitioning/replication for
you. **Cassandra** is the closest open-source descendant of the original paper (consistent hashing,
tunable quorums, LWW). Know the distinction: "the Dynamo *paper*" (AP, vector clocks) vs "DynamoDB the
*product*" (managed, tunable, strong reads available).

---

# PART B — Object / Blob Store (S3-style) ⭐

## B1 — Requirements
**Functional:** `PUT(bucket, key, bytes)` / `GET(bucket, key)` for **large immutable objects** (images,
videos, backups, model artifacts) — organized in **buckets**, retrieved by key.
**Non-functional:**
- **Extreme durability** (**"11 nines"** — never lose data) — *the* priority, even above availability.
- **Scalable to exabytes**, **cheap** (cost per GB matters at this scale).
- **Objects are immutable** (write-once; new version on change) — which **simplifies** everything.
- High **read throughput** (often via CDN).

## B2 — Architecture: separate metadata from data ⭐
The key structural move (same "split metadata from blobs" as m04/m11/m17):
```
  client ─PUT─▶ API/LB ─▶ ┌─ Metadata service ──▶ Metadata store (KV/DB): key → {location, size, hash,
                          │                          version, ...}   (tiny, queryable, the Part-A KV store!)
                          └─ Data path ─▶ Storage nodes: the actual bytes (chunked, replicated/erasure-coded)
  GET: look up metadata → fetch bytes from storage nodes (→ often via CDN)
```
- **Metadata** (object name → where the bytes live, size, checksum, version) is **small and queryable**
  → a **distributed KV store** (literally Part A, or a sharded DB).
- **Data** (the actual bytes, possibly TBs) → **storage nodes**, chunked for large objects.
- **Multipart upload:** large objects are uploaded in **parts** (parallel, resumable) and assembled —
  essential for multi-GB files.

## B3 — Durability: replication vs erasure coding ⭐⭐ (the cost insight)
How do you guarantee 11 nines without paying 3× for storage everywhere?
- **Replication (e.g. 3×):** store 3 full copies across racks/AZs. Simple, fast reads, but **200% storage
  overhead** (3× the cost).
- **Erasure coding (Reed-Solomon):** split an object into **k data shards** + **m parity shards**; you
  can **reconstruct the original from *any* k of the (k+m) shards** → survive up to **m** failures. E.g.
  **(10+4):** 10 data + 4 parity, survive any 4 losses, with only **40% overhead** (vs 200% for 3×) — and
  *higher* durability. The cost: **CPU to encode/decode** and **more network** to reconstruct on read of a
  lost shard (so it's higher-latency).

> **The senior trade-off:** *"For hot/small data I'd **replicate** (simple, low-latency). For cold/large
> data (the bulk of an object store) I'd use **erasure coding** — it gives the same-or-better durability
> at **~40% overhead instead of 200%**, trading CPU + reconstruction latency for a huge storage cost
> saving. That's why object stores like S3 use erasure coding."* Quote the **40% vs 200%** — it's the
> killer number.

## B4 — Other object-store concerns
- **Immutability simplifies consistency:** objects are write-once → no in-place update → no "consistency
  during update" problem (m04). Updates = a **new version**. (Modern S3 is **read-after-write strongly
  consistent** for new objects.)
- **Integrity:** store a **checksum** (e.g. MD5/ETag) per object; verify on read; **scrub** in the
  background to detect/repair bit-rot (re-create from replicas/parity).
- **CDN for delivery** (m06): hot objects served from the edge — versioned URLs avoid invalidation.
- **Tiering/lifecycle:** move cold objects to cheaper, slower tiers (S3 Glacier) automatically.
- **Distribution:** placement across racks/AZs/regions for durability + availability; consistent hashing
  to map objects→storage nodes.

---

## Step — Bottlenecks & themes (both halves)
- **KV store:** hot keys (m05/m06 — replicate/cache); quorum tuning per workload; conflict resolution
  strategy; gossip convergence time; node add/remove rebalancing (consistent hashing minimizes it).
- **Object store:** durability via replication vs erasure coding (cost); metadata store scaling (it's a
  KV/DB at huge scale → shard it); large-object upload (multipart); bit-rot (scrubbing); read latency for
  erasure-coded data (CDN + caching).
- **Observability (m10):** durability/replication health, quorum latency, conflict rate, scrub repairs,
  per-tier storage cost.

**Wrap-up:** *"A distributed **KV store** is the Dynamo toolkit assembled — **consistent hashing +
virtual nodes** to partition, **N-way replication** for durability, **W+R>N quorums** for tunable
consistency, **vector clocks** to detect conflicts (resolved by LWW/app-merge/CRDT), and **gossip +
hinted handoff** for decentralized, always-available operation. An **object store** layers on
**metadata/data separation** (metadata in a KV store, bytes on storage nodes), **erasure coding** for
~40%-overhead 11-nines durability (vs 200% for 3× replication), **immutability** (write-once + versioning)
to simplify consistency, **multipart upload**, **checksums + scrubbing**, and **CDN** delivery. The
unifying theme: distribute, replicate, and dial consistency to the need."*

---

## State-of-the-art & real-world notes 📚
- **The Amazon Dynamo paper (2007)** — the source of consistent-hashing + quorums + vector clocks +
  gossip + hinted handoff (Part A). **Cassandra** = closest open-source heir; **DynamoDB** = the managed
  product (with strong-read option). **Riak** = another Dynamo descendant.
- **Amazon S3** — the object-store reference (11 nines durability, erasure coding, versioning, lifecycle
  tiers, now strongly consistent). **Google Colossus / GFS**, **Ceph**, **MinIO**, **HDFS** — related
  distributed stores. **Reed-Solomon** is the standard erasure code.
- **Facebook f4 / Haystack** — real photo/blob stores using erasure coding to cut storage cost.

---

## From your systems 🏭
- **MinIO = an object store you ran:** DCE stored **model artifacts (pkl/sav) in MinIO** — an S3-compatible
  object store → you've used Part B's exact pattern (write-once blobs by key, separate from the
  metadata/claims DB). You can speak to "why object storage for model files, not a DB."
- **The m05/m09 assembly is your foundation:** consistent hashing (Redis Cluster slots), replication
  (Mongo replica sets), quorums (Mongo write concern / read concern), failover/gossip-style election —
  you've operated systems that *use* these Dynamo primitives; this module names them as one design.
- **Metadata/data split, lived:** DCE keeps **claims (data) in Mongo** and **model artifacts (blobs) in
  MinIO** — the m04/m18 "metadata in a DB, blobs in object storage" separation in production.
- **Tunable consistency:** Mongo's read/write concerns are the **W/R quorum dial** — you've chosen it per
  operation (strong for critical, relaxed for analytics).

---

## Key concepts (interview-ready)
- **A distributed KV store (Dynamo) = m05 + m09 assembled:** **consistent hashing + vnodes** (partition),
  **N-replication** (durability), **W+R>N quorums** (tunable consistency), **vector clocks** (detect
  conflicts), **gossip + hinted handoff + sloppy quorum** (decentralized, always-available), **read repair
  + anti-entropy/Merkle trees** (convergence). **AP, always-writable.**
- **Conflict resolution** (cost of always-writable): **LWW** (lossy, clock-trusting), **app-merge**
  (semantic — Amazon cart unions items), or **CRDTs** (automatic). Conflicts surfaced at **read time** via
  vector clocks.
- **Dynamo paper (AP, vector clocks) ≠ DynamoDB product (managed, partition+sort key, strong-read
  option).** Cassandra ≈ the open-source paper.
- **Object store (S3):** **separate metadata (KV/DB) from data (storage nodes)**; **objects immutable**
  (write-once + versioning → simple consistency); **multipart upload**; **checksums + scrubbing** for
  integrity; **CDN** delivery; lifecycle tiering.
- **Durability — erasure coding vs replication:** 3× replication = **200% overhead**; **erasure coding
  (k+m, e.g. 10+4)** = reconstruct from any k shards, survive m failures, **~40% overhead** (cheaper,
  same-or-better durability) at the cost of **CPU + reconstruction latency**. The killer cost number.
- **Theme:** distribute, replicate, and **dial consistency/durability to the requirement** — durability
  is the object store's #1 NFR (11 nines).

---

## Go deeper (reading)
- **The Amazon Dynamo paper (DeCandia et al., 2007)** — read it; this module *is* that paper. Plus
  Kleppmann **DDIA Ch. 5–6** (replication/partitioning) and **Ch. 9** (vector clocks/consistency).
- **Amazon S3 design talks / "Building and operating a pretty big storage system"** — durability,
  erasure coding, scrubbing.
- **Facebook f4 paper ("Warm BLOB storage")** — erasure coding to cut blob-storage cost.
- **Reed-Solomon erasure coding** explainers — the k+m math.
- Revisit **m05** (consistent hashing/quorums/replication), **m09** (vector clocks/gossip/CRDTs), **m04/
  m06** (metadata-vs-blobs, CDN).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m18_object_kv_store/`](../../solutions/m18_object_kv_store/README.md).

When you've done the exercises, say **"Module 19"** to design *Video Streaming (YouTube/Netflix)* — the
upload→transcode pipeline, adaptive bitrate, and CDN at massive scale (where this object store holds the
video segments).
