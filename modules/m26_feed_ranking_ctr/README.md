# Module 26 — Feed Ranking & Ads / CTR Prediction ⭐⭐ 🏭

> The deep-dive on the **ranking stage** (m25) — and the **highest-value ML problem on earth**:
> **CTR prediction** (predict P(click)) is what powers Google/Meta/Amazon ads, hundreds of billions in
> revenue riding on a probability. Two related problems: **feed ranking** (the ML version of the news feed,
> m13 — order posts by predicted engagement) and **ads ranking** (CTR + an **auction**). The big concepts:
> **learning-to-rank**, **feature interactions/cross features** (the CTR magic), the **ad auction**, and
> **calibration** (a CTR for an auction must be *accurate*, not just *ordered*). Your **LightGBM scoring +
> 100+ features at DCE** is structurally this exact problem.

> **Format (Part 3 deep-dive):** worked design; **learning-to-rank**, **CTR / feature interactions**, the
> **ad auction**, and **calibration** are the deep-dives.

---

## Step 1 — Requirements

**Functional:**
- **Feed ranking:** given a user + candidate items (from retrieval, m25), **order them by predicted
  engagement** (P(click/watch/like)).
- **Ads:** **predict CTR** for candidate ads and run an **auction** to choose + price which ads to show.

**Non-functional (these drive it):**
- **Ultra-low latency:** ranking is on the page-load path; **ads must score in tens of ms** (the auction
  blocks the page).
- **Massive scale:** **billions of predictions/day**, **billions of sparse feature values** (user/ad IDs).
- **Real-time features:** recency matters ("engagement in the last hour", session signals) → streaming
  features (m23).
- **The business metric:** **engagement** (feed) / **revenue** (ads) — measured **online** (m22).
- **(Ads) calibration:** predicted CTR must be **accurate in absolute terms** (for the auction math).

> **The defining insight:** ranking = predict a score (engagement/CTR) over the **few hundred** candidates
> from retrieval (m25), with the magic in **feature interactions**; for **ads** it additionally must be
> **calibrated** because the **auction multiplies bid × pCTR.**

---

## Step 2 — Learning to Rank ⭐⭐ (how you turn "order items" into ML)

Three formulations:
- **Pointwise:** predict a **score per item independently** (e.g. P(click)), then **sort by score**. Simple,
  scalable, and **what production mostly uses** — **CTR prediction is pointwise** (predict P(click) for each
  ad, sort/auction). *(Your DCE/HDC scoring models are pointwise.)*
- **Pairwise:** learn **which of two items is better** (minimize mis-ordered pairs — e.g. RankNet,
  LambdaRank). Optimizes relative order directly.
- **Listwise:** optimize the **whole list / a ranking metric** (NDCG) directly (e.g. LambdaMART, ListNet).
  Best aligned with the metric, but more complex.

> **The senior take:** *"Production ranking is usually **pointwise** — predict an engagement probability
> per item and sort — because it's simple, scalable, and lets you reuse the prediction (e.g. P(click) ×
> bid in an ad auction). Pairwise/listwise optimize ranking order/metrics more directly but are heavier; I
> reach for them when pure pointwise leaves ranking quality on the table."*

---

## Step 3 — Deep Dive: CTR prediction & feature interactions ⭐⭐ (the magic)

CTR = **predict P(click | user, ad/item, context)**. The features are the whole game:
- **User features** (history, demographics, interests), **item/ad features** (category, advertiser,
  creative), **context** (time, device, placement), and — the magic — **cross features (feature
  interactions)**.

### Feature interactions are the key ⭐
A single feature is weak; the **interaction** is predictive: *"user-interested-in-running × ad-for-running-
shoes × evening"* is far more informative than any one of those alone. The **model evolution is all about
learning interactions:**
- **Logistic regression** over **massive sparse features** + **manual cross features** — the classic,
  scalable baseline (Google's early ads).
- **Factorization Machines (FM):** learn **pairwise interactions** automatically via embeddings (handles
  sparsity).
- **Deep models — Wide & Deep, DeepFM, DCN (Deep & Cross Network):** combine a **"wide" part** (memorized
  manual cross features) with a **"deep" part** (learned generalizing interactions) → the modern standard.
- **GBDTs (XGBoost/LightGBM)** for tabular CTR with engineered features — strong, cheap, common (your DCE
  LightGBM world).

### Massive sparse categorical features ⭐
User IDs, ad IDs, etc. are **billions of distinct values** — you can't one-hot them. Solutions: **embeddings**
(a learned dense vector per ID — giant embedding tables, often **terabytes**) and **feature hashing** (hash
high-cardinality values into a fixed space). This is why large CTR models are dominated by **huge embedding
tables**, a serving + storage challenge of its own.

> **Say this:** *"CTR is a pointwise P(click) model where the **value is in feature interactions** — a
> user feature × an ad feature × context is what predicts a click. The model lineage (**LR + manual
> crosses → factorization machines → Wide&Deep/DeepFM/DCN**) is all about learning those interactions
> automatically over **billions of sparse features** (embeddings + hashing). It's the highest-leverage ML
> problem there is — a 1% AUC gain is enormous revenue."*

---

## Step 4 — Deep Dive: the ad auction ⭐ (ads-specific)

Ads aren't ranked by **CTR alone** — they're ranked by **expected value to the platform**:
- **Rank by `bid × pCTR`** (roughly **eCPM** — expected revenue per thousand impressions): an ad with a
  lower bid but higher predicted CTR can outrank a high-bid, low-CTR ad. This **aligns incentives** — the
  ad shown is one users actually click (good for user, advertiser, *and* platform revenue).
- **Second-price / GSP auction:** the winner pays **just above the next-highest competitor's score** (not
  their own bid) — which makes **truthful bidding** the dominant strategy (you bid your true value). Modern
  systems use variants, but the principle holds.
- So the pipeline: **candidate ads → predict pCTR (calibrated!) → rank by bid×pCTR → second-price auction →
  show + charge.** And **organic + ads are blended** in the final feed (a re-ranking concern).

```
  candidate ads → CTR model → pCTR (calibrated) → score = bid × pCTR → auction (2nd price) → winning ad
                                                   (eCPM ranking aligns user value + revenue)
```

---

## Step 5 — Deep Dive: calibration ⭐ (why ads need more than ranking)

A **ranking** model only needs the **order** right. An **ads** model needs the **predicted CTR to be
accurate in absolute terms** — **calibrated** (if it says 5%, ~5% actually click) — because the auction
**multiplies bid × pCTR** to compute expected revenue and set prices. A model that ranks perfectly but is
**miscalibrated** (right order, wrong magnitude) **breaks the auction economics** (mis-priced ads, wrong
budgets).
- **Measure** with a calibration curve / Expected Calibration Error; **fix** with **Platt scaling /
  isotonic regression** (post-hoc recalibration) or calibration-aware training.

> **The big distinction:** *"Pure ranking (a feed) cares only about **order**; **ads CTR must be
> calibrated** because the auction uses the **absolute probability** (bid × pCTR). So I'd recalibrate the
> CTR model (isotonic/Platt) and monitor calibration in production — a perfectly-ranked but miscalibrated
> model still ruins the auction."* This is a top ads-ML signal most candidates miss.

---

## Step 6 — Real-time features, the full system & evaluation
- **Real-time / streaming features (m23):** recency is hugely predictive — "this post's engagement in the
  last 5 min", "ads this user saw this session", "current trending". These come from **streaming feature
  pipelines** (Flink/Kafka) into the ranker; staleness here directly costs ranking quality.
- **The full ranking system (ties m25 + m13):** **candidate retrieval** (m25) → **ranking (CTR/engagement
  model)** over hundreds → **re-rank/auction + blend organic & ads** → top-k. The **classic feed (m13)**
  built the storage/fan-out; **this** is the model that orders the candidates — together they're the feed.
- **Evaluation:** offline **AUC / log-loss** (CTR) and **NDCG** (ranking) are **proxies**; **A/B test on
  engagement/revenue** online (m22). Watch **position bias** (users click top items regardless → debias,
  m25), **calibration drift**, and the **feedback loop** (you only learn on what you showed).

---

## Step 7 — Bottlenecks, scale & edge cases
- **Latency:** ads score in **tens of ms** → fast models, **feature caching** (m06), batched feature
  lookups (m23), GBDTs/optimized deep models, **dynamic batching** (m24).
- **Scale:** billions of predictions/day + **terabyte embedding tables** → distributed serving, parameter
  servers / sharded embedding lookups, model parallelism for huge models.
- **Calibration drift:** monitor + recalibrate (Step 5).
- **Position/popularity bias + feedback loops:** debias training, exploration (m25).
- **Freshness:** streaming features + frequent retraining (CTR distributions shift fast).
- **Multi-objective:** balance click vs long-term value vs "not interested" vs advertiser/user welfare —
  optimizing raw clicks creates clickbait (m25).
- **Observability (m10/m28):** AUC/log-loss, **calibration**, CTR, revenue, latency, feature drift,
  exploration rate.

**Wrap-up:** *"Ranking = a **pointwise** model predicting **P(engagement/click)** over the hundreds of
candidates from retrieval (m25), with the value in **feature interactions** (user × item × context). The
CTR-model lineage — **LR + manual crosses → factorization machines → Wide&Deep/DeepFM/DCN** — automates
learning those interactions over **billions of sparse features** (embeddings + hashing, terabyte tables).
For **ads**, predictions must be **calibrated** (the auction multiplies **bid × pCTR**, ranks by eCPM, and
prices via a **second-price auction**), and **organic + ads are blended**. **Real-time/streaming features**
(recency) drive quality; evaluate offline with AUC/log-loss/NDCG but **A/B test on engagement/revenue**,
fighting position bias + feedback loops. It's the highest-leverage ML system — exactly the LightGBM scoring
+ heavy feature engineering I did at DCE, at ads scale."*

---

## State-of-the-art & real-world notes 📚
- **CTR model lineage:** Google's logistic-regression ads (Ftrl-Proximal) → **Factorization Machines** →
  **Wide & Deep (Google)**, **DeepFM**, **DCN/DCN-v2 (Deep & Cross)** — the canonical evolution toward
  learned feature interactions.
- **Facebook's "Practical Lessons from Predicting Clicks on Ads"** (GBDT + LR) and **YouTube/Meta ranking**
  papers — production CTR/ranking at scale.
- **Learning-to-rank:** **LambdaMART** (GBDT, listwise) is a classic strong ranker; **RankNet/LambdaRank**
  (pairwise).
- **Auctions:** **GSP / second-price** (Vickrey), eCPM ranking — the ads economics foundation. **Calibration:
  Platt scaling / isotonic regression.**

---

## From your systems 🏭
- **DCE/HDC = a pointwise scoring/ranking model:** you produce a **score/recommendation per claim** from
  **100+ engineered features** with **LightGBM** — structurally a **pointwise ranking/CTR-style model**
  (gradient boosting is *the* common CTR/ranking model). You've done the feature engineering + GBDT scoring
  that this module formalizes at ads scale.
- **Feature engineering = the CTR magic:** your DCE date-overlap windows, membership/provider matching, LSA
  multi-level categorical encoding (HDC) are exactly the **feature-interaction / sparse-categorical** work
  CTR lives on.
- **Calibration awareness:** a claims-risk score that feeds a downstream decision benefits from being
  **calibrated** (a "0.8 risk" should mean ~80%) — the same Step-5 concern.
- **Real-time features:** DCE merges real-time reference data at scoring (m23) — the streaming-feature
  pattern that ranking quality depends on.
- **The feed:** m13 (storage/fan-out) + **this ranker** = the full ML news feed; your Questimate factor/
  z-score ranking is ranking-by-score in a different domain.

---

## Key concepts (interview-ready)
- **Ranking = predict a score over the retrieved candidates (m25), then sort.** **Learning-to-rank:**
  **pointwise** (score each, sort — production default, **CTR is pointwise**), pairwise, listwise (optimize
  NDCG).
- **CTR = P(click | user, item, context); the value is in feature interactions** (user × item × context).
  Model lineage: **LR + manual crosses → factorization machines → Wide&Deep/DeepFM/DCN** (learn
  interactions). **Billions of sparse features** → **embeddings + hashing** (terabyte embedding tables).
- **Ad auction:** rank by **bid × pCTR (eCPM)** — aligns user value + revenue; **second-price/GSP** pricing
  (truthful bidding). Blend **organic + ads**.
- **Calibration (ads-critical):** the auction uses the **absolute** pCTR, so it must be **calibrated** (5%
  means ~5%), not just ordered → **isotonic/Platt** recalibration + monitoring. (Pure ranking needs only
  order.)
- **Real-time/streaming features** (recency) drive quality (m23). **Eval:** AUC/log-loss/NDCG offline are
  proxies → **A/B on engagement/revenue** (m22); fight **position bias** + **feedback loops**.
- **Scale/latency:** tens-of-ms ads → fast models + feature caching + batched lookups + sharded embeddings.
  GBDTs (LightGBM) are a strong, common choice.

---

## Go deeper (reading)
- **Wide & Deep (Cheng et al.), DeepFM, DCN-v2** papers — the feature-interaction model lineage.
- **Facebook "Practical Lessons from Predicting Clicks on Ads" (GBDT+LR)** and **McMahan "Ad Click
  Prediction: a View from the Trenches" (Google, FTRL)** — production CTR.
- **"From RankNet to LambdaRank to LambdaMART"** — learning-to-rank.
- **Calibration:** Platt scaling / isotonic regression primers; **GSP/second-price auction** explainers.
- Revisit **m25** (the funnel/retrieval feeding this), **m13** (the feed storage), **m23** (real-time
  features), **m22** (offline-online eval, feedback loops), **m24** (serving).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m26_feed_ranking_ctr/`](../../solutions/m26_feed_ranking_ctr/README.md).

When you've done the exercises, say **"Module 27"** to design *Fraud & Anomaly Detection* — real-time
low-latency scoring, class imbalance, the rules+ML hybrid, and delayed labels — **your DCE world** (the
last core ML-design module before the ML platform).
