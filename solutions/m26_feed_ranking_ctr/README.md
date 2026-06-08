# m26 — Solutions (Feed Ranking & Ads / CTR Prediction)

> Check the *reasoning*. Recurring lessons: pointwise dominates; CTR magic = feature interactions; ads rank
> by bid×pCTR via second-price; ads need calibration (AUC won't catch it).

---

## Exercise 1 — Learning to rank
1. **Pointwise:** predict a score per item independently, then sort. **Pairwise:** learn which of two items
   is better (minimize mis-ordered pairs). **Listwise:** optimize the whole list / a ranking metric (NDCG).
2. **Pointwise** — simple, scalable, and the prediction is **reusable** (e.g. P(click) × bid in an auction).
3. Because ads need the **probability itself** (pCTR) for the auction (bid × pCTR), not just an order —
   which pointwise gives directly.

---

## Exercise 2 — Feature interactions
1. A single feature ("user likes running" or "ad for shoes") is weakly predictive; the **interaction**
   ("user-likes-running × ad-for-running-shoes × evening") strongly predicts a click — clicks come from the
   **user×item×context match**.
2. **LR + manual cross features** (you hand-craft interactions) → **Factorization Machines** (learn pairwise
   interactions via embeddings) → **Wide&Deep / DeepFM / DCN** (learn higher-order interactions
   end-to-end). Each step **automates more of the interaction learning**.
3. **GBDTs (XGBoost/LightGBM)** model feature interactions via tree splits on engineered features — a
   strong, cheap, common choice for tabular CTR/ranking (my DCE world).

---

## Exercise 3 — Sparse features
1. There are **billions of distinct values** (user IDs, ad IDs) — one-hot would be a billions-wide,
   impossibly sparse vector.
2. **Embeddings** = a learned dense vector per ID (similar IDs → similar vectors); **feature hashing** =
   hash high-cardinality values into a fixed-size space (bounded dimensionality, some collisions).
3. **Giant embedding tables (often terabytes)** dominate model size → you need **sharded embedding lookups
   / parameter servers** to store and serve them, a distributed-systems challenge of its own.

---

## Exercise 4 — The ad auction
1. **`bid × pCTR` (eCPM)** ranks by **expected revenue**, so a relevant ad (high pCTR) can beat a high-bid
   irrelevant one — **aligning user value, advertiser value, and platform revenue**. Bid alone would show
   irrelevant high-bid ads users ignore.
2. **Second-price (GSP):** the winner pays **just above the next competitor's score** (not their own bid) →
   encourages **truthful bidding** (bid your true value), stabilizing the marketplace.
3. **Organic content** — ads and organic results are **blended** into the final feed (a re-ranking concern).

---

## Exercise 5 — Calibration
1. **Calibration:** predicted probability matches reality (says 5% → ~5% click). **Ads need it** because the
   auction multiplies **bid × absolute pCTR**; a **pure feed** only needs the **order** right, so
   calibration is irrelevant there.
2. **Diagnosis:** miscalibration — order is right (high AUC) but absolute probabilities are off, so
   bid×pCTR mis-prices. **Fix:** recalibrate post-hoc (**isotonic regression / Platt scaling**) on a
   validation set (or calibration-aware training) + monitor a calibration curve / ECE in production.
3. **AUC measures only ranking/order** and is **invariant to rescaling** — you can multiply all predictions
   by a constant and AUC is unchanged while calibration breaks → AUC can't reveal it.

---

## Exercise 6 — Pick the metric
1. **NDCG / MAP** (and AUC) — order quality is what matters for an organic feed.
2. **Calibration (ECE / reliability curve) + log-loss** (plus AUC) — the auction needs accurate absolute
   pCTR, not just order.
3. **Performance drift + calibration drift** (and the business metric) — monitor AUC/log-loss/calibration on
   fresh data over time; CTR distributions shift fast (m28).

---

## Exercise 7 — Real-time features
1. "This post's **engagement in the last 5 min**" (catches content blowing up *now*) and "items this user
   **clicked this session**" (current intent) — both capture **recency** that batch features miss.
2. From **streaming feature pipelines** (Flink/Kafka, m23) writing fresh features to the online store.
3. **Staleness directly costs ranking quality** — you'd rank a now-viral post low, or miss the user's
   current intent — so the few high-value recency features justify the streaming cost.

---

## Exercise 8 — Biases & objectives
1. **Position bias:** users click top items just because they're shown first → logs over-credit the current
   ranking. **Correct** with position-aware modeling (click = P(relevance)×P(examined|position) / inverse-
   propensity weighting) + **exploration** (randomize positions sometimes).
2. **Clicks-only → clickbait** (sensational content boosts short-term CTR but hurts long-term satisfaction/
   retention). **Fix:** **multi-objective ranking** (blend click, dwell, share, "not interested",
   conversion) tuned to long-term value.
3. The model **trains on what it showed**, so it entrenches its own choices (feedback loop / filter bubble)
   → needs **exploration + debiasing** to keep learning about un-shown items.

---

## Exercise 9 — Serving at scale
1. **Fast models** (GBDT/optimized deep) + **feature caching** (m06) + **batched feature lookups** (m23) +
   **dynamic batching** (m24) + **sharded embedding lookups / parameter servers**; warm models, horizontal
   replicas.
2. The **feature fetch + embedding-table lookups** are usually the real bottleneck (huge sharded tables +
   the online feature store on the critical path), not the model arithmetic.
3. **Frequent retraining** (sometimes hourly / online learning like FTRL) + **streaming features** for
   recency + **monitoring with auto-retrain + eval gate** (m28); handle new ads/items with content features
   + **exploration** (cold start, m25).

---

## Exercise 10 — Your-systems tie-in 🏭
1. **DCE/HDC = a pointwise scoring model:** **LightGBM** produces a **score/recommendation per claim** from
   **100+ engineered features** — structurally identical to CTR/ranking (gradient boosting is *the* common
   CTR/ranking model), just predicting a claim outcome instead of a click.
2. **Date-overlap windows (30/90/180-day), membership/provider matching, AIM-grouper matching, and HDC's
   LSA multi-level categorical encoding** — these are **feature-interaction + sparse-categorical**
   engineering, exactly the CTR magic.
3. **"The full feed" = retrieval (m25) + ranking (m26) + feed plumbing (m13):** m13 built the **storage +
   fan-out** (who sees what), m25's **retrieval** narrows the candidate posts, and **m26's ranking model
   orders them** by predicted engagement — three views of one system that together produce the modern
   ranked feed.
