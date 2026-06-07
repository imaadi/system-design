# m22 — Solutions (ML System Design Framework)

> Check the *reasoning*. Recurring lessons: frame first (is it ML?); training-serving skew is the silent
> killer; offline metrics are proxies (A/B online); baseline first; monitor for drift.

---

## Exercise 1 — Should it even be ML?
1. **Rules** — a fixed per-user limit is a deterministic threshold; ML adds nothing.
2. **ML** — "what will this user watch next" is a complex personalization pattern with abundant implicit
   feedback (a recommender, m24).
3. **Hybrid** — rules for clear-cut cases + ML for the fuzzy middle (exactly DCE). Pure rules miss
   subtle patterns; pure ML loses explainability/guardrails.
4. **Rules** — tax is a deterministic lookup/calculation, not a prediction.
5. **ML** — churn is a probabilistic pattern over many features with (delayed) labels; a classification
   model fits.

---

## Exercise 2 — Frame it as ML
- **Task:** ranking / classification (rank candidates by P(watch)).
- **Input (X):** user features (history, demographics), video features (topic, length, popularity),
  context (time, device).
- **Output/label (y):** did the user watch (and how long) — a relevance/engagement signal.
- **Labels from:** **implicit feedback** (watches/clicks/dwell) — cheap, plentiful, but biased (you only
  see feedback on what you showed).
- **Online or batch:** **hybrid** — batch-generate candidates, **online re-rank** for fresh context (m24).
- **Business metric:** total **watch time / engagement** (online); **offline proxy:** NDCG / P(watch) AUC
  on held-out logs.

---

## Exercise 3 — Run the framework (fraud)
1. **Frame:** P(fraud) per transaction → binary classification; label = confirmed fraud/chargeback
   (delayed); online <50 ms; business metric = fraud loss prevented vs false-positive friction.
2. **Data:** transaction logs + outcomes; pipeline to a warehouse; **class imbalance** (<1% fraud).
3. **Features:** amount, velocity, location, device, user history — computed consistently online+offline
   (feature store).
4. **Model:** baseline (logistic/GBDT) first; GBDT for tabular + low latency; interpretability matters.
5. **Eval:** offline **precision/recall/PR-AUC** (not accuracy); then **A/B / shadow** online on fraud
   loss + friction.
6. **Serve:** **online**, real-time per transaction, low-latency feature lookups.
7. **Monitor:** drift (PSI/KS), fraud rate, false-positive rate; **retrain** on drift (fraud evolves fast),
   gated; mind feedback loop. (m27 deep-dives this.)

---

## Exercise 4 — Training-serving skew
1. **Skew** = features computed **differently** in the offline training pipeline vs the online serving
   path → the model sees a different distribution in production than it trained on → **silent degradation**
   (no error, just worse predictions). Dangerous because it's invisible — offline metrics still look fine.
2. **Example:** the training pipeline computes "avg purchase last 30 days" in a Spark batch job; the
   serving code computes it in application logic with a slightly different window/rounding/null-handling →
   the same feature has different values in train vs serve.
3. **Feature store fixes it** by **(a)** computing features from **one shared definition** used by both
   training and serving, and **(b)** enforcing **point-in-time correctness** (training rows only use values
   known as-of the event time — no leakage), plus low-latency online lookups.

---

## Exercise 5 — The offline-online gap
1. **No** — higher offline AUC doesn't guarantee B is better in production; offline is a **proxy**.
2. **(a)** offline data is **stale** (past distribution); **(b)** the offline metric **isn't the business
   metric**; **(c)** the model **changes user behavior**, which offline data can't capture (especially
   recommenders/ranking).
3. **Offline eval → shadow mode** (run in parallel, log, don't act) **→ canary** (small % traffic) **→ A/B
   test** (measure business metric vs control, significance) **→ full rollout** — gating each stage on the
   business metric, not the offline proxy.

---

## Exercise 6 — Labels
1. **Explicit** (human-labeled — accurate but expensive/slow); **implicit feedback** (clicks/watches —
   cheap/plentiful but **biased/noisy**); **delayed** (chargeback/churn — truthful but **arrives late**).
2. **Delayed example:** a 30-day chargeback confirms fraud a month later — you can't evaluate today's
   model on today's transactions until labels mature, slowing feedback and complicating online metrics.
3. **Implicit-feedback bias:** you only get feedback on items the model **surfaced** (no impression → no
   click → looks "bad"), so the model can entrench its own choices — a feedback loop that needs exploration/
   debiasing.

---

## Exercise 7 — Online vs batch
1. **Online** — needs the live transaction + fresh context, low latency at checkout.
2. **Batch** (or hybrid) — homepage recs can be precomputed nightly per user and served by lookup (often
   batch candidates + online re-rank).
3. **Online** — ranking depends on the live query + fresh signals; must be real-time.
4. **Batch** — a daily per-account risk score is stable; precompute on a schedule, serve by lookup.

---

## Exercise 8 — Data hazards
1. **Leakage** = a training feature contains info about the label that **won't be available (or differs)
   at prediction time** → the model "cheats" → **flattering offline metrics** (it learned the answer) but
   **fails online** (the signal isn't really available pre-prediction).
2. **Accuracy is useless** at 0.5% fraud — a model predicting "never fraud" is 99.5% accurate but catches
   nothing. Use **precision/recall, PR-AUC, F1** (and handle imbalance via resampling/weights, m27).
3. **Point-in-time correctness** builds each training row using only data **known as-of** the event time —
   so a feature computed *after* the label (the leak) can't sneak in; train mirrors what serving will see.

---

## Exercise 9 — Monitoring & drift
1. **Data drift** = the **input distribution** shifts (e.g. new user demographics) while the model is
   fixed. **Concept drift** = the **X→y relationship** itself changes (fraud patterns evolve) so old
   mappings no longer hold.
2. ML degrades **silently** because the **world changes but the model is frozen** — no error is thrown, the
   model just gets quietly worse. Monitor **data drift (PSI/KS), concept drift (accuracy on fresh labels),
   prediction distribution, and the business metric.**
3. **Gate before promoting** so you **never deploy a worse model** — the retrain might produce a regression
   (bad new data, a pipeline bug), so you require the candidate to **beat the current model on the eval
   set** before it ships (my InsightDesk retrain-trigger + eval-gate).

---

## Exercise 10 — Your-systems tie-in 🏭
1. **DCE through the 7 steps:** **Frame** = recommend DENY/BYPASS/ROUTE/UPDATE per claim (classification),
   rules+ML hybrid; **Data** = claims from Mongo + reference data, Snowflake ETL; **Features** = the
   100+-field preprocessing; **Model** = LightGBM/RF + rule-based models (DS team's pickles + your rules);
   **Eval** = output parity/quality + suppression checks; **Serve** = batch-ish pipeline at 1M/day; 
   **Monitor** = drift + reprocessing + retrain + audit. You operate the whole framework.
2. **Claim-preprocessing (100+ consistent fields) addresses Step 3 (Features) / training-serving
   consistency** — it produces a **single, consistent feature representation** every model consumes, the
   discipline that prevents training-serving skew (one definition, consistent inputs).
3. **InsightDesk offline eval (faithfulness/recall + CI regression gate) = Step 5 offline + Step 7
   retrain-gate**; **drift (PSI/KS) + retrain-trigger = Step 7 monitoring**. **What production adds:** the
   **online** half of Step 5 — an **A/B test against the business metric** (your offline eval proves
   "plausibly better"; production proves "actually better" by measuring real user/business impact live).
