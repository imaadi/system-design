# m24 — Solutions (Model Serving & Inference at Scale)

> Check the *reasoning*. Recurring lessons: choose serving mode by freshness/latency/cost; dynamic batching
> = the GPU latency-vs-throughput dial; never hot-swap (shadow→canary→A/B); registry = rollback.

---

## Exercise 1 — Pick the serving mode
1. **Batch** — precompute per-user recs nightly, serve by lookup (cheap; recs are stable enough).
2. **Online** — needs the live transaction + fresh context, <50 ms per request.
3. **Edge / on-device** — lowest latency + privacy (biometrics shouldn't leave the device); small model.
4. **Streaming** — score events as they flow through the stream (Kafka/Flink), near-real-time.
5. **Batch** — a daily per-customer score is stable; precompute on schedule, serve by lookup.

---

## Exercise 2 — Cold start
1. Loading a model (esp. onto GPU) is **slow** — doing it per request would blow the latency budget and
   waste work.
2. **Load once at startup** (model resident in memory/GPU for all requests) and **warm it** (a dummy
   inference) so the **first real request** isn't slow (frameworks lazily initialize on first call).
3. A **readiness probe** (m07) — the LB only routes traffic to a replica once its model is loaded + warm.

---

## Exercise 3 — Dynamic batching
1. Collect incoming requests for a few ms (or to a max batch size) and run them as **one** inference pass →
   higher **throughput**/GPU utilization at a small **latency** cost. It's the **m02 latency-vs-throughput
   trade**: bigger batch = more efficient but each request waits longer.
2. Set **max batch size + max wait** so the **worst-case added wait** (≈ max wait) plus inference time
   still fits 40 ms — e.g. "batch up to N requests or **~5–10 ms**, whichever first," leaving headroom for
   feature lookup + inference.
3. **GPUs** are massively parallel and far more efficient on a batch than on single items, so batching
   amortizes the per-pass overhead; **CPUs/GBDTs** are already efficient per-request and gain little, so
   batching matters less there.

---

## Exercise 4 — Make it cheaper/faster
- **Quantization** (fp32→int8): smaller/faster, minor accuracy loss.
- **Distillation**: train a small student to mimic the big teacher — much cheaper, near accuracy.
- **Dynamic batching**: better GPU utilization/throughput — costs per-request latency.
- **Caching** repeated predictions: skip inference entirely — only helps repeated inputs.
- (Also: **pruning**, better **hardware**, or move to **batch** inference if real-time isn't truly needed.)
  All are the accuracy-vs-latency-vs-cost trade.

---

## Exercise 5 — Caching predictions
1. **Helps** when the **same input recurs** (deterministic prediction → cache hit skips inference);
   **useless** for unique inputs (~0% hit, pure overhead) and **harmful** when the prediction must reflect
   fresh context the cache would stale out.
2. **InsightDesk `/ask` cache** — a repeated question hit the cache and was **929× faster and $0** (skipped
   the LLM call), a big win because questions repeated.

---

## Exercise 6 — The model registry
1. Stores each **model version's artifact, metrics, and lineage** + a **stage/alias** (champion vs
   challenger). Enables **versioning, reproducibility, safe promotion, and rollback** — the source of truth
   for which model is live.
2. **Instant rollback** = repoint the **alias** (e.g. `champion`) to the **previous version** — serving
   picks up the old model immediately, no rebuild/redeploy.
3. **MLflow registry + `champion` alias**: register a model version, alias it `champion`, and your
   retrain→eval-gate→**PROMOTE** flow flips the alias — exactly this.

---

## Exercise 7 — Safe rollout
1. Because a new model can be **silently worse** (better offline ≠ better online, m22) with **no error** —
   hot-swapping risks a quiet business-metric regression.
2. **Shadow** (parallel, log, no user impact — *does it run correctly/fast on real traffic?*) → **canary**
   (small % of real users — *is it safe on real users?*) → **A/B** (split + measure the business metric vs
   control — *is it actually better?*).
3. Because ML failures are **silent** (no exception, just worse predictions) — a code bug usually throws,
   an ML regression doesn't — so you need staged exposure + the business-metric gate to catch it.

---

## Exercise 8 — Hardware & cost
1. **LightGBM fraud model → CPU** (tabular GBDT is fast/cheap on CPU; a GPU would be wasted). **1B-param
   text model → GPU** (huge parallel matrix math; batching helps).
2. **Cost levers:** dynamic batching (max GPU utilization), **multi-model serving / GPU sharing**,
   **autoscaling** (scale-to-zero spiky workloads), plus shrinking the model (quant/distill) and CPU for
   models that don't need GPU.
3. **Multi-model serving** packs **many models onto shared GPU infra** (load on demand, evict cold) —
   amortizing expensive accelerators across models instead of dedicating a GPU per model (most of which
   would sit idle).

---

## Exercise 9 — Build vs buy + packaging
1. **FastAPI** when you have **one model / moderate scale** and want **control + simplicity** (InsightDesk).
   **A model server (Triton/KServe/TorchServe/BentoML)** when you need **dynamic batching, multi-model,
   versioning, GPU scheduling, metrics** out of the box at scale.
2. **ONNX** is **framework-agnostic** — it decouples the training framework from the serving runtime
   (train in PyTorch, serve in a fast optimized runtime), avoids lock-in, and enables hardware-optimized
   inference; a raw pickle ties you to Python + the training framework.
3. **Pickle can execute arbitrary code** on load (deserialization) — a security risk for untrusted/external
   models; ONNX/SavedModel are safer, data-only formats.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **InsightDesk FastAPI serving:** **lifespan loads models once + warms the embedder** = the **cold-start
   fix** (§6); **`/ask` prediction cache** (929× / $0) = **prediction caching** (§4); streaming + readiness
   = serving best-practices. You built a production model-serving service.
2. **DCE pickle-model integration = §3 packaging/loading** (serialize → load in the service → serve);
   **MLflow `champion` alias = §5 registry** (versioned, aliased, promotable/rollback-able). int8 quant =
   §4 model optimization.
3. **<50 ms fraud serving:** loaded+warmed **small/quantized model** on stateless replicas → on request,
   **one batched low-latency feature lookup** from the online store (m23, ~few ms) → inference (CPU or GPU
   with **dynamic batching** tuned to stay in budget) → return score; **no caching** (unique inputs);
   registry-pinned version with **instant rollback**; monitor feature-lookup latency (the usual
   bottleneck), model latency, and drift. Sub-50 ms = small model + fast feature fetch + minimal hops.
