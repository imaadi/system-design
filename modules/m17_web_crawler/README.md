# Module 17 — Design a Web Crawler ⭐ 🏭

> Systematically download the web (to feed the search index from m16, or for archiving/ML data). It's a
> giant **producer-consumer pipeline** (m08) with three hard parts: the **URL frontier** (what to crawl
> next), **politeness** (don't DoS the sites you crawl — the constraint that *limits* throughput), and
> **deduplication** (URLs *and* near-duplicate content) at billions-of-pages scale. You've built real
> scrapers (Famepilot's 30+ sources, Questimate's NSE/Google-News), so the concepts are familiar — this
> scales them.

> **Format (Part 2):** worked m01 walkthrough; the **URL frontier + politeness + dedup** are the
> deep-dives.

---

## Step 1 — Requirements

**Functional (core):**
1. Start from **seed URLs**, **fetch** each page, **extract links**, and **follow** them (graph traversal
   of the web).
2. **Store** the fetched content (for indexing/archiving).
3. **Respect `robots.txt`** and crawl politely.
4. **Re-crawl** to keep content **fresh** (the web changes).
5. *(Stretch)* prioritization, content-type handling, JS rendering, focused/topic crawling.

**Non-functional (these drive it):**
- **Scalable:** billions of pages → **distributed** crawler fleet.
- **Polite:** **don't overload any host** (a crawler that hammers a site = a DoS → you get banned). This
  is the defining constraint.
- **Robust/fault-tolerant:** handle timeouts, bad HTML, **crawler traps**, malformed/malicious pages.
- **Fresh:** re-crawl based on how often pages change.
- **Efficient (no waste):** don't crawl the same URL/content twice → **dedup**.
- **Extensible:** plug in new content types / parsers.

> **The defining insight:** a crawler is "BFS over the web," but you **can't BFS as fast as possible** —
> **politeness caps your rate per host**, so the real problem is **maximizing global throughput subject to
> per-host rate limits**, with heavy **dedup** and **trap avoidance**.

---

## Step 2 — Estimation
Assume target **1 billion pages/month**, avg page **~500 KB** (raw HTML).
- **Crawl rate:** 1B ÷ (30 × 86,400 ≈ 2.6×10⁶ s) ≈ **~400 pages/sec** avg (Google does *far* more —
  state your target). Higher targets (10B/month → ~4k/s) just scale the fleet.
- **Bandwidth:** 400 × 500 KB ≈ **~200 MB/s** download (significant; the network is a real constraint).
- **Storage:** 1B × 500 KB = **~500 TB** raw (or ~100 TB compressed) → **object storage** (S3/blob).
- **Seen-URL set:** ~1B+ URLs → can't store exactly in memory → **Bloom filter** (Step 6).

> **The estimate frames it:** big bandwidth + PB-scale content (→ object storage), a huge seen-set (→
> Bloom filter), and a rate **bounded by politeness per host** (→ per-host frontier queues), all
> **distributed**.

---

## Step 3 — High-level design (the pipeline) ⭐

```
  seeds ─▶ ┌─────────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────┐
           │ URL Frontier│──▶│ Fetcher  │──▶│ Parser / │──▶│ URL extractor│──▶│  Dedup    │──┐
           │ (per-host   │   │ (HTTP +  │   │ content  │   │  + normalize │   │ (seen?    │  │
           │  queues,    │   │  robots) │   │ store)   │   └──────────────┘   │  Bloom)   │  │
           │  priority)  │   └──────────┘   └────┬─────┘                      └──────────┘  │
           └──────▲──────┘                       │ raw HTML                    new URLs ─────┘
                  └──────────── add new (unseen) URLs back to the frontier ◀──────────────────
                                       Content store (S3) · Link graph · DNS cache
```

**The crawl loop:**
1. **Frontier** hands out the next URL (respecting **politeness** + **priority**).
2. **Fetcher** resolves DNS, checks **robots.txt**, downloads the page (with timeouts/retries).
3. **Parser** extracts content (store raw HTML to **object storage**) and **outlinks**.
4. **URL extractor + normalizer** canonicalizes the links.
5. **Dedup** checks "seen this URL?" (Bloom filter) and "seen this *content*?" → **new** URLs go **back to
   the frontier**.
It's a **producer-consumer pipeline** (m08): the frontier is the queue; fetchers/parsers are workers;
new links are fed back in.

---

## Step 4 — Deep Dive: the URL Frontier ⭐⭐ (the heart)

The frontier decides **what to crawl next** and **at what rate** — it bakes in **politeness +
prioritization**. It's not a single queue; conceptually it has two layers (the **Mercator** design):
- **Front queues (priority):** URLs bucketed by **priority** (importance/freshness — e.g. a news
  homepage > a deep archive page). A prioritizer assigns priority; higher-priority URLs are crawled sooner.
- **Back queues (politeness):** **one queue per host**, each with a **next-allowed-crawl time** (enforcing
  per-host delay). A worker only pulls a host's URL when that host's delay has elapsed → **never hammers a
  single site** while keeping *many different hosts* busy in parallel.

```
  priority front-queues ──(route by host)──▶ per-host back-queues ──(respect crawl-delay)──▶ fetchers
   (WHAT to crawl first)                       (HOW FAST per host = politeness)
```

> **The core tension:** **maximize global throughput while staying polite per host.** You can crawl
> thousands of *different* hosts in parallel (high total throughput), but only **one request at a time
> (with a delay)** to *each* host. Per-host back-queues resolve it: parallel across hosts, paced within a
> host. (It's m12 **rate limiting applied per domain.**)

**BFS not DFS:** the frontier is a (priority) **queue** → **breadth-first**, which spreads the crawl
across many hosts and gets broad coverage; DFS would tunnel deep into one site (and violate politeness).

---

## Step 5 — Deep Dive: politeness ⭐

Politeness is **mandatory** — ignore it and you DoS sites, get IP-banned, and may face legal issues.
- **`robots.txt`:** fetch and **obey** each site's `robots.txt` (which paths are allowed, `Crawl-delay`).
  Cache it per host (re-fetch periodically).
- **Per-host rate limiting:** at most one (or a few) concurrent requests per host, with a **delay**
  between them (the back-queue's next-allowed time). Often scale the delay to the host's response time
  (slower server → crawl it slower).
- **Identify yourself:** a real **User-Agent** with contact info, so site owners can reach you.
- **Honor `nofollow`, sitemaps, and `Retry-After`** (back off if a site returns 429/503).

> **Why it caps throughput:** politeness means your rate to any single host is tiny, so total throughput
> comes from **breadth** (many hosts at once), not depth. This is *the* design constraint — a "fast"
> crawler isn't one that hammers one site; it's one that politely crawls millions of sites in parallel.

---

## Step 6 — Deep Dive: deduplication (two kinds) ⭐⭐

You must avoid crawling the same thing twice — at **two levels**:

### URL dedup — "have I seen this URL?"
With **billions** of URLs you can't keep an exact set in memory. Use a **Bloom filter** (m06): a
probabilistic set that answers "**definitely not seen**" or "**maybe seen**" with tiny memory (no false
negatives, rare false positives → occasionally you skip a new URL, acceptable). Plus **URL normalization**
(lowercase host, remove default ports, strip tracking params/fragments, resolve relative URLs, sort query
params) so `Example.com/p?b=2&a=1` and `example.com/p?a=1&b=2` are recognized as the **same** URL.

### Content dedup — "is this page a (near-)duplicate?"
The **same content appears at many URLs** (mirrors, print versions, session-ID URLs). Detect it:
- **Exact dup:** hash the content (e.g. MD5) → identical hash = exact duplicate, skip indexing.
- **Near-dup:** **SimHash / MinHash** produce similar fingerprints for *similar* content (small Hamming
  distance) → detect "almost the same page" (boilerplate-heavy near-duplicates) and avoid re-indexing.

> **Two dedup layers** because they catch different waste: URL dedup avoids **re-fetching** the same URL;
> content dedup avoids **re-storing/indexing** the same content reached via different URLs. (Your DCE
> **suppression/dedup** instinct again — don't process the same thing twice.)

---

## Step 7 — Distributed coordination, freshness, traps & bottlenecks

- **Distribute by host (consistent hashing, m05):** **shard the frontier by domain** so each crawler node
  **owns** certain hosts. This both **spreads load** *and* **naturally enforces per-host politeness** (one
  node controls the rate to a host → no cross-node coordination needed to avoid hammering it). Consistent
  hashing lets you add/remove crawler nodes with minimal reshuffling.
- **Freshness / re-crawl:** the web changes → **re-crawl adaptively** by **estimated change frequency**
  (a news homepage hourly; a 2010 archive page rarely). Re-add URLs to the frontier with a priority/
  schedule. (Politeness + freshness + coverage compete for the frontier's limited crawl budget.)
- **Crawler traps ⚠️:** **infinite URL spaces** — a calendar with "next month" forever, session IDs
  generating endless URL variants, deep faceted-filter combinations. Defenses: **URL normalization**,
  **depth limits**, **per-host page caps**, and **trap detection** (a host generating unbounded URLs).
- **DNS at scale:** resolving millions of hostnames is a bottleneck → a **DNS cache** (DNS resolution can
  dominate fetch time, m03).
- **Robustness:** timeouts, retries with backoff (m10), handle bad/huge/malicious pages, content-type
  limits.
- **Storage:** raw pages → **object storage** (S3, PB-scale); metadata + the **link graph** → a DB; the
  index built downstream (m16).
- **Observability (m10):** pages/sec, per-host politeness compliance, dedup/trap rates, frontier size,
  fetch error rates, freshness lag.

**Wrap-up:** *"A distributed producer-consumer pipeline. The **URL frontier** (priority front-queues +
**per-host back-queues** for politeness) decides what to crawl next; **fetchers** obey **robots.txt** and
per-host rate limits; **parsers** store raw HTML to **object storage** and extract+normalize outlinks;
**dedup** uses a **Bloom filter** for URLs and **SimHash** for near-duplicate content; new URLs feed back
in. It's **sharded by host (consistent hashing)** so each node owns hosts and enforces politeness without
coordination, **re-crawls adaptively** by change frequency, and avoids **crawler traps** via
normalization + depth/page limits. The core tension: maximize global throughput **subject to per-host
politeness** — speed comes from breadth, not hammering."*

---

## State-of-the-art & real-world notes 📚
- **Mercator** (the classic crawler design) introduced the **front-queue/back-queue frontier** (priority +
  politeness) — the canonical architecture above. **Googlebot** is the planet-scale version.
- **Apache Nutch** (open-source crawler, feeds Lucene/Solr) and **Common Crawl** (a public PB-scale web
  crawl on S3) are real implementations you can point to.
- **Politeness + robots.txt** is an industry norm (and increasingly legally significant); respectful
  crawling is non-negotiable.
- **SimHash** (Charikar/Google) is the standard near-duplicate detector for web content.

---

## From your systems 🏭 (direct experience)
- **You've built crawlers/scrapers:** **Famepilot** — scraping **30+ review sources** with BeautifulSoup/
  Selenium into data pipelines; **Questimate** — **NSE/BSE scrapers** + **Google News** scraping
  (feedparser/BeautifulSoup) for FinBERT sentiment. This module is those scaled to a distributed, polite,
  deduped fleet — you can speak from real fetch/parse/politeness experience.
- **Frontier = a queue (m08):** the URL frontier is a producer-consumer queue exactly like your **Kafka/
  Celery** pipelines — workers pull URLs, produce new ones back.
- **Dedup = Bloom filter + your suppression instinct:** the seen-URL Bloom filter is the same structure as
  m06 cache-penetration defense; the "don't process the same thing twice" is your DCE dedup/suppression
  reflex.
- **Scheduled re-crawl = Celery Beat:** adaptive re-crawl by change frequency is the Questimate
  Celery-Beat scheduling pattern.

---

## Key concepts (interview-ready)
- **A crawler is BFS over the web as a distributed producer-consumer pipeline** (m08), bounded by
  **politeness**, with heavy **dedup** and **trap avoidance**.
- **URL frontier (the heart):** **priority front-queues** (what to crawl first) + **per-host back-queues**
  (politeness — pace per host). **BFS** for broad coverage. Core tension: **maximize global throughput
  subject to per-host rate limits** (speed = breadth, not hammering one host).
- **Politeness is mandatory:** obey **`robots.txt`** + crawl-delay, **rate-limit per host** (m12 per
  domain), identify your User-Agent, honor 429/503 backoff. Ignoring it = DoS + bans.
- **Dedup at two levels:** **URL dedup** (huge seen-set → **Bloom filter**, + **URL normalization**) and
  **content dedup** (exact = hash; **near-dup = SimHash/MinHash** — same content at many URLs).
- **Distribute by host (consistent hashing, m05)** → spreads load *and* enforces politeness without
  coordination.
- **Freshness:** **adaptive re-crawl by change frequency**; competes with coverage for crawl budget.
- **Crawler traps** (infinite URL spaces) → URL normalization + depth/page limits + trap detection. **DNS
  cache** (resolution is a bottleneck). **Raw pages → object storage**; link graph → DB; index downstream
  (m16).

---

## Go deeper (reading)
- **The Mercator paper ("A scalable, extensible web crawler")** — the front/back-queue frontier design.
- **"Introduction to Information Retrieval" (Manning et al.) — the web-crawling chapter** — frontier,
  politeness, dedup, traps (free online).
- **Apache Nutch** docs and **Common Crawl** — real crawler + public dataset.
- **SimHash (Charikar) / "Detecting near-duplicates for web crawling" (Google)** — near-dup detection.
- Revisit **m05** (consistent hashing/sharding), **m06** (Bloom filters/hot keys), **m08** (queue
  pipeline), **m12** (per-host rate limiting), **m16** (the search index this feeds).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m17_web_crawler/`](../../solutions/m17_web_crawler/README.md).

When you've done the exercises, say **"Module 18"** to design an *Object/Blob Store + Key-Value Store
(S3 / DynamoDB)* — where consistent hashing, quorums, and replication (m05) become the *whole* design, and
where the raw pages from this crawler actually live.
