# m17 — Solutions (Web Crawler)

> Check the *reasoning*. Recurring lessons: frontier = priority + per-host politeness; throughput from
> breadth not hammering; Bloom filter for URLs + SimHash for content; shard by host = load + politeness.

---

## Exercise 1 — Estimation
2B pages/month, 400 KB raw / 80 KB compressed.
1. **Crawl rate:** 2B ÷ (30 × 86,400 ≈ 2.6×10⁶ s) ≈ **~770 pages/sec** avg.
2. **Bandwidth:** 770 × 400 KB ≈ **~300 MB/s** download.
3. **Storage/month:** raw = 2B × 400 KB ≈ **~800 TB**; compressed = 2B × 80 KB ≈ **~160 TB** → object
   storage.
4. The **seen-URL set is billions of entries** — too large to store exactly in RAM → use a **Bloom
   filter** ("definitely not seen" / "maybe seen", tiny memory, rare false positives).

---

## Exercise 2 — The frontier
1. **What to crawl next (priority/importance/freshness)** and **at what rate per host (politeness)**.
2. **Front queues (priority):** URLs bucketed by importance/freshness so high-priority pages crawl first.
   **Back queues (politeness):** one queue per host with a next-allowed-crawl time, so each host is paced
   (one/few requests + delay). Front = *what first*, back = *how fast per host*.
3. **BFS** spreads the crawl across many hosts (broad coverage, parallelism, politeness-friendly); **DFS**
   would tunnel deep into one site (hammering it, poor coverage).

---

## Exercise 3 — Throughput/politeness tension
1. **Maximize global crawl throughput while never overloading any single host** (politeness).
2. **Per-host back-queues** pace each host independently (one request at a time + delay), so you stay
   polite *per host* while running **many hosts in parallel**.
3. From **breadth** — crawling **thousands of different hosts simultaneously**. You can't go fast against
   one host, but summed across millions of hosts the total throughput is high.

---

## Exercise 4 — Politeness
1. **(a)** obey **`robots.txt`** (allowed paths + crawl-delay); **(b)** **rate-limit per host** (one/few
   concurrent + delay); **(c)** send an honest **User-Agent** with contact info; **(d)** honor
   **`Retry-After`/429/503** (back off on overload). (Also: use sitemaps, respect nofollow.)
2. You effectively **DoS the sites** you crawl → get **IP-banned/blocklisted**, harm the sites, and risk
   **legal/abuse** consequences.
3. It's **rate limiting (m12) applied per domain** — the same token-bucket/delay idea, keyed by host.

---

## Exercise 5 — URL dedup
1. A **Bloom filter** stores billions of URLs in tiny memory with **no false negatives** (it never wrongly
   says "seen" for a truly-new URL... actually it can give false *positives*) — the acceptable downside is
   **rare false positives** → you occasionally **skip a genuinely new URL** (fine at web scale; you never
   re-crawl, you just miss a few).
2. **Normalization rules:** lowercase the host; strip default ports/fragments and tracking params; sort
   query params; resolve relative→absolute. They matter so different spellings of the **same** URL collapse
   to one (else you'd treat `Example.com/p?b=2&a=1` and `example.com/p?a=1&b=2` as different and crawl
   both).
3. URL dedup prevents **re-fetching the same URL**.

---

## Exercise 6 — Content dedup
1. URL dedup misses **the same content at different URLs** — e.g. a print version, an AMP version, mirrors,
   or session-ID URLs all serving identical content with different URL strings.
2. **Exact duplicates → a content hash** (MD5/SHA — identical hash = identical content). **Near-duplicates
   → SimHash/MinHash** (similar fingerprint = similar content, detected by small Hamming distance).
3. Content dedup prevents **re-storing/indexing the same content** reached via different URLs.

---

## Exercise 7 — Distribution & traps
1. **Shard the frontier by host** (consistent hashing, m05). Two benefits: **(a)** spreads load across
   nodes, and **(b)** **enforces per-host politeness without coordination** — since one node owns a host,
   it alone paces requests to it (no two nodes hammer the same site).
2. **Traps + defenses:** infinite calendar ("next month" forever) → **depth limit**; **session IDs** in
   URLs → **URL normalization** (strip them); deep **faceted-filter** combinations / link mazes →
   **per-host page caps + trap detection** (flag hosts emitting unbounded distinct URLs).
3. **DNS** is a bottleneck because resolving millions of hostnames is a network round trip that can
   dominate fetch time → **DNS cache** (cache resolutions per host with TTL) so most fetches skip the
   lookup.

---

## Exercise 8 — Freshness
1. **Adaptive re-crawl by estimated change frequency** — frequently-changing pages (news, forums)
   re-crawled often; static/archival pages rarely (informed by observed change rate + `Last-Modified`/
   `ETag`/sitemaps).
2. **New-page coverage** and **politeness** — all three compete for the **limited crawl budget** (the
   frontier can only fetch so much per host per unit time).
3. **Conditional requests:** `If-Modified-Since`/`If-None-Match` (ETag) → the server returns **304 Not
   Modified** with no body if unchanged (m06), so you skip re-downloading unchanged pages.

---

## Exercise 9 — Storage & feeding search
1. **Raw pages → object storage (S3/blob)** (large, numerous, write-once blobs); **metadata + link graph →
   a DB** (URL, status, content hash; who-links-to-whom). Split because bytes belong in cheap durable blob
   storage while queryable metadata/graph belong in a DB (m04 "split metadata from blobs").
2. Crawled pages are **tokenized/normalized and built into the inverted index + BM25** (and embeddings) of
   **m16** — the crawler is the **data source** for search.
3. **Flow:** crawler (frontier→fetch→parse→dedup→store) → **indexing pipeline** (tokenize/normalize/embed)
   → **inverted index + vector index** (m16) → **search/retrieval** (BM25 + vector + RRF + rerank) →
   **RAG** (retrieved chunks → LLM, m30).

---

## Exercise 10 — Your-systems tie-in 🏭
1. From scraping 30+ sources (Famepilot) and NSE/Google-News (Questimate): **(a)** **politeness/rate-
   limiting per host** (you respected source sites / API limits) and **(b)** **dedup + parsing pipelines**
   (you normalized/deduped scraped data) — scaled here to a distributed, polite, deduped fleet.
2. **Frontier = a producer-consumer queue** (your Kafka/Celery pipelines — workers pull URLs, produce new
   ones); **seen-set = a Bloom filter** (the same structure as m06 cache-penetration defense).
3. "Don't crawl/index the same content twice" is the **DCE dedup/suppression** instinct — stop redundant/
   already-processed items from going through the pipeline again (URL dedup + content dedup = two
   suppression gates).
