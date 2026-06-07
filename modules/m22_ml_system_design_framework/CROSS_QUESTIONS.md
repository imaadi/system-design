# m22 — Cross-Questions ("if-and-buts") on the ML System Design Framework

> Answer out loud in 2–3 sentences before reading the model answer. Framing, the offline-online gap, and
> training-serving skew are where ML-SD interviews are won.

---

### Q1. How is ML system design different from a classic system design question?
**A.** It's **classic SD + the ML lifecycle**. You still design data stores, queues, serving, and
monitoring (Parts 0–2), but you **also** frame the problem as ML, design the **data and feature
pipelines**, **train and evaluate** (offline *and* online), serve predictions, and close the **feedback/
drift loop**. The mindset shift: it's a **production system that has a model in it**, and the hard parts
are the **data/feature pipelines, evaluation strategy, and drift/feedback** — not the model architecture.

---

### Q2. The interviewer says "design a system to detect fraud." What's your first move?
**A.** **Frame it.** Clarify: is ML even the right tool (vs rules — often a hybrid)? What exactly are we
**predicting** (fraud probability per transaction → binary classification)? What's the **input** and where
do **labels** come from (confirmed fraud / chargebacks — which are **delayed**)? **Online or batch**
(real-time scoring, <50 ms)? And what's the **business metric** (fraud loss prevented vs false-positive
friction)? Only after framing do I design data/features/model/serving — same "requirements first"
discipline as m01.

---

### Q3. Should every problem be solved with ML?
**A.** No — **ask "should this be ML?" first**. A **heuristic/rules baseline** is often simpler, cheaper,
more explainable, and good enough, and it sets a bar to beat. ML is worth it when patterns are too complex
for rules **and** you have data + a feedback signal. Many production systems are **rules+ML hybrids** (DCE
is — rules for clear cases + guardrails, ML for the fuzzy middle). Reaching for ML when rules suffice is
over-engineering; the senior move is to justify ML, not assume it.

---

### Q4. What does it mean to "frame a problem as ML," concretely?
**A.** Turn a fuzzy goal into a concrete ML task: define the **task type** (regression/classification/
ranking/recommendation), the **input (X)**, the **output/label (y)** and **where labels come from**
(explicit, implicit-feedback, or delayed), whether prediction is **online or batch**, and the **business
metric** plus the **offline proxy** you'll optimize. E.g. "recommend videos" → "rank candidate videos by
P(watch) for this user (classification/ranking), label = did they watch (implicit), online re-rank, business
metric = watch time." Framing wrong dooms everything downstream.

---

### Q5. Where do labels come from, and why is that often the hard part?
**A.** Three sources: **explicit** (human annotation — accurate but expensive/slow), **implicit feedback**
(clicks/purchases/watches — cheap and plentiful but biased and noisy), and **delayed** labels (a 30-day
chargeback, a churn signal — you only learn the truth much later). Labels are often **the bottleneck**
because good labels are scarce/costly, implicit labels are biased (you only see feedback on what you
showed — a feedback loop), and delayed labels slow learning and complicate evaluation. The label strategy
shapes the whole system.

---

### Q6. What is training-serving skew and why is it the #1 ML production bug?
**A.** Features are computed in **two places** — the **offline training pipeline** (batch over historical
data) and the **online serving path** (live per request). If they're computed **differently** (different
code, different data freshness, a subtle transform mismatch), the model sees a **different feature
distribution in production than it trained on** → **silent degradation** (offline metrics look fine, real
predictions get worse). It's the #1 bug because it's **silent** — nothing errors, the model just quietly
underperforms. The fix is a **feature store** with shared feature definitions + point-in-time correctness
(m23).

---

### Q7. How does a feature store prevent training-serving skew?
**A.** It computes/serves features from **one place with one definition**, used by **both** training and
serving — so the feature a model trains on is computed identically to the feature it serves on. It also
enforces **point-in-time correctness** (when building a training row for time T, only use feature values
known *as of* T — no leaking future data) and provides low-latency online lookups for serving. So the two
data paths stay consistent by construction (m23). Without it, two teams writing two feature
implementations almost guarantees skew.

---

### Q8. Why aren't offline metrics enough to decide a model is better?
**A.** Because **offline metrics are proxies**, not the business outcome. Offline data is **stale**, the
metric (AUC/RMSE) **isn't the business metric** (revenue/engagement/fraud loss), and crucially the model
**changes user behavior** in ways offline data can't capture (a recommender that surfaces different items
generates different future data). So a model with **better offline AUC can be *worse* in production**.
Offline says "plausibly better"; only an **online A/B test** against the business metric proves real value.

---

### Q9. How do you actually validate a model will improve the business?
**A.** **A/B test online.** Route a fraction of real traffic to the new model (treatment) vs the current
one (control), measure the **actual business metric** with **statistical significance**, and only ship if
it wins. De-risk first with **shadow mode** (run the new model in parallel, log predictions but don't act —
catches gross errors + lets you compare) and **canary** (small % first, m10). So the flow is **offline
eval → shadow → canary/A-B → full rollout**, gating on the business metric, not the offline proxy.

---

### Q10. Online vs batch inference — how do you choose?
**A.** **Online (real-time)** when predictions need **fresh context / per-request inputs** or must reflect
live state (fraud scoring at checkout, search ranking) — within a **latency budget**, needing low-latency
serving + feature lookups. **Batch** when predictions are **stable** and you can **precompute** them in
bulk on a schedule and serve by lookup (nightly recommendations, risk scores) — cheaper/simpler but
**stale**. Many systems are **hybrid**: batch-generate candidates, **online re-rank** (m24). DCE is
batch-ish over claims; a feed re-ranks online.

---

### Q11. What's a feedback loop / data flywheel, and what's the danger?
**A.** Predictions **influence user behavior**, which **generates new training data**, which **retrains**
the model — a self-reinforcing **flywheel** (more usage → better model → more usage). The danger is
**feedback loops amplifying bias**: you only get labels for what the model **surfaced** (a recommender
never shows item X → no clicks on X → it learns X is bad → never shows it), so the model can entrench its
own mistakes and narrow over time. You mitigate with **exploration** (occasionally show non-top items),
debiasing, and monitoring for distributional collapse.

---

### Q12. Why "baseline first," and what baseline?
**A.** Because a **simple baseline** (a heuristic, logistic regression, or gradient-boosted trees) is fast
to build, sets a bar, is interpretable, and is **often surprisingly competitive** — and you need it to
know whether a complex model is even worth the cost. **Don't default to deep learning** (on tabular data a
GBDT often matches or beats a neural net — I found "NN == logreg" in my own work). You add complexity only
when the baseline plateaus *and* the gain justifies the latency/cost/maintenance. Start simple, measure,
then escalate.

---

### Q13. How do you choose the model type?
**A.** By the **task + constraints**. Task: **GBDTs (XGBoost/LightGBM)** for tabular (most enterprise ML —
my DCE/HDC world), **deep nets** for text/images/sequences, **two-tower** for large-scale retrieval (m24),
**transformers/LLMs** for language. Constraints: **latency** (a <50 ms fraud model rules out a huge
ensemble), **interpretability** (a regulated credit decision needs explainability → simpler/explainable
models), **cost**, and **data volume**. You weigh **accuracy vs latency vs interpretability vs cost** — the
"best" model is the best *for these constraints*, echoing m01.

---

### Q14. What can go wrong in the data, and how do you catch it?
**A.** **Class imbalance** (fraud <1% → accuracy is useless, use precision/recall/PR-AUC, resampling — m27),
**label noise**, **bias** (unrepresentative data), **data leakage** (a feature secretly containing the
label → great offline, useless online), **drift** (distribution shifts over time), and **privacy/PII**. You
catch leakage by suspicion of too-good offline metrics + audit; imbalance by inspecting the label
distribution; drift by monitoring (m28). Data problems cause most ML failures — "garbage in, garbage out"
is the literal first-order concern.

---

### Q15. What is data leakage and why is it dangerous?
**A.** **Leakage** is when a feature available at **training** time secretly contains information about the
**label** that **won't be available (or is different) at prediction time** — e.g. using a "has_chargeback"
field to predict fraud, or a feature computed *after* the event. It makes **offline metrics look amazing**
but the model **fails in production** because that signal isn't really available pre-prediction. It's
dangerous precisely because it's flattering (you think you have a great model) — you guard against it with
**point-in-time correctness** (only use data known as-of the prediction time) and skepticism of
suspiciously high offline scores.

---

### Q16. How do ML systems degrade silently, and what do you monitor?
**A.** The **world changes but the model is frozen** — input distributions shift (**data drift**), the X→y
relationship shifts (**concept drift**), so the model gets quietly worse with **no error thrown**. You
monitor: **data drift** (input distribution vs training — PSI/KS, my InsightDesk work), **concept drift**
(accuracy on freshly-labeled data), **prediction distribution** (is the model's output shifting?), and the
**business metric**. Then you **retrain** (scheduled or drift-triggered) **with an eval gate** so you never
promote a worse model (m28). Silent degradation is why ML needs *more* monitoring than classic systems.

---

### Q17. When and how do you retrain a model?
**A.** **Triggers:** a schedule (daily/weekly), a **drift alarm**, or a **performance drop** on fresh
labels. **How:** an automated **training pipeline** produces a candidate, you **evaluate** it (offline) and
**gate** it — only promote if it beats the current model on the eval set (don't auto-deploy a regression) —
then **shadow/canary/A-B** before full rollout, with the model in a **registry** (m28). The cadence depends
on how fast the world drifts (fast-moving fraud → frequent; stable domains → rare). My InsightDesk
retrain-trigger + eval-gate is this in code.

---

### Q18. How do all the Parts 0–2 building blocks show up in an ML system?
**A.** Everywhere: **data lake/warehouse** (m04 OLAP) for training data; **queues/streams** (m08, Kafka)
for events + feature pipelines; **caching** (m06) for feature/prediction lookups; **load balancing +
serving** (m07) for the model service; **sharding** (m05) for feature stores at scale; **reliability/
monitoring** (m10) for the serving SLOs + drift. ML system design is **assembling the toolkit you already
have** and adding the ML-specific layer (data/features/eval/drift). That's why Parts 0–2 came first —
you can't design an ML *system* without them.

---

### Q19. Walk me through the end-to-end architecture of an ML system.
**A.** **Two paths.** **Offline (training):** data sources → data pipeline (ETL, m04/m08) → **feature
store** → training pipeline → **model registry** (m28) → safe deploy (shadow/canary/A-B, m10). **Online
(serving):** request → **feature lookup** (same feature store) → **model service** (low-latency) →
prediction → action. A **monitoring/feedback loop** watches drift + the business metric and feeds new
labels back to retraining (the flywheel). The **feature store** is the linchpin keeping the two paths
consistent; the **registry + safe deploy + monitoring** make it a production system, not a notebook.

---

### Q20. What makes someone strong at ML system design specifically?
**A.** Treating it as a **system, not a model**: framing the problem precisely (task/label/business
metric), designing the **data + feature pipelines** as first-class (and naming **training-serving skew**),
understanding the **offline-online evaluation gap** (A/B against the business metric), choosing **online vs
batch** deliberately, and designing the **drift/retrain/feedback loop** — all on top of solid distributed-
systems fundamentals. The tell is someone who says "the model is the easy part; the data, evaluation, and
feedback loop are where it lives or dies" — which is exactly what running a 1M-claims/day pipeline teaches
you.
