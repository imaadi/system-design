# m30 — Exercises (Production RAG at Scale)

> Do these on paper / out loud before checking `solutions/m30_production_rag/`. Chunking, the retrieval
> funnel, grounding, and evaluation are the core — and you built all of it in InsightDesk.

---

## Exercise 1 — RAG vs alternatives
1. When do you use RAG vs fine-tuning vs a bigger prompt?
2. State the "RAG = ___, fine-tune = ___" rule and the escalation order.
3. Give one knowledge type that *must* be RAG (not fine-tune) and why.

---

## Exercise 2 — The pipeline
1. List the indexing (offline) steps and the query (online) steps.
2. Which earlier modules' funnel is the query path, and what are its stages?
3. Where do citations come from and why do they matter?

---

## Exercise 3 — Chunking
1. Why is chunking the #1 lever?
2. Describe the boundary problem with a concrete example (cite your own numbers).
3. Name two fixes.

---

## Exercise 4 — Retrieval
1. Hybrid retrieval: what two methods, fused how, and why both?
2. What does the cross-encoder reranker add?
3. Why is retrieval recall "the #1 quality lever"?

---

## Exercise 5 — Grounding & anti-hallucination
1. What three instructions go in a grounded prompt?
2. "Most hallucinations are retrieval misses" — explain and what to debug first.
3. What's the top-k tradeoff and "lost in the middle"?

---

## Exercise 6 — Evaluation
1. Name the four RAG metrics and what each measures.
2. Which is usually the bottleneck, and why?
3. Answers are wrong: faithfulness high but context recall low — retrieval or generation problem?

---

## Exercise 7 — Production: freshness & multi-tenancy
1. A document changes. How do you update RAG's knowledge, and why is this RAG's superpower?
2. How do you ensure tenant A never sees tenant B's documents?
3. Where is that isolation enforced?

---

## Exercise 8 — Cost & latency
1. What part of RAG is the cost, and what part is cheap?
2. List four cost levers.
3. How does streaming help, and which metric does it improve?

---

## Exercise 9 — Security & scale
1. What is prompt injection in RAG, and why is your own indexed data the attack vector?
2. How do you scale retrieval to tens of millions of docs?
3. Name two advanced retrieval techniques and when you'd add them.

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map your **InsightDesk RAG** (chunking 0.55-vs-0.79, RAGPipeline, citations/refusal, top-k) to §2–§4.
2. Map your **evaluation work** (faithfulness ~0.96, context recall ~0.72, RAGAS-from-scratch, CI gate) to
   §6.
3. Design a production RAG for **healthcare claim/policy Q&A** end-to-end (chunking, retrieval, grounding,
   multi-tenancy, guardrails, eval) — leaning on what you built.

---

When done, check `solutions/m30_production_rag/README.md`, then say **"Module 31"**.
