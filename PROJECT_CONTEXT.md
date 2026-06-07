# PROJECT_CONTEXT — System Design Learning System (handoff & memory)

> **Purpose:** if an AI assistant (or future-me) starts fresh on this repo, read this first. It
> captures who the learner is, what we're building, the conventions, and what's done vs pending —
> so work continues without re-deriving context.

---

## 0. TL;DR
A **5+ year Python/Django backend & platform engineer** (Aaditya Kumar Dixit) is preparing for
**AI/ML/LLM-engineering interviews** and wants to master **system design** — basic to advanced,
deeply, with all the cross-questions ("if-and-buts") an interviewer can throw. We're building a
**36-module curriculum** (`modules/` + separate `solutions/`) in 5 parts: core SD → building
blocks → classic problems → ML system design → LLM system design → capstone/interview.

This mirrors his existing **AI/ML notes** (in `../ai_ml_llm/`, 14 modules, done) in spirit: deep,
practical, interview-ready, honest, with practice + separate solutions.

---

## 1. Who the learner is (constraints & preferences)
- **Background:** 5+ yrs. Python, Django/DRF, Flask, FastAPI. REST microservices, rule-based
  systems. **PostgreSQL, MongoDB, Redis, MySQL.** Celery, **Kafka, RabbitMQ**. Docker, **Kubernetes /
  OpenShift, ArgoCD**, CI/CD (GitHub Actions), nginx, uWSGI, AWS (working knowledge). Strong on
  backend + ops; this is a real advantage for system design.
- **Has shipped real systems at scale** (use these as running examples — his biggest edge):
  - **Questimate** (Sabertooth): **4 Django/DRF microservices** (qsocial, risk_factor,
    daily_return, stock_sentiment), decoupled by REST, **Postgres + Redis + MongoDB**, **Celery
    Beat** scheduled pipelines, **Redis-backed JWT with instant revocation**, parallel batch
    precompute into snapshot tables, FinBERT NLP. Quant analytics (factor risk model, VaR,
    optimization, Monte-Carlo, BDT bond pricing). Deployed on Linode behind uWSGI/nginx.
  - **DCE / Digital Claims Examiner** (Carelon): **~1M+ US healthcare claims/day** auto-adjudicated
    (DENY/BYPASS/ROUTE/UPDATE). ML models + **rule layers** (exclusions at entry, rule-based
    models in the middle, suppressions at the end). **MongoDB** (claims store), **Redis** (real-time
    reference data), **Kafka** (claims streaming), **Snowflake/Snowpark** (ETL), **MinIO** (model
    artifacts), **AWS EKS / OpenShift, ArgoCD**. Migrations: ENSO→K8s, OpenShift→EKS, Azure→AWS,
    Lambda/Glue→Snowpark. Huge cost-savings story ($675K→$50K/mo; team 20→4). LightGBM/RF, LSA.
- **Goal:** AI/ML/LLM, MLOps, or backend-on-an-AI-product roles. So interviews = **classic
  distributed-systems design + ML system design + LLM system design**. Curriculum is **balanced**
  across all three (his explicit choice).
- **How he likes to learn (KEEP DOING THIS):**
  1. **Depth.** Explain every term fully — what/why/when/when-NOT/the trade-off. No hand-waving.
  2. **Real & practical examples**, including **his own systems** (Questimate/DCE callouts).
  3. **Cover all cross-questions / "if-and-buts"** — the follow-ups an interviewer asks.
  4. **Practice problems with solutions in a SEPARATE tree** (`solutions/`) so he practices honestly.
  5. **Diagrams/flowcharts** (ASCII in the .md files) where they help.
  6. **One module at a time**, finish + verify, then say "Module N" to continue.
  7. Quote **numbers** (the senior signal). Reusable estimation numbers live in m02.

---

## 2. What we're building — the 36-module curriculum
See `README.md` for the full table. Five parts:
- **Part 0 (m01–m02):** Interview framework + core fundamentals (the vocabulary).
- **Part 1 (m03–m10):** Building blocks — networking, databases, replication/sharding, caching,
  load balancing, messaging, distributed-systems theory, reliability/observability.
- **Part 2 (m11–m21):** Classic problems — URL shortener, rate limiter, news feed, chat,
  notifications, typeahead, crawler, object/KV store, video, proximity/ride-hailing, payments.
- **Part 3 (m22–m28):** ML system design — framework, feature pipelines/stores, model serving,
  recsys, feed ranking/ads, fraud/anomaly, ML platform/MLOps.
- **Part 4 (m29–m34):** LLM system design — serving/inference, production RAG, chatbot, agents,
  LLM gateway/platform, eval & safety.
- **Part 5 (m35–m36):** His systems as case studies; mock-interview drills + cheat-sheets.

---

## 3. Conventions (match these in every new module)
```
modules/mNN_topic/
  README.md           ← concept deep-dive. Structure: intro → "Key terms in depth" →
                         diagrams → real-world examples → "From your systems" callout →
                         "Key concepts (interview-ready)" → "How to go deeper / improve" →
                         pointers to CROSS_QUESTIONS + EXERCISES.
  CROSS_QUESTIONS.md  ← 15–40 follow-up Qs ("if-and-buts") each with a tight model answer.
  EXERCISES.md        ← practice problems (design tasks / estimations / "spot the flaw").
solutions/mNN_topic/
  README.md           ← worked solutions to that module's EXERCISES.
```
**Style:** ASCII diagrams; tables for trade-offs; **bold** the term being defined; every claim
that can have a number gets one; honesty about trade-offs (there is no "best", only "best for X").
Use GitHub-flavored markdown. Keep the "say Module N to continue" footer.

For **design-problem modules** (Part 2+), the README itself contains the **step-by-step worked
design** (requirements → estimation → API → data model → high-level → deep-dives → bottlenecks →
trade-offs), and CROSS_QUESTIONS pushes the follow-ups.

---

## 4. Status — done vs pending
**DONE ✅**
- `README.md` (roadmap + how-to-use), `PROJECT_CONTEXT.md` (this file).
- **m01 — Interview Framework** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m02 — Core Fundamentals** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m03 — Networking & Communication** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m04 — Databases & Storage** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m05 — Replication, Partitioning & Sharding** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m06 — Caching & CDNs** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m07 — Load Balancing & Proxies** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m08 — Messaging, Queues & Stream Processing** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m09 — Distributed Systems Theory** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m10 — Reliability, Observability & Operations** (README + CROSS_QUESTIONS + EXERCISES + solutions).
  → **PART 0 + PART 1 COMPLETE (m01–m10): foundations + all building blocks.**
- **m11 — URL Shortener (TinyURL)** (README + CROSS_QUESTIONS + EXERCISES + solutions). *Part 2 begins —
  design-problem modules: the README is a full worked m01-framework walkthrough; CROSS_QUESTIONS drills
  follow-ups; EXERCISES has extensions/variations.*
- **m12 — Rate Limiter** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m13 — News Feed / Twitter Timeline** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m14 — Chat System (WhatsApp/Slack)** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m15 — Notification System** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m16 — Typeahead / Autocomplete + Search** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m17 — Web Crawler** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m18 — Object Store + Key-Value Store (S3 / DynamoDB)** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m19 — Video Streaming (YouTube/Netflix)** (README + CROSS_QUESTIONS + EXERCISES + solutions).
- **m20 — Proximity / Ride-hailing (Uber/Yelp/Maps)** (README + CROSS_QUESTIONS + EXERCISES + solutions).

**PENDING ⬜**
- m21 (Payments/Ledger — closes Part 2), then m22–m36 (Part 3 ML, Part 4 LLM, Part 5 capstone). Build **on
  the learner's signal** ("Module 21", etc.), one at a time, in depth.

---

## 5. How to continue
- Build the next module in full (all 4 files) when he says "Module N".
- Keep weaving in **Questimate/DCE** examples — they're the interview differentiator.
- Don't dump everything at once; depth + one module at a time is the agreed rhythm.
- The context source files (resume + Questimate/DCE write-ups) are kept **outside the repo** at
  `../for_context_purpose/` (gitignored, not published). Their lessons live on in the 🏭 callouts.
