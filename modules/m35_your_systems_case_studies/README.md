# Module 35 — Your Systems as Case Studies ⭐⭐ 🏭 — *Part 5: your biggest edge*

> Most candidates **fake** scale ("imagine 100M users…"). **You've operated it** — a **~1M-claims/day ML
> pipeline (DCE)** and a **4-service quant platform (Questimate)**. This module turns your real work into
> **interview-grade system-design walkthroughs** you can *walk in and own*: the 90-second pitch, the
> architecture, the **decisions + trade-offs**, the **numbers**, the **"hardest part" stories**, and the
> cross-questions you'll face. When an interviewer asks *"have you actually built this?"*, you say **"yes —
> on a system processing a million claims a day, I…"** — and that single sentence separates you from the
> field.

> **Format (Part 5):** the two case studies as walkthroughs (here), the cross-questions on *your own*
> systems (CROSS_QUESTIONS), rehearsal drills (EXERCISES), and a **story bank** mapping interview prompts →
> your stories (solutions).

---

## 0. Why this is your edge (and how to wield it)
- **Lived scale + correctness + ops** beats memorized designs: you've made real trade-offs under cost/
  scale/compliance pressure, run migrations, and operated on-call. Interviewers *probe* memorized answers
  and *reward* lived ones.
- **Speak in the curriculum's language.** You did the work; now **name the patterns** — "that's a **rules+ML
  hybrid decision engine** (m27)," "we **precomputed into snapshot tables** (m13/m23 materialized view),"
  "**Redis-backed JWT** = stateless verify + instant revocation (m02/m03)." Naming patterns = senior.
- **Quote a number, always** — 1M claims/day, $675K→$50K/mo, team 20→4, 4 services, ~90 APIs, the migration
  savings. (The market's #1 senior signal.)
- **Two reusable threads to pull:** *correctness at scale* (DCE) and *clean architecture + lived trade-offs*
  (Questimate). Decide which to lead with based on the role (ML/correctness → DCE; backend/architecture →
  Questimate).

---

## 1. DCE (Digital Claims Examiner) — a 1M-claims/day ML pipeline 🏭⭐⭐

### The 90-second pitch (memorize the shape, not the words)
> *"I work on **DCE**, an AI pipeline that **auto-adjudicates ~1M US healthcare claims a day** —
> recommending **DENY / BYPASS / ROUTE / UPDATE** for each. It's a **rules + ML hybrid**: **exclusion
> rules** filter claims at entry, **ML models + rule-based models** produce recommendations together, and
> **suppression rules** at the exit block incorrect recommendations before they ship. Data flows through
> **Kafka**; claims live in **MongoDB**, hot reference/eligibility data in **Redis**, ETL in **Snowflake**,
> model artifacts in **MinIO**; it runs on **Kubernetes/EKS**. The thing I'm proudest of: it prioritizes
> **correctness + auditability** — a wrong auto-deny has real consequences, so the suppression layer is a
> safety net — and we cut run cost from **~$675K to ~$50K/month** and the team from **20 to 4** while
> holding throughput, via platform migrations I worked on."*
Then offer a thread: *"Happy to go deep on the suppression layer, the feature pipeline, or the migrations."*

### As a system-design walkthrough (the m01 framework)
- **Requirements/NFRs:** FR = recommend an action per claim; **dominant NFRs = correctness + auditability**
  (CP-leaning, m02/m21) + throughput at scale + explainability (regulated). *Down-beats-wrong* (m21).
- **Estimation:** ~1M/day ≈ **~12/sec average**, **bursty** around payer cycles → size for peak (m02).
  The challenge is **per-claim correctness + compute**, not raw QPS.
- **Data model → storage by access pattern (m04 polyglot):** **MongoDB** = claims (nested, variable JSON,
  read/written per-claim); **Redis** = hot eligibility/auth/reference data (sub-ms lookup at scoring);
  **Snowflake** = ETL/analytics (columnar OLAP); **MinIO** = model blobs.
- **High-level (the pipeline):** ingest (Kafka, m08 async) → **claim-preprocessing → 100+ structured
  features** (m23 feature engineering + training-serving consistency) → **exclusions → ML+rule models →
  suppressions** (m27 rules+ML hybrid + guardrails) → decision + audit (Mongo).
- **Deep-dive options (pick one):** the **suppression layer** (output guardrail, m27/m34), the **feature/
  preprocessing pipeline** (m23, point-in-time eligibility), the **eligibility model** you built, or the
  **migrations** (ENSO→K8s, OpenShift→EKS, Azure→AWS, Lambda/Glue→Snowpark — m10 safe deploy + cost).
- **Bottlenecks/evolution (step 7, lived):** the migrations + **parity-validated `compare_results.py`**
  (= reconciliation, m21) + reprocessing scripts (= replay, m08) + drift/monitoring (m28).

### Concepts it demonstrates (name these → map to modules)
rules+ML hybrid + 3-way decision (m27) · output guardrails/suppression (m34) · polyglot persistence (m04) ·
Kafka async + idempotent reprocessing/replay (m08) · feature engineering + consistency (m22/m23) ·
real-time feature lookup via Redis (m23) · CP/correctness-first (m02/m21) · reconciliation/parity (m21) ·
safe migrations + cost-as-SLO (m10/m28) · MLOps ops at scale (m28).

### "Hardest part / a bug you fixed" stories (have 2 ready)
- **Correctness safety net:** designing **suppressions** so a model's incorrect recommendation can't ship
  (block at the exit) — "in a domain where a wrong auto-deny has consequences, I treated correctness as the
  #1 NFR and built a guardrail layer, with everything logged to Mongo for audit."
- **A migration done safely:** validating **claim-processing parity** across old/new environments with
  comparison scripts before cutover — "I assumed something would diverge and verified against a source of
  truth" (reconciliation thinking, m21).

---

## 2. Questimate — a 4-service quant platform 🏭⭐⭐

### The 90-second pitch
> *"I **architected and built the entire backend from scratch** for **Questimate**, a quantitative
> investing platform — **4 Dockerized Django/DRF microservices** (qsocial = portfolio/risk analytics,
> risk_factor, daily_return, stock_sentiment), decoupled by **REST APIs** with env-injected URLs, backed by
> **PostgreSQL + Redis + MongoDB** and orchestrated with **Celery Beat**. qsocial alone is ~40 models, ~90
> APIs: a **multi-factor risk model** (EWMA covariance + Ledoit-Wolf shrinkage), VaR/risk decomposition,
> SLSQP portfolio optimization, Brinson attribution, Monte-Carlo, and a **Black-Derman-Toy callable-bond
> pricer**. Two design decisions I'm proud of: **Redis-backed JWT** — stateless verification plus
> **instant revocation** by deleting the key — and **precomputing risk into snapshot tables in parallel**
> so the API is an O(1) read instead of a heavy recompute."*
Then: *"Happy to go deep on the microservice mesh, the auth, the precompute, or the BDT pricer."*

### As a system-design walkthrough (the m01 framework)
- **Requirements/NFRs:** FR = portfolio/risk analytics + real-time trading; **dominant NFR = fast reads on
  expensive analytics** → drove the **precompute-into-snapshots** architecture (m13/m23 materialized view).
- **High-level (the mesh):** clients (JWT bearer) → **qsocial (API gateway-ish)** → REST to **risk_factor /
  daily_return / stock_sentiment** (east-west, m03) → Postgres/Redis/Mongo; **Celery Beat** runs the daily
  pipeline (price refresh → returns → **risk-decomposition precompute, 4 parallel threads** → snapshots).
- **Deep-dive options:** **Redis-backed JWT** (stateless + instant revocation — m02/m03 options/trade-off
  story), **snapshot precompute** (do expensive compute off the request path → O(1) reads, m13/m23),
  **microservice decoupling** (each analytical concern independently deployable/backfillable — m03/m08),
  or the **BDT callable-bond pricer** (a genuinely hard algorithm — calibrate a binomial short-rate tree to
  the zero curve via bisection, backward-induct with an embedded call).
- **Trade-offs to articulate:** **REST between services** (simple, language-agnostic, debuggable) **vs
  gRPC** (you'd move hot paths to gRPC if latency/throughput demanded — m03); **stateless app tier** (JWT/
  session in Redis → horizontal scale — m02/m07); **eventual consistency** for analytics snapshots vs
  **strong** for auth revocation (m02 — pick the model per feature).

### Concepts it demonstrates
microservices + REST data mesh / east-west (m03) · API-gateway-ish entry + JWT-at-edge (m03) · stateless
app tier + Redis session (m02/m07) · **precompute/materialized view for read latency** (m13/m23) · polyglot
persistence (m04) · async scheduled pipelines + parallel batch compute (m08) · consistency-per-feature
(m02) · the throughput-for-latency trade (m02).

### "Hardest part" story
- **The BDT callable-loan pricer:** "calibrating a recombining short-rate tree to the live zero curve via
  bisection, then backward-inducting with an embedded issuer call (`V = min(continuation, call_price)`) to
  separate the call-option value — genuinely advanced fixed-income engineering I built from the math up."

---

## 3. How to use these in an interview (the meta-skill)
- **Bring them up proactively** when asked "tell me about a system you built" / "a hard technical problem" /
  "a trade-off you made" — and **weave them in** during a design problem ("I actually did this on DCE…").
- **Match the system to the role:** ML/data/correctness role → **lead with DCE**; backend/architecture/
  fintech → **lead with Questimate**. Have both ready; pivot on the interviewer's interest.
- **Drive the walkthrough** (m01): pitch (90s) → "want the architecture or a deep-dive?" → go 3 levels deep
  on **one** component → name the **trade-off + the number**. Don't monologue; check in.
- **Turn "have you done X?" into a yes:** almost every curriculum concept maps to your work (caching,
  async, sharding-awareness, idempotency, monitoring, safe deploys, ML serving, RAG, agents). Say *"yes —
  on [system] I…"*
- **Handle "what would you do differently?"** (shows seniority): DCE → more real-time scoring / a formal
  feature store (m23) / richer online eval (m22). Questimate → harden prod hygiene (the known DEBUG/
  SECRET_KEY items), gRPC on hot internal paths (m03), a managed vector DB if you added semantic search.
- **Bridge to AI/ML/LLM:** you *also* built **InsightDesk** (RAG + agents + eval + serving + MLOps) — use
  it for LLM/ML-platform roles (m28–m34), and use DCE/Questimate for the *production-scale + correctness*
  credibility underneath it.

---

## 4. Your metrics cheat-sheet (quote one every time)
| System | Number | One-liner |
|---|---|---|
| DCE scale | **~1M+ claims/day** auto-adjudicated | lived production ML at scale |
| DCE impact | run cost **~$675K → ~$50K/mo**; team **20 → 4** | cost/ops maturity, held throughput |
| DCE ETL | Snowpark migration **~60% ETL cost cut** ($1.2M→$400K) | platform optimization |
| DCE infra | OpenShift→EKS **$75K → $20K/mo** | infra migration savings |
| DCE quality | **>85% test coverage**, 100+ tests/release | production rigor |
| Questimate | **4 microservices**, qsocial **~40 models / ~90 APIs** | full-stack architecture from scratch |
| Questimate auth | **Redis-JWT**: stateless verify **+ instant revocation** | best-of-both auth design |
| Questimate perf | risk decomposition **precomputed in parallel → O(1) API reads** | latency via precompute |
| InsightDesk (AI) | RAG **faithfulness ~0.96, context recall ~0.72**, CI-gated | measured, production-grade ML |

---

## 5. Key concepts (interview-ready)
- **Your real systems are your biggest edge** — lived scale + correctness + ops beats memorized designs.
  **Name the patterns** (rules+ML hybrid, materialized view, Redis-JWT, polyglot, async, reconciliation)
  and **quote a number** every time.
- **DCE = a correctness-first, rules+ML hybrid, 1M-claims/day pipeline** (m27/m34 + m04/m08/m22/m23 + m10/
  m28 + the migration/cost story). Lead for ML/correctness roles.
- **Questimate = a clean 4-service architecture with lived trade-offs** (m03 mesh + m02/m07 stateless+Redis-
  JWT + m13/m23 precompute + m04 polyglot + the BDT algorithm). Lead for backend/architecture roles.
- **Drive each as the m01 framework:** 90s pitch → offer a thread → one 3-level deep-dive → trade-off + number.
- **Turn "have you done X?" into "yes, on [system] I…"** — almost every concept maps to your work. Have a
  **"hardest part" story** and a **"what I'd do differently"** for each.
- **Bridge:** DCE/Questimate = production credibility; **InsightDesk** = the modern AI/ML/LLM stack on top.

---

## 6. Go deeper (your own materials)
- **`for_context_purpose/`** (now at `../for_context_purpose`): your **DCE work summary**, **Questimate
  technical overview**, and resume — the source facts for these walkthroughs. Re-read them and tighten each
  pitch.
- **Your AI/ML notes (`../ai_ml_llm/` + `m14_capstone/MOCK_INTERVIEW.md`)** — the InsightDesk pitch +
  metrics cheat-sheet for the AI/ML/LLM half.
- This whole curriculum (**m01–m34**) is your **pattern vocabulary** — re-skim the "From your systems 🏭"
  callouts; they already map your work to each concept.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md) (the cross-questions you'll face on *your own* systems),
then do [`EXERCISES.md`](EXERCISES.md) (rehearse the pitches + walkthroughs) and check the **story bank** in
[`solutions/m35_your_systems_case_studies/`](../../solutions/m35_your_systems_case_studies/README.md).

When you've rehearsed these, say **"Module 36"** to build the **final** module — *Mock Interview Drills +
Cheat-Sheets + Rapid-Fire* — which completes the 36-module curriculum.
