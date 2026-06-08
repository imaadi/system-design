# m29 — Exercises (LLM Serving & Inference)

> Do these on paper / out loud before checking `solutions/m29_llm_serving/`. The memory-bound insight, the
> KV cache, continuous batching, and cost are the core.

---

## Exercise 1 — Why LLMs are different
1. Name the three ways LLM serving differs from normal model serving (m24).
2. What does "memory-bound, not compute-bound" mean, and why does it flip the optimizations?
3. Why can't you parallelize generation within one request?

---

## Exercise 2 — Latency metrics
1. Define TTFT and TPOT and which phase each maps to.
2. Write the total-latency formula.
3. How does streaming change which metric users feel? (Cite your own numbers.)

---

## Exercise 3 — The KV cache
1. What does the KV cache store and why does it make decode cheap?
2. Why does it dominate GPU memory, and what two things does it limit?
3. What is PagedAttention and what waste does it fix?

---

## Exercise 4 — Batching
1. Why is static batching wasteful for LLMs specifically?
2. What does continuous/in-flight batching do, and roughly what throughput gain?
3. Which two techniques together give vLLM its advantage?

---

## Exercise 5 — Cost
1. Write the cost back-of-envelope. Why is cost the dominant constraint?
2. List five levers to reduce cost.
3. What's the difference between response caching and prefix/prompt caching?

---

## Exercise 6 — Optimizations
1. What is quantization and why does it help a *memory-bound* workload?
2. Explain speculative decoding in two sentences.
3. Tensor vs pipeline parallelism — when do you need either?

---

## Exercise 7 — Self-host vs API
1. Two reasons to use an API; two reasons to self-host.
2. For a healthcare app handling PHI, which way do you lean and why?
3. What does a hybrid approach look like?

---

## Exercise 8 — Long context
1. A request needs 100K tokens of context. What breaks?
2. List three mitigations.
3. How does RAG (m30) relate to this?

---

## Exercise 9 — Tuning to the use case
For each, how do you tune batch size (latency vs throughput)?
1. An interactive chatbot.
2. A nightly batch job summarizing 1M documents.

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map your **TTFT 1.81s vs 2.77s total + SSE streaming** to this module's metrics + perceived latency.
2. Map your **$5 budget + per-call cost tracker + "embeddings local, generation API"** to the cost levers
   + hybrid routing.
3. Map your **8 GB no-GPU + quantized Ollama** to the memory-bound insight + quantization. Then: API vs
   self-host — which did you use for what, and why?

---

When done, check `solutions/m29_llm_serving/README.md`, then say **"Module 30"**.
