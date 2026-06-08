# Module 25 — Design a Recommendation System ⭐⭐ 🏭

> Recommend items (videos, products, posts) a user will engage with — from **millions of candidates, in
> <200 ms.** The headline insight: **you can't score millions of items per request, so you use a
> multi-stage funnel** — cheap high-**recall** retrieval narrows millions → hundreds, then an expensive
> high-**precision** ranker orders them. The retrieval workhorse is the **two-tower model + ANN search**
> — which is **literally your embeddings + vector-DB work** (AI/ML m05–m06), and the retrieve-then-rerank
> shape is **your hybrid-search pattern** (m16). This is where serving (m24), features (m23), and ranking
> all come together.

> **Format (Part 3 deep-dive):** worked design; the **multi-stage funnel**, **two-tower retrieval**, and
> **cold start** are the deep-dives.

---

## Step 1 — Requirements

**Functional:** given a **user + context**, return the **top-k items** they're most likely to engage with,
ranked. (Homepage feed, "recommended for you", "up next".)

**Non-functional (these drive it):**
- **Low latency:** the whole pipeline in **<~200 ms** (it's on the page-load path).
- **Massive scale:** **millions of items × millions of users** — you **cannot** score every item per
  request.
- **Personalization + relevance + diversity + freshness:** relevant to *this* user, not monotonous, and
  surfacing new content.
- **The business metric:** **engagement** (watch time, clicks, purchases, retention) — measured **online**
  (m22), with offline ranking metrics (NDCG) as a proxy.
- **Handle cold start** (new users/items with no history).

> **The defining insight:** the latency × scale constraint (millions of items, <200 ms) **forces a
> multi-stage funnel** — you can't run a heavy model over millions of items, so you stage it. *(Same
> retrieve-then-rerank shape as your hybrid search, m16.)*

---

## Step 2 — The multi-stage funnel ⭐⭐ (THE core architecture)

You progressively **narrow + refine**: each stage handles **more items, more cheaply**; later stages
handle **fewer items, more precisely.**

```
   millions of items
        │  ① CANDIDATE GENERATION / RETRIEVAL  (cheap, high RECALL)
        ▼     millions → ~hundreds          e.g. two-tower + ANN, collaborative filtering
   ~hundreds of candidates
        │  ② RANKING  (expensive, high PRECISION)
        ▼     hundreds → ~dozens            a heavy model scores each: P(engage)
   ~dozens, ordered
        │  ③ RE-RANKING  (business logic)
        ▼     diversity, freshness, dedup, filtering, business rules
   top-k  →  shown to the user
```

> **The senior framing:** *"You can't score millions of items in 200 ms, so a **funnel**: a **cheap,
> high-recall retrieval** stage narrows millions to hundreds (don't miss good items), then an
> **expensive, high-precision ranking** model orders those few, then **re-ranking** applies diversity/
> business rules. **Retrieval optimizes recall over many; ranking optimizes precision over few** — the
> same retrieve-then-rerank pattern as hybrid search (m16)."*

---

## Step 3 — Deep Dive: candidate generation / retrieval ⭐ (recall over millions)
Get **hundreds of decent candidates** from millions, **fast**. Approaches (often **combined** — multiple
sources):
- **Collaborative filtering (CF):** "users who liked X also liked Y" — from the **user-item interaction
  matrix**. Classic: **matrix factorization** (learn user + item latent vectors; recommend by dot
  product). Powerful but needs interaction history (cold-start weak).
- **Content-based:** recommend items **similar to ones the user liked**, by item **features/embeddings**
  (genre, text, image). Works for new items (uses content, not interactions).
- **Two-tower model ⭐⭐ (the modern retrieval workhorse):** two neural encoders — a **user tower**
  (user features/history → a user embedding) and an **item tower** (item features → an item embedding) —
  trained so a user and the items they engage with are **close in the same embedding space**.
  - **Serving:** **precompute all item embeddings** (offline, batch) and **index them in a vector DB
    (ANN: HNSW/IVF)**; at request time, **embed the user** (online, one tower pass) and do an **ANN
    nearest-neighbor search** for the closest items → hundreds of candidates in **~ms**.
  - **Why it's brilliant:** it **decouples** the expensive item encoding (offline, once) from the fast
    user query (online), so retrieval over millions is a **vector ANN lookup**, not a model over every
    item. *This is exactly your embeddings + vector DB + HNSW/IVF work (AI/ML m05–m06).*

```
  Two-tower:  [user tower] → user_emb ─┐
                                        ├─ ANN search in vector DB of PRECOMPUTED item_embs → top-N items
              [item tower] → item_emb ─┘  (item embeddings indexed offline; user embedded online)
```

---

## Step 4 — Deep Dive: ranking ⭐ (precision over hundreds)
Now **precisely order the few hundred candidates** with a **heavier model** and **rich features**:
- A model (GBDT or deep net) **scores each candidate** for **P(engagement)** — P(click), P(watch-time),
  P(purchase) — using **user features, item features, context (time/device), and cross features**
  (user×item interactions). This is **learning-to-rank / CTR prediction** (deep-dived in **m26**).
- **Why a separate stage?** Retrieval is cheap/approximate over millions; ranking can afford an
  **expensive, feature-rich, precise** model because it only runs on **hundreds**. (Mirrors m16's
  BM25-retrieve + **cross-encoder rerank** — exactly your reranking work.)
- Often **multi-objective:** balance several predictions (a click vs a long watch vs a "not interested")
  into one score — modern feeds optimize a blend, not one metric.

---

## Step 5 — Deep Dive: re-ranking & cold start

### Re-ranking (business logic on the final list)
Pure relevance gives a **monotonous, gameable** list. Re-ranking adjusts the top items for:
- **Diversity** (don't show 10 near-identical items — MMR/diversity penalties), **freshness** (surface
  new content), **dedup**, **filtering** (already seen/purchased, blocked, ineligible), and **business
  rules** (promotions, sponsored, policy).

### Cold start ⭐ (the classic recsys problem)
New entities have **no interaction signal**:
- **New user** (no history): fall back to **popularity** (trending), **content/onboarding** (ask
  preferences), demographic priors, and **exploration**. The user embedding improves as they interact.
- **New item** (no interactions): **content-based** (use its features/embeddings — the item tower works
  from content!), boost into exploration so it earns interaction data.
> Two-tower helps cold start because the **towers use features/content**, not just IDs — a brand-new item
> still gets an embedding from its content.

---

## Step 6 — Feedback loops, exploration & evaluation ⭐
- **The feedback loop (data flywheel + filter bubble):** recommend → user interacts → those interactions
  **train** the next model. But **you only get feedback on what you showed** → the model **narrows**
  (echo chamber / filter bubble) and never learns about un-shown items. **Fix: exploration** — deliberately
  show some non-top items (**ε-greedy / multi-armed bandits**) to keep discovering and gathering unbiased
  data. The **exploration-exploitation** trade is core to recsys.
- **Evaluation (the m22 gap):** offline **ranking metrics** (NDCG, MAP, recall@k) are **proxies**; a model
  with better offline NDCG can **engage users less** online (it changes behavior). The truth is an
  **online A/B test on the business metric** (engagement/retention). Shadow/canary first (m24).

---

## Step 7 — Serving, scale & bottlenecks
- **Serving (hybrid batch + online, m24):** **item embeddings precomputed** (batch) + **indexed in a
  vector DB**; **user embedding computed online**; **ANN retrieval** → **online ranking** → re-rank. Some
  systems **precompute** recommendations for less-active users (batch) and compute **online** for active
  ones.
- **Latency:** the funnel *is* the latency solution — retrieval ~ms (ANN), ranking ~tens of ms (hundreds
  of items), re-rank cheap. Cache user embeddings / candidate sets where possible (m06).
- **Scale:** the vector DB (m18-style ANN) shards/replicates for millions of items; the ranking service
  scales like any model server (m24).
- **Freshness:** re-index new item embeddings continuously; refresh user embeddings as behavior changes
  (streaming features, m23).
- **Bottlenecks:** cold start, feedback loops/filter bubbles, the offline-online eval gap, position bias
  (users click top items regardless → biases training data), and popularity bias (popular items dominate).
- **Observability (m10/m28):** engagement metrics, recall@k, diversity, latency per stage, embedding/
  feature drift, exploration rate.

**Wrap-up:** *"A **multi-stage funnel** because you can't score millions of items in 200 ms: **(1)
candidate generation** narrows millions → hundreds cheaply with high **recall** (combine sources, but the
workhorse is a **two-tower model** — precompute item embeddings into a **vector DB / ANN index**, embed the
user online, **nearest-neighbor retrieve**); **(2) ranking** precisely orders the hundreds with a heavy,
feature-rich model predicting **P(engagement)** (learning-to-rank, m26); **(3) re-ranking** applies
**diversity, freshness, dedup, filtering, business rules**. Handle **cold start** with content/popularity/
exploration, fight the **filter-bubble feedback loop** with **exploration (bandits)**, and **A/B test on
engagement** because offline NDCG is just a proxy. Served as hybrid batch+online (item embeddings offline,
user query online). It's literally my embeddings + vector-DB + rerank work, staged into a production funnel."*

---

## State-of-the-art & real-world notes 📚
- **The multi-stage funnel** is universal: **YouTube** (candidate generation + ranking deep nets),
  **Instagram/Facebook feed**, **Netflix**, **TikTok** (the famous engagement flywheel), **Amazon**
  (item-to-item CF). **Two-tower + ANN retrieval** is the modern retrieval standard.
- **Matrix factorization** (the Netflix Prize era) → **deep two-tower / neural CF** → **transformers for
  sequential recs** (session-based). **ANN libraries:** FAISS, ScaNN, HNSW (your vector-DB work).
- **Bandits / exploration** (e.g. contextual bandits) are standard for the exploration-exploitation +
  cold-start problem. **YouTube's "Deep Neural Networks for YouTube Recommendations"** paper is the
  canonical funnel reference.

---

## From your systems 🏭 (the retrieval stack is yours)
- **You built the retrieval engine:** your AI/ML notes cover **embeddings** (m05), **vector databases**
  (Chroma/pgvector, **HNSW/IVF**, m06), and **hybrid retrieve + cross-encoder rerank** (m06). The
  **two-tower → ANN retrieval** (Step 3) *is* your embeddings + vector DB; the **ranking/rerank** stage
  (Step 4) *is* your cross-encoder reranking. You can speak to the recsys funnel from having built its
  pieces.
- **Retrieve-then-rerank = your hybrid search (m16/AI-m06):** candidate generation (recall) → ranking
  (precision) is the exact shape of BM25/vector retrieve → rerank you implemented. Same pattern, recsys
  domain.
- **Multi-stage funnel = your DCE pipeline instinct:** narrow with cheap filters (exclusions), then apply
  expensive logic (models/rules) on the survivors — the same "cheap-then-expensive staged" thinking.
- **Sennzo movie platform:** a content platform (500k+ titles) where this recsys would power "recommended
  movies" on top of your search/filters.
- **Precompute-then-serve:** precomputing item embeddings (batch) for fast online retrieval is the
  snapshot/precompute pattern (Questimate, m13/m23).

---

## Key concepts (interview-ready)
- **Multi-stage funnel (the core):** you can't score millions in 200 ms → **candidate generation
  (recall, cheap, millions→hundreds) → ranking (precision, expensive, hundreds→dozens) → re-ranking
  (business logic)**. Same retrieve-then-rerank shape as m16.
- **Candidate generation:** collaborative filtering (matrix factorization), content-based, and the
  **two-tower model** — user tower + item tower → embeddings; **precompute item embeddings into a vector
  DB (ANN/HNSW), embed the user online, nearest-neighbor retrieve** (your embeddings + vector-DB work).
- **Ranking:** a heavy, feature-rich model scores **P(engagement)** over the few hundred candidates
  (learning-to-rank / CTR, m26); often multi-objective. = your cross-encoder rerank role.
- **Re-ranking:** **diversity, freshness, dedup, filtering (seen), business rules.**
- **Cold start:** new user → popularity/onboarding/exploration; new item → content-based (towers use
  features). **Feedback loop / filter bubble** → fight with **exploration (ε-greedy / bandits)**;
  exploration-exploitation trade.
- **Eval:** offline NDCG/recall@k are **proxies**; **A/B test on engagement** (m22). Watch position/
  popularity bias.
- **Serving:** hybrid — item embeddings + ANN index offline, user embedding + ranking online (m23/m24).

---

## Go deeper (reading)
- **"Deep Neural Networks for YouTube Recommendations" (Covington et al.)** — the canonical two-stage
  (candidate gen + ranking) funnel.
- **"Sampling-Bias-Corrected Neural Modeling..." / Google's two-tower retrieval papers**; **FAISS / ScaNN
  / HNSW** docs (your vector-DB tools).
- **Netflix / Instagram / TikTok engineering blogs** on multi-stage recs + ranking.
- **Chip Huyen "Designing ML Systems"** + your own **AI/ML notes (m05 embeddings, m06 vector DBs/rerank)** —
  the retrieval + rerank stack.
- Revisit **m16** (retrieve-then-rerank / hybrid search), **m23** (features), **m24** (serving), **m26**
  (the ranking/CTR model), **m22** (offline-online eval, feedback loops).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m25_recommendation_system/`](../../solutions/m25_recommendation_system/README.md).

When you've done the exercises, say **"Module 26"** to design *Feed Ranking & Ads / CTR Prediction* — the
deep-dive on the **ranking** stage (learning-to-rank, CTR models, real-time features), the ML version of
the news feed (m13).
