# m27 — Cross-Questions ("if-and-buts") on Fraud & Anomaly Detection

> Answer out loud in 2–3 sentences before reading the model answer. Imbalance, the rules+ML hybrid, delayed
> labels, and the decision system are where this is won — and it's your DCE world.

---

### Q1. What makes fraud detection different from a normal classification problem?
**A.** It **inverts the standard playbook**: **extreme class imbalance** (fraud <1% → accuracy is useless),
**delayed/noisy labels** (you learn it was fraud weeks later via chargeback), **adversarial concept drift**
(fraudsters actively adapt to evade you → fast-shifting distribution), and **asymmetric costs** (a missed
fraud vs blocking a paying customer differ greatly). So you can't just train a classifier and optimize
accuracy — you design a **rules+ML hybrid real-time decision engine with a human-review feedback loop**
(my DCE pipeline).

---

### Q2. Fraud is 0.5% of transactions. Why is accuracy a useless metric?
**A.** Because a model that predicts **"never fraud"** is **99.5% accurate** and catches **zero fraud** —
accuracy is dominated by the trivially-predictable majority. You must use **precision** (of flagged, how
many are actually fraud), **recall** (of fraud, how many you caught), **PR-AUC** (the right summary for
imbalance), and **F1**, and choose the operating point by the **asymmetric cost** of misses vs false
positives — not a default accuracy or 0.5 threshold.

---

### Q3. How do you handle the extreme class imbalance?
**A.** **(1) Right metrics** (precision/recall/PR-AUC, not accuracy). **(2) Resampling** — oversample the
minority (duplicate or **SMOTE**-synthesize fraud) or undersample the majority, to give the model signal.
**(3) Class weights** — penalize missing the rare positive more heavily in the loss. **(4) Cost-sensitive
threshold tuning** (not 0.5 — set by the cost of FN vs FP). **(5)** Optionally **anomaly detection**, which
sidesteps imbalance entirely by modeling "normal." You combine several; imbalance handling is a core part
of the design, not an afterthought.

---

### Q4. Why is the decision threshold not 0.5?
**A.** Because the **costs are asymmetric**. A **false negative** (missed fraud) costs the fraud **loss
($X)**; a **false positive** (blocking a legit transaction) costs **lost revenue + customer friction +
churn**. The optimal threshold balances these *specific* costs, which are almost never symmetric — so you
slide the threshold along the **precision-recall curve** to the **cost-minimizing** operating point (and
often use a **3-way** allow/review/block, not a single cut). The threshold is a **business/cost decision**,
not an ML default.

---

### Q5. Walk me through the rules + ML hybrid and why it beats either alone.
**A.** **Rules** (velocity checks, blocklists, thresholds) are **fast, explainable, instantly deployable**
(react to a new known threat *now*), and act as **guardrails** — but they're **rigid and gameable** and miss
subtle/novel fraud. **ML** learns **complex patterns** — but needs **labels**, only catches fraud **similar
to the past**, and is **less explainable**. The **hybrid covers each other's gaps**: rules for known/clear +
explainability + instant response, ML for the fuzzy/evolving middle, anomaly detection for the novel. It's
**the production answer** — and **literally DCE**: exclusion rules → ML + rule-based models → suppression
rules → decision.

---

### Q6. Why are delayed labels such a problem, and how do you deal with them?
**A.** You don't learn the truth until **weeks later** (a chargeback), some fraud is **never labeled**, and
review labels are **noisy** — so **training data matures slowly** and you **can't immediately evaluate**
recent predictions. You handle it by: **evaluating on matured data** (accounting for the label lag),
**using manual-review outcomes** as faster ground truth, **sampling/exploring** to get labels on auto-
allowed cases, and **avoiding leakage** (never use the post-event chargeback as a *feature* to predict the
event — point-in-time correctness, m23). The label-maturity window shapes your whole eval + retrain cadence.

---

### Q7. What is label leakage in fraud, and why is it dangerous?
**A.** Using a signal that's only known **after** the event — like the **chargeback** itself, or a
"fraud-confirmed" flag — as a **feature** to predict whether the event is fraud. It makes **offline metrics
look amazing** (the model "sees the answer") but the feature **isn't available at prediction time**, so it
**fails in production**. It's especially tempting in fraud because the post-hoc labels are so predictive.
You prevent it with **point-in-time correctness** (only features known as-of the transaction time, m23) and
skepticism of suspiciously perfect offline scores.

---

### Q8. Fraud is adversarial. What does that mean for the system?
**A.** Unlike a static problem, **fraudsters actively adapt to evade your model**, so the data distribution
**shifts faster and on purpose** — it's a **cat-and-mouse game**. Consequences: you **retrain frequently**
(stale models decay fast), **monitor aggressively** (feature drift, fraud-rate, performance — PSI/KS),
lean on **human review** to spot new patterns quickly, and use **anomaly detection** to catch attack types
you've never seen. You also avoid over-exposing your rules (which get reverse-engineered). The defense is
never "done" — it's continuous adaptation.

---

### Q9. What's the role of anomaly (unsupervised) detection here?
**A.** It catches **novel fraud you've never seen** — there are **no labels** for a brand-new attack, so a
supervised model (trained on past fraud) misses it, but an **anomaly detector** (isolation forest,
autoencoder reconstruction error, outlier detection) flags it as **unlike normal behavior**. It also
**sidesteps the imbalance problem** (it models "normal," not "fraud"). The downside is it's **noisy**
(anomalous ≠ fraudulent — a legit unusual transaction also looks anomalous) and hard to threshold — so it
**complements** the supervised model + rules rather than replacing them.

---

### Q10. Why a 3-way decision (allow/block/review) instead of binary?
**A.** Because of the **asymmetric cost + the uncertain middle**. Confidently-low-risk → **allow**;
confidently-high-risk → **block**; the **ambiguous middle** (where a false block is costly and a missed
fraud is costly) → **route to human review**. The review queue (1) handles the cases the model can't
confidently decide, (2) provides a **safety valve** (a human catches model mistakes), and (3) **generates
labels** that feed retraining (the feedback loop). It's exactly **DCE's ROUTE** — send the uncertain case
to a human/downstream examiner.

---

### Q11. What does the human-review loop give you beyond catching the case?
**A.** It's a **label factory and a safety valve.** The reviewers' decisions become **ground-truth labels**
(faster + cleaner than waiting for chargebacks) that feed **retraining**, and they catch **model mistakes
and new fraud patterns** that the model would have auto-allowed. It also lets you **sample/explore**
auto-allowed traffic to gather unbiased labels (else you only learn about what you flagged — a feedback
loop). The review queue is therefore central to **keeping the model fresh against adversarial drift**, not
just a per-case fallback.

---

### Q12. What real-time features matter most for fraud, and where do they come from?
**A.** **Velocity / recency features** are the strongest signal: "transactions on this card in the last 5
min", "logins from this IP in the last hour", "amount vs the user's historical average", "new device/
location for this account", and **graph signals** (shared device/card across accounts). They come from
**streaming feature pipelines** (Flink/Kafka, m23) maintaining real-time aggregates, fed to the inline
scorer. Fraud is often a **burst** (a stolen card used rapidly), so the last-few-minutes signal catches
what batch features miss — exactly why DCE merges real-time reference data at scoring.

---

### Q13. How do you serve fraud scoring within the latency budget?
**A.** It's **online, inline, in tens of ms** (you must decide before the money moves), so: a **fast model**
(GBDT/LightGBM or an optimized net), the **rules engine as a cheap first filter** (block obvious cases
instantly), **fast feature lookups** (cached + streaming velocity features from the online store, m23,
batched), and warm models on stateless replicas (m24). The rules give immediate coverage; the ML adds the
nuanced score; the whole thing fits the budget because the model is light and the features are
pre-aggregated. Block before commit.

---

### Q14. How do you evaluate a fraud model given delayed labels?
**A.** Carefully, on **matured data** — only score periods old enough that labels (chargebacks/review
outcomes) have mostly arrived, and account for the **unmatured recent window** (don't conclude "no fraud"
just because labels haven't come yet). Use **PR-AUC / precision@recall / recall@precision** at the operating
threshold (not accuracy), track **$ loss prevented vs false-decline cost**, and **A/B test online** (m22)
since offline can't capture the adversary's reaction. You also watch the **label-maturity curve** to know
when recent performance is trustworthy.

---

### Q15. Why is explainability especially important in fraud (and your domain)?
**A.** Because decisions have **real consequences** (a blocked customer, a denied claim) and often face
**regulation/audit** (finance, healthcare) — you must be able to say **why** something was flagged. **Rules
are inherently explainable** (which rule fired); for **ML**, you use **SHAP / feature attributions** to
explain a score. In healthcare claims (my DCE world) you **log which edits/rules fired** to an audit trail
— both for compliance and so a human reviewer can act on the reason, not just the score. Explainability is
a hard requirement, not a nicety, which is another reason the **hybrid** (with explainable rules) wins.

---

### Q16. How do you monitor a fraud system in production?
**A.** **Fraud catch rate (recall)** and **false-positive/decline rate (precision)** at the operating
threshold; **$ loss prevented vs friction/decline cost**; **review-queue volume** (a spike signals a new
attack or a drifting model); **feature drift + score-distribution shift** (PSI/KS — my InsightDesk work);
and **label-maturity**. Alerts: a recall drop or a decline-rate spike = the model is failing one side of the
trade. Because the adversary adapts, monitoring is **aggressive and continuous**, feeding **fast
retraining** (m28).

---

### Q17. How would you detect a fraud *ring* (coordinated accounts)?
**A.** Per-transaction models miss coordination, so you go **graph-based**: build a graph of **entities and
shared attributes** (accounts, devices, cards, IPs, addresses) and look for **dense/suspicious connected
components** — "this card shares a device with 50 flagged accounts", "these 200 signups share an address."
**Graph features** (degree, shared-attribute counts, community membership) feed the model, or a **GNN**
learns ring patterns directly. It catches what isolated-transaction scoring can't — the **connections** are
the signal, not any single transaction.

---

### Q18. A fraud model is silently getting worse. What's likely and how do you catch it?
**A.** Most likely **adversarial concept drift** — fraudsters found a new tactic the model wasn't trained
on — or a **feature pipeline break** (m23). You catch it via **monitoring**: **rising review-queue volume /
chargebacks**, **falling recall on matured labels**, **feature drift (PSI/KS)**, and **score-distribution
shifts**. The defense is **frequent retraining on fresh (incl. review) labels with an eval gate** (m28),
**anomaly detection** for the novel pattern, and human review to characterize and rule-patch the new attack
**immediately** (the rules' fast-response advantage) while the ML retrains.

---

### Q19. How is DCE an instance of this exact design?
**A.** Directly: DCE is a **rules+ML hybrid 3-way+ decision engine** for healthcare claims. **Exclusion
rules** at entry = guardrails/rules-first filter; **ML models + rule-based models** producing
recommendations together = the **hybrid scoring**; **suppression rules** at the exit = guardrails that
**block wrong outputs**; the decision **DENY / BYPASS / ROUTE / UPDATE** is the **multi-way decision** where
**ROUTE = human review**. It runs **inline-ish at 1M/day**, on **real-time reference features** (Redis),
with **audit logging** (explainability) and **drift/retrain** — every concept in this module, in
production. The HDC high/low-risk classifier with review routing is the imbalanced-scoring + threshold +
review piece.

---

### Q20. What's the single biggest "think differently" insight about fraud systems?
**A.** That **fraud isn't a model problem, it's a *system + adversary* problem**: the data is imbalanced,
the labels are late and noisy, and **an intelligent adversary actively breaks your model** — so a single
classifier optimized for accuracy is the wrong frame. The right frame is a **continuously-adapting
rules+ML hybrid with a human-in-the-loop**, where the **human review generates the labels** that fight the
**adversarial drift**, the **rules give instant response + explainability**, and the **threshold is a
cost decision**. Recognizing it as a **closed-loop, cost-sensitive, adversarial decision system** (which
DCE is) — not a Kaggle classifier — is the senior insight.
