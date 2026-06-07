# Module 22 — The ML System Design Framework ⭐⭐ 🏭 — *Part 3 begins*

> ML system design = **everything from Parts 0–2 (the distributed-systems toolkit) + an ML lifecycle
> layered on top.** It's a *different interview* with a different framework: you still design data stores,
> queues, serving, and monitoring — but now you also **frame the problem as ML, design the data and
> feature pipelines, train and *evaluate* (offline vs online), serve predictions, and close the feedback
> loop.** This module is the **method**; m23–m28 apply it (feature stores, serving, recsys, ranking,
> fraud, MLOps). **This is your home turf** — you run a 1M-claims/day ML pipeline (DCE) and built the full
> InsightDesk ML stack.

> **Format (Part 3 opener):** like m01 but ML-specific — the framework steps in depth, with the
> ML-specific concepts (online vs batch, **offline-vs-online eval**, **training-serving skew**, feedback
> loops) woven in. `CROSS_QUESTIONS` drills the ML-SD follow-ups.

---

## 1. How ML system design differs from both classic SD and ML modeling

| | Classic SD (Parts 0–2) | ML *modeling* (Kaggle) | **ML system design** |
|---|---|---|---|
| Focus | scale, latency, availability | model accuracy on a fixed dataset | **the whole ML *system* in production** |
| Data | given/structured | given/clean | **you design the data + feature pipelines** |
| Success | meets NFRs | offline metric | **business metric (measured online)** |
| Lifecycle | build & run | train once | **data → features → train → eval → serve → monitor → retrain** |

> **The senior framing:** *"An ML system is a **production software system that happens to have a model in
> it.** I design it like any system (Parts 0–2: storage, queues, serving, monitoring) **plus** the ML
> lifecycle — and I treat the **data and feature pipelines, the evaluation strategy, and the feedback/drift
> loop** as the first-class hard parts, not the model architecture."* That sentence separates ML
> *engineers* from ML *enthusiasts*.

---

## 2. The ML system design framework (the steps) ⭐

Run these in a design interview (the ML analog of m01's 7 steps):

```
  1. FRAME      — clarify the problem + frame it as ML (is ML even right? what's predicted/the label?)
  2. DATA       — sources, collection, labeling, the data pipeline
  3. FEATURES   — feature engineering, the feature store, training-serving skew
  4. MODEL      — baseline FIRST, then model selection
  5. EVALUATION — offline metrics + ONLINE metrics (A/B), the gap between them
  6. SERVING    — online vs batch inference, latency, the serving infra
  7. MONITOR    — drift, retraining, the feedback loop / data flywheel
```

---

### Step 1 — Frame the problem as ML ⭐⭐ (the most important step)
First ask: **"Should this even be ML?"** Often a **heuristic/rules baseline** is simpler, cheaper, more
explainable, and good enough — and ML is only worth it when patterns are too complex for rules *and* you
have data. *(DCE is a **rules+ML hybrid** for exactly this reason — you know when each wins.)*

If ML: **frame it precisely.**
- **What are we predicting?** → defines the **ML task**: regression (a number), **classification** (a
  label), **ranking** (an order), **recommendation** (top-k items).
- **What's the input (X) and the output (y / the label)?** And **where do labels come from?** (explicit
  human labels, **implicit feedback** like clicks/purchases, or **delayed** labels like a 30-day
  chargeback). *Labels are often the bottleneck.*
- **Online or batch prediction?** (real-time per request vs precomputed in bulk — Step 6).
- **What's the *business* metric** the model must move (revenue, fraud loss, engagement), and what
  **offline proxy** will you optimize? (Step 5 — these differ!)

> The framing question an interviewer is really testing: *can you translate a fuzzy product goal ("reduce
> fraud", "recommend videos") into a concrete ML task with a defined input, output, label source, and a
> business metric?* Get this wrong and the rest is moot — same as "requirements first" in m01.

---

### Step 2 — Data
The model is only as good as the data. Design the **data pipeline**:
- **Sources:** where does training data come from? (logs, transactions, user events, third-party.)
- **Collection + labeling:** how are labels obtained (human, implicit, delayed)? How much data, how fresh?
- **Pipeline:** ingest → clean → store (a **data lake/warehouse**, m04 OLAP) → produce training datasets.
  This is a **batch data-engineering** problem (your DCE Snowflake/ETL + claim-preprocessing world).
- **Issues to name:** **class imbalance** (fraud is <1% — m27), **label noise**, **bias**, **leakage**
  (a feature that secretly contains the answer), privacy/PII.

---

### Step 3 — Features & the training-serving skew problem ⭐⭐
**Features** are the model's inputs, derived from raw data (feature engineering — often more impactful than
the model, your AI/ML "features beat algorithms" lesson). The system-design crux:
- Features are computed in **two places**: the **offline training pipeline** (batch, over historical data)
  and the **online serving path** (live, per request). If they're computed **differently**, you get
  **training-serving skew** — *the #1 ML production bug* — the model sees a different feature distribution
  in production than it trained on → **silent degradation** (metrics look fine, predictions get worse).
- **The fix: a feature store** (m23) — compute features **once**, share the **same definitions** between
  training and serving, and ensure **point-in-time correctness** (don't leak future data into past
  training rows). *(Your DCE claim-preprocessing layer that produces 100+ consistent fields for the models
  is exactly this consistency discipline.)*

> **Carry this:** ML systems have **two data paths** (offline training + online serving), and the silent
> killer is **feature inconsistency between them.** Naming training-serving skew + the feature-store fix is
> a top ML-SD signal.

---

### Step 4 — Model (baseline first) ⭐
- **Start with a baseline:** a simple model (logistic regression, gradient-boosted trees) or even a
  heuristic. It's fast, sets a bar, and is often surprisingly good. **Don't reach for deep learning
  first** — your AI/ML lesson (NN == logreg on tabular → "don't default to DL"). Add complexity only when
  the baseline plateaus *and* it's worth the cost.
- **Model selection** by the task + constraints: **GBDTs (XGBoost/LightGBM)** for tabular (your DCE/HDC
  world), **deep nets** for text/images/embeddings, **two-tower** for retrieval (m24). Weigh **accuracy
  vs latency vs interpretability vs cost** (a fraud model needs <50 ms; a regulated decision needs
  explainability).
- The **training pipeline** itself is infra (m28): reproducible, versioned, scheduled.

---

### Step 5 — Evaluation: offline vs online ⭐⭐ (the insight most people miss)
**Offline metrics are *proxies*; the only truth is the online business metric.**
- **Offline evaluation:** on a held-out dataset, measure task metrics — accuracy/AUC/precision/recall
  (classification), RMSE (regression), NDCG/MAP (ranking). Cheap, fast, but it's a **proxy** for real
  impact.
- **The gap:** a model with **better offline AUC can be *worse* in production** — offline data is stale,
  the metric isn't the business metric, and the model **changes user behavior** (which offline data can't
  capture). *Offline tells you a model is plausibly better; it does **not** prove business value.*
- **Online evaluation (the real test): A/B testing.** Ship the new model to a fraction of traffic, measure
  the **actual business metric** (revenue, engagement, fraud loss) vs the control, with statistical
  significance. Use **shadow mode** (run the new model in parallel, log but don't act) and **canary** (m10)
  to de-risk first.

> **The senior line:** *"Offline metrics are necessary but not sufficient — they're proxies. I'd validate
> a model offline, then **A/B test it online against the business metric**, because better offline AUC
> doesn't guarantee better business outcomes (the offline-online gap). Shadow + canary first to de-risk."*
> *(Your AI/ML evaluation work — faithfulness/recall on a fixed set, the regression gate — is the offline
> half; the online A/B is what production adds.)*

---

### Step 6 — Serving: online vs batch inference ⭐
How predictions reach users — a core ML-SD decision:
- **Online (real-time) inference:** predict **per request, synchronously**, within a latency budget (e.g.
  fraud scoring <50 ms, m27). Needs a low-latency serving service (m24/m23), feature lookups, caching.
- **Batch (offline) inference:** **precompute** predictions in bulk on a schedule, store them, serve by
  lookup (e.g. nightly recommendation precompute). Cheaper, simpler, but **stale** (no fresh-context
  predictions). *(This is the precompute-into-snapshots pattern again — m13/Questimate.)*
- **The choice:** real-time freshness/context needed → **online**; predictions stable + latency-critical
  reads → **batch** (or a hybrid: batch candidates + online re-rank, m24). DCE runs a **batch-ish pipeline**
  over claims; a feed re-ranks **online**.

---

### Step 7 — Monitoring, drift & the feedback loop ⭐
ML systems **degrade silently** — the world changes but the model doesn't. So (m28):
- **Monitor:** data **drift** (input distribution shifts — your PSI/KS from InsightDesk), **concept
  drift** (the X→y relationship changes), prediction distribution, and the **business metric**.
- **Retrain:** triggered by drift/schedule; with an **eval gate** (don't promote a worse model — your
  InsightDesk retrain-trigger + gate).
- **The feedback loop / data flywheel:** predictions influence user behavior → which generates new training
  data → which retrains the model. Powerful (the flywheel) but **dangerous**: feedback loops can
  **amplify bias** (you only get labels for what the model surfaced) — so you watch for it.

---

## 3. The reference ML system architecture (ties it together)
```
   ┌──────────── OFFLINE (training) ────────────┐      ┌──────── ONLINE (serving) ────────┐
   data sources → data pipeline → feature store ──┬──▶ feature lookup → model service → prediction
   (logs/events)   (ETL, m04/m08)   (m23) ────────┘            (low latency)        │
                          │                                                          ▼
                    training pipeline → model registry (m28) → deploy (shadow/canary/A-B, m10) → users
                          ▲                                                          │
                          └──────── monitor (drift/metrics, m28) ◀── feedback/labels ┘  (the flywheel)
```
Two paths (offline training + online serving), one **feature store** keeping them consistent, a **registry**
+ **safe deploy**, and a **monitoring/feedback** loop. **Every box is a system you've already studied** —
ML system design is assembling them with the ML lifecycle.

---

## 4. From your systems 🏭 (you live this)
- **DCE = a production ML system at scale:** 1M claims/day, **rules + ML hybrid** (you know "should it be
  ML?"), a **batch-ish pipeline** (data → preprocess/features → models+rules → suppressions → decision),
  **drift/monitoring**, **reprocessing/retraining**, **audit**. You've operated every step of this
  framework.
- **Training-serving consistency, lived:** your **claim-preprocessing layer** produces **100+ consistent
  fields** the models consume — the discipline that prevents training-serving skew (Step 3).
- **Offline + online eval:** InsightDesk's **faithfulness/recall on a fixed set + a CI regression gate**
  is the offline half; you understand that production needs the **online** check (A/B). Your MLflow +
  **drift (PSI/KS) + retrain-trigger + eval-gate** is Step 7 in code.
- **Baseline-first:** your "**NN == logreg → don't default to DL**" finding *is* Step 4's lesson.
- **Online vs batch:** DCE's batch pipeline vs a real-time fraud score — you can argue both (Step 6).

---

## 5. Key concepts (interview-ready)
- **ML system design = classic SD + the ML lifecycle.** It's a production system with a model in it; the
  **data/feature pipelines, evaluation, and feedback/drift loop are the hard parts**, not the architecture.
- **Framework:** **Frame** (is it ML? what's predicted/the label/the business metric?) → **Data** → 
  **Features** (+ feature store) → **Model** (baseline first) → **Eval** (offline + online) → **Serve**
  (online vs batch) → **Monitor** (drift, retrain, feedback).
- **Frame first:** ask "should this be ML?" (rules baseline often wins); define **task, input, output,
  label source, business metric.**
- **Training-serving skew** (the #1 production bug): features computed differently offline vs online →
  silent degradation → **feature store** + point-in-time correctness fixes it. **Two data paths.**
- **Offline vs online evaluation** (the key insight): offline metrics are **proxies**; better offline AUC
  ≠ better business — **A/B test online** against the business metric (shadow/canary to de-risk).
- **Baseline first** (don't default to DL); model choice = accuracy vs latency vs interpretability vs cost.
- **Online vs batch inference:** real-time per-request vs precomputed bulk (the precompute pattern).
- **Monitor for drift** (data/concept), retrain with an **eval gate**, mind the **feedback loop / data
  flywheel** (powerful but can amplify bias).

---

## 6. Go deeper (the well-researched reading list)
- **"Designing Machine Learning Systems" — Chip Huyen** — the definitive ML-system-design book; this
  module is its framework distilled (framing, data, features, eval, deployment, monitoring).
- **"Machine Learning Design Patterns" (Lakshmanan et al.)** and **Google's "Rules of ML" (Martin
  Zinkevich)** — practical patterns + the baseline-first / "do the simple thing" wisdom.
- **"The ML Test Score" (Google) / "Hidden Technical Debt in ML Systems" (Sculley et al.)** — why the model
  is the small part and the system is the hard part.
- **Your own AI/ML notes (`../ai_ml_llm/` m09 evaluation, m11 serving, m12 MLOps)** — the offline-eval,
  serving, drift/retrain pieces you already built; revisit them as the implementation of this framework.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) (framing
problems as ML, offline-vs-online, skew) before checking
[`solutions/m22_ml_system_design_framework/`](../../solutions/m22_ml_system_design_framework/README.md).

When you've done the exercises, say **"Module 23"** to design *Data & Feature Pipelines / Feature Stores* —
the deep-dive on Step 3 (batch vs streaming features, **training-serving skew**, point-in-time correctness),
the foundation of every ML system.
