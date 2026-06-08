# Module 27 — Design Fraud & Anomaly Detection ⭐⭐ 🏭 (your DCE world)

> Detect fraud/anomalies in **real time** — block the bad, allow the good, route the uncertain. This is
> the ML problem that **inverts the usual rules**: **extreme class imbalance** (fraud <1% → accuracy is
> useless), **delayed/noisy labels** (you learn it was fraud weeks later via chargeback), **adversarial
> drift** (fraudsters actively adapt), and **asymmetric costs** (a missed fraud vs a blocked paying
> customer). The production answer is a **rules + ML hybrid** with a **3-way decision** (allow / block /
> **review**) — which is **exactly DCE**: exclusion rules → ML + rule-based models → suppression rules →
> DENY/BYPASS/**ROUTE**/UPDATE. You don't study this module; you've *operated* it at 1M claims/day.

> **Format (Part 3 deep-dive):** worked design; **class imbalance**, the **rules+ML hybrid**, **delayed
> labels**, and the **decision system** are the deep-dives.

---

## Step 1 — Requirements

**Functional:** for each event (transaction, claim, login, signup), **score fraud/anomaly risk** and
**decide: allow / block / route-to-review.**

**Non-functional (these — esp. the inverted ones — drive it):**
- **Real-time, low latency:** scoring is **inline** with the transaction → **tens of ms** (you can't make
  the customer wait, and you must block *before* the money moves).
- **High recall AND precision under asymmetric cost:** catch fraud (recall) **without** blocking legit
  customers (precision) — and the costs differ (a missed fraud vs a false decline).
- **Adapt to evolving/adversarial fraud:** fraudsters change tactics → **fast retraining + monitoring.**
- **Explainability / compliance:** especially regulated domains (healthcare, finance) — *why* was this
  flagged? (Your DCE auditability.)
- **Handle extreme class imbalance + delayed labels** (the data realities).

> **The defining insight:** fraud **inverts standard ML** — imbalance makes accuracy meaningless, labels
> arrive late and noisy, the adversary adapts, and costs are asymmetric. The system is a **hybrid
> (rules+ML) real-time decision engine with a human-in-the-loop feedback loop** — which is the DCE shape.

---

## Step 2 — The defining challenges ⭐⭐ (why fraud is special)

1. **Extreme class imbalance:** fraud is often **<1% (sometimes <0.1%)**. **Accuracy is useless** — a model
   predicting "never fraud" is 99%+ accurate and catches **nothing**. You **must** use **precision/recall/
   PR-AUC/F1** and reason about costs, not accuracy.
2. **Delayed & noisy labels:** you don't know a transaction was fraud until a **chargeback weeks later**;
   some fraud is **never labeled**; "labels" from manual review are noisy. So training data **matures
   slowly**, evaluation lags, and you must avoid **label leakage** (using the chargeback as a feature —
   m23 point-in-time).
3. **Adversarial / concept drift:** fraud is an **adversary that actively adapts to evade you** → the
   distribution shifts **faster** than ordinary drift. It's **cat-and-mouse** — retrain frequently, monitor
   aggressively.
4. **Asymmetric costs + the precision-recall trade:** a **missed fraud** costs $X (the loss); a **false
   positive** blocks a **paying customer** (lost revenue + friction + churn). The **threshold is a
   cost/business decision**, not a default 0.5.

> **Say this:** *"Fraud breaks the normal ML playbook: imbalance kills accuracy (use PR/recall), labels are
> **delayed and noisy** (chargebacks weeks later), the adversary **adapts** (fast drift → frequent
> retraining), and **costs are asymmetric** (a missed fraud vs a declined good customer → a **cost-
> sensitive threshold**). So I design a **rules+ML hybrid** with a **human-review loop**, not a single
> classifier."*

---

## Step 3 — The approaches ⭐ (and why hybrid wins)

- **Rules (expert-defined):** velocity checks ("> N txns / 5 min"), blocklists, thresholds, known-pattern
  flags. **Fast, explainable, instant to deploy** (react to a new known threat *now*) — but **rigid** and
  **gameable** (fraudsters learn the rules), and can't catch subtle/novel patterns. *(DCE's exclusion +
  suppression + rule-based models.)*
- **Supervised ML (classification on labeled fraud — GBDT/deep):** learns **complex patterns** from
  features; strong when you have labels. But needs **labels** (delayed), and only catches fraud **similar
  to the past** (struggles with novel attacks). *(DCE/HDC LightGBM.)*
- **Unsupervised / anomaly detection (no labels):** flag **outliers** — **isolation forests**,
  **autoencoders** (high reconstruction error = anomalous), clustering, statistical outliers. Catches
  **novel fraud you've never seen** (no labels needed) — but **noisy** (anomaly ≠ fraud) and hard to
  threshold.
- **Graph-based (fraud rings):** fraud is often **coordinated** (rings of connected accounts/devices/
  cards) → **graph features / GNNs** catch patterns a per-transaction model misses (e.g. "this card shares
  a device with 50 flagged accounts").

### The rules + ML hybrid ⭐⭐ (THE production answer = DCE)
Real systems **combine** them: **rules** for known patterns, instant response, hard guardrails, and
explainability; **ML** for the fuzzy/evolving/subtle patterns; **anomaly detection** for the novel; with
the scores/flags **combined** into a decision.
- **Why hybrid:** pure ML loses explainability + can't react instantly to a newly-known threat + needs
  labels; pure rules can't catch novel/subtle fraud + gets gamed. The **hybrid covers each other's gaps.**
- *This is **literally DCE**: **exclusion rules** at entry (keep clear cases out) → **ML models + rule-
  based models** producing recommendations together → **suppression rules** at the exit (block wrong
  outputs / guardrails) → a decision (DENY/BYPASS/**ROUTE**/UPDATE).* You built the canonical hybrid.

---

## Step 4 — Deep Dive: handling class imbalance ⭐
With <1% positives:
- **Metrics (not accuracy):** **precision** (of flagged, how many are fraud), **recall** (of fraud, how
  many caught), **PR-AUC** (precision-recall AUC — the right summary for imbalance), **F1**. Plot the
  **precision-recall curve** and pick the operating point by **cost**.
- **Resampling:** **oversample** the minority (duplicate/**SMOTE**-synthesize fraud) or **undersample** the
  majority, to give the model signal. **Class weights:** penalize missing the minority more in the loss.
- **Threshold tuning (cost-sensitive):** the decision threshold is **not 0.5** — set it by the **asymmetric
  cost** of a false negative (missed fraud loss) vs false positive (blocked customer). Often a **3-way
  threshold** (allow / review / block).
- **Anomaly detection** sidesteps imbalance entirely (it doesn't need fraud labels — it models "normal").

---

## Step 5 — Deep Dive: delayed labels & the decision system ⭐

### Delayed/noisy labels
- You learn the truth **late** (chargebacks weeks later) → training data **matures over a window**;
  evaluation must account for **unmatured** recent data. **Avoid leakage:** don't use a post-event signal
  (the chargeback) as a feature for predicting the event (m23 point-in-time correctness).
- **Manual review generates labels:** the human-review queue (below) produces ground truth that feeds
  retraining (the feedback loop) — and exploration/sampling helps get labels on cases the model would
  auto-allow.

### The decision system (3-way + human-in-the-loop) ⭐
Not a binary block/allow — a **3-way decision**: **allow** (low risk), **block** (high risk), **review**
(uncertain → a human queue). The **review queue**:
- Handles the ambiguous middle (where false-positive cost is high), **generates labels**, and provides a
  **safety valve** (a human catches model mistakes). *(DCE's **ROUTE** = exactly this — send to a human/
  downstream examiner.)*
- The **thresholds** between allow/review/block are **cost-tuned** and adjustable (a knob you turn as the
  fraud landscape / review capacity changes).

```
  event → features (incl. real-time velocity) → [rules + ML + anomaly] → combined risk score
        → threshold → ALLOW | REVIEW (human queue → label) | BLOCK
        → review outcomes + matured labels (chargebacks) → retrain (feedback loop)
```

---

## Step 6 — Real-time features, serving & the full system
- **Real-time / streaming features (m23) are critical:** **velocity** ("txns/card in last 5 min", "logins/
  IP in last hour", "amount vs user's history"), session/device signals, graph signals. Recent behavior is
  the strongest fraud signal → **streaming feature pipelines** (Flink/Kafka) feed the scorer. *(DCE merges
  real-time reference data at scoring.)*
- **Serving (m24):** **online, inline, <tens of ms** — a fast model (GBDT/optimized) + fast feature lookups
  + the rules engine, all in the transaction path. Block **before** the money moves.
- **The pipeline:** event → real-time + batch features → **rules engine + ML model + anomaly detector** →
  combined score → decision → (review queue) → **feedback loop** (review outcomes + matured labels →
  retrain). Plus the **explainability** layer (which rules/features fired → for review + compliance).

---

## Step 7 — Bottlenecks, scale & edge cases
- **Latency:** inline, tens of ms → fast models, cached/streaming features, the rules as a cheap first
  filter.
- **Imbalance + threshold:** PR-metrics + cost-sensitive thresholds + resampling/weights.
- **Adversarial drift:** **frequent retraining**, monitoring (PSI/KS on features, fraud-rate, model
  performance — your InsightDesk drift work), and human review to catch new patterns fast.
- **Label delay:** evaluate on matured data; use review labels; avoid leakage.
- **Explainability:** rules are inherently explainable; for ML use SHAP/feature attributions — **essential
  in regulated domains** (your healthcare world). DCE logs which rules/edits fired (audit).
- **False-positive cost:** monitor the **decline rate** + customer impact; the review queue absorbs the
  uncertain.
- **Observability (m10/m28):** fraud catch rate (recall), false-positive/decline rate (precision), review-
  queue volume, feature drift, score-distribution shift, $ loss prevented vs friction.

**Wrap-up:** *"Fraud inverts standard ML — **imbalance** (accuracy useless → PR/recall + cost-sensitive
threshold), **delayed/noisy labels** (chargebacks weeks later → matured-data eval + review labels, no
leakage), **adversarial drift** (frequent retraining + monitoring), **asymmetric costs**. So a **rules+ML
hybrid**: **rules** for known patterns/instant response/explainability/guardrails, **supervised ML** for
learned patterns, **anomaly detection** for novel fraud, **graph** for rings — scores combined into a
**3-way decision (allow / block / review)** served **inline in tens of ms** on **real-time velocity
features**, with a **human-review queue** that generates labels and feeds **retraining**. It's literally my
DCE pipeline — exclusions → ML+rule models → suppressions → DENY/BYPASS/ROUTE/UPDATE — at 1M claims/day."*

---

## State-of-the-art & real-world notes 📚
- **Stripe Radar** (payments fraud ML), **bank/card fraud systems**, **Feedzai/Sift** — production
  rules+ML hybrids with real-time velocity features + human review.
- **Imbalance:** SMOTE, class weighting, PR-AUC; **anomaly detection:** **Isolation Forest**, autoencoders,
  one-class SVM. **Graph:** **GNNs for fraud rings** (a hot area).
- **The hybrid rules+ML decision engine with human review** is the universal production pattern (fraud,
  claims, content moderation, AML) — your DCE is a textbook instance.

---

## From your systems 🏭 (you built the canonical version)
- **DCE *is* this module:** **exclusion rules** (entry guardrails) + **ML models + rule-based models**
  (combined scoring) + **suppression rules** (exit guardrails / block wrong outputs) → **DENY / BYPASS /
  ROUTE / UPDATE** — a **rules+ML hybrid 3-way+ decision engine** with **ROUTE = human review**. You
  operate the exact architecture this module designs.
- **HDC = imbalanced risk scoring + review routing:** high/low-risk **classification** with **LSA features**
  and low-risk overrides — class-handling + cost-tuned thresholds + routing the high-risk for review.
- **Real-time features:** merging **Redis reference/eligibility data at scoring** = the velocity/real-time
  feature pattern (m23).
- **Explainability + audit:** logging **which edits/rules fired** to Mongo = the compliance/explainability
  layer fraud demands.
- **Adversarial drift + retraining:** your **drift monitoring (PSI/KS) + retrain/eval-gate** (InsightDesk)
  + DCE reprocessing is the cat-and-mouse retraining loop.
- **LightGBM** = the standard fraud/risk model — your DCE/HDC choice.

---

## Key concepts (interview-ready)
- **Fraud inverts standard ML:** **extreme imbalance** (<1% → **accuracy useless → PR/recall/PR-AUC/F1 +
  cost-sensitive threshold**, resampling/SMOTE/class-weights), **delayed/noisy labels** (chargebacks
  weeks later → matured-data eval, review labels, **no leakage**), **adversarial concept drift** (fraudsters
  adapt → **frequent retraining + monitoring**), **asymmetric costs** (missed-fraud vs false-decline).
- **Approaches:** **rules** (fast/explainable/instant/guardrails, but rigid/gamed), **supervised ML**
  (learned patterns, needs labels, past-only), **anomaly detection** (novel fraud, no labels, noisy),
  **graph** (fraud rings). **The hybrid (rules + ML + anomaly) is the production answer** = DCE.
- **3-way decision + human-in-the-loop:** **allow / block / REVIEW** (uncertain → human queue → generates
  labels = the feedback loop + safety valve). Cost-tuned thresholds. (DCE = ROUTE.)
- **Real-time velocity features (m23)** are the strongest signal; serve **inline, tens of ms** (m24);
  **explainability** (SHAP / rule attributions) essential in regulated domains.
- **It's the universal rules+ML decision-engine pattern** (fraud/claims/moderation/AML) — your DCE at scale.

---

## Go deeper (reading)
- **Stripe Radar / Feedzai / Sift engineering blogs** — production fraud ML (hybrid, real-time, review).
- **Imbalanced-learn (SMOTE), Isolation Forest, autoencoder anomaly detection** docs/primers.
- **"Graph Neural Networks for fraud detection"** surveys (fraud rings).
- **Cost-sensitive learning + PR-AUC** references; calibration (m26) matters here too (a risk score feeding
  a decision).
- Revisit **m22** (framing/imbalance/eval), **m23** (real-time features/point-in-time), **m24** (inline
  serving), **m26** (scoring models/calibration), **m28** (drift/retrain) — and your **DCE work** is the
  living example.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m27_fraud_anomaly/`](../../solutions/m27_fraud_anomaly/README.md).

When you've done the exercises, say **"Module 28"** to design the *ML Platform / MLOps End-to-End* — the
training infra, experiment tracking, model registry, drift/monitoring, and retraining that ties **all of
Part 3 together** (and which you built in InsightDesk + operate in DCE).
