# Module 24 — Model Serving & Inference at Scale ⭐⭐ 🏭

> You've framed the problem (m22), built feature pipelines (m23) — now **get the model into production**:
> serve predictions reliably, within a latency budget, at scale, and **roll out new models safely**. The
> hard parts: the **online-vs-batch** spectrum, **latency optimization** (dynamic batching, quantization,
> caching), the **model registry**, and **safe rollout** (shadow → canary → A/B) — because a bad ML deploy
> fails **silently** (m22), so you need *more* caution than a code deploy. This is **your InsightDesk
> FastAPI serving** (load-once + warm + cache) and **DCE pickle-model integration** made systematic.

> **Format (Part 3 deep-dive):** worked design of the serving system; **dynamic batching**, the **model
> registry**, and **safe rollout** are the deep-dives.

---

## 1. Requirements for a serving system
- **Latency:** predictions within a budget (e.g. fraud <50 ms, search ranking <100 ms, a feed re-rank
  <200 ms). The budget drives everything.
- **Throughput:** the QPS the model tier must sustain.
- **Scalability + availability:** scale replicas, survive failures (Parts 0–2 apply).
- **Cost:** GPUs/accelerators are **expensive** → cost is a first-class constraint.
- **Safe deployability:** ship new model versions without breaking production (silently).
- **Consistency with training:** serve the **same features** the model trained on (m23 — skew).

---

## 2. The serving spectrum: batch → online → streaming → edge ⭐

How predictions reach users — a core decision (recap m22, now the full spectrum):
- **Batch (offline) inference:** precompute predictions in bulk on a schedule, store them, **serve by
  lookup**. Cheapest, simplest, highest throughput — but **stale** (no fresh context). *(Nightly recs;
  the precompute-into-snapshots pattern — m13/Questimate.)*
- **Online (real-time) inference:** predict **per request, synchronously**, within a latency budget.
  Fresh, context-aware — but needs low-latency serving + feature lookups + more infra. *(Fraud, search.)*
- **Streaming inference:** predict on events as they flow through a stream (Kafka/Flink) — near-real-time,
  for event-driven scoring.
- **Edge / on-device inference:** run the model **on the user's device** (phone, browser). Lowest latency,
  works offline, privacy-preserving — but constrained by device compute/memory (needs small/optimized
  models).

```
   batch ──────────────▶ online ──────────────▶ streaming ─────────▶ edge/on-device
   cheap, stale          real-time, fresh        event-driven         lowest-latency, constrained
   (precompute+lookup)   (per-request)           (Kafka/Flink)        (model on device)
```

> **Choose by:** freshness need, latency budget, cost, and where the data/context lives. Many systems are
> **hybrid** (batch candidates + online re-rank — m24/m25). DCE is **batch-ish** over claims; a fraud
> score is **online**.

---

## 3. The serving architecture & model packaging
```
   request ─▶ Model Service (API, e.g. FastAPI/Triton)
                 │ 1. fetch features (online feature store, m23)
                 │ 2. run model.predict() (CPU/GPU, dynamic batching)
                 │ 3. (optional) cache the prediction
                 ▼
              prediction ─▶ caller
   Model registry (m28) ──▶ which version is live? ──▶ load model on startup (warm it)
```
- **Model packaging/serialization:** the trained model is saved (**pickle/joblib**, **ONNX**,
  **TorchScript**, TF **SavedModel**) and loaded by the service. **ONNX** is a portable format that
  decouples training framework from serving runtime (avoids lock-in). *(DCE integrates the DS team's
  **pickle** model files — you've done exactly this packaging-and-loading step.)*
- **Load once + warm at startup** (kill cold-start, §6): load the model into memory/GPU when the service
  boots, and warm it (a dummy inference) so the first real request isn't slow. *(Your InsightDesk
  `lifespan` that loads models once + warms the embedder — this exact lesson.)*

---

## 4. Deep Dive: latency optimization ⭐⭐

The serving challenge is meeting the latency budget cheaply. Levers:

### Dynamic (adaptive) batching ⭐ — the GPU lever
GPUs are most efficient processing a **batch**, but online requests arrive **one at a time**. **Dynamic
batching** collects incoming requests for a few milliseconds (or until a max batch size) and runs them as
**one batch** → much higher **GPU throughput** at a small **latency cost**.
- This is the **m02 latency-vs-throughput trade** made concrete: **bigger batch = better GPU utilization +
  throughput, but higher per-request latency** (each request waits for the batch to fill). You tune the
  **max batch size + max wait time** to the latency budget.

```
  requests: r1 r2 r3 ...  → collect for ~5 ms (or until batch=N) → ONE GPU forward pass → split results
  ↑ throughput (GPU efficient)   ↑ latency (wait to fill the batch)   → tune batch size & wait to the SLO
```

### Model optimization (make the model cheaper to run)
A big accurate model may be too slow/expensive → shrink it (accuracy-vs-latency-vs-cost trade):
- **Quantization:** lower precision (fp32 → **int8**) → smaller + faster, minor accuracy loss. *(Your AI/ML
  int8 quant work.)*
- **Distillation:** train a **small "student" model to mimic a big "teacher"** → much cheaper, near the
  accuracy.
- **Pruning:** remove unimportant weights → smaller/faster.
- **Hardware:** GPU/TPU for big models; CPU for small/tabular (GBDTs run fine on CPU — most enterprise ML).

### Caching predictions
If the same input recurs, **cache the prediction** (m06) → skip inference entirely. *(Your InsightDesk
`/ask` cache: a hit was 929× faster and $0.)* Great when inputs repeat; useless for unique inputs.

---

## 5. Deep Dive: the model registry & safe rollout ⭐⭐ (the ML deploy)

### The model registry
A **central catalog of model versions** — the source of truth for "which model is live." It stores each
version's **artifact, metrics, lineage** (what data/code produced it), and **stage/alias** (e.g.
`champion`/`production` vs `challenger`/`staging`). It enables **instant rollback** (point the alias back
to the previous version) and reproducibility. *(Your MLflow registry + `champion` alias work — m12.)*

### Safe rollout — NEVER just swap a model ⭐
A new model can be **silently worse** (m22 — better offline ≠ better online), so you roll out gradually,
the same staged approach as m10 but gated on the **business metric**:
1. **Shadow deployment:** run the **new model in parallel** with the live one — send it real traffic,
   **log its predictions, but don't act on them**. Catches crashes/latency issues + lets you compare
   predictions, **zero user impact**.
2. **Canary:** route a **small % of real traffic** to the new model, watch metrics/SLOs; ramp up or roll
   back (m10).
3. **A/B test:** split traffic and measure the **actual business metric** vs control with significance
   (m22 — the only real proof of value).
4. **Full rollout**, with **instant rollback** (registry alias) if anything regresses.
- **Multi-armed bandit (advanced):** adaptively shift more traffic to the better-performing model as
  evidence accumulates (vs fixed A/B splits) — faster, but more complex.

> **The senior line:** *"I never hot-swap a model. I promote it through the **registry** and roll out
> **shadow → canary → A/B**, gating on the **business metric** with **instant rollback** — because a bad ML
> deploy fails **silently** (good offline, worse online), so I need *more* caution than a code deploy, not
> less. Shadow gives zero-risk real-traffic validation; A/B gives the only real proof it's better."*

---

## 6. Cold start, scaling & multi-model serving
- **Cold start:** loading a large model (esp. onto GPU) is slow → **load once at startup + warm it**
  (a dummy inference), keep replicas warm, and use **readiness probes** (m07) so traffic only hits a
  loaded model. *(Your m11 lifespan lesson.)*
- **Scaling:** **horizontally replicate** the stateless model service behind a load balancer (m07);
  **autoscale** on QPS/latency; for GPUs, scale carefully (expensive — scale-to-zero for spiky/dev
  workloads, keep warm for latency-critical).
- **Multi-model serving / model-as-a-service:** serve **many models on shared infra** (GPU sharing,
  multi-tenant model servers) to amortize expensive hardware — load models on demand, evict cold ones.
- **Build vs buy:** roll-your-own (**FastAPI** + your model — your InsightDesk approach, full control) vs
  **dedicated model servers** (**TF Serving, TorchServe, NVIDIA Triton, KServe, BentoML, Seldon**) that
  give batching, multi-model, versioning, metrics out of the box. Use a server when you need their
  features at scale; FastAPI when simplicity/control wins.

---

## 7. Bottlenecks, cost & edge cases
- **Latency vs cost (the GPU batching trade):** dynamic batching + right-sized models + caching keep
  latency in budget without over-spending on GPUs. GPUs idle = money burned → batch + multi-model + autoscale.
- **Feature-lookup latency:** the online feature fetch (m23) is on the critical path → fast KV + batched
  lookups (else features, not the model, are your latency).
- **Model size / memory:** big models → quantize/distill, or shard the model (model parallelism for huge
  ones — m29 for LLMs).
- **Silent failures:** monitor **prediction distribution, latency, feature drift** (m28) — a model can
  degrade with no error (the m22 lesson).
- **Versioning consistency:** serve the model with the **feature version it trained on** (m23) — a
  mismatch reintroduces skew.
- **Observability (m10):** p99 inference latency, throughput, GPU utilization, cache hit rate, error rate,
  prediction-distribution drift; alert on latency-budget breach + drift.

**Wrap-up:** *"A model service that **loads the model once + warms it** (kills cold-start), **fetches
features from the online store** (m23), runs inference (CPU/GPU), and optionally **caches predictions**.
Latency is met cheaply via **dynamic batching** (the GPU latency-vs-throughput trade), **model optimization**
(quantization/distillation/pruning), and **caching**. Models are tracked in a **registry** (version,
lineage, champion alias → instant rollback) and **never hot-swapped** — rolled out **shadow → canary → A/B**
gated on the **business metric**, because bad ML deploys fail **silently**. Scale = stateless horizontal
replicas + autoscaling + multi-model GPU sharing for cost. It's the m06 (cache) + m07 (LB/scaling) + m10
(safe deploy) toolkit specialized for inference — exactly my InsightDesk FastAPI serving, made
production-grade."*

---

## 8. From your systems 🏭 (you built this)
- **InsightDesk FastAPI serving = this module:** **lifespan loads models ONCE + warms the embedder** (kills
  cold-start, §6); **`/ask` prediction cache** (929× faster, $0 on a hit, §4 caching); **streaming**
  responses; readiness for "models loaded." You built a real model-serving service.
- **DCE pickle-model integration = model packaging + serving:** you **integrate the DS team's pickle model
  files** into the production pipeline (§3 packaging/loading) and serve them **batch-ish at 1M/day** (§2
  batch inference).
- **MLflow registry + champion alias = §5 registry:** versioning + a `champion` alias + retrain→eval-gate
  →**PROMOTE** is exactly the registry + safe-promotion flow.
- **int8 quantization** (§4 model optimization) — you've done it. **Shadow/canary/A-B** concepts from your
  MLOps work (§5).
- **Online feature lookup on the hot path:** DCE merges Redis reference data into each claim at scoring
  (§7 feature-lookup latency) — the m23 online store feeding the serving path.

---

## 9. Key concepts (interview-ready)
- **Serving spectrum:** **batch** (precompute+lookup, cheap/stale) → **online** (real-time/per-request) →
  **streaming** → **edge/on-device** (lowest latency/private, constrained). Choose by freshness/latency/
  cost; often **hybrid** (batch candidates + online re-rank).
- **Architecture:** model service **loads once + warms** (kills cold-start), **fetches features (online
  store, m23)**, runs inference, optionally **caches predictions** (m06). Package via pickle/ONNX/
  TorchScript (**ONNX** decouples train/serve framework).
- **Latency optimization:** **dynamic batching** (GPU efficiency — the **m02 latency-vs-throughput** trade;
  tune batch size + max wait to the SLO), **model optimization** (**quantization/distillation/pruning**),
  **prediction caching**, right hardware (GPU for big, CPU for tabular/GBDT).
- **Model registry** = source of truth for the live version (artifact/metrics/lineage/champion alias →
  **instant rollback**).
- **Safe rollout (NEVER hot-swap):** **shadow** (parallel, log, no impact) → **canary** (small %) → **A/B**
  (business metric) → full, with rollback. Bandits = adaptive. *More caution than a code deploy* (silent
  failure).
- **Scale:** stateless horizontal replicas + autoscale; **multi-model GPU sharing** for cost. **Build vs
  buy:** FastAPI (control) vs Triton/KServe/TorchServe/BentoML (batching/multi-model/versioning built-in).
- **Monitor** latency/throughput/GPU-util/drift (m28); serve with the **matching feature version** (no skew).

---

## 10. Go deeper (reading)
- **NVIDIA Triton / KServe / TorchServe / BentoML / Seldon docs** — production model servers (dynamic
  batching, multi-model, versioning).
- **Chip Huyen "Designing ML Systems" (deployment/inference chapters)** — batch vs online, model
  optimization, the serving spectrum.
- **MLflow Model Registry docs** — versioning + aliases (your champion-alias flow).
- **Quantization/distillation** primers (and your own AI/ML m10 int8 work).
- Revisit **m06** (caching), **m07** (LB/scaling/readiness), **m10** (canary/shadow/rollback), **m22**
  (online vs batch + offline-online eval), **m23** (feature lookup), **m28** (monitoring), **m29** (LLM
  serving — the extreme case).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m24_model_serving/`](../../solutions/m24_model_serving/README.md).

When you've done the exercises, say **"Module 25"** to design a *Recommendation System* — the candidate-
generation → ranking → re-ranking pipeline, two-tower retrieval, embeddings, and cold start (where serving,
features, and ranking all come together).
