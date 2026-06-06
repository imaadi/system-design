# m16 — Exercises (Typeahead + Search)

> Do these on paper / out loud before checking `solutions/m16_typeahead_search/`. The trie precompute and
> the inverted index + BM25 are the core; the hybrid/semantic bridge is your edge.

---

## Exercise 1 — Estimation
Assume **200M searches/day**, ~**18 keystrokes/query** with typeahead, debounce halves keystroke fires.
1. Search QPS (avg + peak 5×)?
2. Typeahead QPS after debouncing?
3. Which path is the latency/throughput-critical one, and what does that imply?

---

## Exercise 2 — The trie
1. Draw a trie for: cat, car, card, dog. Mark complete words.
2. For prefix "car", what's the lookup cost in terms of the trie?
3. Why is a naive trie too slow for ranked suggestions, and what's the fix?

---

## Exercise 3 — Building & serving typeahead
1. Where do the popularity rankings come from, and how often do you rebuild?
2. How do you serve <100 ms at high QPS? (Name three techniques.)
3. Why not `LIKE 'abc%'` in SQL — give two concrete reasons.

---

## Exercise 4 — Inverted index
1. Given doc1={fresh,coffee,beans}, doc2={coffee,mug}, doc3={tea,fresh}, write the inverted index.
2. For the query "fresh coffee" (AND), which docs match?
3. Why is this O(1)-ish lookup vs scanning all documents?

---

## Exercise 5 — The indexing pipeline
1. List the steps to turn raw documents into an inverted index.
2. What do stop-word removal and stemming each do, and why?
3. How fresh is search, and why is that acceptable? Name the pipeline that keeps it fresh.

---

## Exercise 6 — Ranking with BM25
1. What do TF and IDF each contribute to a relevance score?
2. Why does a match on a rare term rank higher than a match on a common one?
3. Why does BM25 normalize by document length?

---

## Exercise 7 — Sharding the index
1. Compare sharding by document vs by term (cost of a query in each).
2. Which is the common choice and why?
3. How is this the same trade you saw in m05 (secondary indexes)?

---

## Exercise 8 — Keyword vs semantic vs hybrid (your edge)
1. Give one query keyword search nails but semantic misses, and one where it's the reverse.
2. What is hybrid search, and how do you combine the two rankings?
3. What does a cross-encoder reranker add, and why can't you run it over the whole corpus?

---

## Exercise 9 — Edge cases
1. How do you handle a typo like "cofee"?
2. A movie site needs filters (genre, year, rating) plus text search. How do you design it?
3. A query is searched millions of times — how do you serve it cheaply?

---

## Exercise 10 — Tie-in 🏭
1. Connect this module to **RAG**: what part of m16 is the "R" in RAG, and what's the modern recipe?
2. Map this to two things you've built: **Sennzo multi-filter search** and your **AI/ML hybrid+rerank**
   pipeline.
3. How is the trie's precomputed top-k the same instinct as Questimate's snapshot tables?

---

When done, check `solutions/m16_typeahead_search/README.md`, then say **"Module 17"**.
