# m06 — Cross-Questions ("if-and-buts") on Caching & CDNs

> Answer out loud in 2–3 sentences before reading the model answer. Caching follow-ups are where
> people reveal whether they've actually *operated* a cache (the failure modes) or just heard of one.

---

### Q1. Why cache at all — what does it actually buy you?
**A.** Two things: **lower latency** (serve from RAM/nearby Redis ~sub-ms instead of a DB query in ms
or a cross-continent trip in ~150 ms — moving hot data up the m02 ladder) and **reduced load on the
expensive tier** (every cache hit is a DB query you didn't run, protecting the hard-to-scale database
with the cheap-to-scale cache). It works because traffic is **skewed** — a small hot set gets most of
the reads.

---

### Q2. Your cache hit ratio is 90%. Is improving it to 99% worth the effort?
**A.** Hugely — it's **non-linear**. Misses are what hit the DB, and going 90%→99% drops the miss rate
from 10% to 1%, i.e. **10× fewer DB queries**. The DB-load reduction near the top is enormous, which is
also why a **cache outage** (hit ratio → 0) can mean **100×+ sudden DB load** and cascade into an
outage. So I always track hit ratio and treat the cache as load-bearing infrastructure, not a nice-to-
have.

---

### Q3. "Just add a cache" — which cache? Walk me through the tiers.
**A.** Caches exist at every layer: **browser** (per-user, HTTP headers), **CDN** (edge, near users),
**app-local in-process** (fastest server-side, but per-server and inconsistent across servers),
**distributed cache like Redis** (shared, consistent across servers, a network hop), and the **DB
buffer pool**. I'd name *which* tier solves the problem — e.g. static assets → CDN/browser; shared hot
data across app servers → Redis; the very hottest keys → a small local tier in front of Redis.

---

### Q4. Local (in-process) cache vs distributed cache — trade-offs?
**A.** **Local** is fastest (no network hop) but **per-server**, so servers can disagree (cross-server
inconsistency), it duplicates memory, and it's lost on restart. **Distributed** (Redis) is **shared and
consistent** across all servers, survives restarts, and scales independently — at the cost of a network
round trip and another system to operate. Many systems use **both** (a near-cache: tiny local L1 +
shared Redis L2) — local for the hottest keys, Redis for the rest.

---

### Q5. Explain cache-aside vs read-through vs write-through vs write-back.
**A.** **Cache-aside** (most common): the app checks the cache, on miss reads the DB and populates the
cache — simple, memory-efficient, cache failure ≠ outage, but first read misses and you must invalidate
on writes. **Read-through:** the cache library loads from the DB on miss transparently (app only talks
to cache). **Write-through:** write to cache + DB synchronously → cache always fresh, but slower writes
and caches unread data. **Write-back:** write to cache only, flush to DB async → very fast writes but
**data-loss risk** if the cache dies before flushing.

---

### Q6. When would you use write-back despite the data-loss risk?
**A.** When **write throughput matters more than durability of each write** and some loss is
tolerable: high-volume **counters, metrics, view counts, analytics events** — where batching async
flushes to the DB gives huge write performance and an occasional lost increment is acceptable. I'd
mitigate with a **durable cache** (Redis AOF/replication) or periodic checkpoints. For money/inventory I
would **never** use write-back — the temporary "cache is the source of truth" window is unacceptable.

---

### Q7. Eviction vs invalidation — what's the difference?
**A.** **Eviction** is "I'm out of memory, drop *something*" — governed by a policy (LRU/LFU/TTL) and
about **capacity**. **Invalidation** is "this data **changed**, so the cached copy is now **wrong** —
remove it" — about **correctness**. People conflate them, but they're different jobs: a cache evicts to
free space and invalidates to stay correct. TTL straddles both (bounds staleness *and* frees memory).

---

### Q8. LRU vs LFU — when each?
**A.** **LRU** (evict least-recently-used) is the default — it matches **temporal locality** (recently
accessed → likely accessed again) and is cheap to approximate; good for shifting hot sets. **LFU**
(evict least-frequently-used) is better when popularity is **stable over time** (keeps long-term hot
items and resists a burst of one-off accesses evicting them), but it needs frequency tracking and can
cling to once-popular-now-cold items. Modern caches use refinements (W-TinyLFU, segmented LRU) that
blend both.

---

### Q9. Why is cache invalidation called one of the hardest problems?
**A.** Because the moment you cache a copy, the **source of truth can change** and your copy silently
goes **stale** — and knowing *exactly when and where* to invalidate, across many cache tiers and
servers, without races, is genuinely hard. Invalidate-on-write is the **dual-write problem** (DB write +
cache delete can partially fail); TTL bounds staleness but trades freshness for load; and a cache turns
a strongly-consistent DB into eventually-consistent reads. There's no clean general solution, only
trade-offs.

---

### Q10. On a write, should you update the cache or delete (invalidate) it?
**A.** **Prefer delete (invalidate) + lazy reload** over update-in-place. Updating the cache directly
risks **races** (two concurrent writers updating the cache in the wrong order → the cache ends up with a
stale value even though the DB is correct) and caches data that may never be read. Deleting is simpler
and safe: the next read repopulates from the DB. "Invalidate, don't update" is the common rule (with the
caveat that the delete itself can fail — the dual-write hazard, ideally handled via CDC/outbox, m04/m08).

---

### Q11. How do you keep a cache and the database consistent?
**A.** Honestly, you keep them **eventually** consistent and bound the window — perfect consistency
between two systems isn't free (dual-write problem). Options: **TTL** (accept staleness up to the TTL),
**invalidate-on-write** (delete the key when the DB changes; fresher but can partially fail),
**write-through** (update both synchronously), or for strong guarantees, **CDC/outbox** (tail the DB log
to update the cache reliably, m04/m08). For data that truly needs strong consistency, I either don't
cache it or read past the cache for that operation.

---

### Q12. What is a thundering herd / cache stampede, and how do you prevent it?
**A.** When a **hot key expires** (or the cache restarts), thousands of concurrent requests all **miss
at once** and hit the DB with the same expensive query simultaneously → the DB overloads → an outage
*caused by* the cache. Fixes: **single-flight/locking** (only the first miss recomputes, others wait for
that result → one DB hit), **stale-while-revalidate** (serve the stale value while one background task
refreshes), **probabilistic early expiration** (refresh hot keys slightly before expiry), and
**jittered TTLs** so keys don't expire in lockstep.

---

### Q13. What's cache penetration and how is it different from a stampede?
**A.** **Penetration** is requests for keys that **don't exist anywhere** (random/invalid IDs, or
malicious probing) — they always miss the cache *and* the DB finds nothing, so the cache provides zero
protection and the DB takes every hit. (A stampede is about a *real* key expiring; penetration is about
*non-existent* keys.) Fixes: **negative caching** (cache the "not found" with a short TTL), a **Bloom
filter** of existing keys to reject definitely-absent lookups before touching the DB, and input
validation/rate limiting.

---

### Q14. What's a cache avalanche?
**A.** When **many keys expire at the same moment** (e.g. you bulk-loaded them all with the same TTL) or
the **cache cluster goes down**, so a flood of misses hits the DB simultaneously and overwhelms it.
Fixes: **jittered/randomized TTLs** so expirations spread out, **cache HA/replication** so the cluster
doesn't fully drop, **circuit breakers / load shedding** (m10) to protect the DB during a cache outage,
and **warming** the cache before traffic arrives.

---

### Q15. One key (a viral post) is overwhelming a single cache node. What do you do?
**A.** That's the **hot-key** problem — you can't shard a single key, so all its traffic lands on one
node (same celebrity problem as m05). Fixes: **replicate the hot key** across multiple cache nodes and
read from any; add a **local in-process cache tier** so each app server serves the hot item without
hitting the shared cache; or **push it to the CDN** if it's cacheable. Detect it first (hot-key metrics)
so you special-case only what's actually hot.

---

### Q16. When should you NOT cache something?
**A.** When caching adds risk without payoff: **write-heavy / low read-reuse** (nothing to re-serve),
**low locality** (every key read once → ~0% hit ratio, pure overhead + a consistency problem), **strong-
consistency data** (balances, inventory, permissions — the staleness window is unacceptable), or data
that's **already cheap** to fetch. For strong-consistency data I either skip the cache, use a very short
TTL + write-through, or bypass the cache for the critical read.

---

### Q17. What is a CDN and when do you use one?
**A.** A **Content Delivery Network** is a global mesh of **edge servers (PoPs)** that cache content
**near users**, so requests are served in ~ms from a nearby edge instead of a slow trip to a distant
origin — it's caching applied to **geography** (you can't beat the speed of light, m03, so move the data
to the user). Use it for **static assets** (JS/CSS/images), **video/large files**, and increasingly
**dynamic/edge-computed** content; it also gives **DDoS protection, TLS termination, and origin
offload**. Routing to the nearest edge uses anycast/GeoDNS.

---

### Q18. Push vs pull CDN?
**A.** **Pull** (the default): the CDN fetches from your origin on the **first request** for a piece of
content (cache-aside at global scale), then serves the cached copy to later users in that region — easy,
but the first user per region pays the origin miss. **Push:** you **proactively upload** content to the
CDN ahead of demand — good for large, known-in-advance content (a video release, a game patch) to avoid
any first-hit penalty, at the cost of managing what's stored where.

---

### Q19. How do you invalidate content on a CDN?
**A.** Two ways. **Explicit purge:** tell the CDN to evict a URL/path — but global purges are slow and
error-prone. **Far better: versioned/content-addressed URLs** (`app.9f3a.css`, `image?v=7`) — when
content changes, the **filename/URL changes**, so it's a brand-new cache key and the old one just ages
out; you **never invalidate**. So I design static assets with content hashes in the URL and reserve
purging for the rare cases versioning can't cover.

---

### Q20. What does `Cache-Control: no-cache` actually mean?
**A.** **Not** "don't cache" — it means "you *may* store it, but **revalidate with the origin before
using it**" (via `ETag`/`If-None-Match`, getting a cheap **304 Not Modified** if unchanged). The "don't
store at all" directive is **`no-store`** (for sensitive data). Other key ones: `max-age` (the TTL),
`public` (shared caches/CDN may store) vs `private` (browser only), `immutable` (never revalidate). The
`no-cache` vs `no-store` confusion is a classic interview trap.

---

### Q21. How do ETag / Last-Modified reduce load even when content might be stale?
**A.** They enable **conditional requests (revalidation)**: the client sends `If-None-Match: <etag>` (or
`If-Modified-Since`); if the resource is unchanged the server replies **304 Not Modified** with **no
body** — so the client reuses its cached copy and you save the **bandwidth** of re-sending it, while
still guaranteeing freshness. It's a middle ground between "cache blindly for max-age" and "refetch
everything" — cheap freshness checks instead of full transfers.

---

### Q22. Redis vs Memcached — which and why?
**A.** I default to **Redis**: it's a rich **data-structure server** (strings, hashes, lists, sets,
**sorted sets**, streams, pub/sub) with **TTLs, optional persistence, and HA/replication (Sentinel/
Cluster)** — so it does far more than caching (rate-limit counters, leaderboards, queues, sessions,
pub/sub fan-out). **Memcached** is a simpler, **multi-threaded**, pure key→value cache that can be
marginally better for plain large-scale caching with no extra features. Unless I specifically want
Memcached's simple multi-threaded model, Redis wins on versatility.

---

### Q23. Does putting a cache in front of a strongly-consistent database weaken consistency?
**A.** Yes — it makes reads **eventually consistent** within the TTL/invalidation window, because the
cache can serve a value that's already been changed in the DB. That's usually a fine trade for read-
heavy data, but for anything needing strong consistency (account balance, inventory count, permission
checks) I either **don't cache it**, use a **very short TTL + write-through**, or **bypass the cache**
for that specific read. The key insight: **adding a cache is a consistency decision, not just a
performance one** — it's the same staleness trade as denormalization (m04).

---

### Q24. Your DB is getting hammered. Before sharding it, what cheaper things do you try?
**A.** Caching levers first (cheaper than resharding): add/grow a **distributed cache (Redis)** in front
of hot reads, push static/cacheable content to a **CDN**, add **read replicas** (m05) to spread read
load, and check for **N+1 queries / missing indexes** (m04) that inflate DB work. Quantify with the
**hit ratio** — often a good cache + replicas removes the need to shard entirely (sharding is the last,
most painful resort because the shard key is hard to change, m05). Only shard when **writes** or
**dataset size** genuinely exceed one primary.
