# m26 — Exercises (Feed Ranking & Ads / CTR Prediction)

> Do these on paper / out loud before checking `solutions/m26_feed_ranking_ctr/`. Learning-to-rank, feature
> interactions, the auction, and calibration are the core.

---

## Exercise 1 — Learning to rank
1. Define pointwise, pairwise, listwise in one line each.
2. Which dominates production and why?
3. Why is pointwise especially natural for ads?

---

## Exercise 2 — Feature interactions
1. Why is a single feature weak but an interaction predictive? Give an example.
2. Trace the CTR model lineage and what each step adds re: interactions.
3. Where do GBDTs fit?

---

## Exercise 3 — Sparse features
1. Why can't you one-hot user IDs and ad IDs?
2. Name the two techniques and what each does.
3. What system challenge do giant embedding tables create?

---

## Exercise 4 — The ad auction
1. Why rank ads by `bid × pCTR` instead of bid alone?
2. What is a second-price auction and what behavior does it encourage?
3. What gets blended into the final feed besides ads?

---

## Exercise 5 — Calibration
1. Define calibration. Why do ads need it but a pure feed might not?
2. A model ranks great (high AUC) but the auction misprices. Diagnose + fix.
3. Why doesn't AUC reveal a calibration problem?

---

## Exercise 6 — Pick the metric
For each, name the metric(s) you'd watch:
1. Ordering an organic feed.
2. Pricing ads in an auction.
3. Whether the model still works a month after deploy.

---

## Exercise 7 — Real-time features
1. Give two recency features that boost ranking and why.
2. Where do they come from (which pipeline, m23)?
3. What's the cost of staleness here?

---

## Exercise 8 — Biases & objectives
1. What is position bias and how do you correct it?
2. Why does optimizing clicks alone hurt, and what's the fix?
3. How does the feedback loop apply to ranking?

---

## Exercise 9 — Serving at scale
1. How do you serve a CTR model in tens of ms at billions/day?
2. What's usually the real bottleneck (model math or feature/embedding lookups)?
3. CTR drifts fast — how do you keep the model fresh?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Argue that **DCE/HDC scoring** is structurally a pointwise ranking/CTR model. Name the model + features.
2. Which DCE features are "feature-interaction / sparse-categorical" engineering?
3. Connect this module to **m25 (the funnel)** and **m13 (the feed)** — what is "the full feed"?

---

When done, check `solutions/m26_feed_ranking_ctr/README.md`, then say **"Module 27"**.
