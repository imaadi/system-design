# m28 — Cross-Questions ("if-and-buts") on the ML Platform / MLOps

> Answer out loud in 2–3 sentences before reading the model answer. The code+data+model versioning, drift
> monitoring, the eval gate, and CI/CD/CT are where this is won — and you built it in InsightDesk.

---

### Q1. What is MLOps and how is it different from DevOps?
**A.** MLOps is **DevOps for ML, plus the data and model dimensions.** In DevOps you **version, test, and
deploy code**; in MLOps you do that for **code + data + model** (all three must align for reproducibility),
and you add **Continuous Training (CT)** because the **model degrades silently as data drifts.** So MLOps
needs extra machinery DevOps doesn't — feature stores, drift monitoring, model registries, automated
retraining with eval gates — because an ML system is a **production software system *plus* a live model
that rots.**

---

### Q2. Why is ML harder to operate than normal software?
**A.** Three reasons: **(1) it fails silently** — a drifted model returns 200s with confidently-wrong
predictions, no error (m22); **(2) three things version** (code + data + model), so reproducibility is far
harder; and **(3) the model is the small part** — "Hidden Technical Debt in ML Systems" shows the ML code
is a tiny box surrounded by data/feature/serving/monitoring/config infrastructure. So you need more
operational tooling (drift monitoring, retraining, lineage) than software, precisely because the failure is
invisible and the system is mostly *not* the model.

---

### Q3. What does it mean that "code + data + model" all version?
**A.** A model's behavior depends on the **code** that trained it, the **data** it trained on, and the
resulting **model artifact** (+ config/hyperparameters). To **reproduce** a result or **roll back** safely,
you must pin **all of them together** — the same code on different data gives a different model; the same
data with different code does too. So reproducibility requires versioning **code (git) + data (snapshots/
versioned datasets) + model (registry) + config**. This three-way (really four-way) versioning is the
defining MLOps complexity that software's "just version the code" doesn't have.

---

### Q4. What's the "Hidden Technical Debt in ML Systems" insight?
**A.** That the **actual ML model code is a tiny box** in a huge system — the surrounding **data
collection, feature engineering, data verification, serving infra, monitoring, configuration, and
pipelines** are the vast majority of the work and the debt. The lesson: **the model is the easy part; the
platform is the hard part.** It reframes ML engineering from "build a good model" to "build a reliable
*system* around the model" — which is why MLOps (the platform) exists and why this whole module matters
more than the modeling.

---

### Q5. What are the core components of an ML platform?
**A.** **Experiment tracking** (log runs/params/metrics/artifacts → reproducibility), a **model registry**
(version/lineage/stage-alias → rollback), a **feature store** (consistent batch+online features, m23),
**reproducible training pipelines/orchestration**, **model serving** (online/batch, safe rollout, m24),
**monitoring** (drift + performance), **CI/CD/CT**, and **governance** (lineage/audit/fairness). Every ML
system (recsys/fraud/ranking) runs **on** this shared platform — you build it once and many models reuse
it. I built these pieces in InsightDesk (MLflow + PSI/KS + retrain gate + GitHub Actions).

---

### Q6. Why does an ML system need monitoring beyond latency and errors?
**A.** Because ML **fails silently** (m22) — when the world shifts but the model is frozen, it returns
**confident, wrong predictions with no error**, so normal latency/error monitoring sees a "healthy"
service. You **also** monitor the **ML-specific signals**: **data drift** (input distribution vs training —
PSI/KS), **concept drift** (X→y relationship — accuracy on fresh labels), **prediction drift** (output
distribution), and **performance + business + cost** metrics. Without this layer you'd never notice the
model decaying until the business metric tanked.

---

### Q7. What is data drift and how do you detect it?
**A.** **Data drift** is when the **input feature distribution in production shifts away from the training
distribution** (e.g. a new user demographic, a holiday-season pattern) while the model stays fixed — so it's
operating on data unlike what it learned. You detect it per feature with statistical tests: **PSI
(Population Stability Index)** — a threshold like >0.2 flags significant drift — or the **KS test**. In my
InsightDesk work, **PSI was 0.018 when stable and 2.075 during a surge** — a clear drift signal that should
trigger investigation/retraining.

---

### Q8. Data drift vs concept drift vs prediction drift?
**A.** **Data drift:** the **inputs** shift (P(X) changes). **Concept drift:** the **relationship** shifts
(P(y|X) changes — the same inputs now map to different outcomes, e.g. fraud tactics evolve). **Prediction
drift:** the **model's outputs** shift (e.g. positive-rate jumps — my POSRATE_SHIFT alert), which can
signal either input drift or a problem. You monitor all three (plus actual performance when labels arrive),
because each is a different early-warning of silent degradation — and concept drift is the dangerous one
since the model's learned mapping is now wrong.

---

### Q9. When and how do you retrain a model?
**A.** **Triggers:** **scheduled** (daily/weekly), **drift-triggered** (a PSI/KS alarm), or **performance-
triggered** (accuracy drop on fresh labels) — cadence matched to how fast the domain drifts (fraud =
frequent). **How:** an **automated pipeline** pulls fresh data → features → trains a **candidate** → it
passes through the **eval gate** (Q10) → **shadow/canary/A-B** (m24) → promote, with registry rollback. In
InsightDesk this was a **retrain-trigger + eval-gate + PROMOTE** flow (skewed 0.71 → retrain 0.967 →
promote).

---

### Q10. What's the eval gate and why is it non-negotiable?
**A.** The **eval gate** requires a retrained **candidate to beat the current model on a fixed evaluation
set before it's promoted** — so you **never auto-deploy a regression.** It's non-negotiable because
**automated retraining is dangerous**: a bad data batch, a pipeline bug, or label noise could produce a
worse model, and without the gate you'd silently ship it. In my work a **degraded config FAILED the
faithfulness gate (0.83 < 0.85) → exit 1** — the gate caught it. Gate first, then shadow/canary/A-B for the
online check.

---

### Q11. What is CI/CD/CT for ML?
**A.** **CI** (Continuous Integration): test **code + data + model** — not just unit tests, but **data
validation** (schema/distribution/nulls) and **model validation** (does the candidate meet the bar / not
regress?). **CD** (Continuous Delivery): deploy the serving service safely (shadow/canary/A-B, m24/m10).
**CT** (Continuous Training): the **ML addition** — **automatically retrain** on new data/triggers (with the
eval gate). CT exists because the model rots as data drifts, so unlike software you must continuously
*re-create* the artifact, not just redeploy code. My CI was lint → test → docker build; CT was the retrain
loop.

---

### Q12. What does "testing" mean in ML beyond code unit tests?
**A.** Three layers: **code tests** (normal unit/integration tests on the pipeline code), **data
validation** (schema checks, distribution/range checks, null rates, freshness — catch a broken upstream
feed before it corrupts the model, m23), and **model validation** (does the candidate meet quality
thresholds, not regress vs current, behave on slices/edge cases, stay calibrated?). You also test for
**training-serving skew** and **fairness**. Because ML bugs are often **data bugs** (not code bugs),
data + model validation is as important as code tests — "The ML Test Score" (Google) formalizes this.

---

### Q13. What are the MLOps maturity levels?
**A.** **Level 0:** **manual** — notebook-to-production, manual training/deploy, no automation (where most
orgs start). **Level 1:** an **automated training pipeline** with a feature store, registry, and monitoring
— you can **continuously train** (CT) on new data. **Level 2:** **full CI/CD/CT** automation — automated
build/test/deploy of the *pipelines themselves*, rapid reliable experimentation. Naming where an org sits
and the next concrete step (e.g. "we're level 0 — first add experiment tracking + a registry + drift
monitoring to reach level 1") is a senior signal.

---

### Q14. How do you make an ML result reproducible?
**A.** **Pin everything:** **code** (git commit), **data** (a versioned/immutable snapshot — same rows, with
point-in-time correctness, m23), **model** (the exact artifact + version in the registry), and **config/
hyperparameters/environment** (containerized, e.g. Docker). With all four pinned, you can re-run training
and get the same model, **reproduce** any past result, and **roll back** confidently. Reproducibility is
the hardest MLOps infra problem precisely because it requires versioning **data and models**, not just code
— and it's the foundation of debugging, auditing, and safe rollback.

---

### Q15. What's the DS↔Eng handoff problem and how does the platform solve it?
**A.** A classic failure: **data scientists build a model in a notebook** with their own data/feature code,
then **engineers can't productionize it** (different feature computation → skew, no serving contract, not
reproducible). The platform **bridges the seam**: a **shared feature store** (same features train+serve),
**standardized training pipelines**, a **model registry** (a clean handoff artifact), and a **serving
contract** — so a model moves to prod reliably and consistently. I sit at exactly this seam — **integrating
the data-science team's models into the DCE production pipeline** — which is what the platform exists to
make reliable.

---

### Q16. Your model's accuracy looks fine but the business metric dropped. What's going on?
**A.** Several possibilities: **concept drift** (the world changed — the model's still "accurate" on stale
labels but wrong on current reality), a **feature pipeline break** (m23 — silently corrupted inputs),
**training-serving skew**, or the **offline-online gap** (m22 — the offline metric isn't the business
metric, or the model changed behavior). You'd investigate via **drift monitoring** (PSI/KS on features +
prediction distribution), check feature freshness/validity, verify the feature versions match, and look at
the **business-metric A/B** — the divergence between "model looks fine" and "business dropped" is the
classic silent-failure signature MLOps monitoring exists to surface.

---

### Q17. How is cost handled in an ML platform?
**A.** **Cost is a first-class monitored SLO** — training compute (esp. GPUs), serving inference cost, and
data/storage. You **monitor and alert** on it (my **OVER_BUDGET** alert treats cost like reliability),
**right-size** retraining cadence (don't retrain more than drift demands) and serving (model optimization,
batching, autoscaling, m24), and track **cost per prediction / per training run**. In LLM systems (m29/m33)
cost dominates even more. Treating cost as a monitored objective — not an afterthought — is a mature MLOps
practice (and especially relevant on a small budget).

---

### Q18. How does the platform serve many models/teams (multi-tenancy)?
**A.** The platform is **shared infrastructure**: a **multi-tenant feature store** (teams reuse + discover
features, m23), a **registry** (all models versioned in one place), **shared serving infra** (multi-model
serving / GPU sharing, m24), and **common monitoring + CI/CD/CT pipelines**. Build it **once**, and every
ML system (recsys, fraud, ranking) runs **on** it — gaining reuse, governance, and consistency, instead of
each team reinventing the lifecycle. This is the "platform" idea (Uber Michelangelo, Meta FBLearner) — it
turns ML from artisanal to industrial.

---

### Q19. How do all of m22–m27 sit on this platform?
**A.** The platform is the **substrate** for every Part-3 system: **m22's framework** is the lifecycle the
platform automates; **m23's feature store**, **m24's serving + registry**, and the monitoring/retraining are
platform components; and **m25 (recsys), m26 (ranking/CTR), m27 (fraud)** are **applications that run on
it** — each reusing the shared feature store, registry, serving, drift monitoring, and eval-gated
retraining. So m28 doesn't add a new system; it's the **operational platform that makes all the others
reliable, repeatable, and governable** — which is why it closes Part 3.

---

### Q20. You built this in InsightDesk. Summarize it as your MLOps story.
**A.** *"In InsightDesk I built an end-to-end MLOps stack: **MLflow** for **experiment tracking + a model
registry** with a **champion alias** for rollback; **drift detection** from scratch (**PSI** 0.018→2.075,
KS, concept-drift accuracy 0.964→0.640) feeding a **monitoring loop** with alerts (**DATA_DRIFT /
POSRATE_SHIFT / OVER_BUDGET** — including **cost as an SLO**); a **retrain-trigger + eval-gate + PROMOTE**
flow that **refused to ship a regression** (a degraded config failed the gate → exit 1); and **CI/CD**
(GitHub Actions: lint → test → docker build) on a hermetic, reproducible app. And at **DCE** I operate this
at scale — integrating the DS team's models, reprocessing, monitoring, and platform migrations to run 1M
claims/day with a team of 4. The model was always the small part; the platform is what I actually
build and run."* — that's the senior MLOps answer, and it's true.
