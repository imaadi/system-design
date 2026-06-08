# m26 — Cross-Questions ("if-and-buts") on Feed Ranking & CTR

> Answer out loud in 2–3 sentences before reading the model answer. Learning-to-rank, feature interactions,
> the auction, and calibration are where this is won.

---

### Q1. What are the three learning-to-rank approaches, and which dominates production?
**A.** **Pointwise** (predict a score per item independently, then sort), **pairwise** (learn which of two
items is better — RankNet/LambdaRank), and **listwise** (optimize the whole list / a ranking metric like
NDCG — LambdaMART). **Pointwise dominates production** because it's simple, scalable, and the prediction is
**reusable** (e.g. P(click) × bid in an ad auction). **CTR prediction is pointwise.** Pairwise/listwise
align better with ranking metrics but are heavier — used when pointwise leaves quality on the table.

---

### Q2. What is CTR prediction and why is it such a big deal?
**A.** CTR = **predict P(click | user, ad/item, context)** — a pointwise probability per candidate. It's
the **highest-value ML problem on earth** because **ad ranking + pricing ride on it** (Google/Meta/Amazon
ads = hundreds of billions in revenue), so a **1% accuracy gain is enormous money**. It also has to be
**fast (tens of ms)**, **huge scale (billions of predictions, billions of sparse features)**, and (for
ads) **calibrated**. It's where ML engineering meets real economics.

---

### Q3. Why are feature interactions ("cross features") so important in CTR?
**A.** Because a **single feature is weak, but the interaction is predictive**: "user-likes-running" alone
or "ad-for-shoes" alone says little, but **"user-likes-running × ad-for-running-shoes × evening"** strongly
predicts a click. Clicks come from the **match between user and item in context**, which is inherently an
**interaction**. The whole CTR model lineage — LR + manual crosses → factorization machines →
Wide&Deep/DeepFM/DCN — is about **learning these interactions automatically** instead of hand-crafting
every one.

---

### Q4. Walk me through the CTR model evolution.
**A.** **Logistic regression** over massive sparse features + **manually-engineered cross features** —
scalable, the classic baseline (Google's early ads). **Factorization Machines** — learn **pairwise
interactions** automatically via embeddings, handling sparsity. **Deep models — Wide & Deep** (a "wide"
memorization part + a "deep" generalization part), **DeepFM**, **DCN (Deep & Cross)** — learn higher-order
interactions end-to-end. And **GBDTs (XGBoost/LightGBM)** for tabular CTR with engineered features (strong,
cheap, common — my DCE world). The arc is "**from hand-crafted to learned feature interactions.**"

---

### Q5. How do you handle billions of distinct categorical values (user IDs, ad IDs)?
**A.** You **can't one-hot** billions of values. Two tools: **embeddings** — a **learned dense vector per
ID** (so similar IDs get similar vectors), which produces **giant embedding tables (often terabytes)** that
dominate the model's size; and **feature hashing** — hash high-cardinality values into a **fixed-size
space** (accepting some collisions) to bound dimensionality. This is why large CTR systems are really
**embedding-table systems** — serving them needs sharded embedding lookups / parameter servers, a scaling
problem of its own.

---

### Q6. How are ads ranked — just by predicted CTR?
**A.** No — by **expected value to the platform: `bid × pCTR`** (roughly **eCPM**, expected revenue per
thousand impressions). So a **lower-bid ad with higher predicted CTR can outrank a high-bid, low-CTR ad** —
which **aligns incentives**: the ad shown is one users actually click, good for the user, advertiser, and
platform revenue. Ranking by bid alone would show irrelevant high-bid ads users ignore; multiplying by pCTR
makes the marketplace efficient. (Then pricing is via a second-price auction.)

---

### Q7. What's a second-price (GSP) auction and why use it?
**A.** The **winning advertiser pays just above the next-highest competitor's score** (not their own bid).
Why: it makes **truthful bidding the dominant strategy** — you bid your **true value** because over- or
under-bidding can't improve your outcome — which stabilizes the marketplace and simplifies advertiser
strategy. Modern ad systems use **generalized second-price (GSP)** variants over eCPM scores. It's the
economic mechanism that, combined with pCTR ranking, makes ad auctions efficient and (reasonably)
incentive-compatible.

---

### Q8. What is calibration and why do ads need it (but a feed might not)?
**A.** A **calibrated** model's predicted probability matches reality — if it says **5% CTR, ~5% actually
click**. A pure **ranking/feed** model only needs the **order** right (who's above whom), so calibration
doesn't matter. But an **ads** model **multiplies bid × pCTR** in the auction to compute expected revenue
and set prices, so it needs the **absolute probability** to be accurate — a model that **ranks perfectly
but is miscalibrated** (right order, wrong magnitude) **breaks the auction economics** (mis-priced ads,
wrong budgets). So ads CTR must be calibrated; feed ranking need not be.

---

### Q9. Your CTR model ranks well but the auction is mispricing ads. What's wrong and how do you fix it?
**A.** It's almost certainly a **calibration** problem — the model's **order** is right but its **absolute
probabilities are off**, so `bid × pCTR` is wrong and pricing breaks. Fix: **recalibrate** the outputs
**post-hoc** with **Platt scaling** (a logistic fit) or **isotonic regression** (a monotonic fit) on a
validation set, or use calibration-aware training; then **monitor a calibration curve / Expected
Calibration Error** in production (CTR distributions drift, so calibration drifts). Ranking metrics like AUC
won't catch this — AUC is calibration-invariant — which is why you monitor calibration separately.

---

### Q10. Why does AUC not tell you whether a model is calibrated?
**A.** Because **AUC only measures ranking/ordering** — the probability that a random positive is scored
higher than a random negative. You can **rescale all predictions** (e.g. multiply by 0.1) and **AUC is
unchanged** while calibration is completely broken. So a model can have great AUC and terrible calibration.
For **feed ranking** (order only) AUC suffices; for **ads** (absolute pCTR in the auction) you need a
**calibration metric (ECE/reliability curve) + log-loss** alongside AUC. Different jobs, different metrics.

---

### Q11. Why do real-time / streaming features matter so much for ranking?
**A.** Because **recency is hugely predictive** — "this post's engagement in the last 5 minutes", "what
this user just clicked this session", "current trending" — captures fresh signal that a daily batch feature
misses entirely. A post blowing up *right now* should rank higher *right now*. So you compute these via
**streaming feature pipelines** (Flink/Kafka, m23) and feed them to the ranker, and **staleness directly
costs ranking quality**. It's why the m23 batch-vs-streaming decision lands hard here: the few high-value
recency features justify the streaming cost.

---

### Q12. How does this module relate to the recommendation funnel (m25) and the feed (m13)?
**A.** This **is the ranking stage of m25's funnel** — retrieval narrows millions → hundreds, then **this
CTR/engagement model precisely orders those hundreds**. And it's the **ML half of the news feed (m13)**:
m13 built the **storage + fan-out** (who sees what); **this** is the **model that orders** the candidate
posts by predicted engagement. Together — retrieval (m25) + ranking (m26) + the feed plumbing (m13) — you
have the complete modern feed. They're three views of one system.

---

### Q13. What is position bias and how does it corrupt CTR training?
**A.** Users **click top-ranked items more just because they're shown first**, regardless of true
relevance — so click logs over-credit whatever the model already ranked high. Train naively on those clicks
and the model **learns position, not relevance**, entrenching its own ranking (a feedback loop). You correct
with **position-aware modeling** (model click = P(relevance) × P(examined | position), or **inverse-
propensity weighting**) and **exploration** (occasionally randomize positions to get unbiased signal). It's
a core reason "clicks ≠ relevance" in ranking/CTR.

---

### Q14. How do you serve a CTR model in tens of milliseconds at billions/day?
**A.** **Fast models** (GBDTs or optimized deep nets), **feature caching** (m06) + **batched feature
lookups** from the online store (m23), **dynamic batching** (m24), and **sharded embedding lookups /
parameter servers** for the terabyte embedding tables. You precompute/refresh features (streaming for
recency), keep the model warm (m24), and scale horizontally. The hard parts are the **embedding-table
lookups** (huge, sharded) and **feature freshness** under the latency budget — the model arithmetic itself
is usually not the bottleneck; the feature fetch + embedding lookups are.

---

### Q15. Optimizing for clicks alone causes problems. What and how do you fix it?
**A.** It causes **clickbait** — the model learns to maximize the immediate click signal (sensational,
misleading content/ads), which boosts short-term CTR but **hurts long-term satisfaction, trust, and
retention**. Fix with **multi-objective ranking**: predict several outcomes (click, **dwell/long-watch**,
share, **"not interested"/hide**, downstream conversion) and **combine them** with weights tuned to
long-term value, plus negative-feedback signals and diversity. The metric you optimize *becomes* the
product — so you optimize **healthy engagement**, not raw clicks (the m25 lesson, again).

---

### Q16. How do you evaluate a feed-ranking / CTR model?
**A.** **Offline:** **AUC and log-loss** (CTR — log-loss also rewards calibration), **NDCG/MAP** (ranking
order). But these are **proxies** (m22). **Online:** **A/B test on the business metric** — engagement
(feed) or **revenue + long-term value** (ads) — with significance, shadow/canary first (m24). You also
monitor **calibration** (ads), **position bias**, and the **feedback loop**. The key discipline: a model
with better offline AUC can lose online (behavior change), so the A/B on the real metric is the decision —
offline just gates what's worth testing.

---

### Q17. Pointwise CTR vs a true ranking loss — when would you switch?
**A.** **Pointwise CTR** (predict P(click) per item, sort) is the default — simple, scalable, and the
probability is **reusable** (the auction needs the calibrated pCTR, not just an order). I'd consider a
**pairwise/listwise** loss (LambdaMART/LambdaRank) when **ranking quality (NDCG) is the goal and pointwise
under-delivers** — e.g. organic feed ranking where you don't need a calibrated probability, just the best
order. For **ads** I'd stay **pointwise + calibrated** (the auction needs the absolute pCTR). So: ads →
pointwise calibrated; pure organic ranking → consider listwise.

---

### Q18. The CTR distribution shifts fast (new ads, trends). How do you keep the model good?
**A.** **Frequent retraining** (CTR drifts faster than most domains — sometimes hourly/continuous online
learning, e.g. FTRL), **streaming features** for recency (m23), and **monitoring** for both **performance
drift** and **calibration drift** with auto-retrain + eval gate (m28). New ads/items face **cold start**
(no click history) → handle with content features + **exploration** (m25). The combination of fast
retraining, fresh features, and exploration keeps the model tracking a moving target — staleness shows up
fast as falling CTR/revenue.

---

### Q19. How is your DCE/HDC work structurally the same as CTR/ranking?
**A.** Very directly: I produce a **score/recommendation per claim from 100+ engineered features using
LightGBM** — that's a **pointwise scoring model with heavy feature engineering**, exactly the shape of
CTR/ranking (gradient boosting is *the* common CTR/ranking model). My **date-overlap windows, membership/
provider matching, and LSA multi-level categorical encoding** are **feature-interaction + sparse-
categorical** engineering — the CTR magic. A claims-risk score also benefits from **calibration** (a "0.8
risk" should mean ~80%), the same Step-5 concern. So I've done the core ranking/CTR engineering, in
healthcare instead of ads.

---

### Q20. What's the single biggest "think differently" insight in this module?
**A.** That **ads CTR is fundamentally different from feed ranking because of calibration** — a feed only
needs the **order** right, but an ad auction multiplies **bid × absolute pCTR**, so the **probability must
be accurate, not just well-ranked.** A perfectly-ranked but miscalibrated model destroys the auction
economics, and **AUC won't catch it** (AUC is calibration-invariant). Recognizing that the **same
prediction (P(click)) has different requirements depending on how it's *used*** — order-only vs absolute —
is the senior insight, and it's why ads systems obsess over calibration while feed systems often don't.
