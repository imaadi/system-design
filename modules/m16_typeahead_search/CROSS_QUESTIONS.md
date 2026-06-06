# m16 — Cross-Questions ("if-and-buts") on Typeahead + Search

> Answer out loud in 2–3 sentences before reading the model answer. The trie precompute, the inverted
> index + BM25, and the keyword-vs-semantic bridge are where this is won.

---

### Q1. Why not just use `SELECT ... WHERE name LIKE 'abc%'` for autocomplete?
**A.** Because it doesn't meet the requirements: a `LIKE 'abc%'` can use an index for the prefix but
**can't efficiently rank by popularity**, can't hit **<100 ms at typeahead QPS** (keystroke-multiplied),
and won't do fuzzy/personalized suggestions. Autocomplete needs a **purpose-built structure** — a **trie
with precomputed top-k per node** — that matches the access pattern (prefix → ranked completions). It's
the m04 lesson: match the data structure to the query, don't force a general DB scan.

---

### Q2. How does a trie make prefix lookup fast?
**A.** A **trie** stores strings by shared prefixes — each node is a character, and a path from the root
spells a prefix. Looking up a prefix is just **walking down character by character — O(prefix length)**,
*independent* of how many words are stored (a billion words doesn't slow a 3-char prefix lookup). That's
why it beats scanning a list/DB for matches.

---

### Q3. What's the key optimization that makes typeahead actually fast?
**A.** **Precompute the top-k completions at each trie node.** A naive trie would, on prefix "ca", have to
**traverse the entire subtree** to find and rank completions — too slow. Instead you store the **top-k
most-popular completions directly at each node** (computed offline). Then a query is "walk to the prefix
node → return its cached top-k" — O(prefix length), basically instant. It's the recurring theme: **do the
expensive ranking offline so the read is O(1).**

---

### Q4. How do you build and update the trie, and how fresh does it need to be?
**A.** Build it **offline from query logs**: aggregate how often each completion is searched (frequency),
plus recency, and compute the top-k per node, then **rebuild or incrementally update** the trie on a
schedule (a batch/stream pipeline). Freshness can be **near-real-time, not instant** — a trending term
appearing in suggestions a few minutes late is fine (eventual, m02). You don't update the trie on every
keystroke; you update the *counts* and periodically refresh the precomputed top-k.

---

### Q5. How do you scale typeahead to high QPS and low latency?
**A.** **Precompute top-k** (so reads are O(prefix)), **cache hot prefixes** (Redis/CDN — prefix
popularity is heavily Zipf-skewed, so a few prefixes serve most traffic), **shard the trie by prefix**
across nodes, and **debounce** client-side (only fire after a brief typing pause) to cut keystroke QPS.
Hot prefixes are a hot-key problem (m06), handled by caching/replication. The combination keeps p99 well
under 100 ms at tens of thousands of QPS.

---

### Q6. What is an inverted index and why is it the core of search?
**A.** It **inverts** "document → its words" into **"term → list of documents containing it"** (the
posting list). So "find documents containing *coffee*" becomes an **O(1) lookup** of that term's posting
list instead of scanning every document. It's the foundational search data structure because it makes
**finding documents by term** fast — the exact access pattern search needs (m04: model the access
pattern). Multi-term queries intersect/union the posting lists.

---

### Q7. Walk me through the indexing pipeline.
**A.** Documents → **tokenize** (split into terms) → **normalize** (lowercase, remove **stop words** like
"the/a", apply **stemming/lemmatization** so "running"→"run") → build the **inverted index** (term →
posting list, often with term frequencies + positions). New/updated docs flow through a **near-real-time
indexing pipeline** (e.g. Kafka → indexer), so search freshness is **eventual**. The same tokenize+
normalize steps are applied to the **query** at search time so terms match.

---

### Q8. How do you rank search results? What's BM25?
**A.** With a relevance score; the standard is **BM25** (an improved TF-IDF). **TF (term frequency):** the
more a query term appears in a document, the more relevant (with saturation — diminishing returns).
**IDF (inverse document frequency):** **rarer terms count more** (matching "espresso" is more meaningful
than "the"). BM25 also **normalizes by document length** so long documents don't unfairly accumulate
matches. You compute BM25 over the candidate docs (from the posting-list intersection) and return the top
page.

---

### Q9. Why do rare terms matter more in ranking (IDF)?
**A.** Because common terms (stop words, very frequent words) appear in almost every document, so matching
them tells you **almost nothing** about relevance — whereas a **rare term** that appears in few documents
is **highly discriminating** (if a doc contains "espresso," it's probably actually about espresso). IDF
=`log(total_docs / docs_containing_term)` gives rare terms a high weight and common terms near-zero, so
the score is driven by the meaningful, specific matches.

---

### Q10. How do you shard the inverted index?
**A.** Two ways (the m05 trade). **Shard by document** (each shard holds the complete index for a *subset
of documents*): a query **scatter-gathers** across all shards and merges results — easy to add documents,
every query touches all shards (the **common** choice). **Shard by term** (each shard owns some *terms'*
posting lists): a query hits only the shards for its terms, but load is **skewed** (hot terms) and
multi-term queries get complex. You also **replicate** shards for read throughput and availability.

---

### Q11. How fresh is search, and why is that acceptable?
**A.** **Eventually fresh** — because the inverted index is **built by a pipeline**, a new/updated
document isn't searchable until it's been indexed (seconds to minutes via a near-real-time pipeline).
That's acceptable for almost all search use cases (a new product/post being searchable a few seconds late
harms nothing), and the trade buys you a precomputed index that makes **reads fast**. If you needed
instant searchability you'd pay heavily; almost nobody does.

---

### Q12. Keyword (BM25) vs semantic (vector) search — what's the difference?
**A.** **Keyword/lexical (BM25 over an inverted index)** matches **exact terms** — high precision, fast,
but **blind to meaning** (a search for "laptop" misses "notebook computer"). **Semantic (vector) search**
embeds query and documents into vectors and finds the **nearest by cosine similarity** (ANN: HNSW/IVF) —
it captures **meaning/paraphrase** (great recall) but can **miss exact-term/rare-keyword** matches and is
heavier. They have complementary strengths, which is why the modern answer is to combine them.

---

### Q13. What's hybrid search and how do you combine the two?
**A.** **Hybrid search** runs **both** BM25 (keyword) and vector (semantic) retrieval and **fuses their
rankings** — commonly with **Reciprocal Rank Fusion (RRF)** (combine by reciprocal of each result's rank
in each list) — then often **reranks** the top candidates with a **cross-encoder** for precision. You get
keyword precision + semantic recall. This is the **state-of-the-art retrieval recipe** and the heart of
modern **RAG** (m30) — and it's exactly the hybrid+rerank pipeline I built in my AI/ML work.

---

### Q14. What's a cross-encoder reranker and why use it?
**A.** Retrieval (BM25/vector) is fast but approximate — it scores query and doc somewhat independently.
A **cross-encoder** takes the **query and a candidate together** and scores their relevance jointly
(more accurate but expensive). So you **retrieve a larger candidate set cheaply**, then **rerank the top
N** (say 100→10) with the cross-encoder for a precise final order. It's the "cheap recall, then expensive
precision on a small set" pattern — you can't run a cross-encoder over millions of docs, but over 100
it's affordable and lifts quality a lot.

---

### Q15. How do you handle typos / fuzzy matching?
**A.** Several ways: **edit-distance/fuzzy matching** (match terms within 1–2 character edits), **n-gram
indexing** (index character n-grams so "cofee" overlaps "coffee"), a **"did you mean" / spell-correction**
step (correct the query against a dictionary of known terms before searching), and **phonetic** matching.
Notably, **semantic/vector search** is naturally somewhat typo-tolerant (the embedding of a slightly
misspelled query is still close). Most engines (Elasticsearch) offer fuzzy queries built-in.

---

### Q16. How would you design search for a movie site with filters (genre, year, rating)?
**A.** Combine **full-text relevance** with **structured filters**: the inverted index handles the text
query (title/description match + BM25), and **filters** (genre=X, year>2000, rating≥4) restrict the
candidate set — implemented as additional indexed fields / Elasticsearch filter clauses (or a secondary
structure). You apply filters to narrow, then rank the matches by relevance (and maybe popularity). This
is exactly the **multi-filter search** pattern (I built one for Sennzo) — text match + faceted filters +
ranking.

---

### Q17. The same popular query is searched millions of times. How do you handle it?
**A.** **Cache the query results** (m06) — a hot query's result page is identical for everyone (modulo
personalization), so serve it from a result cache instead of re-running retrieval+ranking each time. For
typeahead, the **hot prefixes** are similarly cached/replicated. It's the standard hot-key treatment:
cache the popular reads, and the long tail goes to the index. Watch for cache stampede on a hot query's
expiry (single-flight, m06).

---

### Q18. Where does ranking become a machine-learning problem?
**A.** Beyond BM25, real search/feeds use **learning-to-rank (LTR)**: a model scores candidates using many
features (BM25 score, semantic similarity, popularity, freshness, personalization, click history) trained
on **click/engagement data** to optimize relevance. So the pipeline is **candidate retrieval (inverted
index + vector) → ML ranking → re-ranking**. That ranking model is its own system (we design it in **m26**
— feed ranking / CTR). For this module, retrieval is the focus; ranking quality is where ML enters.

---

### Q19. What are the main bottlenecks and how do you address them?
**A.** **Typeahead latency/QPS** (precompute top-k + cache hot prefixes + shard by prefix + debounce);
**index size** (shard by document + replicate); **index freshness** (near-real-time pipeline, accept
eventual); **hot queries/prefixes** (result caching — hot-key handling); **relevance** (BM25 → hybrid →
rerank → LTR). Observability (m10): p99 typeahead/search latency, cache hit ratio, **index lag**
(freshness), zero-result rate, click-through rate (the relevance signal you optimize against).

---

### Q20. How does this connect to RAG / your AI-ML work?
**A.** **Retrieval-Augmented Generation (RAG)** is exactly this search problem feeding an LLM: you
**retrieve** the most relevant chunks for a question (the *modern hybrid recipe* — BM25 + vector/HNSW,
fused with RRF, **reranked** with a cross-encoder), then pass them to the LLM to generate a grounded
answer. So m16's retrieval *is* the "R" in RAG (which I built in my AI/ML notes, and which we design at
scale in **m30**). The inverted index + BM25 here is the classic foundation; embeddings/vector DBs/rerank
are the modern layer — and RAG combines them.
