# Module 16 — Design Typeahead / Autocomplete + Search ⭐⭐

> Two related, **latency-obsessed read** problems: **(A) typeahead/autocomplete** (suggest as you type —
> must respond in **<100 ms on every keystroke**) and **(B) full-text search** (find relevant documents
> for a query). The lesson: **purpose-built data structures** (the **trie** and the **inverted index**)
> crush general-purpose DB queries for these access patterns, and the trick — as always — is to
> **precompute the expensive work offline** so the read is cheap. The bonus: this bridges classic IR
> (BM25) to the **semantic/vector search + RAG** you already built in your AI/ML notes.

> **Format (Part 2):** worked m01 walkthrough; the **trie** (typeahead) and **inverted index + ranking**
> (search) are the deep-dives.

---

## Step 1 — Requirements

**Functional (core):**
1. **Autocomplete:** given a **prefix**, return the **top-k suggestions** (ranked by popularity), updating
   as the user types.
2. **Full-text search:** given a query, return **relevant ranked documents**.
3. *(Stretch)* personalization, spelling correction / fuzzy match, filters/facets, semantic search.

**Non-functional (these drive it):**
- **Ultra-low latency (typeahead):** must feel **instant** — **<100 ms** — and it fires on **every
  keystroke**, so it's both latency- *and* throughput-critical.
- **Relevance:** suggestions/results must be good (ranking quality matters as much as speed).
- **High QPS, read-heavy:** search is read-dominated; typeahead multiplies QPS by keystrokes.
- **Fresh-ish:** new content/queries should appear soon (near-real-time), but **eventual is fine** (a
  doc searchable a few seconds late is OK).
- **Scalable:** millions–billions of documents/queries.

> **The defining insight:** these are **read-latency** problems solved by **specialized indexes built
> offline** (trie for prefixes, inverted index for terms) — *not* by `SELECT ... LIKE 'abc%'` on a SQL DB
> (a slow scan that can't rank well).

---

## Step 2 — Estimation
Assume **100M searches/day**, each query ~**20 keystrokes** with typeahead.
- **Search QPS:** 100M ÷ 10⁵ = **~1,200/sec** avg (~6k peak).
- **Typeahead QPS:** ~20× searches (one suggestion request per keystroke) = 2B/day ÷ 10⁵ = **~23,000/sec**
  avg (~115k peak). **Typeahead is the high-QPS, latency-critical path** (you reduce it with **debouncing**
  — fire only after a short pause — but it's still large) → heavy **caching** + precomputed structures.
- **Index size:** depends on the corpus; e.g. 100M docs × ~1 KB text → the inverted index is a similar
  order (terms + posting lists, compressed) → **sharded** across nodes.

> **The estimate frames it:** typeahead's keystroke-multiplied QPS at <100 ms → **precompute top-k per
> prefix + cache hot prefixes**; search's large corpus → **sharded inverted index**.

---

## Step 3 — API
```
GET /autocomplete?q=<prefix>&limit=10     -> { suggestions: ["...", ...] }    (must be <100ms)
GET /search?q=<query>&page=...&filters=... -> { results: [...], total, facets }
```
- **Debounce** client-side (wait ~50–150 ms after typing stops) to cut typeahead QPS.
- Search supports **pagination** (cursor/keyset) and **filters/facets**.

---

## Step 4 — Deep Dive: Typeahead with a trie ⭐⭐

### The trie (prefix tree)
A **trie** stores strings by **shared prefixes**: each node = a character, a path from the root = a
prefix, and nodes mark complete words. Looking up a prefix = walking down the tree character by character
— **O(length of prefix)**, independent of how many words exist.
```
        (root)
        /   \
       c     t
       |     |
       a     e
      / \    |
     t   r   a  → "tea"
     |   |
   "cat" "car"
```

### The key optimization: precompute top-k at each node ⭐
A naive trie, on prefix "ca", would **traverse the whole subtree** to find and rank all completions —
**too slow** at query time. Instead, **store the top-k most-popular completions *at each node***
(precomputed). Then a query is: **walk to the prefix node → return its cached top-k** — O(prefix length),
basically instant. This is the recurring theme: **do the expensive ranking offline; make the read O(1).**

### Ranking & building the trie
- **Rank suggestions** by **popularity/frequency** (how often a completion is searched), with **recency**
  and optional **personalization** as factors.
- **Build/update offline:** aggregate **query logs** (what people actually search) into frequency counts,
  then **rebuild or incrementally update** the trie (a batch/stream pipeline). Trending terms update the
  top-k. Real-time perfect freshness isn't needed — near-real-time is fine.
- **Sharding:** partition the trie by **prefix** (e.g. by first letter/letters) across nodes; **cache hot
  prefixes** (Redis/CDN) since prefix popularity is heavily skewed (Zipf) — a few prefixes dominate.

### Why not just `LIKE 'abc%'` in SQL?
A DB `LIKE 'abc%'` can use an index for the prefix but **still can't efficiently rank by popularity**,
struggles at typeahead QPS/latency, and won't do fuzzy/personalized suggestions well. The **trie +
precomputed top-k** is purpose-built for exactly this access pattern (m04: match the structure to the
query).

> **Say this:** *"Typeahead is a trie with the **top-k completions precomputed and cached at each node**,
> built offline from query-log frequencies, sharded by prefix and fronted by a cache — so a keystroke is
> an O(prefix-length) walk to a precomputed list, well under 100 ms. I wouldn't use SQL `LIKE` — it can't
> rank by popularity at this latency/QPS."*

---

## Step 5 — Deep Dive: Full-text search with an inverted index ⭐⭐

### The inverted index (the core data structure)
Instead of "document → its words," **invert** it to **"term → list of documents containing it"** (the
**posting list**). Now "find docs containing *coffee*" is an **O(1) lookup** of the term's posting list,
not a scan of every document.
```
  Forward:  doc1 → {coffee, beans, fresh}   doc2 → {coffee, mug}
  Inverted: "coffee" → [doc1, doc2]   "beans" → [doc1]   "mug" → [doc2]   ("fresh" → [doc1])
```

### The indexing pipeline (offline)
documents → **tokenize** (split into terms) → **normalize** (lowercase, remove **stop words** like
"the/a", **stemming**/lemmatization so "running"→"run") → build the **inverted index** (term → posting
list, often with positions + term frequencies). New/updated docs flow through a **near-real-time indexing
pipeline** (e.g. Kafka → indexer) — so freshness is **eventual** (m02).

### Query processing & ranking
1. **Tokenize + normalize** the query the same way.
2. **Look up** each term's posting list; **intersect** (AND) or **union** (OR) them to get candidate docs.
3. **Rank** the candidates by **relevance** — the standard is **BM25** (an improved TF-IDF):
   - **TF (term frequency):** more occurrences of the term in a doc → more relevant.
   - **IDF (inverse document frequency):** **rare terms matter more** (a match on "espresso" is more
     meaningful than on "the"). BM25 adds saturation + length normalization (long docs don't unfairly win).
4. Return the top-ranked page.

### Sharding the index (the m05 trade returns)
- **Shard by document** (each shard holds the full index for a *subset of docs*): a query **scatter-
  gathers** across all shards then merges — easy to add docs, query hits everyone. **(The common choice.)**
- **Shard by term** (each shard owns some *terms'* posting lists): a query hits only the shards for its
  terms — but load is skewed (hot terms) and multi-term queries get complex. *(This is the local vs global
  secondary-index trade from m05.)*
- **Replicate** shards for read throughput + availability (m05).

> **Elasticsearch / Apache Lucene** is the industry-standard inverted-index search engine — it *is* this
> design productized (tokenization, BM25, sharding, replication, facets).

---

## Step 6 — Bridge to AI/ML: keyword vs semantic vs hybrid search ⭐ 🏭 (your strength)

This is where your AI/ML notes connect directly. The inverted index above is **lexical/keyword search** —
it matches **exact terms** (great precision, but **misses synonyms/paraphrases**: a search for "laptop"
won't match "notebook computer"). The modern complement:
- **Semantic (vector) search:** embed the query and documents into vectors (embeddings) and find the
  **nearest** by cosine similarity in a **vector DB** (ANN: HNSW/IVF) — captures **meaning**, finds
  paraphrases, but can miss exact-term/rare-keyword matches. *(Your AI/ML m05–m06.)*
- **Hybrid search (state of the art):** run **both** BM25 (keyword) and vector (semantic), then **combine
  the rankings** — commonly with **Reciprocal Rank Fusion (RRF)** — and optionally **rerank** the top
  results with a **cross-encoder** for precision. *(Your AI/ML m06: "hybrid BM25+vector via RRF +
  cross-encoder reranking" — you literally built this.)*

> **The senior line (and your edge):** *"Classic search is a BM25 inverted index — fast, exact-term,
> high-precision, but blind to meaning. I'd add **semantic vector search** (embeddings + an ANN index like
> HNSW) for synonym/paraphrase recall, then **fuse** them (hybrid, e.g. RRF) and **rerank** the top
> candidates with a cross-encoder — the modern recipe. I've built exactly this hybrid+rerank pipeline."*
> This connects m16 → your m05–m07 AI work → RAG (m30 here).

---

## Step 7 — Bottlenecks, scale & edge cases

- **Typeahead latency/QPS:** precompute top-k per node + **cache hot prefixes** (CDN/Redis) + **debounce**
  + shard by prefix. Hot prefixes are a hot-key problem (m06).
- **Index freshness:** the index is built offline → new docs aren't instantly searchable → **near-real-
  time indexing pipeline** (Kafka → indexer); accept **eventual** searchability.
- **Spelling/fuzzy match:** typos ("cofee") → edit-distance/fuzzy matching, n-grams, or "did you mean"
  (and embeddings naturally handle some of this in semantic search).
- **Hot queries:** cache popular query results (m06).
- **Personalization & ranking:** ranking is itself an ML problem (learning-to-rank) — for a real product
  it's a ranking system (m26) on top of candidate retrieval.
- **Scale:** shard the inverted index (by doc), replicate; the trie sharded by prefix.
- **Multi-filter search** (genre/year/rating — like a movie site): combine the inverted-index match with
  **structured filters** (often a secondary structure / Elasticsearch filters), then rank.
- **Observability (m10):** p99 typeahead/search latency, cache hit ratio, index lag (freshness), zero-
  result rate, click-through (relevance signal).

**Wrap-up:** *"Two read-latency systems backed by **purpose-built, offline-built indexes**. Typeahead = a
**trie with precomputed top-k at each node** (built from query logs, sharded by prefix, cached) → an
O(prefix) lookup under 100 ms. Search = an **inverted index** (tokenize→normalize→posting lists) with
**BM25** ranking, **sharded by document** and replicated, fed by a **near-real-time indexing pipeline**
(eventual freshness). Modern recipe: **hybrid keyword (BM25) + semantic (vector/HNSW) search fused with
RRF and reranked** — the pipeline I built in my AI/ML work. Key theme: precompute the expensive ranking
offline so reads are cheap; match the data structure to the access pattern (don't use SQL `LIKE`)."*

---

## State-of-the-art & real-world notes 📚
- **Elasticsearch / Apache Lucene / OpenSearch / Solr** — the inverted-index search standard (BM25,
  sharding, replication, facets, fuzzy). **Algolia** — a hosted low-latency search/typeahead engine.
- **BM25** is the default lexical ranker (TF-IDF's better successor). **Vector DBs** (FAISS, **Chroma**,
  pgvector, Pinecone, Weaviate, Milvus) + **HNSW/IVF** ANN power semantic search. **RRF + cross-encoder
  rerank** is the standard hybrid recipe — and the heart of modern **RAG** (m30).
- **Google autocomplete** ranks by query popularity + freshness + personalization + safety filters — a
  trie/precompute + ranking system at planet scale.

---

## From your systems 🏭 (strong tie-in)
- **You built the modern half:** your AI/ML notes cover **embeddings + semantic search** (m05), **vector
  databases** (Chroma/pgvector, HNSW/IVF, m06), **hybrid BM25+vector via RRF + cross-encoder reranking**
  (m06), and **RAG** (m07). m16 is the **classic IR foundation** under that — together you can speak to
  *both* lexical and semantic search, which most candidates can't.
- **Sennzo movie platform (Questimate-era):** you built **multi-filter search** (genre/year/rating/
  runtime) — a real search system → exactly Step 7's "inverted-index match + structured filters + rank."
- **DCE cross-cutting query:** "find all claims with edit code 597" (m04 exercise) is an **inverted-index
  need** (term → matching claims) — the same invert-the-access-pattern idea.
- **Precompute-for-fast-reads:** the trie's precomputed top-k and the offline-built index are the same
  **materialize-offline-so-reads-are-cheap** instinct as Questimate's snapshot tables (m04/m13).

---

## Key concepts (interview-ready)
- **Latency-obsessed reads → purpose-built, offline-built indexes** (not SQL `LIKE`); **precompute the
  expensive ranking offline so the read is O(1)** (the recurring theme).
- **Typeahead = a trie** with **top-k completions precomputed + cached at each node** (built from query
  logs), sharded by prefix, cached for hot prefixes; lookup = O(prefix length), <100 ms; debounce keystrokes.
- **Search = an inverted index** (term → posting list): pipeline = tokenize → normalize (lowercase,
  stop-words, stemming) → posting lists; query = look up + intersect/union → **rank with BM25** (TF×IDF,
  rare terms matter more). **Shard by document** (scatter-gather, common) vs by term (m05 trade); replicate.
- **Freshness is eventual** — near-real-time indexing pipeline (Kafka → indexer).
- **Keyword (BM25) vs semantic (vector/HNSW) vs hybrid:** lexical = exact terms (precision, misses
  meaning); semantic = meaning (recall, misses exact terms); **hybrid (RRF) + cross-encoder rerank = SOTA**
  — the RAG retrieval recipe (your AI/ML work, → m30).
- **Edge cases:** fuzzy/typo correction, hot-query caching, personalization (→ ranking, m26), multi-filter.

---

## Go deeper (reading)
- **"Introduction to Information Retrieval" (Manning, Raghavan, Schütze) — free online** — the canonical
  inverted-index + ranking text (BM25, tokenization).
- **Elasticsearch / Lucene docs** — the productized inverted index (analysis, BM25, sharding).
- **System Design Interview (Alex Xu): "Design a search autocomplete system"** — the trie/precompute
  treatment.
- **Your AI/ML notes (`../ai_ml_llm/` m05 embeddings, m06 vector DBs/hybrid/rerank, m07 RAG)** — the
  semantic half of Step 6; revisit them alongside this.
- Revisit **m04** (match structure to access pattern), **m05** (sharding/secondary indexes), **m06**
  (caching/hot keys), **m26** (learning-to-rank), **m30** (RAG).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m16_typeahead_search/`](../../solutions/m16_typeahead_search/README.md).

When you've done the exercises, say **"Module 17"** to design a *Web Crawler* (BFS frontier, politeness,
dedup, distributed coordination, freshness) — which *feeds* the search index you just built.
