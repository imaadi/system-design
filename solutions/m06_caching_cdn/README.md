# m06 — Solutions (Caching & CDNs)

> Check the *reasoning*. Recurring lessons: hit ratio is the metric (and non-linear); invalidation is
> the hard part; every cache failure mode = the DB shield dropping; versioned URLs beat purging.

---

## Exercise 1 — Hit-ratio math
1. 50,000 reads/sec × miss rate: **95% hit → 5% miss = 2,500 DB q/s**; **99% → 1% = 500 q/s**;
   **99.9% → 0.1% = 50 q/s**. (Each +"nine" of hit ratio = ~10× fewer DB queries.)
2. Cache down → **100% miss = 50,000 DB q/s** — a **100× jump** vs the 99% case (500 → 50,000). The DB
   almost certainly can't take that → cascading outage.
3. Treat the cache as **load-bearing, critical infrastructure** (not optional): make it **HA/
   replicated**, add **circuit breakers/load shedding** to protect the DB if it drops, **warm** it
   before traffic, and alert on hit-ratio drops. A cache you can't afford to lose must be engineered
   like any other critical tier.

---

## Exercise 2 — Pick the tier
1. **CDN (+ browser cache)** — static, global; serve from the edge with a long max-age + versioned URL.
2. **App-local in-process cache** — read on every request by every server; tiny, rarely-changing →
   local avoids a network hop. (Accept brief cross-server staleness, or use a short TTL / pub-sub
   invalidation.)
3. **Distributed Redis** — must be **shared and consistent** across all scoring workers; sub-ms; this is
   exactly DCE's pattern.
4. **Browser cache** (private) — only that user sees it; no need to occupy shared server memory.
5. **App-local tier in front of Redis** (near-cache) for the 10 hottest keys — protects the single Redis
   node from a hot-key meltdown by serving them from each app server's memory.

---

## Exercise 3 — Choose the pattern
1. **Cache-aside read + invalidate-on-write (or write-through)** — read-heavy, occasional updates, few
   seconds stale OK; cache-aside is simplest and TTL bounds staleness.
2. **Write-back** — millions of increments/sec, occasional lost increment acceptable → batch async
   flushes to the DB for throughput (use a counter in Redis). Write-through would bottleneck on DB writes.
3. **Write-through (or warm/preload on import)** — must be fresh immediately after import, so populate
   the cache as part of the write so the first read is a hit and never stale.
4. **Don't cache the balance for the critical read** (or cache-aside with a **very short TTL +
   write-through**, and read past the cache for the actual transfer) — strong consistency required; a
   stale balance enables double-spend.

---

## Exercise 4 — Invalidate or update?
1. **Delete the key** (invalidate), don't update in place. Race: two concurrent writers update the cache
   in a different order than the DB (writer A's DB commit lands last but its cache write lands first) →
   the cache ends up with a **stale value** while the DB is correct. Deleting avoids this — the next read
   repopulates from the (correct) DB.
2. **Divergence:** DB write succeeds, then the **cache delete fails** (network blip / crash between the
   two) → cache serves the old value until TTL. Underlying problem: the **dual-write problem** (m04) —
   two systems, no shared transaction.
3. **TTL** — even if invalidation has a bug, every entry expires after N seconds, so staleness is
   **bounded** and the cache self-heals. (TTL is the safety net under any invalidation scheme.)

---

## Exercise 5 — Name the failure mode
1. **Hot key** → replicate the key across nodes / add a local near-cache tier / CDN it.
2. **Cache penetration** → negative caching of "not found" + a Bloom filter of existing ids + input
   validation/rate limiting.
3. **Cache avalanche** (mass simultaneous expiry) → **jittered/randomized TTLs** so they don't expire in
   lockstep (+ HA, load shedding).
4. **Stampede / thundering herd** → **single-flight/locking** (one recompute) or **stale-while-
   revalidate**.

---

## Exercise 6 — Kill the stampede
1. **Single-flight (request coalescing):** on a miss, the **first** request acquires a per-key lock and
   recomputes; concurrent requests for the same key **wait** for that in-flight result instead of each
   hitting the DB → exactly **one** DB query regardless of how many concurrent misses.
2. **Stale-while-revalidate:** keep serving the **stale** cached value immediately (users never block)
   while **one** background task refreshes the entry; once refreshed, new reads get the fresh value. No
   user waits and the DB gets a single refresh.
3. **Jittered TTL** adds randomness to expiry times so many keys don't expire at the same instant
   (addresses **avalanche**/synchronized expiry). **Probabilistic early refresh** recomputes a hot key a
   little *before* it expires (randomly), so it's refreshed by one request ahead of the mass-miss
   (addresses the single-hot-key **stampede**).

---

## Exercise 7 — CDN design
1. **JS/CSS/images and product videos → CDN** (static/large, global, cacheable). **Avatars → CDN** too
   (static blobs). Dynamic/personalized API responses generally stay closer to origin (or edge-computed).
2. **Content-addressed/versioned filenames** — `app.[contenthash].css`. A new deploy produces a new
   filename → a brand-new cache key, so users fetch the new one and the old just ages out; **no global
   purge needed.**
3. **Push** for launch videos — you know they'll be in huge demand immediately, so pre-load them to
   edges to avoid every region's first user paying an origin miss (a pull CDN would cold-miss per region
   at launch).
4. Beyond latency: **origin offload** (CDN absorbs most traffic), **DDoS protection**, **TLS termination
   at the edge**, improved **availability**.

---

## Exercise 8 — HTTP caching headers
1. `Cache-Control: public, max-age=31536000, immutable` — the content hash guarantees the URL changes
   when content changes, so it's safe to cache **forever** and never revalidate.
2. `Cache-Control: private, no-store` (or at least `private, no-cache`) — private so shared CDNs don't
   store it; `no-store` for sensitive personal data so it isn't persisted anywhere.
3. The client sends **`If-None-Match: <etag>`** (or `If-Modified-Since`); if unchanged the server returns
   **304 Not Modified** with no body, so the client reuses its cached copy (saves bandwidth).
4. **`no-cache`** = you may store it but must **revalidate** with the origin before using it.
   **`no-store`** = don't store it **at all**, anywhere.

---

## Exercise 9 — When NOT to cache
1. **No** — write-heavy + almost never re-read → ~0% hit ratio, pure overhead and a consistency liability.
2. **Mostly no** — near-unique queries = no locality = near-0 hit ratio (you might cache *popular* queries
   only, or cache at a different layer like the search index itself).
3. **No (or bypass for this read)** — strong consistency required; a stale balance on a transfer screen
   could allow double-spend. Read from the source of truth.
4. **Yes** — read 100× more than written = strong locality + read-heavy → ideal cache candidate
   (cache-aside + invalidate/TTL).

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **DCE Redis reference data = cache-aside (a shared distributed cache).** A **Redis flush** (or cluster
   loss) → **cache avalanche**: every scoring request misses and hits the backing store at once. Protect
   the DB with: **HA/replicated Redis**, **jittered TTLs / staggered warming** after a flush, **circuit
   breakers/load shedding** during a cache outage, and pre-**warming** critical reference keys before
   processing resumes.
2. **Snapshot tables = a materialized cache** of an expensive join+compute. **Cached:** precomputed risk
   decomposition per portfolio. **"TTL"/refresh:** the nightly (or scheduled) pipeline recompute.
   **Risk:** the snapshot is **stale between refreshes** (a write/consistency window), and if the refresh
   pipeline fails, reads serve **stale risk numbers** — the classic denormalized-copy staleness trade.
3. **Redis-backed JWT sessions = caching auth state** for fast, stateless verification (no DB hit per
   request). **Invalidation becomes a feature:** deleting the Redis token key **instantly revokes** the
   session — you get stateless JWT performance *plus* server-side, immediate invalidation, which pure
   stateless JWTs can't do.
4. **Order of levers** (cheapest/least-disruptive first): **(1) add/grow Redis cache** for hot reads
   (biggest win for read-heavy, smallest change), **(2) add read replicas** to spread remaining read load
   (mind read-your-writes), **(3) add a CDN** for any static/cacheable content, **(4) shard Postgres**
   *last* — only if writes or dataset size outgrow one primary, since the shard key is hard to change
   (m05). For a read-heavy analytics product, steps 1–2 usually suffice without sharding.
