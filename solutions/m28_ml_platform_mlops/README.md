# m28 — Solutions (ML Platform / MLOps End-to-End)

> Check the *reasoning*. Recurring lessons: MLOps = DevOps + data/model; pin code+data+model; ML fails
> silently → drift monitoring; eval-gate retraining; you built this in InsightDesk.

---

## Exercise 1 — MLOps vs DevOps
1. **MLOps = DevOps + the data and model dimensions** — you version/test/deploy **code + data + model**,
   not just code.
2. **Code, data, model (+ config/environment)** — because a model's behavior depends on all of them; the
   same code on different data (or vice versa) yields a different model, so reproducibility needs all
   pinned.
3. **CT (Continuous Training)** — automated retraining. ML needs it because the **model degrades as data
   drifts**, so you must continuously *re-create* the artifact, not just redeploy code.

---

## Exercise 2 — Hidden technical debt
1. The **ML model code is a tiny box** in a huge system of data collection, feature engineering, serving,
   monitoring, config, and pipelines — the surrounding infra is the vast majority of the work/debt.
2. **Spend most engineering effort on the *system* (data/features/serving/monitoring/reproducibility), not
   on squeezing the model** — the platform is where reliability (and most failures) live, so that's where
   the leverage is.

---

## Exercise 3 — Platform components
**Experiment tracking** (log runs/params/metrics/artifacts) ✅MLflow · **Model registry** (version/lineage/
champion alias → rollback) ✅MLflow · **Feature store** (consistent batch+online features, m23)
✅Snowflake+Redis · **Training pipeline/orchestration** (reproducible, scheduled) ✅DCE pipelines ·
**Serving** (online/batch, safe rollout, m24) ✅FastAPI · **Monitoring** (drift/perf/business) ✅PSI/KS ·
**CI/CD/CT** (test code+data+model; continuous training) ✅GitHub Actions · **Governance** (lineage/audit/
fairness) ✅DCE audit. (You've touched every box.)

---

## Exercise 4 — Drift monitoring
1. Because ML **fails silently** — a drifted model returns 200s with **confidently-wrong** predictions and
   no error, so latency/error monitoring sees a "healthy" service.
2. **Data drift:** inputs shift (P(X) changes). **Concept drift:** the X→y relationship changes (P(y|X)).
   **Prediction drift:** the model's output distribution shifts.
3. **PSI (Population Stability Index)** (or KS) per feature; **PSI > ~0.2 flags significant drift** — in my
   InsightDesk work **PSI was 0.018 stable vs 2.075 on a surge** (and concept drift dropped accuracy
   0.964→0.640).

---

## Exercise 5 — Retraining & the eval gate
1. **Scheduled** (daily/weekly), **drift-triggered** (PSI/KS alarm), **performance-triggered** (accuracy
   drop on fresh labels).
2. The **eval gate** requires the retrained **candidate to beat the current model on a fixed eval set
   before promotion** — non-negotiable because automated retraining could otherwise **silently ship a
   regression** (bad data/pipeline bug). (My degraded config FAILED the gate → exit 1.)
3. **Shadow → canary → A/B** (m24) — the online check on the business metric — before full rollout, with
   registry rollback.

---

## Exercise 6 — CI/CD/CT
1. **Code tests** (pipeline code) **+ data validation** (schema/distribution/nulls/freshness) **+ model
   validation** (meets bar, doesn't regress, calibrated, fair) — ML bugs are often **data** bugs, so you
   test data and models, not just code.
2. **CI:** test code+data+model. **CD:** deploy serving safely (shadow/canary/A-B). **CT:** automatically
   retrain on new data/triggers (with the eval gate).
3. Because the **model rots as data drifts** — software just needs the code redeployed, but ML must
   **continuously re-create the model artifact** to stay accurate, which CT automates.

---

## Exercise 7 — Reproducibility
1. **Code** (git commit), **data** (immutable/versioned snapshot, point-in-time correct), **model** (exact
   artifact + registry version), and **config/hyperparameters/environment** (containerized).
2. Software reproduces from **code + deps**; ML adds **data and model** versioning — data is huge/mutable
   and models are derived from a specific data+code+config combo, so pinning all of them is much harder.
3. With everything pinned, you can **re-train to the same model**, **reproduce any past result**, and
   **roll back** to a known-good (code,data,model,config) tuple confidently.

---

## Exercise 8 — Maturity & governance
1. **Level 0:** manual, notebook-to-prod, no automation. **Level 1:** automated training pipeline (CT) +
   feature store + registry + monitoring. **Level 2:** full CI/CD/CT automation (automated build/test/
   deploy of pipelines).
2. **Level 0.** First improvement: add **experiment tracking + a model registry + drift monitoring** (and
   a reproducible training pipeline) to move toward Level 1 — get out of "notebook by hand."
3. **Lineage** (what produced this model — debugging/audit), **reproducibility** (re-run/roll back),
   **fairness/bias + audit trails** — critical in **healthcare** (your domain): regulated decisions need to
   be explainable, auditable, and non-discriminatory.

---

## Exercise 9 — Diagnose
**Likely causes:** **concept drift** (world changed — model still "accurate" on stale labels but wrong on
current reality), a **feature pipeline break** (m23 — silently corrupted/stale inputs), **training-serving
skew** (feature version mismatch), or the **offline-online gap** (the offline metric ≠ the business metric,
or the model changed behavior). **Investigate:** check **feature drift (PSI/KS) + prediction-distribution
shift**, verify **feature freshness/validity** and that **feature versions match the model**, look at the
**business-metric A/B**, and check for an upstream data change. The "accuracy fine but business dropped"
divergence is the classic **silent-failure** signature.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **InsightDesk → platform components:** **MLflow** = experiment tracking + **model registry (champion
   alias → rollback)**; **PSI/KS + concept-drift + monitoring loop (DATA_DRIFT/POSRATE_SHIFT/OVER_BUDGET)** =
   monitoring; **retrain-trigger + eval-gate + PROMOTE** = CT + safe promotion; **GitHub Actions (lint→test→
   docker build)** = CI/CD; hermetic reproducible app = reproducibility. You built §2–§5 in code.
2. **DCE = operating an ML platform at scale:** model integration (DS pickles), reprocessing, drift/
   monitoring, audit/lineage, and platform migrations (ENSO→K8s, OpenShift→EKS, Lambda/Glue→Snowpark) —
   running **1M claims/day with a team of 4** *requires* the automation/monitoring/retraining a platform
   provides.
3. **The DS↔Eng seam is literally your job** — you **integrate the data-science team's models into the DCE
   production pipeline.** The platform makes it reliable via a **shared feature representation** (consistent
   preprocessing), a **clean model artifact/registry handoff**, **standardized pipelines**, and a **serving
   contract** — so the model moves from DS to prod without skew, surprises, or one-off glue.
