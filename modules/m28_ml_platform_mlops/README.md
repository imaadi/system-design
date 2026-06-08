# Module 28 — ML Platform / MLOps End-to-End ⭐⭐ 🏭 — *closes Part 3*

> The **platform** that lets teams build, deploy, monitor, and **retrain** ML systems **reliably and
> repeatedly** — not one-off notebook models. It ties m22–m27 together: data → features (m23) → training →
> registry → serving (m24) → **monitoring → retraining → loop.** The core insight: **MLOps = DevOps +
> two new dimensions (data & model)** — you version/test/deploy **code + data + model**, and you add
> **Continuous Training (CT)** because models **degrade silently** as data drifts. You **built this** in
> InsightDesk (MLflow registry + PSI/KS drift + retrain-trigger + eval-gate + GitHub Actions CI/CD) and
> **operate it** at DCE (1M claims/day, team of 4 = heavy automation).

> **Format (Part 3 finale):** worked design of the platform; **the MLOps components**, **drift
> monitoring**, **CI/CD/CT**, and **eval-gated retraining** are the deep-dives.

---

## 1. Why MLOps exists (and why ML is harder to operate than software)
- **"Hidden Technical Debt in ML Systems" (Sculley et al.):** the **ML model code is a tiny box** in a vast
  system of **data collection, feature engineering, serving, monitoring, configuration, and pipelines.**
  *The model is the easy part; the surrounding platform is the hard part* (the m22 theme, as infra).
- **ML degrades silently** (m22): the world changes, the model doesn't, and **no error fires** — so ML
  needs **more** operational machinery (drift monitoring, retraining) than normal software, not less.
- **Three things version, not one:** **code + data + model** must align for **reproducibility** — a result
  is only reproducible if you pin all three. This is the defining MLOps complexity.

> **The framing:** *"MLOps is DevOps for ML, plus the **data and model** dimensions. In software you
> version, test, and deploy **code**; in ML you do that for **code + data + model**, and you add
> **Continuous Training** because the model rots as data drifts. The platform exists so every model team
> reuses one reliable lifecycle instead of reinventing it — and because the system, not the model, is
> where ML lives or dies."*

---

## 2. The ML platform — the components ⭐⭐ (ties m22–m27 together)
A platform every ML system (recsys, fraud, ranking) runs **on** — build once, many models reuse:

```
   data (m22) → DATA PIPELINE → FEATURE STORE (m23) → TRAINING PIPELINE → EXPERIMENT TRACKING
                                                              │                    │
                                                              ▼                    ▼
                                                        MODEL REGISTRY (m24) ◀── metrics/artifacts
                                                              │ promote (eval gate)
                                                              ▼
                                          SERVING (m24, shadow/canary/A-B) → predictions → users
                                                              │
                                  MONITORING (drift/perf/business) ──▶ RETRAIN TRIGGER ──▲ (the loop)
                                  CI/CD/CT · governance/lineage · orchestration
```

| Component | What it does | Your tool |
|---|---|---|
| **Experiment tracking** | log runs, params, metrics, artifacts → **reproducibility + comparison** | **MLflow** ✅ |
| **Model registry** | version, lineage, stage/alias (champion), **rollback** (m24) | **MLflow** ✅ |
| **Feature store** | consistent batch+online features, point-in-time (m23) | (DCE Snowflake+Redis) ✅ |
| **Training pipeline / orchestration** | reproducible, scheduled, automated training | Airflow/Kubeflow (DCE pipelines) ✅ |
| **Serving** | online/batch, dynamic batching, safe rollout (m24) | **FastAPI** ✅ |
| **Monitoring** | drift, performance, business metrics — catch silent failure | **PSI/KS** ✅ |
| **CI/CD/CT** | test+deploy code/data/model; **continuous training** | **GitHub Actions** ✅ |
| **Governance** | lineage, reproducibility, audit, fairness | (DCE audit) ✅ |

> You've touched **every box** — InsightDesk built the platform pieces; DCE operates them at scale.

---

## 3. Deep Dive: monitoring & drift ⭐ (the ML-specific observability)
A normal service monitors **latency/errors/saturation** (m10). An ML system **also** monitors the
ML-specific signals, because it **fails silently** (m22):
- **Data drift:** the **input distribution shifts** (vs training) → measure with **PSI (Population
  Stability Index)** or **KS test** per feature. *(Your InsightDesk: PSI 0.018 stable vs 2.075 on a
  surge.)*
- **Concept drift:** the **X→y relationship changes** → accuracy on fresh labels drops. *(Your concept-
  drift demo: 0.964 → 0.640.)*
- **Prediction drift:** the model's **output distribution** shifts (e.g. positive-rate jumps — your
  **POSRATE_SHIFT** alert).
- **Performance + business metrics:** the actual metric (when labels arrive), plus business KPIs and
  **cost** (your **OVER_BUDGET** alert — cost as a monitored SLO).
- **Alerting:** a **monitoring loop** raises alerts (DATA_DRIFT / POSRATE_SHIFT / OVER_BUDGET — your exact
  alerts) → triggers investigation/retraining.

> **The point:** **you cannot detect ML degradation with normal monitoring** — a drifted model returns
> 200s with confident-but-wrong predictions. Drift monitoring (PSI/KS + prediction/perf) is the
> **ML-specific layer** of observability. (This is your literal InsightDesk monitoring loop.)

---

## 4. Deep Dive: retraining & the eval gate ⭐
Because models drift, you **retrain** — but automated retraining is **dangerous** (a bad retrain could
silently ship a worse model). So:
- **Triggers:** **scheduled** (daily/weekly), **drift-triggered** (a PSI/KS alarm), or **performance-
  triggered** (accuracy drop on fresh labels). Fast-drifting domains (fraud, m27) retrain often.
- **The automated retrain pipeline:** fresh data → features → train a **candidate** → **evaluate** it.
- **The eval gate ⭐ (never skip):** the candidate must **beat the current model on a fixed eval set**
  before promotion — **don't auto-deploy a regression.** Then **shadow/canary/A-B** (m24) before full
  rollout, with **registry rollback**. *(Your InsightDesk: skewed retrain 0.71 → retrain 0.967 → eval gate
  passes → PROMOTE; and a degraded config FAILED the faithfulness gate 0.83<0.85 → exit 1.)*
- **Continuous Training (CT):** the automation of this loop — the ML addition to CI/CD (§5).

```
  drift/schedule trigger → retrain candidate → EVAL GATE (beat current?) → yes: shadow/canary/A-B → promote
                                                                          → no: keep current, alert
```

---

## 5. Deep Dive: CI/CD/CT for ML
Software has **CI/CD**; ML adds **CT (Continuous Training)** and a broader notion of testing:
- **CI (Continuous Integration):** test **code** *and* **data** *and* **models** — not just unit tests:
  **data validation** (schema, distributions, nulls), **model validation** (does the candidate meet the
  quality bar / not regress?), plus normal code tests. *(Your CI: lint → test → docker build.)*
- **CD (Continuous Delivery):** deploy the model-serving service safely (shadow/canary/A-B, m24/m10).
- **CT (Continuous Training):** **automatically retrain** on new data / triggers (with the eval gate) —
  the unique ML pipeline that keeps the model fresh against drift.
- **Reproducibility:** pin **code (git) + data (versioned/snapshot) + model (registry) + config** so any
  result can be reproduced and any deploy rolled back. *(Your hermetic, CI-buildable InsightDesk app.)*

---

## 6. ML maturity levels & governance
- **MLOps maturity (Google's levels):** **Level 0** — manual, notebook-to-prod, no automation (where most
  orgs start); **Level 1** — **automated training pipeline** (CT) with a feature store + registry +
  monitoring; **Level 2** — full **CI/CD/CT** automation (automated build/test/deploy of pipelines).
  Naming where an org sits — and the next step — is a senior signal.
- **Governance:** **model lineage** (what data/code/config produced this version), **reproducibility**,
  **audit trails** (your DCE compliance), **fairness/bias** checks, **access control**, and **model cards**
  / documentation. Essential in regulated domains (your healthcare world).

---

## 7. Bottlenecks, the org reality & edge cases
- **Reproducibility** (code+data+model+config) is the hardest infra problem — solved by versioning
  everything + the registry + pinned data snapshots.
- **The DS↔Eng handoff:** a classic failure — DS builds in a notebook, Eng can't productionize. The
  platform **bridges it** (standardized pipelines, the feature store, the registry, a serving contract) so
  models move to prod reliably. *(You sit at this seam — integrating the DS team's models into the DCE
  pipeline.)*
- **Retraining cost** (compute) vs freshness — retrain as often as drift demands, no more.
- **Monitoring the monitors:** drift alerts can be noisy → tune thresholds (m10 alert fatigue).
- **Scale:** the platform serves many models/teams → multi-tenant feature store, registry, serving (m24).
- **Observability (m10) + ML-specific (§3):** infra metrics **plus** drift/perf/business + cost.

**Wrap-up:** *"An ML platform is **DevOps + the data & model dimensions**: **experiment tracking** + a
**model registry** (version/lineage/champion alias → rollback), a **feature store** (consistent batch+
online features, m23), **reproducible training pipelines**, **safe serving** (shadow/canary/A-B, m24),
**drift + performance monitoring** (PSI/KS — because ML fails silently), and **CI/CD/CT** (test code *and*
data *and* models; **Continuous Training** with an **eval gate** so you never auto-promote a regression),
all reproducible by pinning **code+data+model+config**. Every ML system (recsys, fraud, ranking) runs **on**
this platform — built once, reused. The hard parts are **reproducibility**, **silent-degradation
monitoring**, and the **DS↔Eng handoff** — exactly what I built in InsightDesk (MLflow + PSI/KS drift +
eval-gated retrain + CI/CD) and operate at DCE."*

---

## State-of-the-art & real-world notes 📚
- **"Hidden Technical Debt in Machine Learning Systems" (Sculley et al., Google)** — the "model is the
  small box" paper; the founding MLOps text. **"The ML Test Score" (Google)** — what to test in ML.
- **Tools:** **MLflow / Weights & Biases** (tracking + registry), **Kubeflow / Airflow / Metaflow / Dagster**
  (orchestration), **Tecton/Feast** (feature store), **SageMaker / Vertex AI / Databricks** (managed
  platforms), **Evidently / WhyLabs** (drift monitoring).
- **Google's "MLOps: Continuous delivery and automation pipelines in ML"** — the maturity levels (0/1/2) +
  CI/CD/CT.
- **Uber Michelangelo / Meta FBLearner** — the canonical internal ML platforms.

---

## From your systems 🏭 (you built and operate this)
- **InsightDesk = an MLOps stack you built:** **MLflow** (experiment tracking + **model registry** +
  **`champion` alias**), **drift detection** (**PSI** 0.018→2.075, **KS**, concept-drift accuracy
  0.964→0.640), a **monitoring loop** with alerts (**DATA_DRIFT / POSRATE_SHIFT / OVER_BUDGET**), a
  **retrain-trigger + eval-gate + PROMOTE** (skewed 0.71 → retrain 0.967 → promote; a degraded config
  **FAILED** the gate → exit 1), and **CI/CD** (GitHub Actions: lint→test→docker build) on a hermetic,
  reproducible app. That's §2–§5 in code.
- **DCE = operating an ML platform at scale:** model integration (DS team's pickles), **reprocessing**,
  **drift/monitoring**, **audit/lineage**, and the **platform migrations** (ENSO→K8s, OpenShift→EKS,
  Lambda/Glue→Snowpark) — running 1M claims/day with a **team of 4** *requires* heavy MLOps automation.
- **The DS↔Eng seam is your job:** you **integrate the data-science team's models into production** — the
  exact handoff the platform exists to make reliable.
- **Cost as an SLO:** your **OVER_BUDGET** alert = monitoring cost like reliability (m10) — a mature MLOps
  touch.

---

## Key concepts (interview-ready)
- **MLOps = DevOps + the data & model dimensions:** version/test/deploy **code + data + model**;
  reproducibility = pinning all of code+data+model+config. **The model is the small box** (Sculley) — the
  platform is the hard part.
- **Platform components:** **experiment tracking**, **model registry** (version/lineage/champion alias →
  rollback), **feature store** (m23), **reproducible training pipelines/orchestration**, **serving** (m24,
  safe rollout), **monitoring**, **CI/CD/CT**, **governance**.
- **Monitoring is ML-specific** because ML **fails silently** (m22): **data drift (PSI/KS)**, **concept
  drift**, **prediction drift**, **performance + business + cost** — normal latency/error monitoring can't
  catch a confidently-wrong model.
- **Retraining + the eval gate:** triggers (schedule/drift/perf) → retrain candidate → **must beat current
  on a fixed eval set before promotion** (never auto-deploy a regression) → shadow/canary/A-B → rollback
  via registry. **Continuous Training (CT)** automates this.
- **CI/CD/CT:** test **code + data + model** (data validation, model validation); CT = automated retraining
  — the ML addition to CI/CD.
- **Maturity levels 0→1→2** (manual → automated pipeline → full CI/CD/CT). **Governance** = lineage,
  reproducibility, audit, fairness. **The platform unifies m22–m27** (build once, many models reuse) and
  bridges the **DS↔Eng handoff**.

---

## Go deeper (reading)
- **Sculley et al., "Hidden Technical Debt in Machine Learning Systems"** — the foundational MLOps paper.
- **Google Cloud, "MLOps: Continuous delivery and automation pipelines in machine learning"** — maturity
  levels + CI/CD/CT (the canonical framework).
- **Chip Huyen "Designing ML Systems" (deployment + data distribution shifts + monitoring chapters).**
- **MLflow / Kubeflow / Evidently docs**; **Uber Michelangelo** — real platforms.
- **Your own AI/ML notes (`../ai_ml_llm/` m12 MLOps, m13 Docker/CI-CD)** — you implemented this module;
  revisit them as the worked example.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m28_ml_platform_mlops/`](../../solutions/m28_ml_platform_mlops/README.md).

🎉 **This completes PART 3 — ML System Design (m22–m28):** framework, feature pipelines, serving, recsys,
ranking/CTR, fraud, and the MLOps platform. Say **"Module 29"** to begin **PART 4 — LLM System Design** with
*LLM Serving & Inference* (tokens, KV cache, continuous batching, vLLM, quantization, the cost math) — the
2026 differentiator, building on everything you already know about embeddings/RAG/agents.
