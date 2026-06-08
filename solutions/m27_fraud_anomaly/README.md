# m27 — Solutions (Fraud & Anomaly Detection)

> Check the *reasoning*. Recurring lessons: fraud inverts ML (imbalance/delayed-labels/adversarial/
> asymmetric-cost); rules+ML hybrid + 3-way decision + human review; DCE is the textbook instance.

---

## Exercise 1 — Why fraud is special
1. **Extreme class imbalance** (fraud <1%); **delayed/noisy labels** (chargebacks weeks later); **adversarial
   concept drift** (fraudsters adapt); **asymmetric costs** (missed fraud vs blocked customer).
2. **Imbalance → PR-metrics + resampling/weights + cost-sensitive threshold; delayed labels → matured-data
   eval + review labels + no leakage; adversarial drift → frequent retraining + monitoring + anomaly
   detection; asymmetric costs → cost-tuned threshold + 3-way decision.**

---

## Exercise 2 — Imbalance
1. A model predicting **"never fraud"** is **99.7% accurate** and catches **zero fraud** — accuracy is
   dominated by the trivial majority and tells you nothing about catching the rare positive.
2. **Precision, recall, PR-AUC, F1** (and cost-weighted variants) — at the operating threshold.
3. **Resampling** (oversample minority / **SMOTE** / undersample majority), **class weights** (penalize
   missing fraud more), and **anomaly detection** (model "normal," sidestepping imbalance). (Also cost-
   sensitive thresholds.)

---

## Exercise 3 — The threshold
1. Because **costs are asymmetric** — a false negative (missed fraud) and a false positive (blocked
   customer) cost very different amounts, so the cost-minimizing cut is rarely the symmetric 0.5.
2. **$200 miss ≫ $5 false-decline** → lean the threshold **lower** (flag more aggressively) → **higher
   recall, lower precision** (catch more fraud, accept more false declines, since each is cheap relative to
   a miss).
3. Three thresholds give a **3-way decision** (allow / **review** / block) — the **uncertain middle** goes
   to human review instead of forcing a costly auto-block or auto-allow.

---

## Exercise 4 — The hybrid
1. **Rules strengths:** fast/cheap, **explainable**, instantly deployable (react to a known threat now),
   guardrails. **Weakness:** rigid + **gameable**, miss subtle/novel fraud.
2. **ML strengths:** learns **complex patterns**, adapts to data, scales. **Weakness:** needs **labels**,
   only catches fraud **similar to the past**, less explainable.
3. **Hybrid covers each other's gaps:** rules for known/clear + explainability + instant response, ML for
   the fuzzy/evolving middle. **Anomaly detection adds** coverage of **novel, never-seen fraud** (no labels
   needed) that supervised ML misses.

---

## Exercise 5 — Delayed labels
1. Because the **truth arrives late** — you don't know a transaction was fraud until a **chargeback weeks
   later** (or never), so recent predictions have **unmatured labels** and can't be scored yet.
2. **Leakage** = using a post-event signal (the chargeback / "fraud-confirmed" flag) as a **feature** to
   predict the event — **tempting** because it's hugely predictive, **dangerous** because it isn't available
   at prediction time → great offline, fails online.
3. **Manual-review outcomes** (faster ground truth) and **sampling/exploration** of auto-allowed traffic
   (to get unbiased labels) — plus targeted investigations.

---

## Exercise 6 — The decision system
1. **3-way:** confidently-low-risk → **allow**; confidently-high-risk → **block**; **uncertain middle** →
   **review** (human queue).
2. The review queue: **(a)** decides the ambiguous cases, **(b)** is a **safety valve** (humans catch model
   mistakes / new patterns), **(c)** **generates labels** that feed retraining (the feedback loop).
3. **DCE's ROUTE** = exactly this — send the uncertain case to a human/downstream examiner.

---

## Exercise 7 — Adversarial drift
1. Because fraud is **adversarial** — fraudsters **actively adapt to evade** your model (on purpose), so the
   distribution shifts faster and deliberately, unlike passive concept drift.
2. **Frequent retraining** (on fresh + review labels), **aggressive monitoring** (feature drift, fraud-rate,
   performance — PSI/KS), and **human review + anomaly detection** to catch new patterns fast.
3. **Rules deploy instantly** — when a human review identifies a new attack, you **patch a rule
   immediately** to block it **now**, while the ML model **retrains** (which takes time + matured labels).
   Rules = fast response; ML = durable learning.

---

## Exercise 8 — Real-time features & serving
1. **"Txns on this card in last 5 min"** (catches a stolen-card burst), **"logins from this IP in last
   hour"** (credential-stuffing), **"amount vs the user's historical average"** (out-of-pattern spend) —
   recent behavior is the strongest fraud signal.
2. From **streaming feature pipelines** (Flink/Kafka, m23) maintaining real-time aggregates → the online
   store; serving must be **online, inline, in tens of ms** (decide before the money moves).
3. The **rules engine is a cheap first filter** — it **blocks obvious cases instantly** (and short-circuits
   the expensive ML scoring), keeping the average latency in budget.

---

## Exercise 9 — Fraud rings & explainability
1. Per-transaction models look at **one event in isolation**, but a ring's signal is the **connections**
   (shared devices/cards/IPs/addresses across many accounts). **Graph features / GNNs** (degree, shared-
   attribute counts, community detection) catch the coordinated structure.
2. Decisions have **real consequences + regulatory/audit requirements** (finance/healthcare) → you must say
   **why**. **Rules are inherently explainable**; for an **ML score** you use **SHAP / feature
   attributions** to show the contributing features (and DCE logs which edits/rules fired to an audit
   trail).

---

## Exercise 10 — Your-systems tie-in 🏭
1. **DCE = textbook instance:** **exclusion rules** (entry guardrails / rules-first filter) → **ML models +
   rule-based models** (the **hybrid scoring**) → **suppression rules** (exit guardrails that **block wrong
   outputs**) → **DENY / BYPASS / ROUTE / UPDATE** (the **multi-way decision**, with **ROUTE = human
   review**). Inline-ish at 1M/day on real-time reference features, with audit + drift/retrain. Every
   concept here, in production.
2. **HDC** classifies claims **high/low risk** with **LSA features** and low-risk overrides — the
   **imbalanced-scoring + cost-tuned threshold + route-high-risk-for-review** piece.
3. **Drift monitoring (PSI/KS)** = the adversarial-drift defense (catch the distribution shift fast →
   retrain); **audit logging** (which edits/rules fired → Mongo) = the **explainability/compliance** layer
   fraud (and healthcare) demands.
