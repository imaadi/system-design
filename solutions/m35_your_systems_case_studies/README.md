# m35 — The Story Bank (model answers for your systems)

> Map of **interview prompt → which of your stories to tell + the one-line frame.** Memorize the *shape*,
> tell it in *your* words with *your* numbers. This is the highest-leverage page in the curriculum for
> *your* interviews.

---

## The story bank: prompt → story

| Interviewer asks… | Tell this | The frame (one line) |
|---|---|---|
| "Tell me about a system you built" | **DCE** or **Questimate** (role-match) | 90s pitch + number + a thread to pull |
| "Hardest technical problem?" | **BDT pricer** (Questimate) or **DCE correctness safety net** | specific, hard, yours; problem→why-hard→what-you-did→outcome |
| "A trade-off you made?" | **Redis-JWT** (stateless + revocation) or **REST-vs-gRPC** | "X buys A, costs B; I picked X because…" |
| "A bug you fixed / a failure?" | **migration parity** ( `compare_results.py` caught a divergence) or **a suppression you added** | reconciliation thinking; correctness-first |
| "How do you handle scale?" | **DCE ~1M/day** (but ~12/s — correctness is the hard part) | estimation maturity: 1M/day ≠ high QPS |
| "Caching?" | **Redis cache-aside (DCE) / snapshot precompute (Questimate)** | cache-aside + materialized view (m06/m13) |
| "Async / messaging?" | **Kafka (DCE) / Celery Beat (Questimate)** | decouple + burst absorption (m08) |
| "Idempotency / exactly-once?" | **reprocessing claims / a retried trade order** | at-least-once + idempotent = effectively-once (m08) |
| "ML in production / MLOps?" | **DCE pipeline** + **InsightDesk** (MLflow/drift/gate) | rules+ML hybrid + the full MLOps loop (m27/m28) |
| "RAG / LLM / agents?" | **InsightDesk** (faithfulness ~0.96, hybrid+rerank, LangGraph agent) | measured, production-grade (m30/m32/m34) |
| "What would you do differently?" | DCE → feature store + online eval; Questimate → harden prod + gRPC hot paths | shows the maturity gap (senior) |
| "Cost / efficiency?" | **$675K→$50K/mo, ETL 60%, $75K→$20K** | cost-as-a-first-class concern (m10/m28/m29) |

---

## Exercise solutions (the model frames)

### Ex 1 — The pitches
Use the **README §1 (DCE)** and **§2 (Questimate)** pitches verbatim as your starting scripts; tighten to
*your* voice. Both ≤ 90s, end with **"happy to go deep on X, Y, or Z."** (Recording check: zero silence,
≥ 1 number, a thread offered.)

### Ex 2 — Framework on your system
**DCE:** requirements (recommend action/claim; **correctness+auditability** dominant NFR) → estimate (~1M/
day ≈ 12/s, bursty → size for peak; correctness is the challenge) → data (Mongo claims / Redis reference /
Snowflake OLAP / MinIO blobs — polyglot) → high-level (Kafka → preprocess/features → exclusions → ML+rule
models → suppressions → decision+audit) → deep-dive (suppression layer: output guardrail, rules over
ERR_CDS/LOB, logged for audit) → bottlenecks (Kafka/serving SPOF → replication + reprocessing + suppression
degrades safely; migrations + parity). **Questimate:** analogous (README §2).

### Ex 3 — Deep-dives (component → mechanism → trade-off)
1. **Suppression:** an exit guardrail → rules over claim error-codes/LOB/type, vectorized in pandas, logged
   to Mongo → trade-off: catches wrong outputs (correctness) at the cost of maintaining many rules + some
   valid recommendations possibly suppressed (tuned).
2. **Redis-JWT:** bearer token → key looked up in Redis → JWT verified (HS256) → stateless verify + instant
   revocation (delete key) → trade-off: a Redis dependency on the auth path (mitigate: replicate Redis) for
   the benefit of revocability pure-JWT lacks.
3. **Snapshot precompute:** Celery Beat nightly → risk decomposition for all portfolios in parallel
   (ThreadPoolExecutor) → cached to snapshot tables → O(1) API reads → trade-off: **staleness** between
   refreshes (fine for analytics) for fast reads.
4. **Feature pipeline:** raw claim JSON → 100+ structured fields (membership overlap, date windows, NPI/
   diagnosis matching, Redis reference merge) → one consistent representation every model consumes →
   trade-off: heavy preprocessing, but prevents training-serving skew (m23).

### Ex 4 — Name the pattern
1. Output guardrail / suppression (m27/m34). 2. Online feature lookup (m23) / cache-aside (m06). 3.
Materialized view / precompute (m13/m23). 4. Reconciliation (m21). 5. Replay from the log (m08). 6.
Stateless auth + Redis session (m02/m03/m07). 7. Microservice REST data mesh / east-west (m03).

### Ex 5 — "Yes, on [system] I…"
- **Caching:** "Redis cache-aside for hot reference data (DCE); snapshot-table precompute (Questimate)."
- **Async:** "Kafka claim ingestion (DCE); Celery Beat daily pipelines (Questimate)."
- **Idempotency:** "reprocessing claims is idempotent so a replay doesn't double-adjudicate; a retried trade
  order mustn't double-execute."
- **Monitoring:** "claim/exclusion counts + parity reports (DCE); missing-returns detection + backfill
  (Questimate); PSI/KS drift + alerts (InsightDesk)."
- **Safe deploys:** "parity-validated platform migrations on a live 1M/day system."
- **Feature engineering:** "100+ consistent claim features from variable JSON (DCE)."
- **Cost control:** "drove run cost $675K→$50K/mo; per-call LLM cost tracking on a $5 budget (InsightDesk)."
- **Reconciliation:** "`compare_results.py` parity between old/new environments."
- **RAG / agents / serving:** "InsightDesk — hybrid retrieve+rerank RAG (faithfulness ~0.96), a LangGraph
  multi-tool agent, FastAPI serving with load-once+warm + caching."

### Ex 7 — Hardest-part stories
- **DCE:** "Making correctness the #1 NFR on a 1M/day pipeline — I designed the **suppression layer** so an
  incorrect model recommendation can't ship, with everything audited. The hard part was getting the rules
  right across LOBs/states/edit-codes without over-suppressing valid ones — tuned + unit-tested per
  scenario." (Or the migration-parity story.)
- **Questimate:** the **BDT pricer** (README §2 / CROSS Q12) — calibrate a short-rate tree to the zero curve
  via bisection, backward-induct with an embedded call.

### Ex 8 — What I'd do differently
- **DCE:** real-time scoring where latency matters; a formal **feature store** (point-in-time + reuse, m23);
  richer **online eval** beyond offline parity (m22); automated **retrain + eval gates** (m28).
- **Questimate:** harden the **known prod-hygiene items** (DEBUG=True, hard-coded SECRET_KEY, ALLOWED_HOSTS
  — "known items to harden for production," framed honestly as maturity, not a flaw); **gRPC** on hot
  internal paths (m03); read replicas as load grows (m05).

### Ex 9 — Handle the challenge (curiosity + both sides)
1. **REST not gRPC:** "Coarse-grained, non-latency-critical inter-service calls → REST's simplicity +
   debuggability won; I'd move a hot path to gRPC if it became a bottleneck (binary + HTTP/2 multiplexing)."
2. **One big DB:** "The access patterns differ ~100× — variable claim docs (Mongo), sub-ms reference lookups
   (Redis), columnar analytics (Snowflake); one DB can't be great at all three (polyglot, m04)."
3. **Single Postgres primary:** "It's read-heavy analytics, so reads are absorbed by snapshots + cache +
   (next) read replicas; I'd add replication + failover for HA and only shard if write volume demanded it
   (m05) — it doesn't yet."
4. **Stale snapshots:** "Acceptable for risk analytics where a few-minutes-old number is fine; it's the
   precompute-for-latency trade. For anything needing freshness (auth, live trading) I use strong
   consistency / live calls — consistency per feature (m02)."

### Ex 10 — Role-matching + bridge
1. (a) **DCE** (ML at health-tech — domain + production ML + correctness). (b) **Questimate** (fintech
   backend — from-scratch architecture + quant). (c) **InsightDesk** lead, DCE/Questimate as production
   credibility (AI/LLM startup). (d) **DCE + InsightDesk** (platform/MLOps — operating ML at scale +
   building the MLOps stack).
2. **Bridge to InsightDesk:** "Beyond the production systems, I built the modern AI stack end-to-end —
   **RAG (hybrid retrieve+rerank, faithfulness ~0.96, CI-gated), a LangGraph agent, FastAPI serving, MLflow
   + drift monitoring + retrain gates** — so I bring both **production-scale credibility (DCE/Questimate)**
   and **hands-on AI/ML/LLM system-building (InsightDesk)**." That combination is rare and exactly what
   AI/ML/LLM roles want.
