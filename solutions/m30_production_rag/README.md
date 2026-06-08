# m30 — Solutions (Production RAG at Scale)

> Check the *reasoning*. Recurring lessons: RAG = knowledge (re-index, don't retrain); chunking is the #1
> lever; most hallucinations are retrieval misses; gate on faithfulness + context recall.

---

## Exercise 1 — RAG vs alternatives
1. **RAG** for fresh/private/large/citable **knowledge** (update by re-indexing); **fine-tuning** for
   **behavior/style/format**; **bigger prompt** when the relevant data is bounded + affordable (but costly
   + lost-in-the-middle).
2. **"RAG = knowledge, fine-tune = behavior."** Escalation: **prompt → RAG → fine-tune** (try the cheaper
   first; escalate when it plateaus).
3. **Fast-changing/private facts** (e.g. today's policy docs, internal KB) — must be RAG because
   fine-tuning bakes facts in expensively, can't cite, and can't be updated without retraining.

---

## Exercise 2 — The pipeline
1. **Indexing (offline):** chunk → embed → store in vector DB (+ metadata) [+ BM25 index]. **Query
   (online):** embed query → retrieve (hybrid) → rerank → build grounded prompt → generate.
2. The **m16/m25 retrieve-then-rerank funnel**: **hybrid retrieve (BM25 + vector, RRF) → cross-encoder
   rerank → top-k** (then generate instead of returning a list).
3. **Citations** come from tracking which retrieved chunk/source each claim used; they matter because they
   make the answer **verifiable** (and are required in regulated domains) and reinforce grounding/anti-
   hallucination.

---

## Exercise 3 — Chunking
1. Because **retrieval quality depends on chunk quality**, and retrieval determines whether the LLM even
   has the answer — bad chunks → retrieval misses → hallucinations.
2. **Boundary problem:** a fixed-60-char split sliced **"14 business days"** across chunks → the top chunk
   couldn't answer, sim **0.55**; **sentence-aware** chunking kept the fact whole → sim **0.79**.
3. **Overlap** chunks (a straddling fact appears whole in one) and **structure/sentence-aware splitting**
   (split on sentence/paragraph/section boundaries).

---

## Exercise 4 — Retrieval
1. **BM25 (keyword/lexical)** + **vector (semantic/ANN)**, **fused via RRF** — both because BM25 nails
   exact terms/rare keywords and vector nails meaning/paraphrase; together = precision + recall.
2. The **cross-encoder reranker** scores **query+chunk jointly** (more accurate than independent retrieval
   scores) to **precisely reorder the top candidates** → big precision lift on a small set.
3. Because **"most hallucinations are retrieval misses"** — if retrieval doesn't surface the answer, the
   LLM can't ground on it and makes something up; so retrieval recall directly bounds answer quality.

---

## Exercise 5 — Grounding & anti-hallucination
1. **Answer ONLY from the provided context**, **cite sources `[n]`**, and **say "I don't know" if the
   answer isn't in the context** (allow refusal).
2. The model hallucinates when it **lacks the right context** (a retrieval failure), so it fills the gap —
   therefore **debug retrieval first** (chunking → hybrid → rerank → measure context recall), generation
   second.
3. **Top-k tradeoff:** more chunks → higher recall but **more tokens (cost)** + **"lost in the middle"**
   (LLMs under-attend to mid-context, burying the relevant chunk) → **right-size top-k**, most-relevant
   first, don't dump context.

---

## Exercise 6 — Evaluation
1. **Faithfulness** (answer grounded in context — anti-hallucination), **answer relevancy** (addresses the
   question), **context precision** (retrieved chunks relevant), **context recall** (retrieval got the
   needed chunks).
2. **Context recall** — it's usually the bottleneck because retrieval is the hardest part (and "most
   hallucinations are retrieval misses"); if the right chunk isn't retrieved, nothing downstream can fix
   it.
3. **Generation problem** — high context recall means retrieval got the right chunks, but low faithfulness
   means the model **ignored them / hallucinated** → fix the grounding prompt / model, not retrieval.

---

## Exercise 7 — Production: freshness & multi-tenancy
1. **Re-embed + re-index just that document** (incremental indexing) → immediately reflected in retrieval.
   It's RAG's superpower because it's **no retraining** — knowledge updates are a cheap index op (vs
   fine-tuning, which would need a retrain).
2. **Metadata filtering** by `tenant_id`: tag every chunk with its tenant and **filter retrieval to the
   requesting tenant**, so a query only ever retrieves that tenant's docs (ideally separate indexes for
   sensitive data).
3. **At the retrieval layer (the filter)** — not just in the prompt — so tenant B's documents are never
   even fetched into tenant A's context (a hard security boundary).

---

## Exercise 8 — Cost & latency
1. **Generation is the cost** (tokens × price, m29); **retrieval (embeddings + ANN) is cheap/local.**
2. **Right-size top-k** (fewer context tokens), **cache** answers for repeats (my `/ask` 929×, $0),
   **prefix-cache** a shared system prompt, **smaller model / route** (m33). (Also: minimize output tokens.)
3. **Streaming** shows tokens as generated → user sees output after **TTFT** (improves **perceived
   latency**) even if total time is unchanged.

---

## Exercise 9 — Security & scale
1. **Prompt injection** = a malicious instruction hidden in a **retrieved document** ("ignore your
   instructions…") that hijacks the model; **your own indexed data is the attack vector** because RAG
   inserts retrieved content into the prompt — treat retrieved text as **untrusted data, not
   instructions** (m34).
2. **Shard + replicate the vector DB** (ANN/HNSW, sub-linear search, m18), run **distributed indexing**
   (m08), use **metadata filtering** to narrow search, and **cache** hot queries.
3. **Query rewriting/HyDE** (when recall is still the bottleneck after hybrid+rerank) and **agentic/
   GraphRAG** (for complex multi-hop questions) — added after the high-ROI hybrid+rerank.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **InsightDesk RAG → §2–§4:** chunking (**fixed-60 0.55 vs sentence-aware 0.79** = the boundary problem),
   **`RAGPipeline` (retrieve → rerank → generate)** = the funnel, **citations + grounding + refuse-when-
   unknown** = anti-hallucination, **top-k cost tradeoff** = the budget/lost-in-the-middle lever.
2. **Evaluation → §6:** **faithfulness ~0.96, answer_relevancy ~0.89, context_precision ~0.87, context_
   recall ~0.72**, RAGAS-from-scratch, LLM-as-judge (+ biases), and a **CI regression gate** (degraded
   config FAILED faithfulness 0.83<0.85 → exit 1).
3. **Healthcare claim/policy Q&A:** **chunk** policies/claims **structure-aware** (by section/clause) with
   overlap; index in a **vector DB + BM25**, tag with **`tenant_id`/plan/effective-date metadata**; on a
   query, **filter by tenant + date**, **hybrid retrieve → rerank**, build a **grounded prompt** (answer
   only from policy text, **cite the clause**, **say "not covered/unknown"** rather than guess);
   **guardrails** for PHI redaction + prompt injection; **re-index** on policy updates for freshness; and
   **gate on faithfulness + context recall in CI**. Citations + refusal are essential in a regulated
   domain — exactly the grounding discipline I built, applied to my own field.
