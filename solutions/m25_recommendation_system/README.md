# m25 — Solutions (Recommendation System)

> Check the *reasoning*. Recurring lessons: the funnel (recall→precision→business logic); two-tower
> precomputes item embeddings for ANN retrieval; cold start via content/exploration; A/B on engagement.

---

## Exercise 1 — Why a funnel?
1. Running a heavy, feature-rich ranking model on **50M items per request** would take **far more than 200
   ms** (and huge compute) — it's infeasible at that scale × latency.
2. **Candidate generation** (recall, cheap, **millions → ~hundreds**) → **ranking** (precision, expensive,
   **hundreds → ~dozens**) → **re-ranking** (business logic, dozens → top-k).
3. **m16** (typeahead/search): **retrieve** cheaply (BM25 + vector ANN) then **rerank** with a
   cross-encoder — the same retrieve-then-rerank shape (also RAG, AI/ML m06–m07).

---

## Exercise 2 — Two-tower retrieval
1. A **user tower** (user features/history → **user embedding**) and an **item tower** (item features →
   **item embedding**), trained so a user is close to items they engage with in a shared space.
2. **Offline (precomputed):** all **item embeddings** → indexed in a vector DB. **Online:** the **user
   embedding** (one tower pass). The key trick: you **don't run a model over every item per request** —
   you embed the user once and do a **vector lookup** against precomputed item embeddings.
3. An **Approximate Nearest Neighbor (ANN)** index (**HNSW/IVF**) in a **vector DB** — nearest-neighbor
   search returns the closest items in ~ms over millions.

---

## Exercise 3 — Candidate sources
1. **Collaborative filtering** — strength: captures taste from interactions (no content needed); weakness:
   **cold start** (needs history).
2. **Content-based** — strength: works for **new items** + explainable; weakness: **over-specialization**
   (only similar items, no discovery).
3. **Two-tower (embedding retrieval)** — strength: learned, scalable ANN retrieval over the whole catalog,
   blends content + interaction signal; weakness: needs training data + infra, and embeddings can go stale.

---

## Exercise 4 — Ranking
1. Predicts **P(engagement)** (click/watch/purchase) over the **~hundreds** of candidates retrieval
   returned.
2. Because it runs on **hundreds, not millions** — so it can afford a **rich, feature-heavy, expensive**
   model (cross features, deep net) within the latency budget.
3. **Multi-objective ranking** predicts **several** signals (click, long-watch, share, not-interested) and
   **combines them** — used to avoid optimizing clicks alone (clickbait) and instead balance **healthy
   long-term engagement**.

---

## Exercise 5 — Re-ranking
1. **Diversity** (avoid near-duplicates), **freshness** (surface new content), **dedup**, **filtering**
   (already-seen/ineligible), and **business rules** (sponsored/policy) — any four.
2. Pure relevance gives a **monotonous, repetitive, gameable** list (10 near-identical items) that hurts
   long-term engagement and discovery.
3. **Per-item** optimizes each item's relevance independently; **list-as-a-whole** optimizes the **set**
   (diversity, complementarity, constraints) — re-ranking is the list-level stage.

---

## Exercise 6 — Cold start
1. **New user:** recommend by **popularity/trending**, **onboarding** (ask preferences), demographic
   priors, and **exploration**; the **user embedding sharpens** as they interact and generate signal.
2. **New item:** use **content-based** features (the **item tower** produces an embedding from content
   alone, so it's retrievable) and **boost it into exploration** so it earns interaction data.
3. Two-tower helps because the **towers consume features/content**, not just IDs — so a brand-new item
   (or user) still gets a usable embedding from its content/features, unlike pure ID-based CF which has no
   vector for unseen IDs.

---

## Exercise 7 — Feedback loops & exploration
1. Recommendations **train the next model**, but you **only get feedback on what you showed** → the model
   **narrows into a filter bubble**, entrenching its own choices and never learning about un-shown items.
2. **Exploration** = deliberately showing some **non-top items** to gather unbiased data + surface new
   content; techniques: **ε-greedy** and **multi-armed / contextual bandits**.
3. **Position bias:** users click **top-ranked items regardless of true relevance** (because they're shown
   first), so logs say "top items are great" even when a lower item was better → naive training **learns
   position, not relevance**, reinforcing the existing ranking.

---

## Exercise 8 — Evaluation
1. **Not necessarily** — better offline NDCG doesn't prove better engagement; offline is a **proxy** (stale
   data, not the business metric, can't capture behavior change). Validate online.
2. An **online A/B test on the business metric** (watch time / retention / revenue), with significance
   (shadow/canary first).
3. **Position bias, popularity bias, and the feedback-loop/filter-bubble** (data generated by the model's
   own choices) — all make offline replay misleading.

---

## Exercise 9 — Serving & scale
1. **Offline:** item-tower → **item embeddings** → **vector DB (ANN index)**; (optionally) precompute full
   recommendation lists for low-activity users. **Online:** user-tower → **user embedding** → **ANN
   retrieval** → **ranking** (heavy model on hundreds) → **re-rank** → top-k. Cache user embeddings /
   candidate sets.
2. **Shard + replicate the vector DB / ANN index** (m18-style) so nearest-neighbor search stays fast over
   millions/billions (HNSW/IVF = sub-linear search); incrementally re-index new items; scale ranking via
   horizontal model-server replicas (m24).
3. **Precompute (batch)** for **less-active users / stable contexts** (cheap, low-latency lookup, stale);
   **compute online** for **active users / fresh-context** where recency + live session signals matter.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **Retrieval = your embeddings (AI/ML m05) + vector DB (Chroma/pgvector, HNSW/IVF, m06)** — two-tower
   item embeddings indexed for ANN search; **ranking = your cross-encoder reranking (m06)** — an expensive
   model scoring the retrieved few precisely.
2. **Same retrieve-then-rerank funnel:** **hybrid search** (BM25 + vector retrieve → cross-encoder rerank)
   and **RAG** (retrieve chunks → LLM) are the identical recall-then-precision pattern — you've built this
   funnel three times (search, RAG, now recsys), just different final stages.
3. **Sennzo "recommended movies":** **offline** — embed all 500k+ titles (item tower over genre/cast/
   synopsis/embeddings) → index in a vector DB; **online** — build the user's embedding from their watch/
   rating history → **ANN retrieve** ~hundreds of candidate movies → **rank** by P(watch) with user×movie
   features → **re-rank** for **diversity/freshness** + filter already-watched → show top-k. Cold start:
   trending + onboarding genres + content-based for new titles; explore to learn; **A/B test on watch
   time**. (It layers right on top of your existing Sennzo search/filter catalog.)
