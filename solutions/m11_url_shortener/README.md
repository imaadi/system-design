# m11 — Solutions (URL Shortener)

> Check the *reasoning*. Recurring lessons: read-heavy → cache is the design; unique-ID generation is the
> core deep-dive (KGS/Snowflake); mappings are immutable → caching is easy; 302 for analytics.

---

## Exercise 1 — Estimation
500M/day, 50:1, 600 B, 10 yrs, ×3.
1. **Writes:** 500M ÷ 10⁵ = **5,000/s** avg (~25k peak). **Reads:** 50× = **250,000/s** avg (~1.25M
   peak). *Implication:* reads dominate hard → CDN + cache + many read replicas; writes (5k/s) still need
   a non-bottlenecked code generator (KGS) and sharded writes.
2. **Storage/day:** 500M × 600 B = **~300 GB/day** → ×400 ×10 ≈ **~1.2 PB** → ×3 ≈ **~3.6 PB**.
   *Implication:* definitely a **sharded** KV cluster (consistent hashing), with lifecycle/expiry to
   reclaim dead links.
3. (implications inline) — reads → cache/CDN; writes → KGS + sharding; storage → shard + expiry.
4. **Codes needed:** 500M/day × 365 × 10 = **~1.8 trillion** total. **62⁷ ≈ 3.5T > 1.8T** → **7 chars
   suffices** (with headroom); 62⁶ ≈ 57B would run out, so 7 is the floor.

---

## Exercise 2 — Base62 math
1. **62⁶ ≈ 57 billion**, **62⁷ ≈ 3.5 trillion**, **62⁸ ≈ 218 trillion.**
2. **125 in base62** (digits `0-9a-zA-Z`, so 0–9=0-9, 10–35=a-z, 36–61=A-Z): 125 ÷ 62 = **2** remainder
   **1**. So digits are [2, 1] → index 2 = `'2'`, index 1 = `'1'` → **"21"**. (Check: 2×62 + 1 = 125 ✓.)
3. **base62** uses only URL-safe characters (letters+digits); **base64** uses `+`, `/`, and `=` padding,
   which must be percent-escaped in URLs and look ugly/break copy-paste.

---

## Exercise 3 — Pick a code-generation scheme
1. **KGS pre-allocated ranges** — shortest codes (sequential within ranges, base62), zero collisions, no
   per-write coordination → high throughput. (Counter alone bottlenecks/SPOFs.)
2. **Snowflake IDs** — fully distributed, no central coordinator; accepts ~11-char codes for that
   independence.
3. **Hash the URL (+ collision retry)** — same input → same hash → same code = natural dedup. (Accept the
   existence-check-per-write cost.)
4. **Counter** (or even hash) — low traffic means the bottleneck/SPOF concerns don't bite, and simplicity
   wins.

---

## Exercise 4 — The redirect path
1. GET `/code` → **DNS** → **L7 LB** → stateless **app server** → **Redis** (HIT → 302 in sub-ms). MISS →
   **DB read replica** → populate cache → 302 (404 if missing / 410 if expired). Very hot links served
   from the **CDN/edge** before reaching the app tier. Click event emitted **async to Kafka**.
2. Caching is easy because the **mapping is immutable** — a code always points to the same URL — so
   there's essentially **no cache-invalidation problem** (the hard part of caching, m06); you only handle
   deletes/expiry.
3. **Yes, it survives** — reads fall through to **DB read replicas**. Cost: a latency spike and a load
   surge (a potential **stampede**, m06). Protect the DB with **request coalescing/single-flight** on
   misses, enough read-replica capacity sized for fallback, and **cache warming** on recovery.

---

## Exercise 5 — 301 vs 302
1. **302 (temporary)** — it's **not cached** by the browser, so every click hits your server → you can
   **count it**. Analytics requires the click to reach you.
2. **301 (permanent)** — the browser **caches** it, so repeat clicks skip your server → fastest + lowest
   load (but no analytics, and you can't change the target).
3. With **301**, browsers/intermediaries **cache the redirect**, so subsequent clicks go **straight to
   the destination without contacting you** → you never observe them (can't count) and your stored
   mapping is bypassed (can't change/expire reliably).

---

## Exercise 6 — Custom aliases
1. Normalize the alias → **check it's not already taken** (uniqueness constraint, fronted by a Bloom
   filter) and not reserved/invalid → if free, write `{alias → long_url}` with a unique constraint
   (atomic insert) → return it; if taken, **reject** with a clear error (and maybe suggest alternatives).
2. Front the check with a **Bloom filter** of existing aliases/codes (m06): "definitely not present" →
   accept fast without a DB read; "maybe present" → confirm with a DB lookup. This avoids a DB hit for the
   common (available) case.
3. Keep generated codes and custom aliases in the **same key space** with a **uniqueness constraint** —
   generation checks/skips taken keys (or the insert fails and retries). One namespace + unique
   constraint = no cross-collision possible.

---

## Exercise 7 — Add analytics
1. **302** is required — only then does each click reach your server to be recorded.
2. **Pipeline:** click → app emits a **click event** (code, ts, referrer, IP→geo) to **Kafka** →
   **stream processor** aggregates (counts, by time/geo/referrer) → writes to a **columnar/time-series
   store** → **dashboard/stats API** reads it. (A simple total can be an async Redis `INCR`.)
3. Because the redirect is **latency-critical (100k/s)** — doing analytics synchronously would add a
   write to the hot path and couple redirect availability to the analytics store. Emit-and-forget keeps
   redirects sub-ms and lets analytics scale/fail independently (m08).

---

## Exercise 8 — Scale 10× & global
1. The **read path / cache + DB read capacity** saturates first (reads dominate). Scale it with **more
   CDN/edge coverage, a bigger Redis cluster (sharded), and more DB read replicas**; the write path
   (KGS + sharded writes) scales separately.
2. Serve from a **global CDN/edge** — anycast/GeoDNS routes the São Paulo user to a nearby PoP (m03/m06);
   **replicate** the KV store to a South America region so cache misses hit a nearby replica, not
   Virginia. Redirect served in single-digit ms locally.
3. **CDN/edge + replicated hot cache entry** handle it. It's the **easy** hot-key case because the
   mapping is **immutable** — a viral link's redirect is perfectly cacheable at the edge (no consistency/
   invalidation concern), so origin barely sees it.

---

## Exercise 9 — Pastebin
1. **Same:** generate a unique key (base62 + KGS/Snowflake) and **look up content by key**; read-heavy,
   cacheable.
2. **Changes:** the payload is **large text/files**, so **read bandwidth now matters** (you serve real
   content, not a tiny redirect), there are **size limits**, and you may need syntax highlighting/expiry/
   privacy. The store changes: **content → object storage (S3/blob)**, not a row.
3. **Content (the body) lives in object storage (S3/MinIO); metadata + the key live in the DB** (m04
   "split metadata from blobs", m18), and the body is served through a **CDN**. The DB stays small and
   fast; the bytes live in cheap, durable blob storage.

---

## Exercise 10 — Security & abuse + your-systems tie-in 🏭
1. **Abuse + defenses:** (a) **malware/phishing links** → scan destinations (Safe Browsing/blocklists),
   interstitial warnings; (b) **bulk spam creation** → **rate-limit** per user/IP (m12) + CAPTCHA;
   (c) **open-redirect / scheme abuse** (`javascript:`) → validate/whitelist URL schemes, log + monitor.
2. **Consistency:** **eventual is fine** (a new link visible a second late is OK; serve creator from
   primary for read-your-writes). The non-negotiable "-ility" is **durability** — a committed mapping
   must never be lost (replicate). AP-leaning (m02).
3. **Your-systems mapping:** **Redis cache-aside** for the hot read path = DCE reference-data / Questimate
   snapshots (m06); **async click analytics via Kafka** = DCE's Kafka decoupling (m08); **idempotent
   create on (user, long_url)** = your idempotency-key reflex (m03/m08); **KGS handing out ranges to
   avoid per-write coordination** = the same "precompute/allocate to dodge hot-path coordination"
   instinct as Questimate's snapshot precompute (m05/m09).
