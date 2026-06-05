# Module 6 — Caching & CDNs ⭐⭐

> Caching is the **highest-leverage** performance tool in system design: it attacks the slow rungs of
> the m02 latency ladder (memory ≪ disk ≪ network ≪ another continent) by keeping hot data **close**.
> It's also where careless designs blow up — a cache can *cause* an outage (thundering herd) and
> silently serve wrong data (stale reads). This module makes you fluent in **where** caches live,
> **which pattern** to use, the **eviction/invalidation** trade-offs, the **four classic failure
> modes**, and **CDNs/HTTP caching** — and exactly **when not to cache.**

You already do this for real: Redis caching + snapshot tables (Questimate), Redis hot reference data
during scoring (DCE). This turns that into architect-level reasoning.

---

## 1. Why cache — and the hit-ratio math that makes it worth it ⭐

A **cache** is a fast, usually in-memory store that holds a **copy** of data that's **expensive to
fetch or compute**, so repeat requests are served cheaply. Two payoffs:
1. **Latency:** RAM (~100 ns) / a nearby Redis (~0.5 ms) vs a DB query (ms) vs cross-continent
   (~150 ms). You move hot data **up the ladder** (m02 §9).
2. **Load / cost:** every cache hit is a request your database/origin **didn't** serve → you protect
   the expensive, hard-to-scale tier (the DB) with the cheap, easy-to-scale tier (the cache).

**Why it works: locality.** Real traffic is skewed (Zipf/Pareto) — a small set of "hot" items get most
of the requests (the top videos, the popular products). Caching that small hot set captures most reads.

> **Think differently — the hit-ratio leverage is non-linear.** What matters is the **miss rate**,
> because misses hit the DB. Going from a **90% → 99%** hit ratio sounds like "9% better" but it's
> **10× fewer DB queries** (misses drop from 10% to 1%). 99% → 99.9% is *another* 10×. So small
> hit-ratio gains near the top are enormous DB-load reductions — and a **cache outage** (hit ratio →
> 0%) can mean **100×+ sudden DB load**, which is why cache failures cascade (§6). Always quote the
> hit ratio; it's the metric.

---

## 2. The cache hierarchy — caches live at every layer ⭐

"Add a cache" is ambiguous — *which* cache? Data is cached at many tiers between user and DB, each a
trade of proximity vs freshness/shared-ness:

```
  user ─▶ ┌──────────────┐ ─▶ ┌──────┐ ─▶ ┌──────────────┐ ─▶ ┌──────────────┐ ─▶ ┌────────┐
          │ Browser cache │    │ CDN  │    │ App-local     │    │ Distributed  │    │   DB    │
          │ (per user)    │    │(edge)│    │ in-mem cache  │    │ cache (Redis)│    │ + buffer│
          └──────────────┘    └──────┘    │ (per server)  │    │ (shared)     │    │  pool   │
                                          └──────────────┘    └──────────────┘    └────────┘
   closer/faster ◀────────────────────────────────────────────────────────────▶ fresher/shared
```

- **Client/browser cache:** fastest (no network), per-user, governed by HTTP headers (§8). Great for
  static assets and a user's own data.
- **CDN (edge):** caches content in PoPs near users (§7) — beats the speed of light for static/global
  content.
- **Application-local (in-process) cache:** an in-memory map inside each app server (e.g. Caffeine,
  Guava, a Python dict/LRU). **Fastest server-side** (no network hop), but **per-server** → not
  shared, can be **inconsistent** across servers, lost on restart, and limited by one box's RAM.
- **Distributed cache (Redis, Memcached):** a separate shared cache cluster all app servers hit over
  the network (~sub-ms). **Shared & consistent** across servers, survives app restarts, scales
  independently — at the cost of a network hop and an extra system to run. *This is the workhorse
  (your Redis layer).*
- **Database cache (buffer pool):** the DB's own in-memory cache of hot pages; "free" but inside the
  thing you're trying to offload.

> **The senior nuance — local vs distributed is a real trade.** Local caches are faster but cause
> **cross-server inconsistency** (server A's cache says X, server B's says Y) and **duplicate memory**.
> Distributed caches are consistent and shared but add a network hop and a dependency. Many systems use
> **both (a tiered/near cache):** a tiny local L1 in front of a shared Redis L2 — local for the very
> hottest keys, Redis for the rest. Naming this trade is a strong signal.

---

## 3. Caching patterns (read & write paths) ⭐

How the cache, app, and DB interact. Get these names right — they're asked verbatim.

### Read patterns
- **Cache-aside (lazy loading) — the most common.** The *application* manages the cache:
  ```
  read: look in cache → HIT? return it.
                       MISS? read DB → write result to cache (with TTL) → return.
  ```
  **Pros:** only requested data is cached (memory-efficient); cache failure ≠ outage (fall through to
  DB); simple. **Cons:** first read is always a miss (cold-start latency); cache+DB can drift (you must
  invalidate on writes); the **miss path is where stampedes happen** (§6).
- **Read-through.** The *cache library* sits in front of the DB and loads on miss transparently (app
  only talks to the cache). Cleaner app code; same miss/stale concerns; couples you to a cache that
  knows how to load.

### Write patterns
- **Write-through.** Write to cache **and** DB **synchronously** (cache updated on every write).
  **Pro:** cache never stale; reads always fresh. **Con:** write latency (two writes); caches data
  that may never be read (wasted memory).
- **Write-around.** Write **only to the DB**, skip the cache (cache populated lazily on the next read).
  **Pro:** doesn't pollute cache with write-only data. **Con:** a just-written item is a **cache miss**
  on first read (read-after-write latency).
- **Write-back (write-behind).** Write to **cache only**, acknowledge immediately, flush to DB
  **asynchronously** (batched). **Pro:** very fast writes, absorbs bursts, batches DB writes. **Con:**
  ⚠️ **data loss risk** — if the cache dies before flushing, those writes are gone; and the cache is
  temporarily the **source of truth**. Use only where some loss is acceptable (metrics, counters) or
  with a durable cache.

| Pattern | Write goes to | Read freshness | Risk / cost |
|---|---|---|---|
| **Cache-aside** | DB (app invalidates cache) | stale until invalidated/TTL | most common; miss-path stampede |
| **Write-through** | cache + DB (sync) | always fresh | write latency; caches unread data |
| **Write-around** | DB only | miss on first read | read-after-write latency |
| **Write-back** | cache (async to DB) | fresh in cache | **data loss if cache dies** |

> **The common default:** **cache-aside reads + write-through *or* invalidate-on-write** for a good
> balance. Pick the write pattern by how much **staleness** and **write latency** you can tolerate.

---

## 4. Eviction policies — what to drop when the cache is full

A cache has bounded memory; when full, it must **evict**. The policy decides what stays:
- **LRU (Least Recently Used):** evict what hasn't been accessed longest. **The default** — matches
  temporal locality (recently used → likely used again). Cheap to approximate.
- **LFU (Least Frequently Used):** evict the least-*often* accessed. Better when popularity is stable
  (keeps long-term hot items), but needs frequency tracking and can keep stale-but-once-popular items.
- **FIFO:** evict oldest-inserted regardless of use. Simple, usually worse than LRU.
- **TTL (time-based):** evict on expiry (orthogonal — combine with LRU/LFU). The main **freshness**
  control.
- **Random / segmented LRU / W-TinyLFU:** practical refinements (Redis uses approximated LRU/LFU;
  Caffeine uses W-TinyLFU).

**Redis `maxmemory-policy`** options you can name: `allkeys-lru`, `allkeys-lfu`, `volatile-lru`
(only keys with a TTL), `volatile-ttl`, `noeviction` (reject writes when full). Choosing this is a real
config decision — `noeviction` turns a full cache into write errors; `allkeys-lru` is the common pick.

> **Think differently — eviction policy ≠ invalidation.** **Eviction** = "I'm out of room, drop
> something." **Invalidation** = "this data changed, the cached copy is now *wrong*, remove it." They're
> different jobs; people conflate them. TTL does a bit of both (bounds staleness *and* frees memory).

---

## 5. Cache invalidation & consistency — the genuinely hard part ⭐⭐

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil
> Karlton. This is the real difficulty of caching.

The moment you cache a copy, the **source of truth can change** and your copy goes **stale**. The whole
problem is keeping the copy correct enough. Strategies:
- **TTL (expiration):** every entry expires after N seconds → bounded staleness, self-healing, dead
  simple. **The workhorse.** The trade: short TTL = fresher but more misses/DB load; long TTL = less
  load but staler. *TTL is a tunable freshness-vs-load dial* (same idea as DNS TTL, m03).
- **Explicit invalidation on write:** when data changes, **delete** (or update) the cached key so the
  next read repopulates. Fresher than TTL, but it's the **dual-write problem (m04)** — if the DB write
  succeeds and the cache delete fails (or a crash lands between), they diverge.
  - **Delete vs update?** Prefer **delete (invalidate), then lazy-reload** over **update-in-place** —
    updating risks races (two writers updating the cache out of order) and caches data that may not be
    read. "Invalidate, don't update" is a common rule.
- **Write-through:** keeps cache fresh by construction (§3), at write-latency cost.
- **Versioned keys / cache busting:** include a version/hash in the key (`user:42:v7`,
  `app.css?v=abc`) → a new version is a new key; old one ages out. Common for static assets and
  immutable data (no invalidation needed — the URL changes).

> **Think differently — a cache changes your consistency model.** Putting a cache in front of a
> strongly-consistent DB makes reads **eventually consistent** (within the TTL/invalidation window).
> That's usually fine — but for data that needs strong consistency (balances, inventory, permissions),
> either don't cache it, cache with very short TTL + write-through, or read past the cache for the
> critical operation. *Caching is a denormalized copy (m04) — it inherits denormalization's exact
> staleness trade.* Say this; it shows you see caching as a consistency decision, not just speed.

---

## 6. The four classic cache failure modes ⭐⭐ (these are *the* interview gold)

Caching introduces new ways to fail. Knowing these by name + fix is a top senior signal.

### (a) Cache stampede / thundering herd
A **popular** key expires (or the cache restarts). Suddenly **thousands of concurrent requests all
miss at once** and **stampede the database** with the same expensive query → the DB melts → an outage
*caused by* caching. ⚠️
**Fixes:**
- **Locking / single-flight (request coalescing):** only the **first** miss recomputes; others **wait**
  for that result (one DB hit, not thousands). (Go's `singleflight`, a per-key mutex/lock.)
- **Stale-while-revalidate:** serve the **stale** value while **one** background task refreshes it →
  users never wait, DB gets one refresh.
- **Probabilistic early expiration:** refresh a hot key *slightly before* it expires, randomly, so they
  don't all expire and miss simultaneously.
- **Jittered TTLs:** add randomness to TTLs so keys don't expire in lockstep.

### (b) Cache penetration
Requests for keys that **don't exist anywhere** (e.g. malicious lookups of random IDs, or `user:99999`
that was never created). Every such request **misses the cache and hits the DB** (which also finds
nothing) → the cache provides no protection, and an attacker can hammer your DB.
**Fixes:**
- **Negative caching:** cache the "not found" result too (with a short TTL) so repeats are absorbed.
- **Bloom filter:** a tiny probabilistic set of "keys that exist" in front of the cache — if the Bloom
  filter says "definitely not present," reject immediately without touching cache or DB.
- **Input validation / rate limiting** on the lookup.

### (c) Cache avalanche
**Many keys expire at the same time** (e.g. you set the same TTL on a bulk load), or the **cache cluster
goes down**, so a flood of misses hits the DB simultaneously → DB overload. (Like a stampede, but
across many keys at once.)
**Fixes:** **jittered/randomized TTLs** (spread expirations), **cache replication/HA** (so the cluster
doesn't fully drop), **circuit breakers / load shedding** (m10) to protect the DB during a cache
outage, and **warming** the cache before it's needed.

### (d) Hot key (ties back to m05)
A **single key** (a viral video, a celebrity, a flash-sale item) gets so much traffic it overwhelms the
**one cache node** that holds it (you can't shard a single key).
**Fixes:** **replicate** the hot key across multiple cache nodes (read from any), add a **local/
in-process cache tier** in front of the distributed cache (each app server caches the hot item), or
**CDN** it if cacheable. (Same celebrity problem as m05 §6; it recurs in the news feed, m13.)

> **The pattern across all four:** a cache is a **shield for the DB**, and these are the ways the
> shield drops (key expires, key doesn't exist, many keys expire, one key is too hot) and exposes the
> DB to a flood. Senior answers always pair the failure with its mitigation.

---

## 7. CDNs (Content Delivery Networks) ⭐

A **CDN** is a globally-distributed network of **edge servers** (in **PoPs** — Points of Presence) that
**cache content close to users**. It's caching applied to the **geography** problem: you can't beat the
speed of light (m03), so you **move the data to the user** instead of the user to the data.

**How it works:** DNS/anycast (m03) routes the user to the **nearest edge**. If the edge has the content
cached → serves it in ~ms (no trip to origin). If not → fetches from your **origin** server (or a
mid-tier **origin shield**), caches it, and serves it; subsequent users in that region get the cached
copy.

```
   user (Mumbai) ─▶ nearest CDN edge (Mumbai PoP)
                       │ cache HIT → ~5 ms  ✅
                       │ cache MISS → fetch from origin (Virginia, ~200 ms once) → cache → serve
```

**Push vs pull CDN:**
- **Pull (lazy):** CDN fetches from origin on first request (cache-aside at global scale). Default;
  easy; first user in a region pays the miss.
- **Push:** you proactively upload content to the CDN. Good for large files you know will be needed
  (a video release) — no first-hit penalty, but you manage what's there.

**What to put on a CDN:** static assets (JS/CSS/images/fonts), video segments, downloads, and
increasingly **dynamic/personalized** content via **edge compute** (Cloudflare Workers, Lambda@Edge)
and API response caching. **Cache control** uses the same HTTP headers as §8 (TTL, `Cache-Control`),
and you **purge/invalidate** the edge when content changes (or use **versioned URLs** so you never have
to — `app.[hash].js`).

**Why CDNs matter beyond latency:** they absorb huge traffic (offloading origin), provide **DDoS
protection** and TLS termination at the edge, and improve availability. For the m19 video and m18
object-store designs, the CDN is central.

> **Think differently — versioned URLs beat invalidation.** Purging a CDN globally is slow and
> error-prone. Instead, make asset URLs **content-addressed** (`style.9f3a.css`): when content changes,
> the **filename changes**, so it's a brand-new cache key — the old one just ages out and you *never
> invalidate*. This sidesteps the hard problem (§5) entirely for static assets.

---

## 8. HTTP caching headers (how the browser & CDN know what to cache)

Caching at the client/CDN tier is controlled by **HTTP response headers** — worth knowing because
they're concrete and frequently asked:
- **`Cache-Control`** — the main directive: `max-age=3600` (cache 1h), `public` (any cache may store)
  vs `private` (only the browser, not shared CDNs), `no-cache` (must revalidate before use — *not*
  "don't cache"), `no-store` (never store — for sensitive data), `immutable` (never revalidate).
- **Validation (conditional requests):** `ETag` (a content hash/version) and `Last-Modified` let the
  client ask "has it changed?" with `If-None-Match`/`If-Modified-Since`; the server replies **304 Not
  Modified** (no body) if not → saves bandwidth while staying fresh.
- **`stale-while-revalidate` / `stale-if-error`** — serve stale while refreshing in the background / if
  the origin is down (the §6 stampede fix, standardized as a header).

> The distinction interviewers probe: **`no-cache` means "revalidate every time," not "don't cache";
> `no-store` means "don't store at all."** And **`max-age` is the TTL** — the same freshness-vs-load
> dial as everywhere.

---

## 9. When NOT to cache + Redis vs Memcached

**Don't reflexively cache.** It doesn't help (and adds risk) when:
- **Write-heavy / low read-reuse** — little to re-serve; you pay invalidation cost for nothing.
- **Low locality** — every key read ~once (no hot set) → near-0 hit ratio, pure overhead.
- **Strong consistency required** — the staleness window is unacceptable (balances, inventory, auth) →
  don't cache, or cache with very short TTL + write-through, or bypass for the critical read.
- The data is **already fast** to fetch (caching adds a hop + a consistency problem for no gain).

**Redis vs Memcached** (the classic compare):
| | **Redis** | **Memcached** |
|---|---|---|
| Data model | rich: strings, **hashes, lists, sets, sorted sets**, streams, pub/sub, TTLs | simple key→value (strings/blobs) |
| Persistence | optional (RDB snapshots, AOF log) | none (pure cache) |
| Replication/HA | yes (replicas, Sentinel, Cluster) | no built-in |
| Threading | mostly single-threaded (fast, simple) | multi-threaded |
| Use when | you want data structures, persistence, pub/sub, or HA (most cases) | pure, simple, multi-threaded caching at scale |
> Default to **Redis** unless you specifically want Memcached's simple multi-threaded model — Redis's
> data structures (sorted sets for leaderboards, sets for dedup, counters for rate limits, pub/sub for
> fan-out) make it far more than a cache (m04 Q21). *This is why you used Redis for sessions, hot
> reference data, and more.*

---

## 10. From your systems 🏭
- **Distributed cache (cache-aside), lived:** DCE caches **hot reference/eligibility/auth data in
  Redis** and reads it during scoring — a sub-ms shared cache shielding the slower stores, exactly §2/§3
  cache-aside. Your `clear_redis_cache.py` tool is literally **manual invalidation** (§5).
- **Materialized cache / write-through-ish:** Questimate **precomputes results into snapshot tables for
  fast reads** — a cache of an expensive join+compute, refreshed by the pipeline. That's caching with a
  **TTL/refresh** model (and the §5 staleness trade: snapshots are stale between refreshes).
- **Session cache:** Redis-backed **JWT sessions** = caching auth state for fast, stateless verification
  with instant revocation (delete the key) — invalidation as a feature.
- **Hit-ratio leverage:** any time your Redis layer absorbs reads, it's the §1 math protecting Postgres/
  Mongo — you can frame the snapshot tables as "turning an expensive recompute into an O(1) cached read."
- **Failure-mode awareness:** a snapshot refresh or a Redis flush is a place where a **stampede/
  avalanche** could hit the DB — a great place to mention jittered refresh + warming.

---

## 11. Key concepts (interview-ready)
- **Cache = a fast copy of expensive data** → cuts **latency** + **DB load**. The metric is **hit
  ratio**, and its leverage is **non-linear** (90%→99% = 10× fewer DB hits; a cache outage = 100×+ DB
  load).
- **Caches live at every tier:** browser → CDN → app-local → distributed (Redis) → DB buffer pool.
  **Local = faster but inconsistent/per-server; distributed = shared/consistent + a network hop.**
- **Patterns:** **cache-aside** (lazy, most common) / read-through reads; **write-through** (fresh,
  slower) / **write-around** (miss on first read) / **write-back** (fast, **data-loss risk**) writes.
- **Eviction** (LRU default, LFU, TTL, Redis `maxmemory-policy`) ≠ **invalidation** (data changed →
  remove the wrong copy).
- **Invalidation is the hard problem.** **TTL** = simple bounded staleness (tunable freshness-vs-load);
  **invalidate-on-write** is fresher but is the **dual-write problem**; **prefer invalidate over
  update**; **versioned keys** sidestep it. A cache makes reads **eventually consistent** — a
  consistency decision, not just speed.
- **Four failure modes:** **stampede/thundering herd** (hot key expires → fix: single-flight/locking,
  stale-while-revalidate, jitter), **penetration** (missing keys → negative caching, Bloom filter),
  **avalanche** (mass expiry/cluster down → jittered TTL, HA, load shedding), **hot key** (one key too
  hot → replicate it / local tier / CDN).
- **CDN** = cache at the edge near users (beats the speed of light); pull vs push; **versioned URLs
  beat purging**; also DDoS/TLS/offload.
- **HTTP caching:** `Cache-Control: max-age/public/private/no-cache/no-store`, `ETag`/`Last-Modified`
  → **304**, `stale-while-revalidate`. (`no-cache` = revalidate, not "don't cache.")
- **Don't cache** write-heavy, low-locality, or strong-consistency data. **Redis** (rich + HA) is the
  default over **Memcached** (simple multi-threaded).

---

## 12. Go deeper (the well-researched reading list)
- **Kleppmann, DDIA** — caching threads through the replication/derived-data chapters; the
  "derived data must be kept in sync" framing (Ch. 11) is the deep version of §5.
- **Redis docs: "Key eviction" (`maxmemory-policy`) and "Client-side caching"** — authoritative on §4
  and tiered caching.
- **MDN: HTTP caching** (`Cache-Control`, `ETag`, conditional requests) — the source for §8.
- **"Caching at Reddit / Instagram / Facebook (memcache)" engineering posts** — e.g. Facebook's
  *"Scaling Memcache at Facebook"* paper covers stampedes (leases), avalanche, and tiered caching at
  real scale (§6). Excellent, concrete.
- **Cloudflare / Fastly docs on CDN cache-control, purge, and edge compute** — for §7.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m06_caching_cdn/`](../../solutions/m06_caching_cdn/README.md).

When you've done the exercises, say **"Module 7"** to build *Load Balancing & Proxies* (L4 vs L7,
algorithms, reverse proxy, health checks, sticky sessions, service mesh) — the piece that spreads
traffic across all the replicas we've been adding.
