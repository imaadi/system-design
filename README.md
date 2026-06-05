# System Design — From Fundamentals to Architect (for AI/ML/LLM Interviews)

> A deep, practical, interview-grade curriculum. Built for **Aaditya** — 5+ yrs Python/Django
> backend & platform engineer (Questimate quant platform, Carelon DCE 1M-claims/day pipeline),
> now moving into **AI/ML/LLM** roles. The goal isn't to memorize answers; it's to **think like an
> architect** and defend every decision under cross-examination.

---

## Why this exists & how it's different

Most "system design prep" is a list of YouTube-able answers. This isn't. Three principles:

1. **Depth over coverage.** Every term is explained until you *get it* — what it is, why it
   exists, when to use it, when **not** to, and the trade-off it hides. If you can't draw it and
   argue both sides, you don't know it yet.
2. **Real systems, not toy ones.** We use famous designs (Twitter, Uber, S3) **and your own
   shipped systems** (Questimate's 4-service mesh, DCE's 1M-claims/day ML pipeline). Your real
   work is your biggest edge — most candidates have never run anything at scale; you have.
3. **Cover the cross-questions ("if-and-buts").** Interviewers don't grade your first answer; they
   grade how you handle *"what if traffic 100×?", "what if this node dies mid-write?", "why not
   SQL?"* Every module ships a **`CROSS_QUESTIONS.md`** bank that drills exactly these.

---

## How to use this (the working rhythm)

- **One module at a time.** Read the `README.md` (concepts), then test yourself on
  `CROSS_QUESTIONS.md` **out loud**, then do `EXERCISES.md` *before* peeking at `solutions/`.
- **Draw everything.** Keep a notebook (or [excalidraw.com](https://excalidraw.com)). If you can't
  sketch it from memory, re-read.
- **Say a number.** The #1 senior signal is quantifying. We give you the estimation tools and
  reusable numbers; use them every time.
- **Finish + verify, then advance.** When you've done a module's exercises, say **"Module N"** to
  get the next one built. (Same flow as your AI/ML notes.)
- **Anchor to your experience.** Each module has a *"From your systems"* callout linking the
  concept to Questimate/DCE. Rehearse those — they become interview stories.

### Folder layout
```
system_design/
├── README.md                      ← this file (roadmap + how-to-use)
├── PROJECT_CONTEXT.md             ← handoff/memory: who you are, what's built, conventions
├── modules/
│   └── mNN_topic/
│       ├── README.md              ← concept deep-dive (diagrams, real examples, "from your systems")
│       ├── CROSS_QUESTIONS.md     ← the "if-and-buts" / follow-up bank (Q + model answer)
│       └── EXERCISES.md           ← practice problems (try before solutions!)
├── solutions/
│   └── mNN_topic/README.md        ← worked solutions, kept separate so you practice honestly
└── .gitignore                     ← excludes OS junk + personal context from the repo
```
> Personal context (resume, Questimate/DCE write-ups) is kept **outside this repo** (gitignored), so
> nothing private is published. The 🏭 "from your systems" callouts in the modules carry forward the
> relevant lessons.

---

## The curriculum — 36 modules in 5 parts

Legend: ⭐ = especially high-value for AI/ML/LLM interviews · 🏭 = maps directly to your shipped work

### PART 0 — Foundations of the craft
| # | Module | What it builds |
|---|--------|----------------|
| **m01** | **The Interview Framework & Thinking Like an Architect** ⭐ | How to *drive* a design interview: requirements → estimation → API → data → high-level → deep-dive → bottlenecks. Communication, drawing, red flags. |
| **m02** | **Core Fundamentals** ⭐ | The whole vocabulary: scalability, latency vs throughput, availability/SLA, consistency models, CAP & PACELC, vertical vs horizontal, statelessness, the back-of-envelope numbers. |

### PART 1 — The building blocks (the LEGO bricks of every design)
| # | Module | What it builds |
|---|--------|----------------|
| **m03** | **Networking & Communication** | DNS, TCP/UDP, HTTP/1.1·2·3, REST vs gRPC vs GraphQL, WebSockets/SSE/long-polling, API gateway, idempotent HTTP. |
| **m04** | **Databases & Storage** | SQL vs NoSQL (and *which* NoSQL), indexing, B-tree vs LSM, transactions, ACID, isolation levels, schema design. |
| **m05** | **Replication, Partitioning & Sharding** ⭐ | Leader-follower vs multi-leader vs leaderless, quorums, sharding strategies, **consistent hashing**, rebalancing, hot shards. |
| **m06** | **Caching & CDNs** ⭐ | Cache-aside/write-through/write-back, eviction (LRU/LFU/TTL), Redis vs Memcached, CDNs, invalidation, thundering herd, hot keys. |
| **m07** | **Load Balancing & Proxies** | L4 vs L7, algorithms, reverse proxy vs forward proxy, health checks, sticky sessions, service mesh basics. |
| **m08** | **Messaging, Queues & Stream Processing** 🏭 | Queues vs logs, RabbitMQ vs Kafka, pub/sub, delivery guarantees, **idempotency**, ordering, backpressure, outbox/CDC. |
| **m09** | **Distributed Systems Theory** | Consensus (Raft/Paxos), leader election, logical/vector clocks, 2PC vs **Saga**, CRDTs, the "8 fallacies." |
| **m10** | **Reliability, Observability & Operations** | SLA/SLO/SLI & error budgets, retries/backoff/jitter, circuit breakers, bulkheads, graceful degradation, metrics/logs/traces. |

### PART 2 — Classic design problems (apply the bricks)
| # | Module | Teaches |
|---|--------|---------|
| **m11** | **URL Shortener (TinyURL)** | The "hello world": estimation, key generation, KV store, read-heavy caching, redirects. |
| **m12** | **Rate Limiter** | Token/leaky bucket, sliding window, distributed counters, where to enforce. |
| **m13** | **News Feed / Twitter Timeline** ⭐ | Fan-out on write vs read, the celebrity problem, feed ranking, hybrid fan-out. |
| **m14** | **Chat System (WhatsApp/Slack)** | WebSockets, presence, delivery & read receipts, message ordering, group chat. |
| **m15** | **Notification System** 🏭 | Multi-channel (push/SMS/email), fan-out, dedup, rate-limit, priority, retries. |
| **m16** | **Typeahead / Autocomplete + Search** | Tries, prefix sharding, ranking, the search-indexing pipeline (inverted index). |
| **m17** | **Web Crawler** | BFS frontier, politeness, dedup, dist. coordination, freshness. |
| **m18** | **Object Store + Key-Value Store (S3 / DynamoDB)** ⭐ | Blob storage, metadata, replication, consistent hashing, quorum reads/writes. |
| **m19** | **Video Streaming (YouTube/Netflix)** | Upload→transcode pipeline, CDN, adaptive bitrate, metadata at scale. |
| **m20** | **Proximity / Ride-hailing (Uber, Yelp, Maps)** | Geospatial indexing (geohash/quadtree/H3), real-time matching, location updates. |
| **m21** | **Payments & Ledger** 🏭 | Idempotency, exactly-once effects, double-entry ledger, consistency, reconciliation. |

### PART 3 — ML system design ⭐
| # | Module | Teaches |
|---|--------|---------|
| **m22** | **ML System Design Framework** ⭐ | The ML-specific method: framing as ML, data, features, model, **offline vs online eval**, serving, feedback loops. |
| **m23** | **Data & Feature Pipelines / Feature Stores** ⭐ | Batch vs streaming features, **training-serving skew**, point-in-time correctness, the feature store. |
| **m24** | **Model Serving & Inference at Scale** ⭐ | Online vs batch, latency budgets, dynamic batching, model registry, **shadow/canary/A-B**, rollback. |
| **m25** | **Recommendation Systems** ⭐ | Candidate generation → ranking → re-ranking, **two-tower retrieval**, embeddings, cold start. |
| **m26** | **Feed Ranking & Ads / CTR Prediction** ⭐ | Learning-to-rank, CTR models, real-time features, the ML version of the news feed. |
| **m27** | **Fraud & Anomaly Detection (real-time scoring)** 🏭⭐ | Low-latency scoring, class imbalance, rules+ML hybrid (your DCE world), feedback delay. |
| **m28** | **ML Platform / MLOps End-to-End** ⭐ | Training infra, experiment tracking, model registry, **drift & monitoring**, retraining triggers, governance. |

### PART 4 — LLM system design ⭐ (the 2026 differentiator)
| # | Module | Teaches |
|---|--------|---------|
| **m29** | **LLM Serving & Inference** ⭐ | Tokens, **KV cache**, continuous batching, vLLM/TGI, quantization, latency (TTFT/TPOT) vs throughput, cost math. |
| **m30** | **Production RAG at Scale** ⭐ | Ingestion→chunk→embed→vector DB→hybrid→rerank→generate, eval (faithfulness/recall), guardrails, multi-tenancy. |
| **m31** | **LLM Chatbot / Assistant** ⭐ | Conversation memory, context-window management, streaming, moderation, caching, session state. |
| **m32** | **Agentic Systems** ⭐ | Tool/function calling, orchestration, planning, multi-agent, reliability & loop control, cost ceilings. |
| **m33** | **LLM Gateway / Platform** ⭐ | Model routing, semantic caching, rate limiting & quotas, fallbacks, cost control, observability, prompt mgmt. |
| **m34** | **LLM Evaluation & Safety in Production** ⭐ | Eval harnesses, LLM-as-judge, guardrails, **prompt injection**, PII/PHI handling, regression gates. |

### PART 5 — Capstone & interview
| # | Module | Teaches |
|---|--------|---------|
| **m35** | **Your Systems as Case Studies** 🏭⭐ | Turn Questimate & DCE into polished, interview-grade system-design walkthroughs with numbers and trade-offs. |
| **m36** | **Mock Interview Drills + Cheat-Sheets + Rapid-Fire** ⭐ | Timed mocks, the metrics cheat-sheet, rapid-fire concept Q&A, a personal "story bank." |

---

## Reusable numbers you'll quote everywhere (preview — full version in m02)

| Thing | Number to remember |
|---|---|
| L1 cache ref | ~1 ns |
| Main memory (RAM) ref | ~100 ns |
| Read 1 MB from RAM | ~3–25 µs |
| SSD random read | ~16–100 µs (NVMe → SATA) |
| Round trip within a datacenter | ~0.5 ms |
| Read 1 MB from SSD | ~0.3–1 ms |
| Disk (HDD) seek | ~10 ms |
| Round trip CA ↔ Netherlands | ~150 ms |
| 1 day | ~86,400 s ≈ **10⁵ s** |
| "1 million/day" | ≈ **12 writes/sec** average (×~10 for peak) |

> **The one mental model:** *Memory is fast, disk is slow, network is slower, and a different
> continent is slowest. Every design is about keeping hot data close and doing slow things async.*

---

## Status
- ✅ Roadmap + `PROJECT_CONTEXT.md`
- ✅ **m01 — Interview Framework** (built)
- ✅ **m02 — Core Fundamentals** (built)
- ✅ **m03 — Networking & Communication** (built)
- ⬜ m04–m36 (say **"Module 4"** to build the next one)

> Built one module at a time, in depth, with cross-questions and exercises — the way the AI/ML
> notes were built. Quality over speed.
