# m16 — Solutions (Typeahead + Search)

> Check the *reasoning*. Recurring lessons: purpose-built offline-built indexes (trie, inverted index);
> precompute ranking so reads are O(1); BM25 (rare terms matter); keyword+semantic hybrid+rerank = SOTA.

---

## Exercise 1 — Estimation
200M searches/day, ~18 keystrokes, debounce halves.
1. **Search QPS:** 200M ÷ 10⁵ = **~2,000/sec** avg (~10k peak).
2. **Typeahead QPS:** 18 keystrokes × ½ (debounce) = ~9 fires/query → 200M × 9 = 1.8B/day ÷ 10⁵ =
   **~18,000/sec** avg (~90k peak).
3. **Typeahead** is the latency/throughput-critical path (≈9× search volume, must be <100 ms) → implies
   **precomputed top-k + cache hot prefixes + shard by prefix + debounce**; search is lower-QPS but
   larger per-query work (retrieval + ranking over a big index).

---

## Exercise 2 — The trie
1. Trie:
   ```
   (root)─c─a─t*            (cat)
              \─r* ─d*       (car, card)
        └─d─o─g*            (dog)
   ```
   (`*` = complete word.)
2. **O(3)** — walk c→a→r, three steps (length of the prefix), independent of total word count.
3. A naive trie must **traverse the whole subtree** under "car" to collect + rank all completions →
   slow. **Fix: precompute and store the top-k completions at each node**, so the query just returns the
   node's cached list (O(prefix length)).

---

## Exercise 3 — Building & serving typeahead
1. From **query logs** (aggregate how often each completion is searched → frequency, plus recency); rebuild
   /incrementally update the trie on a schedule (**near-real-time**, not per-keystroke).
2. **(a)** precompute top-k at each node (O(prefix) reads); **(b)** **cache hot prefixes** (Redis/CDN —
   Zipf-skewed); **(c)** **shard by prefix** + **debounce** keystrokes client-side.
3. `LIKE 'abc%'`: **(a)** can't efficiently **rank by popularity** (it just matches), **(b)** can't hit
   **<100 ms at typeahead QPS** / no fuzzy/personalized suggestions — wrong tool for the access pattern.

---

## Exercise 4 — Inverted index
1. ```
   fresh  → [doc1, doc3]
   coffee → [doc1, doc2]
   beans  → [doc1]
   mug    → [doc2]
   tea    → [doc3]
   ```
2. "fresh coffee" (AND) = intersect `fresh`[doc1,doc3] ∩ `coffee`[doc1,doc2] = **[doc1]**.
3. Because you **look up each term's posting list directly** (O(1) per term) and intersect short lists,
   instead of scanning **every** document to check if it contains the terms — the index inverts the access
   pattern from "doc→words" to "word→docs."

---

## Exercise 5 — The indexing pipeline
1. documents → **tokenize** (split into terms) → **normalize** (lowercase, remove **stop words**, **stem/
   lemmatize**) → build **inverted index** (term → posting list w/ term frequencies/positions).
2. **Stop-word removal** drops near-useless ultra-common words ("the/a/is") to shrink the index and avoid
   noise; **stemming** reduces words to a root ("running/runs/ran" → "run") so different forms of a word
   match. Both improve match quality + index size.
3. **Eventually fresh** — new docs aren't searchable until indexed (seconds–minutes). Acceptable because a
   new doc searchable slightly late harms nothing, and it buys a precomputed fast-read index. Kept fresh by
   a **near-real-time indexing pipeline** (e.g. Kafka → indexer).

---

## Exercise 6 — Ranking with BM25
1. **TF (term frequency):** more occurrences of the query term in a doc → more relevant (with saturation/
   diminishing returns). **IDF (inverse document frequency):** rarer terms get higher weight (more
   discriminating).
2. Because a **rare term** appears in few documents, so matching it strongly signals the doc is *about*
   that term; a **common term** appears almost everywhere, so matching it says little — IDF down-weights
   common, up-weights rare.
3. So **long documents don't unfairly win** just by containing more words (and thus more incidental term
   matches) — length normalization keeps a focused short doc competitive with a sprawling long one.

---

## Exercise 7 — Sharding the index
1. **By document:** each shard indexes a subset of docs → a query **scatter-gathers** across all shards +
   merges (every query touches all shards, but adding docs is easy). **By term:** each shard owns some
   terms → a query hits only its terms' shards (fewer shards), but **load is skewed** (hot terms) and
   multi-term queries get complex.
2. **By document** is common — adding documents is simple, load spreads evenly across shards, and
   scatter-gather parallelizes well.
3. It's the **m05 secondary-index trade**: by-document ≈ **local index** (cheap writes, scatter-gather
   reads); by-term ≈ **global index** (focused reads, cross-shard writes/skew).

---

## Exercise 8 — Keyword vs semantic vs hybrid
1. **Keyword nails, semantic misses:** a rare exact token / code / name (e.g. "ZX-4500" or "edit 597") —
   exact-term match. **Semantic nails, keyword misses:** a paraphrase ("laptop" vs "notebook computer", or
   "how to fix a slow PC" → an article titled "speeding up your computer") — meaning, not exact words.
2. **Hybrid** runs **both** BM25 and vector retrieval and **fuses the rankings** — commonly **Reciprocal
   Rank Fusion (RRF)** (combine by reciprocal of each item's rank in each list) — getting keyword
   precision + semantic recall.
3. A **cross-encoder** scores the **query+doc jointly** (more accurate than independent retrieval scores),
   adding precision on the final order. You **can't run it over the whole corpus** (millions of doc×query
   scorings = far too expensive); you run it only on the **top-N candidates** from retrieval (e.g. 100→10).

---

## Exercise 9 — Edge cases
1. **Typo "cofee":** fuzzy/edit-distance matching, character **n-gram** indexing (overlap with "coffee"),
   a **spell-correct / "did you mean"** step, or rely on **semantic** search's natural typo tolerance.
2. **Movie site:** inverted index for the **text** query (title/description + BM25) **plus structured
   filters** (genre=X, year>2000, rating≥4) as indexed fields / filter clauses (Elasticsearch); filter to
   narrow, then rank matches by relevance/popularity. (The multi-filter search I built for Sennzo.)
3. **Cache the query's result page** (m06) — identical for everyone (modulo personalization) — and serve
   from cache; protect against stampede on expiry (single-flight). Hot prefixes cached similarly.

---

## Exercise 10 — Tie-in 🏭
1. **RAG:** m16's **retrieval** *is* the "R" — for a question you retrieve the most relevant chunks, then
   feed them to the LLM to generate a grounded answer. The **modern recipe** = hybrid **BM25 + vector
   (HNSW) retrieval, fused with RRF, then reranked with a cross-encoder** (designed at scale in m30).
2. **Sennzo multi-filter search** = inverted-index text match + structured filters + ranking (Step 7 in
   production). **Your AI/ML hybrid+rerank** (notes m06) = exactly the Step 6 SOTA recipe (BM25 + vector +
   RRF + cross-encoder) — you've built both halves: classic IR *and* semantic.
3. The trie's **precomputed top-k per node** and the **offline-built inverted index** are the same
   **materialize-the-expensive-work-offline-so-reads-are-O(1)** instinct as Questimate's **snapshot
   tables** (m04/m13) — precompute the ranking/aggregation, serve a cheap lookup.
