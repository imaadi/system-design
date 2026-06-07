# m24 — Cross-Questions ("if-and-buts") on Model Serving

> Answer out loud in 2–3 sentences before reading the model answer. Online-vs-batch, dynamic batching, the
> registry, and safe rollout are where this is won.

---

### Q1. Online vs batch inference — the full spectrum and how you choose.
**A.** **Batch** (precompute in bulk on a schedule, serve by lookup — cheap, simple, but stale), **online**
(real-time per request within a latency budget — fresh, context-aware, more infra), **streaming** (score
events as they flow through Kafka/Flink), and **edge/on-device** (run on the user's device — lowest
latency + privacy, but compute-constrained). I choose by **freshness need, latency budget, and cost**, and
often go **hybrid** (batch candidates + online re-rank). DCE is batch-ish over claims; a fraud score is
online.

---

### Q2. Why "load the model once and warm it"?
**A.** Loading a model (especially onto a GPU) is **slow** — doing it per request would blow the latency
budget — so you **load it once at service startup** and keep it in memory/GPU. You also **warm it** with a
dummy inference so the **first real request** isn't slow (frameworks lazily init on first call), and use a
**readiness probe** (m07) so the LB only sends traffic once it's loaded. This is the **cold-start** fix —
exactly my InsightDesk `lifespan` that loads models once and warms the embedder.

---

### Q3. What is dynamic batching and why does it help?
**A.** GPUs are most efficient running a **batch**, but online requests arrive **one at a time**. **Dynamic
batching** collects requests for a few milliseconds (or until a max batch size) and runs them as **one GPU
forward pass** → much higher **throughput** + GPU utilization at a small **latency cost**. It's the **m02
latency-vs-throughput trade** made concrete: a **bigger batch = better GPU efficiency but higher per-request
latency** (each waits for the batch to fill). You tune the **max batch size + max wait time** to the
latency SLO.

---

### Q4. What's the trade-off in choosing the batch size for dynamic batching?
**A.** **Bigger batch → higher throughput / better GPU utilization (lower cost per inference) but higher
per-request latency** (requests wait longer to fill the batch); **smaller batch → lower latency but worse
GPU efficiency** (more idle GPU, higher cost). You set **max batch size + max wait** so the worst-case
added latency still fits the budget — e.g. "batch up to 32 requests or 5 ms, whichever first." It's the
classic latency-vs-throughput dial, tuned to the SLO.

---

### Q5. A model is too slow/expensive to serve in budget. What are your options?
**A.** **Optimize the model:** **quantization** (fp32 → int8 — smaller/faster, minor accuracy loss),
**distillation** (train a small student to mimic the big teacher), **pruning** (drop unimportant weights).
**Optimize serving:** **dynamic batching**, **caching** repeated predictions, better **hardware** (GPU for
big models, CPU for tabular/GBDT). **Or relax the requirement:** move to **batch** inference (precompute)
if real-time isn't truly needed. It's the accuracy-vs-latency-vs-cost trade — shrink the model and/or the
work per request.

---

### Q6. When would you cache predictions, and when is it useless?
**A.** **Cache** when the **same input recurs** — identical inputs yield identical predictions, so a cache
hit skips inference entirely (my InsightDesk `/ask` cache: a hit was 929× faster and $0). It's **useless**
when inputs are **unique** (every request different → ~0% hit rate, pure overhead), and **dangerous** when
predictions should reflect fresh context that the cache would stale out. So cache deterministic, repeated-
input predictions; don't cache personalized/fresh-context inference. (Same caching logic as m06.)

---

### Q7. What is a model registry and why do you need one?
**A.** A **central catalog of model versions** — the **source of truth for which model is live**. Each
version records its **artifact, metrics, and lineage** (data/code that produced it), plus a **stage/alias**
(`champion`/`production` vs `challenger`/`staging`). You need it for **versioning**, **reproducibility**,
**instant rollback** (repoint the alias to the previous version), and **safe promotion**. It's what makes a
model deploy auditable and reversible rather than "someone scp'd a pickle file" — my MLflow registry +
`champion` alias is exactly this.

---

### Q8. Why can't you just hot-swap a new model in production?
**A.** Because a new model can be **silently worse** — better offline metrics don't guarantee better online
behavior (m22), and there's no error to alert you; it just degrades the business metric. So you **roll out
gradually**: **shadow** (run in parallel, log, no impact) → **canary** (small % traffic) → **A/B** (measure
the business metric vs control) → full, with **instant rollback**. ML deploys need **more** caution than
code deploys precisely *because* the failure is silent. Hot-swapping risks a quiet, unnoticed regression.

---

### Q9. What is shadow deployment and what does it give you?
**A.** Run the **new model in parallel** with the live one: it receives **real production traffic** and
**logs its predictions, but its outputs are not used** (the old model still serves users). It gives you
**zero-risk real-traffic validation** — you catch crashes, latency problems, and prediction differences
**before any user is affected**, and you can compare the new model's predictions against the old on live
data. It's the safest first step before canary/A-B, because there's no user impact at all.

---

### Q10. Shadow vs canary vs A/B — distinguish them.
**A.** **Shadow:** new model gets real traffic but its output is **logged, not used** — zero user impact,
validates behavior/latency. **Canary:** a **small % of users actually get** the new model's output — limited
real exposure, watch SLOs, ramp or roll back (m10). **A/B test:** split traffic and **measure the business
metric** (treatment vs control) with statistical significance — the only real proof the new model is
*better*, not just *working*. The progression: shadow (does it run?) → canary (does it run on real users
safely?) → A/B (is it actually better?).

---

### Q11. What's a multi-armed bandit in model rollout?
**A.** Instead of a **fixed** A/B split, a **bandit adaptively shifts more traffic to the better-performing
model** as evidence accumulates — minimizing "regret" (traffic wasted on the worse variant). It learns and
exploits the winner faster than a static A/B while still exploring. Trade-off: it's **more complex** and can
complicate clean statistical inference. Use it when you want to optimize outcomes during the rollout (and
have multiple variants); use a clean A/B when you need a rigorous, interpretable result.

---

### Q12. How do you decide CPU vs GPU for serving?
**A.** By **model type + latency/cost**. **GPUs** for **large neural nets** (deep learning, embeddings,
LLMs) where parallel matrix math dominates and batching helps. **CPUs** for **tabular models (GBDTs like
XGBoost/LightGBM — most enterprise ML, my DCE world)** and small models — they're cheaper and fast enough,
and GPUs would be wasted. GPUs are **expensive**, so you only pay for them when the model genuinely needs
them, and then maximize utilization with **batching + multi-model sharing**. Right-size the hardware to the
model.

---

### Q13. GPUs are expensive. How do you keep serving cost down?
**A.** **Maximize GPU utilization** (idle GPU = wasted money): **dynamic batching** (more inferences per
pass), **multi-model serving / GPU sharing** (many models on shared GPUs), and **autoscaling** (scale-to-
zero for spiky/dev workloads, keep warm for latency-critical). **Shrink the model** (quantization/
distillation) so it needs less/cheaper hardware. **Cache** repeated predictions. And **use CPU** for
models that don't need a GPU. The theme: don't let expensive accelerators sit idle, and don't use them when
you don't need to.

---

### Q14. What's the difference between rolling your own serving (FastAPI) and a model server (Triton/KServe)?
**A.** **FastAPI + your model** = full control, simple, great for one model / moderate scale (my InsightDesk
approach) — but you build batching/versioning/multi-model yourself. **Dedicated model servers** (NVIDIA
**Triton**, **KServe**, **TorchServe**, **BentoML**, **Seldon**) give **dynamic batching, multi-model
serving, versioning, GPU scheduling, and metrics out of the box** — worth it at scale or with many models/
GPUs. Build-vs-buy: FastAPI when simplicity/control wins; a model server when you need its production
features and don't want to reinvent them.

---

### Q15. How do you package a model for serving, and why does the format matter?
**A.** Serialize the trained model to a **portable artifact**: **pickle/joblib** (Python-native, simple —
my DCE pickle integration), **ONNX** (framework-agnostic — decouples the training framework from the
serving runtime, avoiding lock-in and enabling optimized runtimes), **TorchScript** (PyTorch), or TF
**SavedModel**. Format matters for **portability** (ONNX lets you train in PyTorch but serve in a fast
optimized runtime), **performance** (compiled formats are faster than pickle), and **security** (pickle can
execute arbitrary code — risky for untrusted models). The service loads this artifact at startup.

---

### Q16. How do you avoid serving features inconsistent with training (skew) in the serving path?
**A.** Fetch features from the **same feature store / definitions** used in training (m23), and pin the
**feature version to the model version** — so the model serves on features computed identically to what it
trained on. The model service does a **low-latency online-store lookup** (m23) for the entity's features,
not a re-implementation in serving code. If you change feature logic, you retrain + redeploy together. This
is why serving and the feature store are co-designed — skew is a serving-time failure you prevent by
construction.

---

### Q17. How do you scale the model-serving tier?
**A.** Keep the model service **stateless** (model loaded from the registry, no per-request state) and
**horizontally replicate** it behind a **load balancer** (m07), **autoscaling** on QPS/latency. For
**GPUs**, scale more carefully (expensive) — keep a warm baseline for latency-critical paths, scale-to-zero
for spiky/dev, and use **multi-model serving** to pack models onto shared GPUs. **Readiness probes** ensure
new replicas only get traffic once the model is loaded + warm. It's the m07 scaling story with model-
loading + GPU-cost wrinkles.

---

### Q18. How do you monitor a serving system, and what's unique vs a normal service?
**A.** Normal-service metrics: **p99 latency, throughput, error rate, GPU/CPU utilization** (m10).
**Unique to ML:** **prediction-distribution monitoring** and **feature drift** (m28) — because the model can
**degrade silently** (m22) with no error, you watch whether its inputs (feature drift) and outputs
(prediction distribution) are shifting, plus the business metric. Also **cache hit rate** and
**model-version** in use. You alert on **latency-budget breach** *and* **drift** — the latter is the
ML-specific early warning a normal service doesn't need.

---

### Q19. Walk me through serving a fraud score in <50 ms.
**A.** Request arrives → **fetch features** from the **online store** (m23) in a **single batched low-latency
lookup** (a few ms) → run the (small, **quantized**, CPU-or-GPU) model — possibly with **dynamic batching**
tuned so added latency stays in budget → return the score; **cache** nothing (each transaction is unique).
The model is **loaded + warmed** at startup, served from **stateless replicas** behind a LB, with the
**registry**-pinned version. Watch: feature-lookup latency (often the bottleneck), model latency, and drift.
Sub-50 ms means a small model + fast feature fetch + minimal hops.

---

### Q20. How does this module connect to the rest of Part 3?
**A.** It's the **production endpoint** of the ML lifecycle: it serves the **model** (m22) using the
**features** from the feature store (m23), and it's where **recommendation/ranking/fraud** systems
(m25–m27) actually run inference (often **batch candidates + online re-rank**). It reuses **caching** (m06),
**load balancing/scaling/readiness** (m07), and **safe deploy** (m10), gated on the **business metric**
(m22), with monitoring feeding **MLOps** (m28). And it's the warm-up for **LLM serving** (m29), the extreme
case (KV cache, continuous batching, huge models) — serving is where ML meets the systems toolkit you
already mastered.
