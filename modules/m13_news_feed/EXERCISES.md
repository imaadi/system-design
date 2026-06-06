# m13 — Exercises (News Feed / Twitter Timeline)

> Do these on paper / out loud before checking `solutions/m13_news_feed/`. The fan-out decision and the
> celebrity problem are the core; later ones are extensions.

---

## Exercise 1 — Estimation & the fan-out number
Assume **200M DAU**, **3 posts/user/day**, **15 timeline reads/user/day**, avg **300 followers**.
1. Posts/sec and timeline-reads/sec (avg + peak at 5×)?
2. If you fan out on **write**, how many timeline writes/sec on average?
3. A celebrity with **50M followers** posts once. How many writes does that single post generate?
4. What do (2) and (3) tell you about the architecture?

---

## Exercise 2 — Fan-out trade-off
1. State the pro and con of fan-out on **write** and fan-out on **read**.
2. For a read-heavy feed, which is the default and why?
3. Why does the default break for celebrities?

---

## Exercise 3 — The hybrid
1. Describe the hybrid fan-out and which users get which path.
2. What does a normal user's assembled feed consist of under the hybrid?
3. What single attribute decides whether an account is push or pull?

---

## Exercise 4 — IDs vs full posts
1. Do you store full posts or post IDs in each timeline? Give three reasons.
2. What does "hydration" mean in the read path?
3. How does storing IDs make edits/deletes easy?

---

## Exercise 5 — Consistency & pagination
1. What consistency model does the feed need, and what's the one nicety you add for authors?
2. Why is offset pagination wrong for a feed? What do you use instead, and why is it stable?

---

## Exercise 6 — The write and read paths
1. Trace step-by-step what happens when a **normal** user posts (include Kafka and async).
2. Trace step-by-step what happens when a user **opens their feed** (include celebrity merge, hydration,
   ranking, paging).
3. Where does the async boundary sit and why?

---

## Exercise 7 — Ranking & the social graph
1. Sketch the candidate-generation → ranking → re-ranking pipeline for a ranked feed (one line each).
2. How do you store the social graph, and what two adjacency lists do you keep per user (and why two)?
3. Which graph query is hot during fan-out on write vs fan-out on read?

---

## Exercise 8 — Failure & idempotency
1. Fan-out runs on at-least-once delivery. What must be true of the append operation, and why?
2. A fan-out worker crashes mid-batch. What happens, and is it safe?
3. What's your alerting signal that delivery is falling behind?

---

## Exercise 9 — Extension: design Instagram's feed
1. What's the same as Twitter, and what's the big addition?
2. Where does the media live, and what tier serves it to users?
3. Why does read bandwidth matter more here than for a text feed?

---

## Exercise 10 — MVP → scale + your-systems tie-in 🏭
1. Describe a sensible evolution: MVP → fast reads → celebrities → ranking. Which fan-out at each stage?
2. Map the **precomputed timeline** to a pattern you built in **Questimate**, and the **async fan-out**
   to a pattern from **DCE**.
3. Why is the celebrity problem the same as the hot-key thread from m05/m06, and what reflex carries
   over?

---

When done, check `solutions/m13_news_feed/README.md`, then say **"Module 14"**.
