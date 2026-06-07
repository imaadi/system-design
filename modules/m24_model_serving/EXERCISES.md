# m24 — Exercises (Model Serving & Inference at Scale)

> Do these on paper / out loud before checking `solutions/m24_model_serving/`. Online-vs-batch, dynamic
> batching, the registry, and safe rollout are the core.

---

## Exercise 1 — Pick the serving mode
For each, choose batch / online / streaming / edge and justify:
1. Nightly "recommended for you" on a homepage.
2. Fraud score at checkout (<50 ms).
3. On-device face unlock.
4. Scoring a stream of IoT sensor events for anomalies.
5. A daily churn-risk score per customer.

---

## Exercise 2 — Cold start
1. Why is loading the model per request a problem?
2. What two things do you do at startup, and why each?
3. Which earlier-module probe ensures traffic only hits a loaded model?

---

## Exercise 3 — Dynamic batching
1. Explain dynamic batching and the trade-off it makes (tie to m02).
2. You have a 40 ms budget. How do you set max batch size + max wait?
3. Why does it help GPUs specifically, and less so CPUs/GBDTs?

---

## Exercise 4 — Make it cheaper/faster
A neural model is accurate but too slow/costly to serve in budget. List four levers (model-side and
serving-side) and the trade-off of each.

---

## Exercise 5 — Caching predictions
1. When does caching predictions help, and when is it useless/harmful?
2. Give an example from your own work where prediction caching paid off.

---

## Exercise 6 — The model registry
1. What does a model registry store, and what does it enable?
2. How does it give you instant rollback?
3. Map it to your MLflow work.

---

## Exercise 7 — Safe rollout
1. Why never hot-swap a model?
2. Order shadow / canary / A-B and say what each proves.
3. Why do ML deploys need *more* caution than code deploys?

---

## Exercise 8 — Hardware & cost
1. CPU or GPU for: a LightGBM fraud model? a 1B-param text model? Justify.
2. GPUs are expensive — name three ways to keep serving cost down.
3. What is multi-model serving and why does it save money?

---

## Exercise 9 — Build vs buy + packaging
1. FastAPI vs a model server (Triton/KServe) — when each?
2. Why might you export to ONNX instead of serving a pickle?
3. Name one security concern with pickle.

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map **InsightDesk's FastAPI serving** (lifespan load+warm, `/ask` cache) to this module's concepts.
2. Map **DCE's pickle-model integration** + **MLflow champion alias** to packaging and the registry.
3. Design serving for a <50 ms fraud score: features, model, batching, caching — end to end.

---

When done, check `solutions/m24_model_serving/README.md`, then say **"Module 25"**.
