# m23 — Exercises (Feature Pipelines & Stores)

> Do these on paper / out loud before checking `solutions/m23_feature_pipelines_stores/`. Batch-vs-
> streaming, the dual-store, and point-in-time correctness are the core.

---

## Exercise 1 — Batch or streaming?
For each feature, choose batch or streaming and justify:
1. A user's average order value over the last 90 days.
2. Number of transactions on this card in the last 5 minutes (for fraud).
3. A merchant's lifetime chargeback rate.
4. Items the user clicked in the current session.
5. A product's total all-time review count.

---

## Exercise 2 — The dual-store
1. Why does the same feature need two different stores?
2. What store type for the offline (training) side, and why?
3. What store type for the online (serving) side, and why?
4. Map this to your DCE setup (Snowflake, Redis).

---

## Exercise 3 — Training-serving skew
1. Define it and why it's silent.
2. Give a concrete example of two diverging implementations.
3. How does a feature store eliminate it?

---

## Exercise 4 — Point-in-time correctness
1. You're building a training row for a transaction from last March. Which feature values must you use?
2. What goes wrong if you use "current" feature values?
3. What feature-store capability makes this correct, and what does it require storing?

---

## Exercise 5 — Spot the leakage
A fraud model trains on a feature `account_total_chargebacks` (the account's lifetime chargeback count) to
predict whether a *given* transaction is fraud.
1. Why might this leak the future?
2. How would you compute it correctly for a past transaction?
3. Why does this produce great offline metrics but fail online?

---

## Exercise 6 — Architecture
1. Sketch the batch and streaming pipelines feeding the offline and online stores.
2. Lambda vs Kappa — define each and name the skew risk Lambda introduces.
3. Why is Kappa increasingly favored?

---

## Exercise 7 — Serving latency
1. A model needs 20 features at inference, with a 30 ms budget. How do you serve the features fast?
2. Where do hot entities (a celebrity user) cause trouble, and how do you handle it?

---

## Exercise 8 — Operational hazards
1. Your upstream pipeline silently changes a field from dollars to cents. What happens, and how do you
   catch it?
2. You add a new feature — what's the complication for training existing models?
3. An embedding model is retrained. What skew risk appears, and how do you avoid it?

---

## Exercise 9 — Do you even need one?
1. When is a full feature store over-engineering?
2. When does it earn its complexity (name three conditions)?
3. Even without the product, which two concepts still apply?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Argue that **DCE is already a feature-store dual-store**: map Snowflake, Redis reference data, and the
   claim-preprocessing layer to feature-store roles.
2. Which DCE features (membership overlap, date-overlap windows, NPI/Tax-ID match) map to §5's aggregation/
   join patterns?
3. Where does point-in-time correctness show up in healthcare claims specifically?

---

When done, check `solutions/m23_feature_pipelines_stores/README.md`, then say **"Module 24"**.
