# m13 — Solutions (News Feed / Twitter Timeline)

> Check the *reasoning*. Recurring lessons: feeds are read-heavy → precompute timelines (push); the
> celebrity skew → hybrid (pull for the tail); timeline = materialized view of post_ids; eventual
> consistency.

---

## Exercise 1 — Estimation & fan-out
200M DAU, 3 posts, 15 reads, 300 followers.
1. **Posts:** 200M × 3 = 600M/day ÷ 10⁵ = **~6,000/sec** avg (~30k peak). **Reads:** 200M × 15 = 3B/day
   ÷ 10⁵ = **~30,000/sec** avg (~150k peak). Read-heavy ~5:1 on actions (more per-read work too).
2. **Fan-out on write:** 600M posts × 300 followers = 180B timeline writes/day ÷ 10⁵ = **~1.8 million
   timeline writes/sec** (avg).
3. **Celebrity (50M followers), one post → 50,000,000 timeline writes** for that single tweet.
4. (2) shows precompute is heavy but feasible *on average* (shard the timeline cache + async fan-out);
   (3) shows it's **catastrophic for celebrities** (a single post = 50M writes) → you **cannot** push for
   them → you need a **hybrid** (pull for high-follower accounts). Reads dominating justifies precompute;
   the celebrity tail forces the hybrid.

---

## Exercise 2 — Fan-out trade-off
1. **Write (push):** *pro* — reads are O(1) (precomputed); *con* — writes cost N (follower count), wasteful
   for inactive followers, **celebrity storm**. **Read (pull):** *pro* — one cheap write per post; *con* —
   reads are expensive (query + merge all followees every scroll), hard to cache.
2. **Push (fan-out on write)** is the default because feeds are **read-heavy** — you optimize the frequent
   operation (reads) by precomputing, paying at the rarer write.
3. It breaks for **celebrities** because push cost scales with follower count → millions of writes per
   post (a fan-out storm) overwhelms the system and delays delivery.

---

## Exercise 3 — The hybrid
1. **Hybrid:** **push** posts to followers' precomputed timelines for **normal users** (fast reads), but
   **don't push** for **celebrities** — store their posts and **pull + merge** them at read time.
2. A normal user's feed = **their precomputed timeline (pushed by normal followees)** + **recently-pulled
   posts of the few celebrities they follow**, merged and ranked.
3. **Follower count** (above a threshold → treat as celebrity → pull path).

---

## Exercise 4 — IDs vs full posts
1. **Post IDs.** Reasons: (a) storing full posts in millions of timelines **duplicates content** and wastes
   memory; (b) **edits/deletes** would require updating every copy — with IDs you just re-hydrate current
   content; (c) IDs are tiny → small, cheap timeline caches.
2. **Hydration** = at read time, take the timeline's post_ids and **batch-fetch the actual post content**
   (cache-first) from the tweet store before returning.
3. Because the timeline holds only a **reference (ID)**, a deleted post simply isn't found on hydration
   (or returns a tombstone), and an edited post returns its **current** content — no need to touch the
   millions of timelines that reference it.

---

## Exercise 5 — Consistency & pagination
1. **Eventual consistency** (AP) — a post appearing in followers' feeds seconds late is fine; availability
   > strict freshness. The nicety: **read-your-own-writes** — inject the author's new post into *their*
   view immediately.
2. **Offset is wrong** because it's O(n) and **unstable** — new posts arriving at the top shift everything,
   so page 2 repeats/skips items. Use **cursor/keyset** pagination ("items after `<last_id/timestamp>`"),
   which is **stable** (top insertions don't affect it) and efficient (indexed range scan).

---

## Exercise 6 — Write and read paths
1. **Post (normal user):** client `POST /posts` → post service writes the tweet to the **sharded tweet
   store** (time-sortable ID) → emits a **fan-out event to Kafka** → **fan-out workers** look up the
   author's followers and **append the post_id to each follower's Redis timeline** (idempotent, skip
   inactive). Author sees their own post immediately. Steps after the store-write are **async**.
2. **Open feed:** timeline service reads the user's **precomputed post_id list** from Redis → **merges in**
   recent posts of celebrities they follow (pull) → **hydrates** post_ids to content (batch, cache-first)
   → **ranks** (chrono/ML) → returns a **cursor-paginated** page.
3. The **async boundary** is right after the tweet is durably stored — **fan-out is async** (Kafka) so the
   poster doesn't wait on potentially millions of timeline writes; delivery happens in the background.

---

## Exercise 7 — Ranking & social graph
1. **Candidate generation** (gather recent posts from followees + injected celebrity posts) → **ranking**
   (score each by predicted engagement/recency/affinity) → **re-ranking** (dedupe, diversity, business
   rules) → page.
2. Store the graph as **adjacency lists** sharded by user_id; keep **two** lists per user: **followers**
   (for fan-out on write — who to push to) and **following** (for fan-out on read — whose posts to pull).
3. **Fan-out on write** needs the author's **followers** list (to push to); **fan-out on read** needs the
   reader's **following** list (to pull from).

---

## Exercise 8 — Failure & idempotency
1. The **append must be idempotent** — fan-out runs on **at-least-once** delivery (m08), so a message may
   be reprocessed; appending must dedupe by post_id (e.g. a sorted set keyed by post_id won't double-
   insert) so a redelivery doesn't put the post in a timeline twice.
2. The in-flight Kafka message wasn't acked, so it's **redelivered** to another worker and reprocessed —
   **safe** because the append is idempotent (already-inserted post_ids are no-ops).
3. **Fan-out lag** — how far behind real-time delivery is (the consumer lag, m08). Rising lag means
   delivery is backing up (e.g. a celebrity storm or under-scaled workers) and posts are appearing late.

---

## Exercise 9 — Instagram's feed
1. **Same:** hybrid fan-out, precomputed timelines of IDs, ML ranking. **Big addition:** it's
   **media-centric** (images/video), so a **blob storage + CDN** tier dominates content storage/delivery.
2. **Media lives in object storage (S3/blob)**; the timeline still stores post IDs, and hydration returns
   **media URLs served from a CDN/edge** (m06/m18).
3. Because you're serving **large images/video** (not a tiny text payload), **read bandwidth** is large and
   latency-sensitive → it must be served from the **edge (CDN)**, unlike a text feed whose payload is
   trivial.

---

## Exercise 10 — MVP → scale + your-systems tie-in 🏭
1. **Evolution:** **MVP = fan-out on read (pull)** (simplest — one write per post, compute feed at read);
   → as reads slow, **add fan-out on write (push)** with precomputed Redis timelines (fast reads); → as
   celebrities appear, **hybrid** (pull for high-follower accounts); → then **ML ranking** and **media/
   CDN**. Match complexity to scale (m01) rather than over-engineering up front.
2. **Precomputed timeline = Questimate's snapshot tables** (do the expensive merge/compute at write time so
   reads are O(1) — same materialized-view trade). **Async fan-out = DCE's Kafka decoupling** (producer
   doesn't block on heavy downstream work; consumers drain it, m08).
3. The celebrity problem **is** the hot-key thread (m05/m06): one entity gets disproportionate write
   (fan-out) and read (views) load → the reflex that carries over is **special-case the hot entity** —
   cache/replicate it and treat the tail differently (here: pull instead of push, and hard-cache its
   posts), i.e. the "hybrid: handle the skewed tail separately" instinct.
