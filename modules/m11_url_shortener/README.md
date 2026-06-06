# Module 11 — Design a URL Shortener (TinyURL / bit.ly) ⭐ — *PART 2 begins*

> The "hello world" of system design — and the **template** for every design problem after it. It looks
> trivial ("just store a mapping") but it cleanly teaches the whole craft: **estimation**, **unique ID
> generation**, **read-heavy caching**, **KV storage at scale**, and the **redirect path**. We solve it
> by **walking the m01 7-step framework end-to-end**, out loud, with numbers and trade-offs at every
> step. Learn the *moves* here; later problems just change the content.

> **How to read this module:** this README *is* a full worked interview answer. Notice the **rhythm** —
> requirements → estimate → API → data → high-level → deep-dive → bottlenecks — and how every decision
> traces back to a requirement or a number (m01). Then `CROSS_QUESTIONS.md` drills the follow-ups.

---

## Step 1 — Requirements (scope it first; never skip this) ⭐

**Clarify, then state assumptions and get a nod.** (m01: most failures happen here, not in the tech.)

**Functional (core):**
1. **Shorten:** given a long URL, return a short URL (`https://sho.rt/aZ3kP9`).
2. **Redirect:** visiting the short URL **redirects** to the original (the hot path).
3. *(Optional, scoped as stretch)* **custom alias** (`sho.rt/my-brand`), **expiry/TTL**, **analytics**
   (click counts), **user accounts**.

> *Say:* "I'll focus on **shorten + redirect**, and treat custom alias, expiry, and analytics as
> extensions — does that scope work?"

**Non-functional (these drive the architecture):**
- **Read-heavy:** redirects ≫ creations — assume **~100:1**. ⇒ optimize the read path (cache/CDN).
- **Low latency:** redirect must be fast (< ~100 ms p99) — it's in the user's critical path.
- **High availability:** a dead redirect breaks **every link ever shared** → target ≥ 99.9%.
  *Availability > strong consistency here* (a new link visible a second late is fine).
- **High durability:** a mapping must **never be lost** — links "live forever."
- **Scalability:** assume ~**100M new URLs/day**; design to grow.
- **Short, non-guessable-ish codes**, and **no collisions** (two long URLs must never share a code).

> **The key NFR insight:** read-heavy + must-not-lose-data + availability-over-consistency → this is a
> **cache + replicated KV store** problem, and the **interesting** part is **generating unique codes**.

---

## Step 2 — Capacity Estimation (numbers → architecture) ⭐

Round hard (m02: 1 day ≈ 10⁵ s; peak ≈ 2–10× avg).

**Throughput:**
- Writes: 100M/day ÷ 10⁵ = **1,000 writes/sec** avg (~5–10k peak). *Trivial for one primary.*
- Reads: 100:1 → **100,000 reads/sec** avg (~500k peak). *This is the number that shapes the design* →
  must be served from **cache/CDN**, not the DB.

**Storage** (per mapping ≈ short_code 7B + long_url ~500B + metadata ~100B ≈ **~600 bytes**, round 500):
- 100M/day × 500 B = **~50 GB/day** → ~18 TB/year → **~90–100 TB over 5 years** → ×3 replicas ≈ **~300
  TB**. *Fits a sharded KV cluster; not a single box at 5 yrs, but no emergency either.*

**Bandwidth:** a redirect response is **tiny** (just headers + a `Location`, ~few hundred bytes — we
don't serve the destination's content). 100k reads/s × ~500 B ≈ **~50 MB/s** egress — modest.

**Cache memory:** by the 80/20 rule, a small fraction of links gets most traffic. Cache the **hot set**
(store just `code → long_url`, ~100–200 B/entry). Caching, say, the ~100M hottest mappings ≈ **~10–20
GB** — a modest Redis cluster serves the bulk of 100k reads/s from memory.

**Key space:** how long must codes be? (See base62 in Step 6.) **62⁷ ≈ 3.5 trillion** > 100M/day × 365 ×
many years → **7 characters is plenty.**

> **The estimate already decided the architecture:** writes are easy (1k/s), **reads are the game**
> (100k/s → cache/CDN + read replicas), storage needs **sharding within a year or two**, and **7-char
> base62 codes** suffice. Say the *implications*, not just the digits.

---

## Step 3 — API / Interface (the contract)

```
POST /api/urls
  body:  { "long_url": "...", "custom_alias?": "...", "expiry?": "2027-01-01" }
  resp:  201 { "short_url": "https://sho.rt/aZ3kP9" }

GET /{short_code}
  resp:  302 Found, Location: <long_url>        (redirect — the hot path)
         404 if not found / 410 if expired

GET /api/urls/{short_code}/stats   (stretch)
  resp:  200 { "clicks": 1234, ... }
```

Decisions to mention:
- **Idempotency:** make `POST /urls` idempotent on `(user, long_url)` (or accept an `Idempotency-Key`,
  m03) so a client retry doesn't create duplicate codes for the same URL.
- **AuthN/rate-limit** at the API gateway (m03/m07/m12) — creation is abuse-prone.
- **301 vs 302** for the redirect — a real design decision (Step 6).

---

## Step 4 — Data Model & storage choice ⭐

**The entity is tiny:**
```
URL:  { short_code (PK), long_url, user_id, created_at, expiry, click_count? }
```
**Access pattern:** the hot path is a **point lookup by `short_code`** (→ `long_url`); creation is a
single write; occasional "list by user." No joins, no complex queries, massive scale, simple relations.

**SQL or NoSQL?** This is the textbook fit for a **key-value / wide-column store** (DynamoDB, Cassandra,
or even a sharded MySQL/Postgres used as a KV): O(1) lookups by key, trivial to **shard by `short_code`**,
and we don't need multi-row transactions. *Say:* "I'd use a KV store keyed by `short_code`; a sharded
relational DB also works at this scale — the access pattern is a key lookup either way" (m04).

**Analytics is a different shape** — append-only, high-volume click events → **don't** put them on the
redirect path. Stream clicks to a separate pipeline (Kafka → a time-series/columnar store, m08/m04 OLAP).
*Separate the hot read path from analytics.*

---

## Step 5 — High-Level Design (the happy path) ⭐

```
                                  ┌─────────── WRITE (create) ───────────┐
  client ─create─▶ DNS ─▶ LB(L7) ─▶ App ─▶ Key-Gen Service ──▶ KV store (short_code → long_url)
                                     │         (unique code)        │
                                     └────────── populate cache ◀───┘

                                  ┌─────────── READ (redirect, 100k/s) ──┐
  client ─GET /aZ3kP9─▶ DNS ─▶ LB ─▶ App ─▶ Cache (Redis) ──HIT──▶ 302 redirect (<1 ms)
                                            │   MISS
                                            ▼
                                       KV store (read replica) ─▶ populate cache ─▶ 302
                                  (optionally a CDN/edge in front for very hot links)
```

**Narrate the two paths:**
- **Create:** client POSTs a long URL → app gets a **unique short code** from the Key-Gen Service (Step
  6) → writes `{code → long_url}` to the KV store (durable, replicated) → returns the short URL (and may
  warm the cache).
- **Redirect (the 100k/s path):** GET hits the L7 LB → an app server checks **Redis**; on a **hit** it
  returns a **302** in ~sub-ms; on a **miss** it reads a **DB read replica**, populates the cache, and
  redirects. Very hot links can also sit in a **CDN/edge** so the redirect never reaches origin.

**Principles applied** (each a prior module): **stateless app tier** behind an **L7 LB** (m07);
**cache the hot read path** + optional CDN (m06); **read replicas** for the DB (m05); **async** analytics
off the hot path (m08).

---

## Step 6 — Deep Dive #1: Unique short-code generation ⭐⭐ (the heart of the problem)

This is *the* interesting decision — interviewers spend most of the time here. Present options →
trade-offs → pick → residual cost (the m01 deep-dive template).

### First, the encoding: base62
A short code is a number rendered in **base62** = `[a–z A–Z 0–9]` (26+26+10 = **62** symbols). Capacity
by length:
- 62⁶ ≈ **56.8 billion** · 62⁷ ≈ **3.5 trillion** · 62⁸ ≈ **218 trillion**.
- ⇒ **7 characters → ~3.5 trillion** unique codes — comfortably enough (100M/day for decades).
- Why base62 not base64? base64 uses `+ /` (and `=`) which aren't URL-safe; base62 is clean in a URL
  (base64**url** swaps in `- _` if you want 64).

### Option A — Hash the long URL (e.g. MD5/SHA), take first 7 chars
- **Pro:** stateless; same URL → same code (natural dedup).
- **Con:** **collisions** (different URLs → same 7-char prefix) → must **check-and-retry** (a read per
  write, and a strategy on collision: append/salt and rehash). Also leaks nothing useful and the
  truncation throws away hash bits. *Workable but the collision handling is fiddly.*

### Option B — Auto-increment counter, base62-encoded
A global counter (1, 2, 3, …) → base62. Unique by construction, short.
- **Pro:** **no collisions**, shortest possible codes, dead simple.
- **Con:** the **counter is a single point of failure and a write bottleneck** (every create must
  increment one global value → contention), and codes are **sequential/guessable** (enumerate all links
  → privacy/security issue, and competitors can count your growth).

### Option C — Key Generation Service (KGS) with pre-allocated ranges ⭐ (the strong answer)
A dedicated service **pre-generates** unique codes (or owns the counter) and **hands out *ranges*** to
app servers (e.g. "you take 1,000,000–1,000,999"). Each app server mints codes **locally** from its range
with **no coordination on the hot path**.
- **Pro:** removes the per-write bottleneck (no global increment per request); scales horizontally; no
  collisions.
- **Con:** another service to run (make it **HA**: replicate it; app servers **pre-fetch the next range**
  so a brief KGS outage doesn't block writes). You can "lose" a range on a crash — fine, the space is
  huge (3.5T). Optionally **pre-generate random codes** into a "available keys" table to also make them
  non-sequential.

### Option D — Snowflake-style distributed IDs
Generate a **64-bit ID** locally on each machine from **timestamp + machine_id + per-ms sequence**, then
base62-encode it. No central coordinator at all.
- **Pro:** fully distributed, no SPOF, roughly time-ordered, high throughput.
- **Con:** **longer codes** (64-bit → ~11 base62 chars) unless you compress, and it leaks timestamp +
  machine info. Great when you want zero coordination and don't mind longer codes.

> **The pick (say it like this):** *"I'd use a **Key Generation Service handing out pre-allocated
> ranges** (Option C): it removes the write-path bottleneck and avoids collisions while keeping codes
> short. I'd make the KGS HA and have app servers pre-fetch their next range so a KGS blip doesn't block
> creates. If I wanted zero central coordination and could accept longer codes, I'd use **Snowflake IDs**
> (Option D). I'd avoid a naive global counter (bottleneck + guessable) and avoid hash-truncation unless
> I'm okay handling collisions."* — Options + trade-offs + pick + justification + residual cost = full
> marks.

---

## Step 6 — Deep Dive #2: the read path (caching) ⭐

Reads are 100:1 and latency-critical, so the read path **is** the design:
- **Cache `code → long_url` in Redis** (cache-aside, m06). With 80/20 traffic, a modest cache absorbs the
  vast majority of the 100k/s, leaving the DB lightly loaded.
- **Eviction:** **LRU** (recently-clicked links stay hot). **TTL** optional (links don't change, so
  entries can live long; expire only if the URL has an expiry).
- **CDN/edge** for *extremely* hot links (a viral campaign): the redirect can be served at the edge
  without touching origin (m06) — and handles the **hot-key** problem (m05/m06).
- **Cache failure** = reads fall through to **DB read replicas** (m05); size them for that fallback +
  use **request coalescing** to avoid a **stampede** (m06) when a hot key misses.
- **Mappings are immutable** (a code always points to the same URL) → caching is *easy* here: almost no
  invalidation problem (the hard part of caching, m06) — you mostly only handle **deletes/expiry**.

---

## Step 6 — Deep Dive #3: 301 vs 302 redirect (a real trade-off) ⭐
- **301 (Permanent):** the browser **caches** the redirect → subsequent clicks **skip your server** and
  go straight to the destination. **Faster + less load**, but you **lose click analytics** (you never see
  the repeat clicks) and you can't change the target.
- **302 (Found / Temporary):** **not cached** → every click **hits your server** first → you can **count
  clicks** and change/expire the target, at the cost of more load + slightly slower.
> **The senior answer:** *"302 if analytics matter (the common choice for link shorteners — they want
> click data); 301 if you purely want speed and the mapping is truly permanent. Most commercial
> shorteners use 302 precisely so they can track clicks."*

---

## Step 7 — Bottlenecks, scale & edge cases (stress your own design) ⭐

Proactively raise these (m01 — the strongest senior signal):

- **Scaling reads (already handled):** cache + CDN absorb the 100k/s; DB read replicas for misses.
- **Scaling writes/storage:** at ~50 GB/day, **shard the KV store by `short_code`** (consistent hashing,
  m05) within a year or two; the KGS removes the create bottleneck.
- **Custom aliases:** a user-chosen alias must be **unique** and not collide with generated codes. Check
  availability on write (a uniqueness constraint / a **Bloom filter** to fast-reject taken aliases before
  a DB read, m06); keep generated codes and custom aliases in the **same key space** (or namespace them).
- **Analytics at scale:** stream click events to **Kafka → a columnar/time-series store** (m08/m04);
  never compute analytics on the redirect path. Use **302** so you capture clicks.
- **Abuse / rate limiting:** creation is spam/abuse-prone → **rate-limit** per user/IP (m12), and
  **validate/scan URLs** (block malware/phishing, e.g. Safe Browsing) — a security must for a public
  redirector (you're a potential open-redirect / malware vector).
- **Expiry & cleanup:** store an `expiry`; on read, return **410 Gone** if expired; reclaim space with a
  **lazy delete** (on access) + a background sweeper (don't scan 100 TB synchronously).
- **SPOFs:** KGS (→ HA + pre-fetched ranges), DB primary (→ replicated w/ failover, m05), cache (→
  replicated; reads survive its loss via DB). LB itself HA (m07).
- **Hot key:** a viral link → CDN/edge + replicate the hot cache entry (m05/m06).
- **Observability (m10):** track redirect p99, **cache hit ratio**, create rate, error rate, KGS range
  consumption; alert on SLO burn.

**Wrap-up (the 3-sentence close):** *"A read-heavy KV system: creates mint a unique base62 code via an
HA Key-Gen Service and write to a sharded, replicated KV store; redirects are served from Redis (and a
CDN for hot links) with DB replicas as fallback, returning a 302 so we can capture analytics async via
Kafka. Key trade-offs: KGS ranges over a global counter (scale + no SPOF), cache-first reads (latency),
and 302 over 301 (analytics over raw speed). With more time I'd add multi-region for availability,
abuse/malware scanning, and load-test the shard rebalance."*

---

## State-of-the-art & real-world notes 📚
- **Real shorteners (bit.ly, TinyURL, t.co):** use **302** redirects to capture click analytics, run
  **malware/phishing scanning** on creation, and serve redirects from a globally-distributed **edge/CDN**
  for latency. bit.ly historically used a base-encoded counter/ID approach with heavy caching.
- **ID generation in industry:** **Twitter Snowflake** (timestamp + worker + sequence) is the canonical
  distributed-ID design (Option D); many shops use Snowflake-likes (Sonyflake, Instagram's sharded ID
  scheme). The **KGS/ticket-server** approach (Option C) is the Flickr "ticket servers" idea.
- **Why this problem is *the* warm-up:** it isolates the core skills (estimation, unique IDs, read-heavy
  caching, KV sharding) with almost no domain complexity — masters of this problem can then layer on the
  harder ones (feed, chat, etc.).

---

## From your systems 🏭
- **Caching the hot read path:** exactly your **Redis cache-aside** pattern (DCE reference data,
  Questimate snapshots) — here it's `code → long_url`, the easy case because mappings are **immutable** so
  there's no invalidation headache (m06).
- **KGS ≈ ID/range allocation:** the "hand out pre-allocated ranges so workers don't coordinate per
  request" idea is the same "precompute/allocate to avoid hot-path coordination" instinct behind
  Questimate's snapshot precompute and the m05/m09 "coordination is the tax on scale" theme.
- **Idempotent create:** the `(user, long_url)` idempotency is your m03/m08 idempotency-key reflex —
  retries mustn't create duplicate codes (just like a claim/trade mustn't double-execute).
- **Async analytics via Kafka:** stream clicks off the hot path → exactly DCE's Kafka decoupling (m08).

---

## Key concepts (interview-ready)
- **Drive the 7 steps** (this module is the template): requirements → estimate → API → data → high-level
  → deep-dive → bottlenecks. Trace every decision to a requirement or number.
- **It's read-heavy (~100:1):** reads are the design → **cache (Redis) + CDN + read replicas**; writes
  (1k/s) are easy. Mappings are **immutable** → caching is easy (no invalidation).
- **Unique code generation is the core deep-dive:** **base62** (62⁷ ≈ 3.5T → 7 chars). Options: hash+
  retry (collisions), global counter (bottleneck + guessable), **KGS pre-allocated ranges (best —
  scalable, HA)**, **Snowflake IDs** (no coordination, longer codes). Pick KGS or Snowflake; justify.
- **Storage:** a **KV store sharded by short_code** (consistent hashing), replicated for durability/HA.
- **301 vs 302:** 301 = cached, fast, **no analytics**; **302 = uncached, trackable** (what real
  shorteners use). 
- **Bottlenecks:** shard for storage; CDN/replicas for hot keys; **Bloom filter** for custom-alias
  checks; **async analytics (Kafka)**; **rate-limit + malware-scan** creation; lazy expiry.

---

## Go deeper (reading)
- **"Designing a URL shortening service" — Grokking the System Design Interview** (the classic writeup of
  this exact problem; KGS approach).
- **Twitter Snowflake** (blog/repo) and **Instagram "Sharding & IDs"** post — real distributed-ID designs.
- **Flickr "Ticket Servers"** — the KGS/ticket idea (Option C) in production.
- Revisit **m05** (consistent hashing/sharding), **m06** (caching/CDN/hot keys), **m08** (async
  analytics) — this problem is those modules applied.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) (extensions
like Pastebin, custom domains, 10× scale) before checking
[`solutions/m11_url_shortener/`](../../solutions/m11_url_shortener/README.md).

When you've done the exercises, say **"Module 12"** to design a *Rate Limiter* (token/leaky bucket,
sliding window, distributed counters) — which you'll also bolt onto *this* design's create endpoint.
