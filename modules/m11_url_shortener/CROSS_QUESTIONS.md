# m11 — Cross-Questions ("if-and-buts") on the URL Shortener

> The interviewer rarely lets the happy path stand — they probe. Answer out loud in 2–3 sentences
> before reading the model answer. This is where the design is won.

---

### Q1. How do you generate the short codes? Walk me through the options.
**A.** Codes are numbers in **base62** (`[a-zA-Z0-9]`, 62 symbols; **62⁷ ≈ 3.5 trillion** → 7 chars is
plenty). Options: **(A) hash the long URL + truncate** — stateless but **collisions** need check-and-
retry; **(B) global auto-increment counter** base62-encoded — no collisions but a **write bottleneck +
SPOF + guessable** sequential codes; **(C) a Key-Gen Service handing out pre-allocated ranges** — app
servers mint codes locally with no per-write coordination (scalable, no collisions); **(D) Snowflake-
style IDs** (timestamp+machine+sequence) — fully distributed, no SPOF, but longer codes. **I'd pick C**
(HA KGS + pre-fetched ranges), or D if I want zero central coordination and accept longer codes.

---

### Q2. Why base62 and not base64? And why 7 characters?
**A.** **base64** includes `+`, `/` (and `=` padding) which aren't URL-safe (you'd have to escape them);
**base62** (letters + digits) is clean in a URL. (base64**url** swaps in `-`/`_` if you want 64.) **7
chars** because **62⁷ ≈ 3.5 trillion** unique codes — at 100M new URLs/day that's decades of headroom,
while staying short and typeable. 6 chars (≈57B) might run out; 8 is unnecessary length.

---

### Q3. What's wrong with a simple global auto-increment counter?
**A.** Three problems: it's a **single point of failure** and a **write bottleneck** (every create must
serialize on incrementing one value → contention at scale), and the codes are **sequential and
guessable** — anyone can enumerate all links (privacy/security leak) and competitors can infer your
volume/growth from the latest ID. You can mitigate (distribute the counter into ranges = the KGS), but a
single naive counter doesn't scale or stay private.

---

### Q4. How does the Key Generation Service avoid being its own bottleneck/SPOF?
**A.** It hands out **ranges**, not single IDs — an app server requests a block (say 1M codes) **once**
and then mints locally with **zero coordination per create**, so the KGS is hit rarely. For HA, you
**replicate** the KGS and have app servers **pre-fetch the next range** before exhausting the current
one, so a brief KGS outage doesn't block creates. If a server crashes mid-range you "lose" some codes —
harmless, since the key space is ~3.5T. You can also pre-generate **random** codes into an "available"
pool so they're not sequential.

---

### Q5. Hash-based codes can collide. How do you handle that?
**A.** On a collision (the truncated hash already maps to a *different* URL), you **retry with a
perturbation** — append a salt/counter to the input and rehash, or take different hash bits — until you
find a free code, checking against the store each time. The cost is a **read (existence check) per
write** and occasional retries, which adds latency and complexity. That overhead is exactly why I prefer
collision-free generation (KGS/Snowflake) unless natural dedup (same URL → same code) is a hard
requirement.

---

### Q6. The same long URL is submitted twice — one code or two?
**A.** It's a product choice. **Dedup (one code)** saves space and is what hash-based generation gives
naturally — good if you want canonical links. **Separate codes** are needed if different users want
**their own** link (e.g. per-user analytics on the same destination). I'd usually **dedup per user**:
make `POST /urls` **idempotent on `(user_id, long_url)`** (or via an `Idempotency-Key`) so retries don't
create duplicates, but two *different* users can each have their own code with separate stats.

---

### Q7. SQL or NoSQL for the store, and why?
**A.** The access pattern is a **point lookup by `short_code`** at massive scale with no joins or multi-
row transactions — a textbook **key-value/wide-column** fit (DynamoDB/Cassandra), trivially **sharded by
short_code**. A **sharded relational DB** works too (it's still a key lookup), so I wouldn't fight over
it — I'd default to a KV store for the simplicity and horizontal scale, and note SQL is fine at this
scale. The analytics data is a *different* shape (append-only events) → a separate columnar/TSDB store,
off the hot path (m04).

---

### Q8. How do you make redirects fast at 100k reads/sec?
**A.** Serve them from **cache, not the DB**: `code → long_url` in **Redis** (cache-aside), which with
80/20 traffic absorbs the vast majority of reads in sub-ms; a **CDN/edge** fronts extremely hot links so
they never reach origin. Cache **misses** fall through to **DB read replicas** (m05). Because mappings
are **immutable**, caching is easy here (no invalidation problem) — the design is essentially "a big,
replicated, cached lookup table."

---

### Q9. 301 vs 302 redirect — which and why?
**A.** **301 (permanent)** is **cached by the browser**, so repeat clicks skip your server → faster and
less load, but you **lose analytics** on those clicks and can't change the target. **302 (temporary)** is
**not cached** → every click hits you → you can **count clicks** and change/expire links. **Most
commercial shorteners use 302** precisely to capture click analytics; I'd choose 301 only if pure speed
matters and the mapping is truly permanent and untracked.

---

### Q10. How do you support custom aliases without collisions?
**A.** A custom alias must be **unique within the same key space** as generated codes. On create, I
**check availability** — a **uniqueness constraint** in the store, fronted by a **Bloom filter** to
fast-reject already-taken aliases without a DB read (m06). To avoid generated codes accidentally
producing a string a user wanted, I either keep one namespace with a uniqueness check, or **reserve**
custom aliases in the same space so generation skips them. Reject on conflict with a clear error.

---

### Q11. How would you implement click analytics without slowing redirects?
**A.** Never compute analytics on the redirect path. Use **302** (so clicks reach you), then **emit a
click event asynchronously** to a stream (**Kafka**) and return the redirect immediately — the
counting/aggregation happens in a **separate pipeline** (stream processor → a columnar/time-series store,
m08/m04). For a simple counter you could increment a Redis counter async, but for rich analytics (geo,
referrer, time-series) you want the event pipeline. The redirect stays sub-ms.

---

### Q12. How do you scale storage as it grows to hundreds of TB?
**A.** **Shard the KV store by `short_code`** using **consistent hashing** (m05) so data spreads evenly
and adding nodes only remaps ~K/N keys; **replicate each shard** for durability/availability. Reads are
mostly absorbed by cache/CDN, so the DB mainly handles writes (1k/s — easy) and cache misses. I'd also
**tier/expire** old, dead links if they have TTLs to reclaim space. Storage scaling here is
straightforward because the data model is a simple, shardable key→value.

---

### Q13. A single link goes viral — millions of clicks/sec to one code. What happens?
**A.** That's a **hot key** (m05/m06): all traffic for one code lands on one cache node/shard. Fixes:
serve it from the **CDN/edge** (a redirect for one URL is perfectly cacheable — the destination doesn't
change), **replicate the hot cache entry** across nodes, and rely on the immutability of the mapping
(no consistency concern). Because the mapping never changes, a viral link is actually the *easy* hot-key
case — push it to the edge and origin barely notices.

---

### Q14. How do you handle link expiration and cleanup?
**A.** Store an `expiry` timestamp; on read, if expired return **410 Gone** (and don't serve the
redirect). For cleanup, use **lazy deletion** (remove on access when found expired) plus a **background
sweeper** that batches deletions of expired keys — never a synchronous scan of hundreds of TB. If using a
store with native **TTL** (Redis, DynamoDB TTL, Cassandra TTL), let it expire entries automatically. The
key point: reclaim space **incrementally/async**, not in a big blocking job.

---

### Q15. Isn't a public redirector a security risk? How do you protect it?
**A.** Yes — you're a potential **open redirect** and a **malware/phishing distribution** vector. Defenses:
**scan/validate destination URLs** on creation (e.g. Google Safe Browsing, blocklists) and re-scan
periodically; **rate-limit creation** per user/IP (m12) to stop bulk abuse; show an **interstitial
warning** for suspicious links; strip/validate the URL scheme (no `javascript:` etc.); and log/monitor
for abuse patterns. Real shorteners invest heavily here because attackers love short links to disguise
malicious destinations.

---

### Q16. What consistency model do you need? Is eventual consistency okay?
**A.** **Eventual consistency is fine** and preferred. A newly-created link being visible on all replicas
a second later is acceptable (the creator can be served from the primary for read-your-writes if needed).
What you **can't** compromise is **durability** — once created, the mapping must never be lost (replicate
it). So this is an **AP-leaning** design (m02): favor availability + low latency, accept a brief
propagation window, but guarantee durability. Strong consistency would add coordination latency for no
real benefit here.

---

### Q17. How do you prevent two concurrent creates from getting the same code?
**A.** With the **KGS range** approach it can't happen — each app server owns a **disjoint range**, so two
servers never mint the same code. With a **counter**, the increment is atomic (one DB/Redis `INCR`). With
**hashing**, the **uniqueness constraint** on `short_code` in the store rejects a duplicate on insert
(and you retry). So uniqueness is guaranteed either by **partitioned ranges** (no coordination) or an
**atomic operation / unique constraint** at write time — never by hoping.

---

### Q18. Walk me through the full request flow for a redirect, including a cache miss.
**A.** GET `/aZ3kP9` → DNS → **L7 load balancer** → a **stateless app server**. It looks up `aZ3kP9` in
**Redis**: on a **hit**, return **302** with `Location: <long_url>` in sub-ms. On a **miss**, read a
**DB read replica** for the code, **populate the cache** (so future clicks hit), and return the 302; if
not found → **404** (or **410** if expired). Concurrently emit a **click event** to Kafka for analytics.
Very hot links are served entirely from the **CDN/edge** and never reach the app tier.

---

### Q19. How is designing Pastebin different from a URL shortener?
**A.** Structurally similar (generate a unique key, store, retrieve by key) but the **payload is large
text/blobs, not a tiny URL** — so you **store the content in object storage (S3/blob), keep only metadata
+ the key in the DB** (m04 "split metadata from blobs", m18), and the **read path serves real content
through a CDN** (so read bandwidth matters now, unlike a redirect's tiny response). You also add
size limits, optional expiry, and syntax/render concerns. Same ID-generation + KV-lookup core, different
storage tier for the body.

---

### Q20. How would you make this globally fast (users worldwide)?
**A.** Multi-region: serve redirects from a **global CDN/edge** (anycast/GeoDNS routes users to the
nearest PoP, m03/m06), and **replicate** the KV store across regions so cache misses hit a nearby replica
rather than crossing continents (m05). Creates can go to a home region (writes are low volume) and
replicate out. Because mappings are immutable, global read replication is easy (no conflict/invalidation
problem). The result: a redirect is served from a PoP near the user in single-digit ms, anywhere.
