# Module 13 — Design a News Feed / Twitter Timeline ⭐⭐

> The first **"big" product design** — and the one that crystallizes the central trade-off of the whole
> course: **precompute vs compute-on-demand**, here as **fan-out on write vs fan-out on read**. It also
> makes the **celebrity / hot-key problem** (the thread from m05/m06) the *main event*. Master this and
> you can design Instagram, Facebook feed, LinkedIn, Reddit, YouTube subscriptions — they're variations.

> **Format (Part 2):** worked m01-framework walkthrough; **fan-out** is the deep-dive; `CROSS_QUESTIONS`
> drills the follow-ups (celebrity problem, ranking, consistency).

---

## Step 1 — Requirements

**Functional (core):**
1. **Post** a tweet/post (text; assume media is a URL to blob storage, m18).
2. **View home timeline:** a feed of recent posts from everyone you **follow**, newest-ish first.
3. **Follow / unfollow** users (the social graph).
4. *(Stretch)* ranking (not pure chronological), likes/replies, media, notifications (m15).

> *Say:* "I'll focus on **post + home-timeline + follow**, treat ranking and engagement as extensions —
> and I'll assume a **home timeline** (posts from people you follow), distinct from a **user timeline**
> (a single user's own posts)."

**Non-functional (these drive it):**
- **Massively read-heavy:** people **scroll** far more than they post. Optimize the read (timeline) path.
- **Low latency timeline load:** feed must load fast (< ~200 ms) — it's the core UX.
- **Eventual consistency is fine:** a new post appearing a few seconds late is acceptable → **AP-leaning**
  (m02). (Read-your-own-writes for *your* posts is nice — show your own tweet immediately.)
- **High availability:** the feed must always render (degrade to slightly-stale rather than error).
- **Huge scale:** hundreds of millions of users, **highly skewed follower counts** (the celebrity tail).

> **The defining insight:** this is a **read-optimization** problem dominated by the **fan-out**
> decision, and the hard part is the **follower-count skew** (celebrities).

---

## Step 2 — Estimation (the fan-out is the scary number) ⭐

Assume **300M DAU**, each posts **~2/day**, reads timeline **~10×/day**, avg **~200 followers**.
- **Posts (writes):** 300M × 2 = 600M/day ÷ 10⁵ = **~6,000 posts/sec** avg (~30k peak).
- **Timeline reads:** 300M × 10 = 3B/day ÷ 10⁵ = **~30,000 reads/sec** avg (~150k peak). *Read-heavy.*
- **⚠️ The fan-out multiplier:** if we **precompute** each follower's timeline on post (fan-out on write),
  each post writes to ~200 follower timelines → 600M × 200 = **120 *billion* timeline writes/day** ≈
  **~1.4 million writes/sec** into timeline caches. *That's the number that shapes everything* — and it's
  an **average**; a **celebrity with 100M followers** generates **100M writes for a single tweet** (a
  "fan-out storm").
- **Storage:** a tweet ≈ ~300 B (text + metadata) × 600M/day ≈ **~180 GB/day** of tweets → ~65 TB/year
  (cheap, shardable). Materialized timelines are larger but **capped** (keep only the latest ~800 per
  user in cache, hydrate older on demand).

> **The estimate already frames the design:** reads dominate (→ precompute timelines), but naive
> precompute (fan-out on write) explodes on celebrities → you'll need a **hybrid**.

---

## Step 3 — API

```
POST /api/posts            { text, media_url? }            -> 201 { post_id }
GET  /api/feed?cursor=...&limit=20                          -> { posts:[...], next_cursor }
POST /api/follow           { target_user_id }              -> 200
```
- **Cursor (keyset) pagination, not offset** — feeds are huge and constantly changing; offset is O(n) and
  skips/duplicates items as new posts arrive. Use a cursor (e.g. last seen post_id/timestamp).
- Posting is a **write** to the tweet store + triggers **fan-out** (Step 6).

---

## Step 4 — Data model

```
User:    { user_id (PK), name, ... }
Post:    { post_id (PK, ~Snowflake/time-sortable), author_id, text, media_url, created_at }
Follow:  { follower_id, followee_id }      ← the social graph (adjacency list)
Timeline (materialized, per user, in cache): ordered list of post_ids (capped, e.g. latest 800)
```
- **Posts:** sharded KV/wide-column by `post_id` (time-sortable IDs help ordering — Snowflake, m11).
- **Social graph:** two adjacency lists per user — **followers** (who follows me — needed for fan-out on
  write) and **following** (who I follow — needed for fan-out on read). Sharded by user_id; this is its
  own scaling challenge (billions of edges).
- **Timeline:** a **per-user precomputed list** in **Redis** (a sorted set / list of post_ids, capped) —
  this is the **materialized view** that makes reads O(1)-ish.

> The timeline cache is a **denormalized, precomputed view** (m04/m06) — exactly Questimate's snapshot-
> table idea applied to feeds.

---

## Step 5 — High-level design

```
  POST a tweet:
   client ─▶ LB ─▶ Post service ─▶ Tweet store (sharded)
                          │
                          └─▶ Fan-out service ──(for each follower)──▶ append post_id to follower's
                              (async, via Kafka)                       Timeline cache (Redis)

  READ timeline:
   client ─▶ LB ─▶ Timeline service ─▶ Timeline cache (Redis: precomputed post_ids)
                          │  HIT → return post_ids
                          └─▶ hydrate: fetch full posts from Tweet store (+ cache) ─▶ rank ─▶ return
```

- **Write path:** post → store the tweet → **async fan-out** (via a queue/Kafka, m08) distributes the
  post_id into each follower's timeline cache. Async because fan-out is heavy and the poster shouldn't
  wait.
- **Read path:** timeline service reads the user's **precomputed post_id list** from Redis (fast),
  **hydrates** the actual post content (from the tweet store, cached), optionally **ranks**, and returns a
  page. The expensive merge work was done *at write time* — so reads are cheap.

---

## Step 6 — Deep Dive: fan-out on write vs read ⭐⭐ (THE decision)

How is each user's home timeline assembled? This is the heart of the problem.

### Fan-out on WRITE (push model) — precompute timelines
When a user posts, **immediately push** the post into **every follower's** materialized timeline (in
Redis). Reads just fetch the ready-made list.
- **Pro:** **reads are extremely fast** (timeline is precomputed — O(1) lookup); great for the read-heavy
  workload.
- **Con:** **writes are expensive** — one post = N writes (N = follower count). **Catastrophic for
  celebrities** (100M followers → 100M writes per tweet = a "fan-out storm" that floods the system and
  delays delivery). Also **wastes work** for inactive followers (you precompute timelines nobody reads).
```
  PUSH:  post by Alice → write to [follower1][follower2]...[followerN] timelines   (read = instant)
```

### Fan-out on READ (pull model) — compute at read time
Store posts only in each author's own timeline. When a user opens their feed, **query all the people they
follow**, fetch each one's recent posts, and **merge** them on the fly (a k-way merge by time).
- **Pro:** **writes are cheap** (one write — to your own timeline); no celebrity write storm; no wasted
  precompute.
- **Con:** **reads are expensive** — a user following 1,000 accounts must query 1,000 timelines and merge
  every time they scroll → slow, heavy, hard to cache. Bad for the read-heavy workload.
```
  PULL:  Bob opens feed → fetch recent posts of all 1,000 people Bob follows → merge by time → return
```

| | **Fan-out on write (push)** | **Fan-out on read (pull)** |
|---|---|---|
| On post | write to all N followers' timelines | write once (your own) |
| On read | O(1) — read precomputed list | O(following) — query + merge |
| Best for | normal users (reads ≫ writes) | celebrities / inactive users |
| Pain | **celebrity fan-out storm**, wasted precompute | **slow reads**, hard to cache |

### The HYBRID — the real answer ⭐ (what Twitter/Instagram do)
**Use push for most users, pull for celebrities:**
- **Normal accounts:** **fan-out on write** — push their posts to followers' timelines (fast reads).
- **Celebrities (huge follower counts):** **do NOT fan out on write** (it'd be a storm). Instead, store
  their posts in their own timeline, and at **read time** the timeline service **merges in** the
  celebrity posts (pull) for followers who follow them.
- So a user's feed = **precomputed timeline (from normal followees, pushed) + merged-in recent posts of
  the few celebrities they follow (pulled)**. Best of both: cheap reads for the bulk, no write storm for
  the tail.

> **Say it like this:** *"I'd use a **hybrid fan-out**: push posts to followers' precomputed timelines
> for normal users (fast reads), but for **celebrities** with millions of followers I'd **not** fan out
> on write (it'd be a fan-out storm) — I'd store their posts and **pull + merge** them at read time. The
> follower-count threshold decides which path. This is exactly how Twitter and Instagram handle the
> celebrity problem."* That's the senior answer.

---

## Step 7 — Other deep-dives & bottlenecks

- **The celebrity / hot-key problem** (m05/m06, now central): celebrities are hot keys both on **write**
  (fan-out storm → solved by hybrid pull) and on **read** (millions read their posts → cache their recent
  posts hard, serve from CDN/edge, replicate the hot entries). The hybrid + caching handles both.
- **Feed ranking:** pure **chronological** is simplest. Real feeds use **ML ranking** (engagement
  prediction — likes/replies/dwell) to reorder; that's a ranking system (we design it in **m26**). For
  now: fetch a candidate set (timeline), then **rank** before returning. Mention it as an extension.
- **Timeline cache management:** cap each timeline (e.g. latest 800 post_ids in Redis) — users rarely
  scroll deep; hydrate older pages from the tweet store on demand. Evict inactive users' timelines (don't
  precompute for users who never log in — a refinement of push).
- **Hydration:** store **post_ids** in the timeline (not full posts) and fetch the content separately
  (cached) — so a deleted/edited post is reflected and you don't duplicate content into millions of
  timelines.
- **Consistency:** **eventual** — a post may take seconds to appear in all timelines (AP, m02). Give the
  author **read-your-writes** by injecting their own new post into their view immediately.
- **The social graph at scale:** billions of follow edges → sharded adjacency lists; "get my followers"
  (for fan-out) and "get my following" (for pull) are the hot queries. A graph DB or a sharded KV; this
  is its own subsystem.
- **Pagination:** **cursor/keyset** (Step 3) — never offset.
- **Async fan-out via Kafka** (m08): decouple posting from delivery; absorb bursts; retry; idempotent
  (don't double-insert a post into a timeline).
- **Bottlenecks/SPOFs:** the fan-out service (scale workers + Kafka partitions), Redis timeline cache
  (shard + replicate), the tweet store (shard by post_id). Observability: timeline read p99, fan-out lag,
  cache hit ratio (m10).

**Wrap-up:** *"A read-optimized, AP feed: posts go to a sharded tweet store and an **async (Kafka)
hybrid fan-out** — pushed into followers' precomputed **Redis timelines** for normal users, **pulled +
merged at read time** for celebrities to avoid the fan-out storm. Reads fetch the precomputed post_id
list, hydrate + rank, and page via cursors. Key trade-offs: push (fast reads, write storm) vs pull (cheap
writes, slow reads) → **hybrid**; eventual consistency for availability; cap + evict timelines. With more
time: ML ranking (m26), media via blob+CDN (m18), and the social-graph subsystem."*

---

## State-of-the-art & real-world notes 📚
- **Twitter** pioneered the **hybrid fan-out** ("fan-out on write with a pull path for high-follower
  accounts"); their timeline lived in Redis. **Instagram** and **Facebook** use similar push/pull hybrids
  with heavy ML ranking. **Facebook's feed** famously moved from chronological (EdgeRank) to ML-ranked.
- **The general principle** — *precompute the common case, compute-on-demand the expensive tail* — recurs
  everywhere (it's the m04 denormalization / m06 caching trade applied to feeds).
- **Time-sortable IDs (Snowflake)** make timeline merging/ordering cheap (sort by ID = sort by time).

---

## From your systems 🏭
- **Precomputed timeline = your snapshot tables:** Questimate **precomputes risk into snapshot tables for
  fast reads** — *identical* pattern to fan-out-on-write (do the expensive merge/compute at write time so
  reads are O(1)). You've built this trade-off in production; the feed is the same idea.
- **Async fan-out via Kafka:** distributing a post to millions of timelines is your **Kafka decoupling**
  (m08) — producer (post) doesn't block on the heavy fan-out work; consumers (fan-out workers) drain it.
- **Hot-key / celebrity instinct:** the celebrity problem is the m05/m06 hot-key thread — your reflex to
  cache/replicate/special-case the hot entity transfers directly (and the "hybrid: special-case the tail"
  is the senior move).
- **Redis sorted sets:** you use Redis for hot state; per-user timelines as capped sorted sets is the same
  tool.

---

## Key concepts (interview-ready)
- **It's a read-optimization problem** dominated by the **fan-out** decision; the hard part is
  **follower-count skew (celebrities)**.
- **Fan-out on write (push):** precompute each follower's timeline on post → **fast reads, expensive
  writes**; **catastrophic for celebrities** (fan-out storm). **Fan-out on read (pull):** merge followees'
  posts at read time → **cheap writes, slow reads**.
- **The hybrid (the answer):** **push for normal users, pull+merge for celebrities** — follower-count
  threshold decides. This is how Twitter/Instagram solve the celebrity problem.
- **Timeline = a materialized/denormalized view** (m04/m06) in Redis (capped sorted set of **post_ids**;
  hydrate content separately). Async **fan-out via Kafka** (m08), idempotent.
- **Eventual consistency** (AP, m02) — posts appear in seconds; give authors **read-your-writes**.
- **Cursor (keyset) pagination, not offset.** **Time-sortable IDs** (Snowflake) for ordering.
- **Ranking** (chronological → **ML-ranked**) is an extension (m26). **Social graph** is its own sharded
  subsystem.

---

## Go deeper (reading)
- **Twitter: "The Infrastructure Behind Twitter: Scale" / timeline (fan-out) talks** — the canonical
  hybrid fan-out source.
- **Instagram engineering** posts on feed + sharded IDs; **"Facebook's TAO"** (the social-graph store).
- **System Design Interview (Alex Xu): "Design a news feed"** — the standard interview treatment.
- Revisit **m04** (denormalization), **m06** (caching/hot keys), **m08** (async fan-out), **m05**
  (sharding/celebrity hot key) — this problem is those applied; **m26** adds ML ranking.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) (Instagram,
ranking, the social graph, scale) before checking
[`solutions/m13_news_feed/`](../../solutions/m13_news_feed/README.md).

When you've done the exercises, say **"Module 14"** to design a *Chat System (WhatsApp/Slack)* —
WebSockets, presence, delivery/read receipts, message ordering — where the **stateful persistent
connection** challenge (m03/m07) becomes the central problem.
