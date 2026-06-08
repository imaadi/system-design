# m29 — Solutions (LLM Serving & Inference)

> Check the *reasoning*. Recurring lessons: LLM serving is memory-bound + autoregressive; KV cache
> dominates memory (PagedAttention); continuous batching = throughput; cost (tokens × price) dominates.

---

## Exercise 1 — Why LLMs are different
1. **Autoregressive** (one token at a time, sequential — N tokens = N forward passes), **two phases**
   (parallel prefill + sequential decode), and **memory-bound, not compute-bound**.
2. The GPU spends most time **moving weights + KV cache to/from memory**, not on FLOPs — so optimizations
   that **shrink memory traffic** (quantization, batching to reuse loaded weights, KV-cache management)
   help, while adding compute doesn't.
3. Each token **depends on all previous tokens**, so you must generate them **in sequence** — you can't
   compute token 5 before token 4 exists within the same generation.

---

## Exercise 2 — Latency metrics
1. **TTFT (Time To First Token)** ≈ **prefill** latency; **TPOT/ITL (Time Per Output Token)** = **decode**
   speed.
2. **Total latency = TTFT + TPOT × (number of output tokens).**
3. **Streaming** shows tokens as generated, so users feel **TTFT** (~1–2s) instead of total — making it
   *feel* fast. I measured **TTFT 1.81s ≪ 2.77s total** with SSE, so the user saw output at 1.81s.

---

## Exercise 3 — The KV cache
1. It stores the **keys and values of all previous tokens**, so each new token computes only its own K/V
   and **attends over the cached rest** — keeping each decode step cheap (avoids O(n²) recompute).
2. It **grows linearly with sequence length** (× layers × heads) per request, so it's large and consumes
   GPU memory alongside the weights → it limits **concurrency** (each request needs its own cache) and
   **context length** (long context → cache explosion).
3. **PagedAttention** stores the KV cache in **non-contiguous pages (like OS virtual memory)** instead of a
   max-length contiguous block — fixing **fragmentation/over-reservation waste** so far more requests fit.

---

## Exercise 4 — Batching
1. LLM requests have **variable output lengths**, so static batching makes **early-finishing slots idle**
   while waiting for the longest request → wasted GPU.
2. **Continuous/in-flight batching** manages the batch **at the token level** — swap finished requests out
   and waiting ones in each step — keeping the GPU saturated; **~10–20× throughput**.
3. **PagedAttention + continuous batching** — together they're the **vLLM** advantage.

---

## Exercise 5 — Cost
1. **cost ≈ tokens (in + out) × price-per-token.** Cost dominates because LLM inference is **expensive**
   (GPU-hours / per-token pricing), so at scale every token is money — the design's main constraint.
2. **Quantization, caching (+ prefix caching), smaller models + routing (m33), continuous batching,
   minimizing output tokens** (and self-hosting at high volume).
3. **Response caching** returns a **stored answer** for an identical request (reuses *results*); **prefix/
   prompt caching** reuses the **KV cache of a shared prompt prefix** (e.g. a long system prompt) across
   requests (reuses *computation* — saves re-prefill).

---

## Exercise 6 — Optimizations
1. **Quantization** = lower-precision weights (int8/int4/GPTQ/AWQ) → **smaller + faster**; it helps a
   **memory-bound** workload because there's **less memory to move** (the actual bottleneck), not just less
   storage.
2. A **small draft model proposes several tokens**, the **big model verifies them in one pass** — if
   correct you get **multiple tokens per big-model pass**, speeding decode without quality loss (the big
   model still decides).
3. **Tensor parallelism** (split computation *within* a layer across GPUs) and **pipeline parallelism**
   (split *layers* across GPUs) — needed when the **model is too big to fit on one GPU** (try quantization
   first to avoid it).

---

## Exercise 7 — Self-host vs API
1. **API:** no infra + best-quality models + fast to start (and spiky/low volume). **Self-host:** cheaper
   at high volume + privacy/control/customization.
2. **Lean self-host** for PHI — **data shouldn't leave your boundary** (HIPAA/compliance), which an
   external API risks; self-hosting keeps sensitive data in-house.
3. **Hybrid:** **route** sensitive/cheap/easy queries to a **self-hosted** model and hard ones to an
   **API** (m33) — best cost/quality/privacy balance.

---

## Exercise 8 — Long context
1. The **KV cache explodes** (grows with length) → **concurrency drops** (fewer requests fit in memory),
   **TTFT rises** (prefill over 100K tokens is heavy), and **cost rises** (more tokens).
2. **Quantize the KV cache**, use **PagedAttention + FlashAttention** (memory-efficient attention), and
   **retrieve only relevant pieces via RAG** (m30) instead of stuffing all 100K tokens.
3. **RAG sidesteps long context** — instead of putting everything in the prompt, you **retrieve the few
   relevant chunks** and pass only those, keeping the context (and KV cache) small (m30).

---

## Exercise 9 — Tuning to the use case
1. **Interactive chatbot:** **cap batch size** to protect **TTFT/TPOT** (latency-critical — users wait on
   each response); favor low latency over max throughput.
2. **Nightly batch summarization:** **maximize batch size** for **throughput / lowest cost-per-token** —
   no user waiting, so trade latency for GPU efficiency.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **TTFT 1.81s vs 2.77s total + SSE** = §2 metrics + perceived latency — streaming made the response *feel*
   fast (user saw tokens at 1.81s), exactly TTFT-driven perceived latency.
2. **$5 budget + per-call cost tracker + "embeddings local/free, only generation calls the API"** = §5 cost
   levers (minimize paid tokens, cache) + §6 **hybrid routing** (cheap work local, expensive work to the
   API) — the cost-obsession LLM serving demands, lived.
3. **8 GB no-GPU + quantized Ollama (deepseek-r1:1.5b)** = the **memory-bound** reality (§1) + **quantization
   as the fix** (§5) — you *felt* that LLM serving is memory-constrained and that quantization makes it
   runnable. **API vs self-host:** **gpt-4o-mini (API)** for quality generation, **Ollama (self-host/local)**
   for free/private demos — you made the §6 decision both ways, by cost and quality.
