# Module 30 — Production RAG at Scale ⭐⭐ 🏭 (your centerpiece)

> Answer questions **grounded in your data** (private/fresh knowledge), **with citations**, using an LLM —
> **without hallucinating.** RAG is the most in-demand LLM-engineering skill, and **you built it end-to-
> end** in InsightDesk: you *measured* the chunking boundary problem, built **hybrid retrieval + rerank**,
> enforced **grounding/citations/refusal**, and **evaluated faithfulness/recall with a CI gate.** This
> module scales that to production. The big lessons (mostly yours): **"most hallucinations are retrieval
> misses — fix retrieval first"**, **"RAG = knowledge, fine-tune = behavior"**, **chunking is the #1
> lever**, and **"any production RAG has a recall metric."**

> **Format (Part 4 deep-dive):** worked design; **chunking**, the **retrieval funnel**, **grounding/anti-
> hallucination**, and **evaluation** are the deep-dives.

---

## 1. Why RAG (vs fine-tuning / just a bigger context) ⭐
- **RAG injects KNOWLEDGE:** fresh/private data the model wasn't trained on, **with citations**, and it's
  **cheap to update** (re-index the changed doc — **no retraining**). Perfect for fast-changing or
  proprietary knowledge.
- **Fine-tuning changes BEHAVIOR:** style, format, tone, a specialized skill — not for injecting facts (it
  bakes them in expensively and can't cite).
- **Bigger context** (stuff everything in the prompt): simple but **expensive** (tokens × price, m29),
  hits **context limits**, and suffers **"lost in the middle"** (§4).
- **The decision (your lesson): prompt → RAG → fine-tune.** Try prompting first; add **RAG** for knowledge;
  **fine-tune** last for behavior or a cheaper specialized model. **"RAG = knowledge, fine-tune =
  behavior."**

---

## 2. The RAG pipeline ⭐⭐ (two halves)

```
  INDEXING (offline):
   documents → CHUNK → EMBED (embedding model) → store in VECTOR DB (+ metadata)   [+ keyword/BM25 index]

  QUERY (online):
   query → EMBED → RETRIEVE top-N (HYBRID: vector + BM25, fused) → RERANK (cross-encoder) → top-k
        → BUILD PROMPT (context + grounding instructions + "cite sources / say I don't know")
        → LLM GENERATE → grounded, cited answer
```
- **Indexing (offline):** split docs into **chunks**, **embed** each (your MiniLM), store vectors in a
  **vector DB** with **metadata**; also build a **keyword/BM25 index** for hybrid.
- **Query (online):** embed the query → **retrieve** candidates (hybrid) → **rerank** → assemble a
  **grounded prompt** → **generate**. **This is the m16/m25 retrieve-then-rerank funnel feeding an LLM** —
  you've now built it three times (search, recsys, RAG).

---

## 3. Deep Dive: chunking ⭐⭐ (the #1, most-underrated lever)
How you **split documents into chunks** determines retrieval quality more than almost anything:
- **Size:** too big → diluted embeddings + wasted context tokens; too small → fragments lose meaning.
- **Overlap:** overlap chunks so a fact spanning a boundary isn't lost (insurance, not a guarantee).
- **The boundary problem ⭐:** a fixed-size split can **cut a fact in half** → retrieval can't find it.
  *(Your measured lesson: a fixed-60-char split sliced "14 business **days**" → top chunk couldn't answer,
  sim **0.55**; **sentence-aware** chunking kept the fact whole → sim **0.79**.)*
- **Structure/semantic-aware chunking:** split on **sentences/paragraphs/sections/markdown** (respect
  document structure), or **semantic** chunking — far better than blind fixed-size.

> **Say this:** *"Chunking is the highest-leverage RAG lever — I split **structure/sentence-aware** with
> overlap so facts aren't cut at boundaries (I measured a fixed split scoring 0.55 vs sentence-aware 0.79
> on a fact that straddled the boundary). Bad chunking causes retrieval misses, which cause
> hallucinations."*

---

## 4. Deep Dive: retrieval, generation & grounding ⭐⭐

### Retrieval (the quality bottleneck) — the m16/m25 funnel
- **Embed + vector search (ANN/HNSW):** semantic similarity — finds meaning/paraphrase (your m05–m06).
- **Hybrid (BM25 + vector, fused via RRF):** keyword precision + semantic recall — better than either
  alone (your m06 hybrid).
- **Rerank (cross-encoder):** precisely reorder the top candidates (your m06 rerank).
- **Metadata filtering:** filter by tenant/date/source (multi-tenancy + freshness — §5).
- **"Most hallucinations are retrieval misses"** → **retrieval recall is the #1 quality lever.** Fix
  retrieval before touching the prompt.

### Generation & grounding (anti-hallucination) ⭐
- **Build a grounded prompt:** put the retrieved context in, and **instruct the model to answer ONLY from
  the context, cite sources `[n]`, and say "I don't know" if the answer isn't there** (your refusal lesson
  — anti-hallucination).
- **Top-k / context budget tradeoff:** more chunks = more recall but **more tokens (cost, m29)** + **"lost
  in the middle"** (LLMs attend less to the middle of a long context) → **right-size top-k**, put the most
  relevant first, don't dump context. *(Your measured top-k plateau.)*
- **Citations** make the answer **verifiable** (and required in many domains — your healthcare world).

---

## 5. Deep Dive: production concerns ⭐ (what makes it "at scale")
- **Scale:** millions of docs → the **vector DB is sharded + replicated** (m18-style ANN at scale); the
  embedding + indexing pipeline runs as a **batch/stream job** (m08).
- **Freshness (RAG's superpower):** on a doc change, **re-embed + re-index just that doc** → immediately
  reflected (no retraining — unlike fine-tuning). Incremental indexing + deletion handling.
- **Multi-tenancy / data isolation:** **metadata filtering** by `tenant_id` so each customer retrieves
  **only their own docs** (a hard security requirement — never leak tenant A's docs to tenant B).
- **Guardrails (m34):** strip/redact **PII/PHI**, defend **prompt injection** (a malicious doc telling the
  model to ignore instructions), moderate output, enforce refusal.
- **Cost + latency (m29):** **retrieval is cheap/local**; **generation is the cost** (tokens × price) →
  **cache** answers for repeated queries (your `/ask` 929×, $0), right-size top-k + the model, stream
  (TTFT).
- **Evaluation (§6) gated in CI** — the thing that makes it production-grade.

---

## 6. Deep Dive: evaluation ⭐⭐ ("any production RAG has a recall metric")
You can't ship RAG you can't measure. The metrics (your m09 / RAGAS-from-scratch work):
- **Faithfulness:** is the answer **grounded in the retrieved context** (not made up)? *(You measured
  ~0.96.)* — the anti-hallucination metric.
- **Answer relevancy:** does the answer **address the question**? *(~0.89.)*
- **Context precision:** are the **retrieved chunks relevant** (not noise)? *(~0.87.)*
- **Context recall:** did retrieval **get the chunks needed** to answer? *(~0.72 — the retrieval-quality
  metric, the usual bottleneck.)*
- **LLM-as-judge** scores these (mind its biases, m34). **Gate in CI** on a fixed eval set — *(your
  degraded config FAILED the faithfulness gate 0.83<0.85 → exit 1).* Decompose: **low faithfulness →
  generation/prompt; low context recall → retrieval/chunking.**

> **The senior line:** *"I evaluate RAG on **faithfulness + context precision/recall + answer relevancy**
> (RAGAS-style, LLM-as-judge) on a fixed set, **gated in CI**. **Context recall is usually the bottleneck**
> — and since **most hallucinations are retrieval misses**, I debug **retrieval (chunking → hybrid →
> rerank) first**, generation second. That measure-and-gate discipline is what makes it production RAG, not
> a demo."*

---

## 7. Advanced RAG & the decision (brief)
- **Query rewriting / multi-query:** rewrite or expand the query (or generate several) for better
  retrieval. **HyDE:** generate a hypothetical answer, embed *that* to retrieve. **Agentic RAG:** an agent
  decides what/whether to retrieve, iterates (m32). **GraphRAG:** retrieve over a knowledge graph for
  multi-hop questions.
- **RAG vs long-context vs fine-tune:** **RAG** for large/fresh/private/citable knowledge; **long-context**
  for a bounded set you can afford to stuff (simpler, but costly + lost-in-the-middle); **fine-tune** for
  behavior. Often **RAG + fine-tune** (knowledge + behavior).

**Wrap-up:** *"RAG = **retrieve relevant context, then generate a grounded, cited answer** — knowledge
without retraining ('RAG = knowledge, fine-tune = behavior'). **Indexing (offline):** chunk → embed →
vector DB + BM25 index. **Query (online):** the **retrieve-then-rerank funnel** (hybrid BM25+vector+RRF →
cross-encoder rerank) → a **grounded prompt** (answer only from context, **cite**, **say I don't know**) →
generate. **Chunking is the #1 lever** (structure-aware + overlap — fixed splits cut facts → retrieval
misses → hallucinations), and **retrieval recall is the bottleneck** ('most hallucinations are retrieval
misses — fix retrieval first'). At **scale**: sharded vector DB, **incremental re-indexing** for freshness
(RAG's superpower), **metadata filtering** for multi-tenancy, **guardrails** (PII/prompt-injection, m34),
**caching** (generation is the cost). And you **gate on faithfulness + context recall in CI** — which makes
it production RAG, not a demo. I built exactly this in InsightDesk."*

---

## State-of-the-art & real-world notes 📚
- **The production RAG stack:** chunk → embed → **vector DB** (Pinecone/Weaviate/pgvector/Chroma/Milvus,
  HNSW) → **hybrid retrieve (BM25+vector, RRF)** → **cross-encoder rerank** → generate; **RAGAS** for eval.
- **"Lost in the Middle" (Liu et al.)** — LLMs under-attend to mid-context; **HyDE**, **query rewriting**,
  **GraphRAG (Microsoft)**, **agentic RAG** are the advanced techniques.
- **The honest truths** (your lessons): "most hallucinations are retrieval misses," "any production RAG has
  a recall metric," "RAG = knowledge, fine-tune = behavior."

---

## From your systems 🏭 (you built the centerpiece)
- **InsightDesk RAG = this module:** naive RAG → **chunking** (you measured the **boundary problem**:
  fixed-60 "14 business days" sim 0.55 vs **sentence-aware 0.79**) → **citations + grounding + refuse-when-
  unknown** (anti-hallucination) → **`RAGPipeline` (retrieve → rerank → generate)** → **top-k cost
  tradeoff**. You built §2–§4 hands-on.
- **Retrieval funnel = your m05–m06:** embeddings (MiniLM), **vector DB** (Chroma/pgvector, HNSW/IVF),
  **hybrid BM25+vector via RRF**, **cross-encoder reranking** — the §4 retrieval stack, yours.
- **Evaluation = your m09:** **faithfulness ~0.96, answer_relevancy ~0.89, context_precision ~0.87,
  context_recall ~0.72**, RAGAS-from-scratch, **LLM-as-judge + biases**, and a **CI regression gate** (a
  degraded config → exit 1). §6, exactly.
- **Cost design:** "embeddings/retrieval local/free, only generation calls the API" + **`/ask` cache**
  (929×, $0) = §5 cost levers (m29).
- **"RAG = knowledge, fine-tune = behavior"** (your m10) = §1/§7 decision.

---

## Key concepts (interview-ready)
- **RAG = retrieve grounded context → generate cited answer** = **knowledge** (fresh/private/citable, cheap
  to update by re-indexing); **fine-tune = behavior**. Decision: **prompt → RAG → fine-tune.**
- **Pipeline:** **indexing (offline):** chunk → embed → vector DB (+ BM25); **query (online):** embed →
  **retrieve (hybrid BM25+vector+RRF) → rerank (cross-encoder)** → grounded prompt → generate. (The m16/m25
  funnel feeding an LLM.)
- **Chunking is the #1 lever** — structure/sentence-aware + overlap; the **boundary problem** (fixed splits
  cut facts → retrieval misses). 
- **Grounding/anti-hallucination:** answer **only from context**, **cite sources**, **say "I don't know"**;
  right-size **top-k** (cost + "lost in the middle"). **"Most hallucinations are retrieval misses — fix
  retrieval first."**
- **Production:** sharded/replicated **vector DB**; **incremental re-indexing** for **freshness** (RAG's
  edge over fine-tune); **metadata filtering** for **multi-tenancy/isolation**; **guardrails** (PII/prompt-
  injection, m34); **caching** (generation = the cost, m29).
- **Evaluation (makes it production):** **faithfulness, context precision/recall, answer relevancy** (RAGAS/
  LLM-judge) **gated in CI**; **context recall = the usual bottleneck**; decompose failures (faithfulness→
  generation, recall→retrieval).

---

## Go deeper (reading)
- **RAGAS** (faithfulness/context-precision/recall — eval) and **"Lost in the Middle" (Liu et al.)**.
- **HyDE, query rewriting, GraphRAG (Microsoft), agentic RAG** — advanced retrieval.
- **Vector DB docs** (Pinecone/Weaviate/pgvector/Chroma) + **hybrid search + reranking** guides.
- **Your own AI/ML notes (`../ai_ml_llm/` m05 embeddings, m06 vector DBs/hybrid/rerank, m07 RAG, m09
  evaluation, m10 RAG-vs-fine-tune)** — you implemented this entire module; revisit them as the worked
  example.
- Revisit **m16** (retrieve-then-rerank/inverted index), **m25** (the funnel), **m18** (vector DB at scale),
  **m29** (LLM serving/cost), **m34** (guardrails/prompt-injection/eval-safety).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m30_production_rag/`](../../solutions/m30_production_rag/README.md).

When you've done the exercises, say **"Module 31"** to design an *LLM Chatbot / Assistant* — conversation
memory, context-window management, streaming, session state, and moderation (the conversational layer on
top of RAG + serving).
