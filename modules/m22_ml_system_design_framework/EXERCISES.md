# m22 — Exercises (ML System Design Framework)

> Do these on paper / out loud before checking `solutions/m22_ml_system_design_framework/`. Framing,
> offline-vs-online, and training-serving skew are the core.

---

## Exercise 1 — Should it even be ML?
For each, decide ML vs rules/heuristic (or hybrid) and justify:
1. Detect if a transaction amount exceeds a per-user limit.
2. Recommend which video a user watches next.
3. Flag a healthcare claim for manual review.
4. Compute sales tax for an order.
5. Predict 30-day customer churn.

---

## Exercise 2 — Frame it as ML
For "recommend videos a user will watch," specify: the **ML task**, the **input (X)**, the **output/
label (y)**, **where labels come from**, **online vs batch**, and the **business metric** + an **offline
proxy**.

---

## Exercise 3 — Run the framework
Pick "detect fraudulent transactions." Walk all 7 steps (Frame → Data → Features → Model → Eval → Serve →
Monitor) in 1–2 lines each.

---

## Exercise 4 — Training-serving skew
1. Define training-serving skew and why it's silent/dangerous.
2. Give a concrete example of how it happens (two teams, two code paths).
3. How does a feature store fix it (name two mechanisms)?

---

## Exercise 5 — The offline-online gap
1. Model B has higher offline AUC than model A. Is B definitely better? Why or why not?
2. Name three reasons offline can disagree with online.
3. Describe the safe rollout sequence from offline eval to full deployment.

---

## Exercise 6 — Labels
1. Name the three label sources and a trade-off of each.
2. "Delayed labels" — give an example and why it complicates evaluation.
3. What's the bias risk with implicit-feedback labels?

---

## Exercise 7 — Online vs batch
For each, choose online or batch inference (or hybrid) and justify:
1. Fraud score at checkout.
2. "Movies you may like" on a homepage.
3. Search results ranking.
4. A daily credit-risk score per account.

---

## Exercise 8 — Data hazards
1. What is data leakage, and why does it produce flattering offline metrics?
2. Fraud is 0.5% of transactions — why is accuracy a bad metric, and what do you use?
3. How does point-in-time correctness prevent leakage?

---

## Exercise 9 — Monitoring & drift
1. Data drift vs concept drift — define each.
2. Why do ML systems degrade "silently," and what do you monitor?
3. When you retrain, why gate before promoting?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map **DCE** to all 7 framework steps (it's a production ML system).
2. Which step does your **claim-preprocessing layer (100+ consistent fields)** address, and how?
3. Your InsightDesk work covered offline eval + drift + retrain-gate. Which framework steps are those, and
   what does production add that your offline work didn't?

---

When done, check `solutions/m22_ml_system_design_framework/README.md`, then say **"Module 23"**.
