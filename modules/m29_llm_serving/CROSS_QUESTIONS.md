# m29 — Cross-Questions ("if-and-buts") on LLM Serving & Inference

> Answer out loud in 2–3 sentences before reading the model answer. The memory-bound insight, the KV cache,
> continuous batching, and cost are where this is won.

---

### Q1. Why is serving an LLM different from serving a normal ML model (m24)?
**A.** Three reasons: it's **autoregressive** (generates **one token at a time, sequentially** — a
500-token reply is 500 forward passes, not one), it has **two phases** (parallel **prefill** of the prompt
then sequential **decode**), and it's **memory-bound, not compute-bound** — the bottleneck is **memory
bandwidth/capacity** (loading huge weights + the growing **KV cache**), not FLOPs. So the optimizations are
about **managing memory and amortizing it across requests (batching)**, not adding compute — a different
discipline from a single-forward-pass model.

---

### Q2. What does "LLM inference is memory-bound" mean, and why does it matter?
**A.** It means the GPU spends most of its time **moving data (weights + KV cache) to/from memory**, not
doing math — the FLOPs are cheap relative to the memory traffic. It matters because it **flips which
optimizations help**: shrinking memory movement (**quantization** to smaller weights, **batching** to
reuse loaded weights across requests, **KV-cache management**) speeds things up, while adding compute
doesn't. It's *the* mental model for LLM serving — and why a quantized model is faster (less memory to
move), not just smaller.

---

### Q3. What are TTFT and TPOT, and why two metrics?
**A.** **TTFT (Time To First Token)** ≈ **prefill latency** — how long until the first output token (the
*perceived* latency, especially with streaming). **TPOT/ITL (Time Per Output Token)** = the **decode
speed** — how fast subsequent tokens stream. Two metrics because the **two phases differ**: prefill is
parallel/compute-heavy (TTFT), decode is sequential/memory-bound (TPOT). **Total latency = TTFT + TPOT ×
output-tokens**, so long outputs are decode-dominated. (I measured TTFT 1.81s ≪ 2.77s total.)

---

### Q4. How does streaming change the latency story?
**A.** With **streaming (SSE, m03)** you display tokens **as they're generated** instead of waiting for the
full response — so the user sees output after **TTFT** (~1–2s) rather than after total latency, making it
**feel fast** even when total time is unchanged. It improves **perceived latency** dramatically (the
ChatGPT-typing effect). So you optimize **TTFT** hard (it's what users feel) and stream — exactly the
`/ask/stream` SSE pattern I built (TTFT 1.81s vs 2.77s total).

---

### Q5. What is the KV cache and why is it essential?
**A.** During decode, attention needs the **keys and values of all previous tokens**; recomputing them
every step would be **O(n²)**. The **KV cache** stores them, so each new token computes only its own K/V
and attends over the cached rest → **each decode step stays cheap**. It's essential because without it
generation would be quadratically slow. The catch: it **grows linearly with sequence length** and is large,
so it **dominates GPU memory** — which becomes the limiting resource (Q6).

---

### Q6. Why does the KV cache limit how many requests / how long a context you can serve?
**A.** Because **each concurrent request needs its own KV cache**, and each cache **grows with its sequence
length** (prompt + generated tokens) × layers × heads. So GPU memory = weights + Σ(per-request KV caches) →
the KV cache caps **how many requests fit at once** (concurrency) and **how long a context** you can handle
(**long context → KV-cache explosion**). It's the central memory-scaling wall in LLM serving, which is why
**PagedAttention** (paged, non-contiguous KV storage) exists — to pack more requests in.

---

### Q7. What is PagedAttention and what problem does it solve?
**A.** Naive KV-cache allocation reserves a **contiguous block sized to the max sequence length** per
request → huge **memory fragmentation/waste** (most requests don't use the max). **PagedAttention** (vLLM)
manages the KV cache like **OS virtual memory — in small non-contiguous "pages"** allocated on demand → far
less waste and **shared pages** (e.g. a common prompt prefix). The result: **many more concurrent requests
on the same GPU** (higher throughput, lower cost/token). It's the key vLLM innovation alongside continuous
batching.

---

### Q8. Why is static batching bad for LLMs, and what's the fix?
**A.** LLM requests have **variable output lengths**, so with **static batching** (form a batch, run to
completion) the requests that finish early **idle their GPU slots** waiting for the longest one → wasted
GPU. The fix is **continuous / in-flight batching**: manage the batch **at the token level** — when a
request finishes, **swap it out and swap a waiting one in** every step — keeping the GPU **fully utilized**.
This yields **10–20× throughput** and is the single biggest LLM-throughput technique (the vLLM advantage,
with PagedAttention).

---

### Q9. What is speculative decoding?
**A.** A technique to speed up the sequential **decode**: a **small, fast "draft" model proposes several
tokens ahead**, and the **big model verifies them all in one forward pass**. If the draft's tokens are
correct (often they are for easy continuations), you get **multiple tokens per big-model pass** instead of
one → faster generation. If wrong, you fall back. It works **around the autoregressive bottleneck** (you
can't parallelize one sequence, but you can *guess-and-verify* in parallel). Net: lower TPOT with no quality
loss (the big model still decides).

---

### Q10. How do you reduce LLM serving cost (the dominant constraint)?
**A.** **cost ≈ tokens (in+out) × price-per-token**, so: **quantization** (int8/int4 → fewer/cheaper GPUs,
and faster since memory-bound), **caching** (response cache for repeats; **prefix/prompt caching** reuses a
shared system-prompt's KV cache), **smaller models + routing** (use the smallest model that's good enough;
route easy queries to a cheap model, hard ones to an expensive one — m33), **continuous batching** (more
throughput/GPU), **minimizing output tokens** (concise prompts/outputs), and **self-hosting at high
volume**. At scale a tiny per-token saving is **enormous money** — which is why LLM systems obsess over
tokens (my $5-budget cost tracker instinct).

---

### Q11. What's prefix/prompt caching and when does it help?
**A.** Many requests share a **common prefix** — a long **system prompt**, a few-shot example block, a RAG
context. **Prefix caching** computes that prefix's **KV cache once and reuses it** across requests, so you
**skip re-prefilling the shared part** → lower TTFT + cost. It helps a lot when you have a big fixed prompt
prepended to every query (very common: long system instructions, shared context). It's distinct from a
**response cache** (which returns a stored answer for an identical request) — prefix caching reuses
*computation*, response caching reuses *results*.

---

### Q12. Quantization for LLMs — what and why?
**A.** **Quantization** runs the model weights (and sometimes activations) at **lower precision** —
**int8, int4, GPTQ, AWQ** — instead of fp16/fp32. Benefits: the model is **smaller** (fits on fewer/cheaper
GPUs or a laptop) **and faster** (less memory to move — and LLM inference is **memory-bound**, so shrinking
memory traffic directly speeds it up), with **modest quality loss** (int8 ≈ lossless; int4 a bit more). It's
why I could run **quantized Ollama models on an 8 GB no-GPU box** — quantization is *the* lever that makes
LLMs runnable on constrained hardware and cheaper at scale.

---

### Q13. When do you self-host an open model vs use an API?
**A.** **API (OpenAI/Anthropic)** for **speed-to-market, best quality, low/spiky volume, no infra** — but
**per-token cost**, **data leaves your boundary** (a problem for healthcare/PII), rate limits, and vendor
lock-in. **Self-host (Llama/Mistral on vLLM/TGI)** for **scale (cheaper at high volume), privacy/compliance,
control, and customization** — but you run GPUs + the serving stack and may trail closed models. The
decision is **volume × privacy × quality × effort**, and many do **hybrid** (sensitive/cheap → self-host,
hard → API; m33). I've done both: gpt-4o-mini (API) + Ollama (self-host).

---

### Q14. A request needs a 100K-token context. What breaks and what do you do?
**A.** The **KV cache explodes** — it grows with sequence length, so a 100K-token context needs a huge
per-request cache, **slashing concurrency** (fewer requests fit in GPU memory) and **raising TTFT** (prefill
over 100K tokens is heavy) and **cost** (more tokens). Mitigations: **quantize the KV cache**, use
**PagedAttention** + long-context-optimized attention (**FlashAttention**), **chunk/summarize** the context
or use **RAG** to retrieve only relevant pieces instead of stuffing everything (m30), and consider models
with efficient long-context attention. Long context is a real scaling wall, not free.

---

### Q15. How do you serve a model too big for one GPU?
**A.** **Model parallelism** — split the model across GPUs. **Tensor parallelism** splits the computation
**within each layer** across GPUs (they collaborate on each forward pass — needs fast interconnect like
NVLink). **Pipeline parallelism** splits **layers across GPUs** (each GPU holds some layers; activations
pass down the pipeline). You combine them (and data parallelism for replicas) for the biggest models. It
adds **inter-GPU communication overhead** and complexity, so you use it only when the model genuinely
doesn't fit — quantization first can sometimes avoid needing it.

---

### Q16. What's the latency-throughput trade-off in LLM serving?
**A.** **Larger batches** (more concurrent requests via continuous batching) → **higher throughput
(tokens/sec, lower cost/token)** but can **increase per-request latency** (more contention, the request
shares GPU time). It's the m02/m24 trade, sharpened because LLM decode is already slow. You **tune to the
SLO**: a latency-critical interactive chatbot caps batch size to protect TTFT/TPOT; a throughput-oriented
batch job (summarize 1M docs) maximizes batch size for cost-efficiency. Same prediction, different tuning
by use case.

---

### Q17. How does this connect to streaming and the chat UI (m14/m03)?
**A.** LLM serving is the **producer** of a token stream; the **transport** is **SSE** (m03 — one-way
server→client, perfect for streaming tokens) feeding the chat UI. The serving layer generates tokens
autoregressively and **streams each as it's produced**, so the client renders them live (low **TTFT** =
fast perceived start). It's a one-way stream (the prompt is a normal POST that *starts* it), so SSE — not
full WebSockets — is the right fit (the m03 lesson). This is exactly the `/ask/stream` endpoint I built.

---

### Q18. What do you monitor for an LLM serving system?
**A.** **Latency:** **TTFT, TPOT**, total, queue depth. **Throughput:** tokens/sec, requests/sec.
**Resource:** **GPU utilization, KV-cache utilization/memory pressure**, batch size. **Cost:** **cost per
request / per token** (the dominant concern — my per-call tracker). And **LLM-quality** signals
(hallucination/eval — m34) since a server can return fluent-but-wrong output (silent failure, the m22
theme). You alert on TTFT/TPOT breaches, KV-cache saturation (concurrency about to drop), and cost spikes.

---

### Q19. Build vs buy for the serving stack?
**A.** **Buy/use a framework** — **vLLM** (PagedAttention + continuous batching, the standard), **TGI**,
**TensorRT-LLM** (fastest on NVIDIA), or **Ollama/llama.cpp** (local/quantized) — almost always, because
rolling your own KV-cache paging + continuous batching is a massive, easy-to-get-wrong effort and these are
battle-tested. You'd hand-roll only for an unusual constraint. The real "build vs buy" is **self-host (run
vLLM on your GPUs) vs API (someone else runs it)** (Q13) — not "write your own inference engine." Use vLLM;
don't reinvent PagedAttention.

---

### Q20. Summarize why LLM serving is its own discipline.
**A.** Because it **inverts the assumptions of normal model serving**: it's **memory-bound** (not compute),
**autoregressive** (sequential token-by-token, not one forward pass), and **huge + expensive**. So the
playbook is LLM-specific — the **KV cache** (cheap decode but memory-dominating, paged via PagedAttention),
**continuous batching** (saturate the GPU despite variable output lengths), **quantization + speculative
decoding + parallelism** (fight memory + sequential latency), **streaming** (perceived latency via TTFT),
and an **obsessive cost focus** (tokens × price). Master m24's serving principles, then flip them for memory
and autoregression — that's LLM serving.
