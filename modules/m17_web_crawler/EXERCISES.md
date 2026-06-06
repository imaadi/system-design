# m17 — Exercises (Web Crawler)

> Do these on paper / out loud before checking `solutions/m17_web_crawler/`. The frontier, politeness, and
> dedup are the core.

---

## Exercise 1 — Estimation
Target **2 billion pages/month**, avg page **400 KB**, compress to ~**80 KB**.
1. Crawl rate (pages/sec, avg)?
2. Download bandwidth?
3. Storage/month (raw and compressed)?
4. Why can't you store the seen-URL set exactly in memory, and what do you use?

---

## Exercise 2 — The frontier
1. What two responsibilities does the URL frontier combine?
2. Describe the front-queue / back-queue design and what each layer is for.
3. Why BFS over DFS?

---

## Exercise 3 — The throughput/politeness tension
1. State the core tension in one sentence.
2. How do per-host back-queues resolve it?
3. Where does total throughput come from if you can only trickle requests to each host?

---

## Exercise 4 — Politeness
1. List four politeness rules a crawler must follow.
2. What happens if you ignore them?
3. How does politeness relate to a concept from m12?

---

## Exercise 5 — URL dedup
1. Why use a Bloom filter for the seen-URL set, and what's the acceptable downside?
2. Give three URL-normalization rules and why they matter.
3. What does URL dedup prevent (vs content dedup)?

---

## Exercise 6 — Content dedup
1. Why isn't URL dedup enough — give an example of duplicate content at different URLs.
2. Exact-duplicate detection vs near-duplicate detection — what tool for each?
3. What does content dedup prevent (vs URL dedup)?

---

## Exercise 7 — Distribution & traps
1. How do you shard the crawler across nodes, and what *two* benefits does sharding by host give?
2. Name three crawler traps and a defense for each.
3. Why is DNS a bottleneck and what's the fix?

---

## Exercise 8 — Freshness
1. How do you decide how often to re-crawl a page?
2. What competes with freshness for the crawl budget?
3. Which HTTP mechanisms help you avoid re-downloading unchanged pages?

---

## Exercise 9 — Storage & feeding search
1. Where do raw pages live vs metadata/link graph, and why the split?
2. How does this crawler connect to m16 (search)?
3. Trace the full data flow "crawl → index → search → RAG."

---

## Exercise 10 — Your-systems tie-in 🏭
1. You scraped 30+ sources at Famepilot and NSE/Google-News at Questimate. Name two crawler concepts
   from this module that scale that experience.
2. Map the **frontier** and the **seen-set** to tools/structures you've used (queue; Bloom filter).
3. How is "don't crawl the same content twice" the same instinct as DCE suppression/dedup?

---

When done, check `solutions/m17_web_crawler/README.md`, then say **"Module 18"**.
