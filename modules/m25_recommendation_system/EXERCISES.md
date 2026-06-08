# m25 — Exercises (Recommendation System)

> Do these on paper / out loud before checking `solutions/m25_recommendation_system/`. The funnel,
> two-tower retrieval, and cold start are the core.

---

## Exercise 1 — Why a funnel?
1. You have 50M items and a 200 ms budget. Why can't you run your best ranking model on all of them?
2. Name the three stages and what each optimizes (recall vs precision) + rough item counts.
3. Which earlier module has the same retrieve-then-rerank shape?

---

## Exercise 2 — Two-tower retrieval
1. Describe the two towers and what they output.
2. What's precomputed offline vs computed online, and why is that the key trick?
3. What data structure / search makes retrieval fast over millions, and what's it called?

---

## Exercise 3 — Candidate sources
For each, name the approach (collaborative filtering / content-based / two-tower) and a strength + weakness:
1. "Users who bought this also bought…"
2. "More movies in the sci-fi genre you watched."
3. A learned embedding retrieval over the whole catalog.

---

## Exercise 4 — Ranking
1. What does the ranking stage predict, and over how many items?
2. Why can ranking afford an expensive model when retrieval can't?
3. What is multi-objective ranking and why use it?

---

## Exercise 5 — Re-ranking
1. Name four things re-ranking does beyond the relevance score.
2. Why does pure relevance produce a bad feed?
3. What's the difference between optimizing per-item vs the list as a whole?

---

## Exercise 6 — Cold start
1. New user, no history — what do you recommend, and how does the user embedding improve?
2. New item, no interactions — how do you get it recommended?
3. Why does two-tower help cold start specifically?

---

## Exercise 7 — Feedback loops & exploration
1. Explain the filter-bubble feedback loop.
2. What is exploration, and name two techniques.
3. What's position bias, and how does it corrupt training data?

---

## Exercise 8 — Evaluation
1. Model B beats model A on offline NDCG. Ship it? Why or why not?
2. What's the only real proof a recommender is better?
3. Name three biases that make offline replay misleading.

---

## Exercise 9 — Serving & scale
1. Sketch the hybrid batch+online serving of the funnel (what's offline vs online).
2. How do you scale retrieval to billions of items?
3. When would you precompute a user's recommendations vs compute online?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map the **retrieval** and **ranking** stages to specific things you built in your AI/ML notes
   (embeddings, vector DB, cross-encoder rerank).
2. How is the recsys funnel the same shape as your **hybrid search** and **RAG** work?
3. Design "recommended movies" for **Sennzo** end-to-end using this funnel.

---

When done, check `solutions/m25_recommendation_system/README.md`, then say **"Module 26"**.
