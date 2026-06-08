# m27 — Exercises (Fraud & Anomaly Detection)

> Do these on paper / out loud before checking `solutions/m27_fraud_anomaly/`. Imbalance, the hybrid,
> delayed labels, and the decision system are the core — and this is your DCE world.

---

## Exercise 1 — Why fraud is special
1. Name the four ways fraud inverts standard ML.
2. For each, name the design response.

---

## Exercise 2 — Imbalance
1. Fraud is 0.3%. Why is a 99.7%-accurate model possibly worthless?
2. Name the metrics you'd use instead.
3. List three techniques to handle the imbalance during training.

---

## Exercise 3 — The threshold
1. Why isn't the decision threshold 0.5?
2. A missed fraud costs $200; a false decline costs ~$5 of friction/churn. Which way do you lean the
   threshold, and what does that do to precision/recall?
3. Why might you use three thresholds instead of one?

---

## Exercise 4 — The hybrid
1. Give two strengths and one weakness of pure rules.
2. Give two strengths and one weakness of pure supervised ML.
3. Why does the hybrid beat both, and what does anomaly detection add?

---

## Exercise 5 — Delayed labels
1. Why can't you evaluate today's predictions today?
2. What is label leakage in fraud, and why is it tempting + dangerous?
3. Name two faster sources of labels than waiting for chargebacks.

---

## Exercise 6 — The decision system
1. Describe the 3-way decision and what goes to each branch.
2. What three things does the human-review queue provide?
3. Map this to a DCE concept.

---

## Exercise 7 — Adversarial drift
1. Why does fraud drift faster than ordinary concept drift?
2. What three defenses keep the system current?
3. How do rules give you a faster response than retraining when a new attack appears?

---

## Exercise 8 — Real-time features & serving
1. Name three velocity/recency features and why each helps.
2. Where do they come from, and what's the serving latency requirement?
3. How does the rules engine help meet the latency budget?

---

## Exercise 9 — Fraud rings & explainability
1. Why do per-transaction models miss fraud rings, and what catches them?
2. Why is explainability a hard requirement here, and how do you explain an ML score?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Argue that **DCE is a textbook instance of this module**. Map exclusion rules, ML+rule models,
   suppression rules, and DENY/BYPASS/ROUTE/UPDATE to the concepts.
2. How is **HDC** the imbalanced-scoring + threshold + review piece?
3. Where do your **drift monitoring (PSI/KS)** and **audit logging** fit?

---

When done, check `solutions/m27_fraud_anomaly/README.md`, then say **"Module 28"**.
