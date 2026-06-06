# m17 — Cross-Questions ("if-and-buts") on the Web Crawler

> Answer out loud in 2–3 sentences before reading the model answer. The frontier, politeness, dedup, and
> traps are where this is won.

---

### Q1. What's the overall architecture of a web crawler?
**A.** A **distributed producer-consumer pipeline** (m08): a **URL frontier** (the queue of what to crawl
next) feeds **fetchers** (download pages, obeying robots.txt + per-host limits) → **parsers** (store raw
HTML to object storage, extract outlinks) → **URL normalizer + dedup** (Bloom filter for "seen?") → new
URLs go **back into the frontier**. It starts from seeds and BFS-traverses the web graph, sharded across
crawler nodes by host, storing content for downstream indexing (m16).

---

### Q2. Why is BFS preferred over DFS for crawling?
**A.** The frontier is a **queue**, which gives **breadth-first** traversal — it **spreads the crawl
across many hosts**, gets broad coverage quickly, and naturally distributes load. **DFS** would tunnel
**deep into a single site** (many requests to one host in a row → violates politeness) and over-crawl one
domain before others. BFS aligns with the politeness model (parallel across hosts) and coverage goals.

---

### Q3. What is the URL frontier and why is it more than a simple queue?
**A.** It decides **what to crawl next and at what rate**, baking in **priority** and **politeness**. It's
two layers (the Mercator design): **priority front-queues** (URLs bucketed by importance/freshness — crawl
a news homepage before a deep archive page) and **per-host back-queues** (one queue per host with a
next-allowed-crawl time, enforcing per-host delay). So it's a prioritized, politeness-aware scheduler, not
a plain FIFO.

---

### Q4. Explain the core tension between throughput and politeness.
**A.** You want **maximum global throughput** (crawl as many pages/sec as possible) but you **must not
hammer any single host** (politeness — or you DoS the site and get banned). These conflict: you can't just
crawl fast against one site. The resolution: **per-host back-queues** — crawl **thousands of *different*
hosts in parallel** (high total throughput) but only **one request at a time with a delay** to **each**
host. **Speed comes from breadth (many hosts), not depth (hammering one).**

---

### Q5. How do you enforce politeness?
**A.** **Obey `robots.txt`** (allowed paths + `Crawl-delay`, cached per host), **rate-limit per host**
(one/few concurrent requests with a delay — the back-queue's next-allowed time, often scaled to the host's
response time), send an honest **User-Agent** with contact info, and **honor `Retry-After`/429/503** (back
off when a site signals overload). It's **m12 rate limiting applied per domain** — politeness is a hard
constraint, not a nicety.

---

### Q6. How do you deduplicate URLs at billions scale?
**A.** You can't keep an exact set of billions of URLs in memory, so use a **Bloom filter** (m06): it
answers "**definitely not seen**" or "**maybe seen**" with tiny memory and no false negatives (rare false
positives just make you occasionally skip a new URL — acceptable). Combine it with **URL normalization**
(canonicalize: lowercase host, strip default ports/fragments/tracking params, sort query params, resolve
relative URLs) so different spellings of the same URL collapse to one.

---

### Q7. Why do you also need *content* deduplication, separate from URL dedup?
**A.** Because the **same content appears at many different URLs** (mirrors, print/AMP versions, session-ID
URLs, trailing-slash variants) — URL dedup won't catch those since the URLs differ. Content dedup catches
it: **hash** the content for exact duplicates, and **SimHash/MinHash** fingerprints for **near-duplicates**
(similar fingerprint = similar content). URL dedup avoids **re-fetching** the same URL; content dedup
avoids **re-storing/indexing** the same content reached via different URLs.

---

### Q8. What is SimHash and why use it?
**A.** **SimHash** maps a document to a fingerprint such that **similar documents get similar fingerprints**
(small Hamming distance) — unlike a normal hash where one byte changes everything. So you can detect
**near-duplicate** pages (e.g. two articles differing only in ads/boilerplate) by comparing fingerprint
distance, and skip re-indexing them. It's essential because a huge fraction of the web is near-duplicate
content, and you don't want to waste storage/index space on it.

---

### Q9. What are crawler traps and how do you avoid them?
**A.** **Crawler traps** are **infinite (or huge) URL spaces** that can trap a crawler forever: a calendar
with an endless "next month" link, **session IDs** in URLs generating infinite variants, deep
**faceted-filter** combinations, or dynamically-generated link mazes. Defenses: **URL normalization**
(strip session params), **depth limits** (max link-depth from seed), **per-host page caps**, and **trap
detection** (flag a host emitting unbounded distinct URLs). Without these, a single trap can consume your
whole crawl budget.

---

### Q10. How do you distribute the crawler across many machines?
**A.** **Shard the frontier by host** (consistent hashing, m05) so each crawler node **owns** a set of
hosts. This does double duty: it **spreads load** across nodes **and naturally enforces per-host
politeness** — since one node controls all requests to a given host, it can pace them without any
cross-node coordination (no two nodes accidentally hammering the same site). Consistent hashing lets you
add/remove nodes with minimal host reshuffling.

---

### Q11. How do you keep crawled content fresh?
**A.** **Re-crawl adaptively by estimated change frequency**: pages that change often (news homepages,
forums) are re-crawled frequently; static/archival pages rarely. You re-add URLs to the frontier with a
priority/schedule based on observed change rate (and signals like HTTP `Last-Modified`/`ETag`, sitemaps).
Freshness competes with new-page **coverage** and **politeness** for the frontier's limited crawl budget,
so it's a prioritization trade-off, not "re-crawl everything."

---

### Q12. Why is DNS a bottleneck, and what do you do about it?
**A.** A crawler resolves **millions of distinct hostnames**, and a DNS lookup is a network round trip
(m03) that can **dominate fetch time** — naive crawling spends more time on DNS than downloading. Fix:
a **DNS cache** (cache resolutions per host with TTL), possibly a dedicated/prefetching resolver, so most
fetches skip the lookup. It's an easy-to-miss bottleneck that materially affects crawl throughput at scale.

---

### Q13. Where do you store the crawled pages and the link graph?
**A.** **Raw pages → object storage (S3/blob)** — they're large, numerous (PB-scale), write-once/read-
later blobs (m18/m04 "split metadata from blobs"). **Metadata** (URL, fetch time, status, content hash) +
the **link graph** (who links to whom) → a **database** (often a wide-column/graph store at scale). The
**search index** is built **downstream** from this content (m16). Keeping bytes in object storage and
metadata/graph in a DB matches the m04 access-pattern split.

---

### Q14. How do you handle a page that times out or returns an error?
**A.** **Retry with exponential backoff** (m10) for transient failures (timeouts, 5xx); after N attempts,
give up and record the failure. Honor **`Retry-After`/429** (the host is asking you to slow down). For
**permanent** errors (404, 410), mark the URL dead and don't retry. Set strict **timeouts** so a slow host
doesn't tie up a fetcher (Little's Law, m02). Robustness against bad/huge/malicious pages (size limits,
content-type checks) is essential at web scale.

---

### Q15. How is the URL frontier like the queues you've used elsewhere?
**A.** It's a **producer-consumer queue** (m08): fetchers/parsers are consumers that pull URLs and
**produce new URLs** back into it — exactly the Kafka/Celery pipeline shape. The differences are the
**politeness-aware scheduling** (per-host back-queues with delays) and **priority** layering on top of a
plain queue. So it reuses queue concepts (backpressure, durability, distribution) with crawler-specific
scheduling.

---

### Q16. A single host has millions of pages you want. How do you crawl it without DoS-ing it?
**A.** Its **per-host back-queue** paces you: one (or a few) request(s) at a time with a **crawl-delay**,
so you trickle through its millions of pages **slowly over time** while crawling *other* hosts in parallel
for throughput. You'd also respect its `robots.txt`/`Crawl-delay`, possibly scale the delay to its
response time, and use its **sitemap** if available (efficient URL discovery). You never speed up just
because one host is big — politeness is per-host regardless of how much you want from it.

---

### Q17. How do you avoid two crawler nodes crawling the same URL?
**A.** **Partition the work by host** (consistent hashing) so a given host (and thus its URLs) is owned by
**exactly one node** — no two nodes crawl the same host. Within that, the **dedup layer** (shared/sharded
Bloom filter of seen URLs) ensures a URL isn't re-added. So ownership-by-host prevents cross-node
duplication of *hosts*, and the seen-set prevents re-crawling *URLs*. (Coordinating a globally-consistent
seen-set is its own challenge — often a sharded/partitioned Bloom filter aligned with host sharding.)

---

### Q18. What ethical/legal considerations matter for a crawler?
**A.** **Respect `robots.txt`** and site terms; **don't overload** sites (politeness — excessive crawling
can be treated as a DoS/abuse); **identify** your crawler (User-Agent + contact); respect **copyright/PII**
in what you store/republish; and follow regional **data-protection** law (GDPR) for personal data. These
aren't just nice-to-haves — ignoring them gets you IP-banned, blocklisted, and potentially sued. A
production crawler treats politeness/compliance as first-class.

---

### Q19. How would you crawl JavaScript-heavy / SPA pages?
**A.** Plain HTTP fetch gets the initial HTML, but **SPAs render content client-side** (the HTML is mostly
empty + JS), so you need a **headless browser** (Puppeteer/Playwright/headless Chrome) to **execute the JS**
and capture the rendered DOM. It's **far more expensive** (CPU/memory/time per page) than a simple fetch,
so you use it **selectively** (only for pages that need it), maintain a separate heavier worker pool, and
cache aggressively. (This is the Selenium-style rendering I did at Famepilot, scaled.)

---

### Q20. What does this crawler feed, and how does it connect to the rest of the course?
**A.** It **feeds the search index (m16)**: crawled pages → tokenize/normalize → inverted index + BM25 (and
embeddings for semantic search). The **raw pages live in an object/KV store (m18)**. It reuses **consistent
hashing/sharding (m05)**, **Bloom filters/caching (m06)**, the **queue pipeline (m08)**, **per-host rate
limiting (m12)**, and **retries/timeouts (m10)**. So it's a capstone of the building blocks applied to a
crawl-the-web problem — and the front of the "crawl → index → search → RAG" data flow.
