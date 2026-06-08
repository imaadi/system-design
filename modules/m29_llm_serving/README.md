# Module 29 — LLM Serving & Inference ⭐⭐ 🏭 — *Part 4 begins*

> Serving LLMs is **fundamentally different from m24 model serving** — and it's the 2026 differentiator.
> LLMs are **autoregressive** (generate token-by-token, sequentially), **huge** (billions of params), and
> — the key surprise — **memory-bound, not compute-bound.** So the optimizations are LLM-specific: the
> **KV cache** (the central data structure), **continuous batching** (the throughput unlock), quantization
> + speculative decoding, and an obsessive focus on **cost** (LLM inference is expensive). You **feel**
> this already — you measured **TTFT vs total latency**, tracked **per-call cost** on a $5 budget, and ran
> **quantized local models** on an 8 GB no-GPU box.

> **Format (Part 4 opener):** worked design; the **memory-bound insight**, the **KV cache**, **continuous
> batching**, and **cost** are the deep-dives.

---

## 1. Why LLM serving is different from normal model serving (m24) ⭐⭐
- **Autoregressive generation:** an LLM generates **one token at a time**, each depending on **all
  previous tokens** → a 500-token response is **500 sequential forward passes**, not one. You **can't
  parallelize within a single generation.** This is the latency killer.
- **Two phases:** **prefill** (process the whole prompt at once — parallel, compute-heavy) then **decode**
  (generate output tokens one-by-one — sequential, memory-bound). They have different performance
  profiles (→ two latency metrics, §2).
- **Memory-bound, not compute-bound (the surprise):** the bottleneck is **memory bandwidth + capacity**
  (loading the huge weights + the growing **KV cache** for each token), **not FLOPs.** So optimizations
  target **memory** (KV-cache management, quantization, batching to amortize weight-loading), not raw
  compute. *This inversion is the core insight that makes LLM serving its own discipline.*
- **Huge models:** billions–trillions of params → often **multiple GPUs** (tensor/pipeline parallelism)
  and serious memory pressure.

> **Say this:** *"LLM serving is **memory-bound and autoregressive**, unlike a normal model's single
> forward pass — so the game is **managing memory (the KV cache + weights) and amortizing it across
> requests (batching)**, plus shrinking the model (quantization), not adding FLOPs. The bottleneck is
> memory bandwidth, which is why the optimizations look so different from m24."*

---

## 2. LLM latency & throughput metrics ⭐ (know these)
- **TTFT (Time To First Token):** how long until the **first** output token appears ≈ **prefill latency**
  (+ queue). Drives **perceived** latency — with **streaming** (SSE, m03), you show tokens as they
  generate, so a low TTFT *feels* fast even if total is slow. *(You measured TTFT 1.81s ≪ 2.77s total.)*
- **TPOT / ITL (Time Per Output Token / Inter-Token Latency):** the **decode** speed — time to generate
  each subsequent token. Determines how fast text streams.
- **Total latency = TTFT + TPOT × (number of output tokens).** Long outputs are dominated by decode.
- **Throughput:** **tokens/sec** (across all requests) and **requests/sec** — the cost-efficiency metric
  (more throughput per GPU = lower cost/token). **There's a latency↔throughput trade** (batching raises
  throughput, can raise per-request latency — m02/m24).

---

## 3. Deep Dive: the KV cache ⭐⭐ (the central LLM-serving data structure)
At each decode step, attention needs the **keys and values of all previous tokens.** Recomputing them
every step would be **O(n²)** wasteful. The **KV cache** stores them so each new token only computes its
*own* K/V and attends over the cached rest → each step is cheap. **But:**
- The KV cache **grows linearly with sequence length** (prompt + generated tokens) **× layers × heads** →
  it's **large** and **dominates GPU memory** alongside the weights.
- So the KV cache **limits two things:** how **many concurrent requests** you can serve (each needs its own
  cache) and how **long a context** you can handle (**long context → KV-cache explosion**).
- **Fragmentation problem:** naive contiguous KV-cache allocation wastes memory (you reserve max length per
  request). **PagedAttention (vLLM)** manages the KV cache like **OS virtual memory — in non-contiguous
  "pages"** → far less waste → **many more concurrent requests** on the same GPU. *This is the key vLLM
  innovation.*

```
  decode step t: compute K/V for token t only; ATTEND over cached K/V of tokens 0..t-1 (the KV cache)
  KV cache size ∝ (prompt + generated) tokens × layers × heads  → grows every token, dominates memory
  PagedAttention: store the KV cache in pages (like virtual memory) → no fragmentation → more concurrency
```

---

## 4. Deep Dive: batching ⭐⭐ (the throughput unlock)
GPUs are efficient on batches, but LLM requests **finish at different times** (different output lengths):
- **Static batching:** wait, form a batch, run it to completion. **Bad for LLMs** — when one request needs
  500 tokens and another needs 10, the finished slots **idle** waiting for the longest → wasted GPU.
- **Continuous / in-flight batching ⭐ (vLLM):** manage the batch **at the token level** — as a request
  **finishes, swap it out and swap a waiting request in**, every step. The GPU stays **fully utilized** →
  **massive throughput gains (often 10–20×)**. *This is the single biggest LLM-throughput technique.*
- Combined with **PagedAttention** (§3), continuous batching is why **vLLM** dramatically out-serves naive
  approaches.

> **The senior line:** *"LLM requests have **variable output lengths**, so **static batching wastes the
> GPU** (finished slots idle). **Continuous (in-flight) batching** adds/removes requests from the batch at
> the token level to keep the GPU saturated — the key throughput unlock — and **PagedAttention** manages
> the KV cache in pages so more requests fit. Together that's the vLLM advantage."*

---

## 5. Deep Dive: model optimization & cost ⭐⭐ (cost is the dominant concern)
LLM inference is **expensive** (GPU-hours / per-token pricing), so **cost is THE design constraint.** The
levers:
- **Quantization:** run weights at lower precision (**int8 / int4 / GPTQ / AWQ**) → smaller (fits on fewer/
  cheaper GPUs) + faster (less memory to move — and it's memory-bound!) with modest quality loss. *(Your
  int8 work; running quantized Ollama models on 8 GB.)*
- **Speculative decoding:** a **small fast "draft" model proposes several tokens**, the **big model
  verifies them in one pass** — if the draft is right, you get **multiple tokens per big-model pass** →
  faster decode (works *around* the sequential bottleneck).
- **Tensor / pipeline parallelism:** **split a huge model across GPUs** (tensor = within a layer; pipeline =
  across layers) when it doesn't fit on one.
- **FlashAttention:** a memory-efficient attention kernel (less memory movement → faster, longer context).
- **Smaller / distilled models + routing:** use the **smallest model that's good enough**, and **route**
  easy queries to a cheap model, hard ones to an expensive one (m33).
- **Caching:** cache responses for repeated prompts (your `/ask` 929×, $0); **prompt/prefix caching**
  reuses the KV cache of a shared prompt prefix (e.g. a long system prompt) across requests.

**Cost math (back-of-envelope):** **cost ≈ tokens (in + out) × price-per-token.** At scale a small per-token
saving (quantization, a smaller model, caching, fewer output tokens) is **enormous money** — which is why
LLM systems obsess over tokens. *(You lived this: "embeddings local/free, only generation calls the API,"
per-call cost tracking — the cost-routing instinct.)*

---

## 6. Self-host vs API + serving frameworks
- **API (OpenAI/Anthropic/etc.):** no infra, instant best models, pay-per-token; but **ongoing per-token
  cost**, **data leaves your boundary** (privacy/compliance — relevant to your healthcare world), rate
  limits, and vendor dependence. *(Your gpt-4o-mini usage.)*
- **Self-host (open models — Llama/Mistral/etc. on vLLM/TGI):** **control, privacy, fixed infra cost**
  (cheaper at high volume), customizable; but you run GPUs + the serving stack and may trail the best
  closed models. *(Your Ollama local serving.)*
- **The decision:** API for speed-to-market/low-volume/best-quality; self-host for **scale (cost), privacy/
  compliance, or customization.** Many do **hybrid** (cheap/sensitive → self-host, hard → API; m33).
- **Frameworks:** **vLLM** (PagedAttention + continuous batching — the standard), **TGI** (HuggingFace),
  **TensorRT-LLM** (NVIDIA, fastest on their GPUs), **Ollama/llama.cpp** (local/quantized — your stack).

---

## 7. Bottlenecks, scale & edge cases
- **Memory (the constraint):** weights + **KV cache** → quantization, PagedAttention, KV-cache limits on
  concurrency/context. **Long context** blows up the KV cache (it grows with length) — a real scaling wall.
- **Latency (autoregressive decode):** continuous batching + speculative decoding + streaming (TTFT) +
  smaller models.
- **Cost (dominant):** quantize, cache, batch, route to smaller models (m33), minimize output tokens,
  self-host at scale.
- **Throughput vs latency trade:** bigger batches = more throughput (cheaper/token) but higher per-request
  latency — tune to the SLO (m02/m24).
- **GPU scarcity/cost:** GPUs are expensive + supply-constrained → maximize utilization (continuous
  batching), autoscale carefully, multi-tenant.
- **Observability (m10/m28):** TTFT, TPOT, throughput (tokens/sec), GPU utilization, KV-cache utilization,
  cost/token, queue depth; plus LLM-quality monitoring (m34).

**Wrap-up:** *"LLM serving is **memory-bound + autoregressive**, so it's its own discipline. The **KV
cache** (storing previous tokens' keys/values) is the central data structure — it makes decode cheap but
**grows with sequence length and dominates memory**, limiting concurrency + context (**PagedAttention**
manages it in pages). **Continuous (in-flight) batching** keeps the GPU saturated despite variable output
lengths (the throughput unlock). I'd measure **TTFT + TPOT** (streaming for perceived latency), cut **cost**
(the dominant constraint) via **quantization, caching/prefix-caching, smaller models + routing, and
minimizing tokens**, use **speculative decoding** for faster decode and **tensor/pipeline parallelism** for
huge models, and choose **API vs self-host** by scale/privacy/cost — serving with **vLLM**. It's m24's
serving principles, inverted for memory and autoregression."*

---

## State-of-the-art & real-world notes 📚
- **vLLM (PagedAttention + continuous batching)** — the standard high-throughput LLM server (UC Berkeley).
  **TGI** (HuggingFace), **TensorRT-LLM** (NVIDIA), **Ollama/llama.cpp** (local/quantized).
- **FlashAttention** (Dao et al.) — memory-efficient attention. **Speculative decoding** (Leviathan et
  al.). **Quantization:** GPTQ, AWQ, int4/int8. **Continuous batching** (Orca/vLLM).
- **The memory-bound reality** is well-documented (e.g. "LLM inference is memory-bandwidth-bound") — it's
  *the* mental model for LLM-serving design.

---

## From your systems 🏭 (you feel these constraints)
- **You measured the right metrics:** **TTFT 1.81s ≪ 2.77s total** with **streaming (SSE)** — exactly §2's
  TTFT-vs-total + perceived-latency lesson. You know streaming makes it *feel* fast.
- **Cost obsession (the dominant constraint):** a **$5 budget**, a **per-call cost tracker**, and the
  design rule **"embeddings local/free, only generation calls the API"** = §5's cost levers + §6's hybrid
  routing instinct, lived.
- **Memory-bound, viscerally:** **8 GB RAM, no GPU** forced you to run **quantized local models (Ollama
  deepseek-r1:1.5b)** — you *feel* that LLM serving is memory-constrained (§1) and that **quantization** is
  the fix (§5).
- **API vs self-host:** you used **gpt-4o-mini (API)** for generation and **Ollama (self-host/local)** for
  free demos — you've made the §6 decision both ways.
- **Caching:** your **`/ask` response cache** (929×, $0) is §5's response caching.

---

## Key concepts (interview-ready)
- **LLM serving is memory-bound + autoregressive** (not compute-bound, not one forward pass) — the core
  inversion vs m24. **Prefill** (parallel, prompt) + **decode** (sequential, token-by-token).
- **Metrics:** **TTFT** (≈ prefill, perceived latency — stream it), **TPOT/ITL** (decode speed), **total =
  TTFT + TPOT×tokens**, **throughput (tokens/sec** = cost-efficiency). Latency↔throughput trade.
- **KV cache (the central data structure):** caches previous tokens' K/V → cheap decode, but **grows with
  sequence length, dominates memory**, limits concurrency + context. **PagedAttention** = manage it in
  pages (less fragmentation → more concurrency).
- **Continuous (in-flight) batching** = the throughput unlock — token-level add/remove keeps the GPU
  saturated despite variable output lengths (10–20×). (vLLM = PagedAttention + continuous batching.)
- **Cost is the dominant constraint:** **cost ≈ tokens × price**; levers = **quantization (int8/int4),
  caching + prefix caching, speculative decoding, smaller models + routing (m33), fewer output tokens**,
  self-host at scale. **Tensor/pipeline parallelism** for huge models; **FlashAttention** for memory.
- **API vs self-host:** API = speed/quality/no-infra but per-token cost + data leaves; self-host = control/
  privacy/cheaper-at-scale but you run GPUs. Often **hybrid**. Framework: **vLLM**.

---

## Go deeper (reading)
- **vLLM / PagedAttention paper ("Efficient Memory Management for LLM Serving")** — KV-cache paging +
  continuous batching (the must-read).
- **"How continuous batching enables 23x throughput" (Anyscale)** and **Orca (continuous batching)** —
  the batching unlock.
- **FlashAttention (Dao et al.)**, **Speculative Decoding (Leviathan et al.)**, **GPTQ/AWQ quantization** —
  the optimization techniques.
- **vLLM / TGI / TensorRT-LLM docs**; your own **AI/ML notes (m07 RAG cost, m10 quantization, m11 streaming/
  cost)** — the cost + streaming + quantization you already implemented.
- Revisit **m24** (serving fundamentals — this inverts them), **m02** (latency/throughput), **m06**
  (caching), **m33** (the LLM gateway: routing/caching/cost across this).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m29_llm_serving/`](../../solutions/m29_llm_serving/README.md).

When you've done the exercises, say **"Module 30"** to design *Production RAG at Scale* — ingestion →
chunking → embeddings → vector DB → hybrid retrieval → rerank → generate → eval/guardrails — the
production version of the RAG you already built (and the retrieval funnel from m16/m25).
