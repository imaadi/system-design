# m25 — Cross-Questions ("if-and-buts") on the Recommendation System

> Answer out loud in 2–3 sentences before reading the model answer. The multi-stage funnel, two-tower
> retrieval, and cold start are where this is won.

---

### Q1. You have millions of items and a 200 ms budget. How do you recommend?
**A.** You **cannot score every item per request**, so a **multi-stage funnel**: **(1) candidate
generation / retrieval** narrows millions → hundreds **cheaply with high recall** (e.g. two-tower + ANN);
**(2) ranking** precisely orders those hundreds with an **expensive, feature-rich model** predicting
P(engagement); **(3) re-ranking** applies diversity/freshness/business rules. Each stage handles **more
items more cheaply, then fewer items more precisely** — the latency × scale constraint forces this
staging.

---

### Q2. Why split retrieval and ranking instead of one model?
**A.** Because they optimize **different things at different scales**. **Retrieval** must be **cheap and
high-recall** over **millions** (don't miss good items) — so it's approximate (an ANN lookup). **Ranking**
can be **expensive and high-precision** because it only runs on the **hundreds** retrieval returned — so it
uses a rich, cross-feature model. One model can't be both cheap-over-millions and precise — you'd either
blow the latency budget or use a weak model. It's the same **retrieve-then-rerank** pattern as hybrid
search (m16).

---

### Q3. What is a two-tower model and why is it the retrieval workhorse?
**A.** Two neural encoders — a **user tower** (user features/history → user embedding) and an **item
tower** (item features → item embedding) — trained so a user and the items they engage with are **close in
the same embedding space**. It's the retrieval workhorse because you can **precompute all item embeddings
offline** and **index them in a vector DB (ANN)**, then at request time just **embed the user** and do a
**nearest-neighbor search** — retrieval over millions becomes a fast **ANN lookup**, not a model over every
item. It decouples expensive item encoding (offline) from the fast user query (online).

---

### Q4. Walk through serving a two-tower retrieval at request time.
**A.** **Offline:** run the **item tower** over the catalog → **item embeddings** → **index them in a
vector DB (HNSW/IVF ANN)**; re-index as items are added/changed. **Online:** run the **user tower** on the
user's features/history → a **user embedding** → **ANN nearest-neighbor search** against the item index →
the **top-N closest items** (hundreds) in ~ms. Those candidates go to ranking. The user embedding can be
cached and refreshed as behavior changes. *This is exactly embeddings + a vector DB + HNSW — my AI/ML
work.*

---

### Q5. What's the difference between collaborative filtering and content-based recommendation?
**A.** **Collaborative filtering** uses the **user-item interaction matrix** — "users who liked X also
liked Y" (e.g. **matrix factorization** learns latent user/item vectors). It's powerful but needs
interaction history → **weak on cold start**. **Content-based** recommends items **similar to ones the user
liked**, by **item features/embeddings** (genre, text, image) — it works for **new items** (uses content,
not interactions) but can over-specialize (only ever shows similar things). Real systems **combine** them
(and two-tower towers can ingest both interaction signals and content features).

---

### Q6. What does the ranking stage actually do?
**A.** It **precisely orders the few hundred candidates** from retrieval by predicting **P(engagement)** —
P(click), P(watch-time), P(purchase) — with a **heavy, feature-rich model** (GBDT or deep net) using
**user, item, context, and cross features** (user×item interactions). It can afford to be expensive because
it runs on **hundreds, not millions**. This is **learning-to-rank / CTR prediction** (m26), and modern
systems are **multi-objective** (blend click vs long-watch vs "not interested" into one score). It's the
**precision** stage — the analog of my cross-encoder reranking.

---

### Q7. What does re-ranking add on top of the ranking score?
**A.** **Business logic the pure relevance score misses:** **diversity** (don't show 10 near-identical
items — a monotonous, gameable list; use MMR/diversity penalties), **freshness** (surface new content),
**dedup**, **filtering** (already-seen/purchased, blocked, ineligible items), and **business rules**
(sponsored/promoted, policy, fairness). Ranking optimizes per-item relevance; re-ranking optimizes the
**list as a whole** (and enforces constraints). Without it, you get a relevant-but-repetitive feed that
hurts long-term engagement.

---

### Q8. What is the cold-start problem and how do you handle it?
**A.** New entities have **no interaction signal** to recommend from. **New user** (no history): fall back
to **popularity/trending**, **onboarding** (ask preferences), demographic priors, and **exploration** —
the user embedding sharpens as they interact. **New item** (no interactions): use **content-based**
features (the item tower produces an embedding from content alone) and **boost it into exploration** so it
earns interaction data. Two-tower helps because the **towers use features/content**, not just IDs — so a
brand-new item still gets a usable embedding.

---

### Q9. What's the feedback loop danger in recommenders?
**A.** Recommendations **train the next model** (recommend → user interacts → those interactions become
training data), but **you only get feedback on what you showed** — so the model **narrows** into a **filter
bubble / echo chamber** and never learns about un-shown items. It can entrench its own biases and reduce
discovery. The fix is **exploration**: deliberately show some **non-top items** (**ε-greedy / multi-armed
bandits**) to gather unbiased data and surface new content — the **exploration-exploitation** trade is core
to recsys.

---

### Q10. Why isn't offline NDCG enough to ship a new recommender?
**A.** Because offline ranking metrics (NDCG, MAP, recall@k) are **proxies** for engagement, not engagement
itself (m22). Offline data is **stale**, the metric isn't the business metric, and a recommender **changes
user behavior** (it surfaces different items → different future interactions → which offline replay can't
capture). So a model with **better offline NDCG can engage users *less*** online. The only real proof is an
**online A/B test on the business metric** (watch time/retention), with shadow/canary first (m24).

---

### Q11. What is position bias and why does it corrupt your training data?
**A.** Users **click the top items regardless of true relevance** simply because they're shown first
(position bias) — so your interaction logs say "top items are great," even if a lower item was actually
better. If you train naively on clicks, the model **learns position, not relevance**, and entrenches
whatever it already ranked high (a feedback loop). You correct for it with **position-debiasing** (model
the click as P(relevance) × P(examined|position), inverse-propensity weighting) and **exploration**
(randomize positions sometimes to get unbiased signal). It's a top reason "clicks ≠ relevance."

---

### Q12. How do you serve a recommender within the latency budget?
**A.** The **funnel is the latency solution**, served as **hybrid batch + online** (m24): **item
embeddings precomputed offline + indexed in a vector DB**, so retrieval is an **ANN lookup (~ms)**; the
**user embedding + ranking** run **online** over only hundreds of candidates (~tens of ms); re-ranking is
cheap. You **cache** user embeddings and candidate sets (m06), and for **less-active users you can
precompute** the whole recommendation list (batch) and serve by lookup, computing **online** only for
active users. Each stage is sized to fit its slice of the 200 ms.

---

### Q13. How do you scale retrieval to millions/billions of items?
**A.** The **vector DB / ANN index** (m18-style) is **sharded and replicated** so nearest-neighbor search
stays fast over millions of item embeddings (HNSW/IVF give sub-linear search — your m06 work: "recall@10
0.99 at ~4% scanned"). Item embeddings are **precomputed in batch** and **incrementally re-indexed** as
the catalog changes. The ranking service scales like any model server (m24 — stateless replicas,
batching). So retrieval scales via ANN + sharding; ranking scales via the funnel (only hundreds per
request) + horizontal replicas.

---

### Q14. How is this the same shape as your hybrid search / RAG work?
**A.** **Identical retrieve-then-rerank funnel.** In hybrid search (m16 / my AI/ML m06): **retrieve**
candidates cheaply (BM25 + vector ANN, high recall) → **rerank** the top few with an expensive
**cross-encoder** (high precision). In recsys: **retrieve** candidates (two-tower + ANN, high recall) →
**rank** with a heavy model (high precision) → re-rank. Same pattern, same recall-vs-precision split, same
"cheap-over-many then expensive-over-few." And in **RAG**, retrieval feeds an LLM — so I've built this
funnel three times in different domains.

---

### Q15. A user follows accounts but you want to recommend *new* content they don't follow. How?
**A.** That's the **exploration / discovery** side — you can't only recommend from their explicit follows
(filter bubble). You blend **multiple candidate sources**: collaborative filtering ("users like you
enjoyed…"), content similarity to what they engaged with, **trending/popular** in their segment, and
**exploration** (some genuinely new items via bandits) — then **rank** them together. The two-tower
embedding naturally surfaces items *similar to their taste but unseen*, and re-ranking injects **diversity/
freshness** so the feed isn't just more of the same. Discovery is a deliberate design goal, not an
accident.

---

### Q16. How do you keep recommendations fresh as user interests and content change?
**A.** **Refresh embeddings + features continuously:** re-index **new item embeddings** as content is added
(so new items are retrievable), and **update the user embedding** from recent behavior — often via
**streaming features** (m23: "items clicked this session") so the recommendation reflects the current
session, not last week. You may recompute the user tower on recent interactions and blend in **real-time
signals** at ranking. Freshness is a feature-pipeline + re-indexing concern; staleness (recommending what
they already moved past) is a common failure you monitor.

---

### Q17. What's multi-objective ranking and why do modern feeds use it?
**A.** Instead of optimizing **one** signal (e.g. clicks), the ranker predicts **several** (P(click),
P(long-watch), P(share), P(not-interested)) and **combines them into one score** with weights tuned to the
**business goal**. Modern feeds use it because optimizing clicks alone produces **clickbait** (high clicks,
low satisfaction/retention) — you want **healthy long-term engagement**, so you balance immediate clicks
against dwell time, diversity, and negative feedback. It's how you avoid the "engagement-maximizing feed
that users come to resent." The weights themselves are a product/policy decision.

---

### Q18. What are the main biases/failure modes in recommenders?
**A.** **Position bias** (clicks reflect rank, not relevance), **popularity bias** (popular items dominate,
the long tail is starved), **feedback-loop / filter-bubble** (you only learn about what you showed →
narrowing), **cold start** (new users/items), and the **offline-online eval gap** (NDCG ≠ engagement). They
share a root: the **data is generated by the model's own choices**, so it's biased toward those choices.
Defenses: **exploration**, **debiasing** (inverse-propensity weighting), **diversity** in re-ranking, and
**online A/B testing** rather than trusting offline replay.

---

### Q19. When would you precompute recommendations vs compute them online?
**A.** **Precompute (batch)** for **less-active users** or stable contexts — compute their recommendation
list nightly and serve by lookup (cheap, low-latency reads, but stale). **Compute online** for **active
users / fresh context** where recency and live signals matter (the session, the current query). Many
systems do **both**: a batch baseline for the long tail + online computation for engaged users — the same
**batch-vs-online + hybrid** decision from m22/m24. You trade freshness for cost per user segment.

---

### Q20. How do all the pieces (m22–m24) come together here?
**A.** Recsys is the **integration point** of Part 3: **framing** (predict P(engagement), label = implicit
feedback, business metric = engagement — m22); **features** (user/item/context features from the feature
store, streaming for freshness — m23); **serving** (hybrid batch+online, ANN retrieval + online ranking,
the funnel for latency — m24); plus the **ranking model** (m26) and **monitoring/feedback** (m28). And the
**retrieval + rerank** is my embeddings + vector-DB + cross-encoder work made into a production funnel —
which is exactly why my AI/ML background is the differentiator for ML system design rounds.
