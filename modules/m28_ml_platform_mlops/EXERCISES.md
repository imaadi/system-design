# m28 — Exercises (ML Platform / MLOps End-to-End)

> Do these on paper / out loud before checking `solutions/m28_ml_platform_mlops/`. Versioning, drift, the
> eval gate, and CI/CD/CT are the core — and you built this in InsightDesk.

---

## Exercise 1 — MLOps vs DevOps
1. State the one-line difference.
2. What three (really four) things must version for reproducibility, and why?
3. What does CT add to CI/CD, and why does ML need it?

---

## Exercise 2 — Hidden technical debt
1. Explain the "model is the small box" insight.
2. What follows for how you should spend engineering effort?

---

## Exercise 3 — Platform components
List the eight platform components and one line on what each does. Mark which you've personally built/used.

---

## Exercise 4 — Drift monitoring
1. Why isn't latency/error monitoring enough for ML?
2. Define data drift, concept drift, prediction drift.
3. Which metric detects data drift, and what threshold roughly flags it? (Cite your own numbers.)

---

## Exercise 5 — Retraining & the eval gate
1. Name three retrain triggers.
2. What is the eval gate and why is it non-negotiable?
3. After the gate passes, what comes before full rollout?

---

## Exercise 6 — CI/CD/CT
1. What does "testing" mean in ML beyond code unit tests?
2. Define CI, CD, CT for ML.
3. Why does ML need CT but classic software doesn't?

---

## Exercise 7 — Reproducibility
1. List everything you pin to reproduce a model.
2. Why is this harder than reproducing a software build?
3. How does it enable safe rollback?

---

## Exercise 8 — Maturity & governance
1. Describe MLOps levels 0/1/2.
2. An org deploys models from notebooks by hand. What level, and what's the first improvement?
3. Name three governance concerns and why they matter in your domain.

---

## Exercise 9 — Diagnose
A model's offline accuracy is unchanged but the business metric dropped. List the likely causes and how
you'd investigate (use the monitoring signals).

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map **every** InsightDesk MLOps piece you built (MLflow, PSI/KS, alerts, retrain+gate, CI/CD) to the
   platform components.
2. How does **DCE** demonstrate operating an ML platform at scale (1M/day, team of 4)?
3. Where do you sit on the **DS↔Eng handoff**, and how does the platform make it reliable?

---

When done, check `solutions/m28_ml_platform_mlops/README.md`, then say **"Module 29"** (Part 4 — LLM System
Design begins).
