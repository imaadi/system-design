# m13 — Cross-Questions ("if-and-buts") on the News Feed

> Answer out loud in 2–3 sentences before reading the model answer. The fan-out trade-off and the
> celebrity problem are the make-or-break of this design.

---

### Q1. Fan-out on write vs fan-out on read — explain both and the trade-off.
**A.** **Fan-out on write (push):** when a user posts, you immediately write that post into **every
follower's precomputed timeline** (in Redis), so reads are O(1) — fast. **Fan-out on read (pull):** you
store posts only in the author's timeline and, at read time, **query everyone the user follows and merge**
their recent posts. The trade-off is the classic precompute-vs-on-demand: **push = fast reads, expensive
writes** (one post = N follower writes); **pull = cheap writes, expensive reads** (merge many timelines
per scroll). Since feeds are read-heavy, push is the default — except it breaks on celebrities.

---

### Q2. Why does fan-out on write break for celebrities?
**A.** Because one post fans out to **every follower**, and a celebrity has millions (or 100M+) — so a
single tweet triggers **millions of timeline writes**, a "**fan-out storm**" that floods the write path,
delays delivery, and wastes huge resources (many followers won't even read it). The cost of push scales
with **follower count**, which is fine for the average user (~hundreds) but catastrophic for the
high-follower tail. This is the **celebrity / hot-key problem** (m05/m06) made central.

---

### Q3. So what's the actual solution to the celebrity problem?
**A.** A **hybrid fan-out**: **push** for normal users (fast reads), but for **celebrities** *don't* fan
out on write — store their posts in their own timeline and **pull + merge** them at read time for their
followers. So a user's feed = **their precomputed timeline (pushed by normal followees) + recently-pulled
posts of the few celebrities they follow**, merged. A follower-count **threshold** decides which path an
account uses. This avoids the write storm while keeping reads fast for the bulk — exactly what Twitter/
Instagram do.

---

### Q4. Which is more read-heavy — and how does that justify the default choice?
**A.** Feeds are **heavily read-heavy** — users **scroll/refresh far more than they post** (e.g. 10
reads : 2 posts per user/day, and each scroll loads many posts). Since reads dominate and must be fast,
you **optimize the read path** by **precomputing timelines (fan-out on write)** so a read is just
fetching a ready list — you move the expensive merge work to write time, which happens far less often.
The celebrity exception flips the cost, hence the hybrid.

---

### Q5. Do you store full posts or post IDs in the timeline? Why?
**A.** **Post IDs** (references), not full content. Reasons: (1) storing full posts in millions of
timelines **duplicates content massively** and wastes memory; (2) you'd have to update every copy on an
**edit/delete** — instead you store IDs and **hydrate** the current content from the tweet store (cached)
at read time, so edits/deletes are reflected automatically; (3) IDs are tiny, keeping timeline caches
small. The read path is "get post_ids from the timeline → batch-fetch + cache the posts → rank → return."

---

### Q6. What consistency model does a feed need?
**A.** **Eventual consistency** is fine (AP-leaning, m02) — a new post appearing in followers' feeds a few
seconds late is perfectly acceptable, and chasing strong consistency would add coordination latency for no
user benefit. Availability matters more (the feed should always render — degrade to slightly stale rather
than error). The one nicety is **read-your-own-writes**: inject the author's own new post into their view
immediately so *they* see it instantly even before fan-out completes.

---

### Q7. Why cursor-based pagination instead of offset (page numbers)?
**A.** Because the feed is **huge and constantly changing**. **Offset** (`LIMIT 20 OFFSET 100`) is **O(n)**
(the DB skips 100 rows) and, more importantly, **breaks when new posts arrive**: a new post at the top
shifts everything, so page 2 **repeats or skips** items. **Cursor/keyset** pagination passes "give me
items after `<last_seen_id/timestamp>`," which is **stable** (new posts at the top don't affect it) and
**efficient** (an indexed range scan). Feeds always use cursors.

---

### Q8. How is the timeline like a cache / materialized view?
**A.** The precomputed per-user timeline **is** a **denormalized, materialized view** (m04) maintained in
a **cache** (Redis, m06): you do the expensive "merge all my followees' posts" computation **once at write
time** and store the result so reads are cheap — exactly the denormalization/precompute trade (fast reads
bought with extra write work and a staleness window). It's the same pattern as Questimate's **snapshot
tables**. And like any cache, you **cap** it (latest ~800) and **evict** inactive users.

---

### Q9. How do you handle inactive users with fan-out on write?
**A.** Pushing posts into the timelines of users who **never log in** is wasted work and memory. Refinements:
**don't precompute (or evict) timelines for inactive users** — when an inactive user returns, **build their
timeline on demand (pull)** and then start maintaining it (push) again. You can track "last active" and
only keep warm timelines for recently-active users. This trims a big chunk of the 1.4M writes/sec the naive
estimate produced.

---

### Q10. How would you rank the feed (not just chronological)?
**A.** Real feeds use **ML ranking**: fetch a **candidate set** (the timeline — recent posts from
followees + injected celebrity posts), then **score each candidate** with a model predicting engagement
(likelihood of like/reply/dwell, recency, affinity), and **reorder** before returning the page. So the
pipeline is **candidate generation → ranking → re-ranking** (dedupe/diversity/business rules). That ranking
system is its own design — we build it in **m26 (feed ranking & CTR)**. For this module, I'd note ranking
as a step after candidate retrieval and keep the storage design ranking-agnostic.

---

### Q11. How do you store and scale the social graph (who follows whom)?
**A.** As **adjacency lists** per user — typically **two**: my **followers** (needed for fan-out on write)
and my **following** (needed for fan-out on read). At billions of edges you **shard by user_id** (a
sharded KV/wide-column, or a graph store like Facebook's TAO). The hot queries are "get a user's followers"
(to fan out a post) and "get a user's following" (to pull/merge). The graph is its own subsystem with its
own hot keys (celebrities have enormous follower lists) and caching.

---

### Q12. Where does Kafka (or a queue) fit in this design?
**A.** In the **fan-out path**: when a user posts, the post service writes the tweet and then **emits a
fan-out event to Kafka** (m08); **fan-out worker** consumers read it and append the post_id into each
follower's timeline. This **decouples** posting from delivery (the poster doesn't wait on millions of
writes), **absorbs bursts**, allows **retries**, and scales workers/partitions independently. Fan-out must
be **idempotent** (don't double-insert a post into a timeline on redelivery — m08).

---

### Q13. A celebrity is both a write problem and a read problem. Explain both and the fixes.
**A.** **Write:** fanning their post to millions = a storm → fix with **hybrid pull** (don't push their
posts; merge at read time). **Read:** millions of users read the celebrity's posts → their recent posts
are a **hot key** (m05/m06) → fix by **caching their recent posts hard** (replicate the hot entries, serve
from a CDN/edge), so the merge-at-read-time step is a cheap cache hit. So celebrities get **pull on write +
heavily-cached reads** — the opposite treatment from normal users, justified by their skew.

---

### Q14. Walk me through what happens when a normal user posts a tweet.
**A.** (1) Client `POST /posts` → **post service** writes the tweet to the **sharded tweet store** with a
time-sortable ID. (2) Post service emits a **fan-out event** to **Kafka**. (3) **Fan-out workers** look up
the author's **followers** (social graph) and **append the post_id** to each follower's **Redis timeline**
(capped, idempotent), skipping inactive users. (4) The author immediately sees their own post
(read-your-writes). All of step 2–3 is **async**, so the poster's request returns fast.

---

### Q15. Walk me through what happens when a user opens their feed.
**A.** (1) **Timeline service** reads the user's **precomputed post_id list** from **Redis** (fast). (2)
It **merges in** recent posts from any **celebrities** the user follows (the pull path) — also cached. (3)
It **hydrates** the post_ids into full content via a **batch fetch** from the tweet store (cache-first).
(4) It **ranks** the candidates (chronological or ML, m26). (5) Returns a **page** (cursor pagination).
The heavy lifting was done at write time, so the read is mostly cache lookups + a hydrate + a rank.

---

### Q16. How do you handle a user with a huge following (follows 10,000 accounts) on the read path?
**A.** For a **push**-based timeline that's fine — their precomputed timeline already contains everything
pushed to them, so reads stay O(1) regardless of how many they follow (the cost was paid at their
followees' write time). The expensive case is **pull** (merging 10,000 timelines), which is exactly why the
default is **push for normal followees**; only the handful of **celebrities** among those 10,000 are
pulled+merged at read time (a small k-way merge), keeping reads cheap.

---

### Q17. How is designing Instagram's feed different from Twitter's?
**A.** Same core (hybrid fan-out + precomputed timelines + ML ranking), but **media-centric**: images/
videos dominate, so you lean hard on **blob storage + CDN** for the content (m18/m06) — the timeline still
stores IDs, but hydration fetches media URLs served from the edge, and **read bandwidth** matters far more
than a text feed. Upload involves a **transcode pipeline** (m19-style) for multiple resolutions. The feed-
assembly logic is nearly identical; the **media storage/delivery** tier is the big addition.

---

### Q18. How do you keep fan-out idempotent and handle failures?
**A.** Fan-out runs on **at-least-once** Kafka delivery (m08), so a worker may reprocess a post → make the
append **idempotent**: use the post's stable ID as a dedup key (e.g. a sorted set keyed by post_id won't
double-insert; or check-before-append). On worker failure, the message is **redelivered** and reprocessed
safely. Poison messages (a follower list that errors) go to a **DLQ**. This is the same idempotent-consumer
pattern from m08 — redelivery must not duplicate posts in a timeline.

---

### Q19. What are the main bottlenecks/SPOFs and how do you address them?
**A.** **Fan-out service** (scale workers + Kafka partitions; hybrid to cap celebrity load); **Redis
timeline cache** (shard by user_id + replicate; cap timelines; evict inactive); **tweet store** (shard by
post_id, replicate); **social graph** (sharded, cached, celebrity follower-lists are hot); **hot
celebrity posts** (CDN/edge + replicated cache). Observability (m10): timeline read p99, **fan-out lag**
(how far behind delivery is), cache hit ratio, error rates — alert on fan-out lag spiking (a delivery
backlog).

---

### Q20. If you had to start simple (MVP) and evolve, what would you do?
**A.** **MVP: pure fan-out on read (pull)** — store posts per author, compute the feed by querying
followees at read time. It's the **simplest** (one write per post, no fan-out infra) and fine at small
scale. As reads grow and become slow, **add fan-out on write (push)** with precomputed Redis timelines for
fast reads. As you hit **celebrities**, add the **hybrid** (pull for high-follower accounts). Then layer
**ML ranking** and **media/CDN**. This "pull → push → hybrid → rank" evolution is a great way to show you'd
match complexity to scale rather than over-engineer up front (m01).
