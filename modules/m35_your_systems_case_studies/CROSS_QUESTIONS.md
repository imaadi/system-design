# m35 — Cross-Questions on YOUR OWN Systems

> These are the follow-ups an interviewer asks about **your real work** — the probing that turns "you built
> it" into "you understand it deeply." Answer out loud in 2–3 sentences, **with a number and a trade-off**.
> (These are the highest-ROI questions to rehearse — they *will* come up.)

---

### Q1. (DCE) Why a rules + ML hybrid instead of pure ML?
**A.** Because **rules give instant response, explainability, and hard guardrails** for clear/known cases
(and a regulated domain demands explainability), while **ML catches the fuzzy, evolving patterns** rules
can't. Pure ML would lose explainability + can't react instantly to a new known edit; pure rules can't
capture subtle patterns. So **exclusions (rules) at entry, ML + rule-based models in the middle, suppressions
(rules) at the exit** — the hybrid covers each other's gaps (m27). In healthcare, a wrong auto-deny has real
consequences, so I want both the ML's reach **and** the rules' control + auditability.

---

### Q2. (DCE) What does the suppression layer do and why is it a good design?
**A.** It's an **output guardrail (m34)** — at the pipeline exit, suppression rules **block incorrect
recommendations from shipping** to downstream adjudication. It's good because it makes **correctness the #1
NFR**: even if an upstream model is wrong, the suppression net catches it before it causes harm, and every
decision is **logged to Mongo for audit** (compliance). It's the same instinct as a payment ledger's
guardrails (m21) — *correctness over throughput, with an immutable record.* Down-beats-wrong for money and
for healthcare claims.

---

### Q3. (DCE) ~1M claims/day — how much QPS is that, and what's actually hard?
**A.** ~1M/day ÷ 10⁵ s ≈ **~12/sec average** — so raw QPS is *trivial*; the challenge is **not throughput,
it's per-claim correctness + compute + auditability** at scale. It's also **bursty** around payer cycles, so
I size for peak. The hard parts are the **feature engineering** (100+ consistent fields from variable claim
JSON), the **rules+ML correctness**, the **suppression guardrails**, and **operating it reliably with a small
team** — not "handle a million QPS." (Naming that ~1M/day ≠ high QPS shows estimation maturity, m02.)

---

### Q4. (DCE) Why MongoDB for claims and Redis for reference data?
**A.** **Polyglot persistence by access pattern (m04):** claims are **deeply nested, variable JSON read/
written as a unit** → a **document store (Mongo)** fits; **eligibility/auth/reference data** is **hot, looked
up sub-ms during scoring** → an **in-memory KV (Redis)**. Merging Redis reference data into each claim at
scoring time is exactly an **online feature lookup** (m23). Each store is chosen for what it's best at —
I wouldn't force both into one DB. Snowflake handles the columnar OLAP/ETL; MinIO holds model blobs.

---

### Q5. (DCE) How did you migrate platforms without breaking a live 1M/day system?
**A.** **Parity validation + safe cutover.** I wrote **comparison scripts (`compare_results.py`)** to verify
**claim-processing output equivalence** between old and new environments (Azure↔AWS, OpenShift↔EKS,
ENSO↔K8s) — running both in parallel and comparing before cutting over. That's **reconciliation thinking
(m21)**: assume something diverged, verify against a source of truth, flag discrepancies. Combined with
**reprocessing scripts** (replay from the log, m08) to recover, and it cut cost dramatically (**$675K→$50K/
mo, OpenShift→EKS $75K→$20K/mo, ETL 60%**) while holding throughput. Safe deploy = validate parity, not
hope (m10).

---

### Q6. (DCE) The labels (was a claim correctly adjudicated) are delayed/noisy. How do you handle eval?
**A.** Carefully — you don't always know immediately if a recommendation was right (it may be reviewed/
corrected later), so I **evaluate on matured outcomes**, use **review/audit feedback** as ground truth,
avoid **leakage** (don't use a post-decision signal as a feature, m23 point-in-time), and **monitor for
drift** (PSI/KS-style) since claim patterns/rules evolve. The **suppression layer + audit trail** also give
a safety net + a record to learn from. It's the delayed-label + monitoring discipline from m27/m28.

---

### Q7. (DCE) What would you do differently / improve?
**A.** Move more toward **real-time scoring** where latency matters, formalize the feature pipeline into a
proper **feature store** (m23 — explicit point-in-time correctness + reuse), add richer **online evaluation**
(m22 — beyond offline parity, measure real adjudication-quality impact), and tighten **drift monitoring +
automated retraining with eval gates** (m28). The architecture is sound; these are the maturity steps from
"works at scale" to "a fully governed ML platform."

---

### Q8. (Questimate) Why microservices, and why REST between them?
**A.** Each **analytical concern** (market data, factor model, sentiment, portfolio analytics) is an
**independently deployable, separately scalable, separately backfillable** service — so I can rebuild the
factor engine without touching the API gateway (m03 east-west). **REST** because it's **simple, language-
agnostic, and easy to debug** (curl), and the inter-service calls (pull factor returns / yield curve /
residuals) are coarse-grained, not latency-critical. The honest trade-off: *"if a hot path became latency/
throughput-critical, I'd move it to **gRPC** (binary + HTTP/2 multiplexing); I chose REST for simplicity
and debuggability"* — that nuance shows I know both (m03).

---

### Q9. (Questimate) Explain the Redis-backed JWT and why it's clever.
**A.** The client sends a bearer token; the **token key is looked up in Redis** to fetch the actual JWT,
which is then **verified (HS256)**. So I get **stateless JWT verification performance PLUS instant server-
side revocation** — deleting the Redis key **immediately kills the session** (which pure stateless JWTs
*can't* do — they're valid until expiry). It's the best of both worlds: JWT speed + session control. It's
also why the app tier stays **stateless** (session state in Redis, not the worker) → horizontal scaling
(m02/m07). Classic "options + trade-off + pick" deep-dive (m01).

---

### Q10. (Questimate) Why precompute risk into snapshot tables?
**A.** Because the risk analytics (factor variance, decomposition, VaR across horizons) are **expensive to
compute**, but the API needs **fast reads**. So I do the heavy compute **off the request path** — a nightly
**Celery Beat** pipeline runs risk decomposition for *every* portfolio **in parallel (ThreadPoolExecutor)**
and **caches results into snapshot tables** — turning the API into an **O(1) read** instead of a heavy
recompute. It's a **materialized view / precompute** pattern (m13/m23) — trading background throughput/work
for hot-path latency (m02). The cost is **staleness** (snapshots are refreshed on a schedule), which is
fine for analytics.

---

### Q11. (Questimate) What consistency model does it use, and where does it vary?
**A.** **Per-feature (m02):** market/analytics **snapshots are eventually consistent** (a few minutes stale
is fine for risk numbers), while **auth/session revocation is strongly consistent** (delete the Redis token
key → *instantly* revoked; you can't tolerate a "revoked-but-still-works" window for security). So I picked
the **weakest model that meets each feature's need** — eventual where staleness is harmless, strong where
correctness/security demands it. That deliberate per-feature consistency choice is a senior signal.

---

### Q12. (Questimate) What was the hardest part?
**A.** The **Black-Derman-Toy callable-loan pricer.** I built a **recombining binomial short-rate tree**,
**calibrated it to the live Indian zero-coupon curve** (solving each step's drift via **bisection** to
match market zero-bond prices), then priced the loan by **backward induction with an embedded issuer call**
(`V = min(continuation, call_price)`) — separating the callable value, non-callable value, and the call-
option value. It's genuinely advanced fixed-income engineering I implemented from the math up — a great
"hardest part" story because it's specific, hard, and mine.

---

### Q13. (Questimate) How does it scale, and what's the bottleneck?
**A.** The **stateless Django app tier scales horizontally** behind **nginx/uWSGI** (session/JWT in Redis,
m02/m07); **reads are absorbed by the snapshot tables + Redis cache** (the precompute, m13); the **Celery
pipeline** does heavy compute off the request path. The bottleneck would be the **nightly batch compute** as
portfolios grow (hence the parallel ThreadPoolExecutor) and a **single Postgres primary** for writes —
which I'd scale with **read replicas first** (it's read-heavy analytics, m05) before sharding, since write
volume is modest. It's a read-heavy system, so I optimized reads (precompute + cache + replicas).

---

### Q14. (Both) Walk me through what happens on a single request.
**A.** *(DCE):* a claim arrives via **Kafka** → **preprocessing** turns raw JSON into 100+ features
(merging Redis reference data) → **exclusion rules** → **ML + rule-based models** score it → **suppression
rules** check the output → the **decision (DENY/BYPASS/ROUTE/UPDATE) + evidence** is written to Mongo
(audit). *(Questimate):* a request hits **nginx → uWSGI → stateless Django**; **JWT** is verified via the
**Redis** token lookup; the handler reads the **precomputed snapshot table** (O(1)) — or, for trading, calls
the broker — and returns. (Trace a request end-to-end = the m01 high-level skill, on your own system.)

---

### Q15. (Both) Where's the single point of failure, and how would you address it?
**A.** *(DCE):* the **Kafka pipeline** and the **model-serving step** — addressed by Kafka replication +
consumer groups (m08), reprocessing/replay for recovery, and the suppression guardrail so failures degrade
*safely*. *(Questimate):* the **single Postgres primary** (writes) and **nginx** (entry) — addressed by
**replication + failover** for Postgres (m05), **redundant nginx behind a VIP/LB** (m07), and Redis
replication for sessions. In both, I'd name the SPOF proactively + its mitigation (the step-7 senior signal,
m01).

---

### Q16. (Both) How do you monitor these in production?
**A.** *(DCE):* claim **volume + exclusion counts** (`claims_count`/`exclusions_count` = metrics),
**suppression/decision reports**, **parity comparisons** (reconciliation), and (in my AI/ML work) **drift
(PSI/KS) + alerts (DATA_DRIFT/POSRATE_SHIFT/OVER_BUDGET) + retrain gates** (m28) — plus the Mongo **audit
trail**. *(Questimate):* admin/ops dashboards that **detect missing portfolio returns/holdings and backfill
them** — production data-integrity tooling. So I monitor **correctness/data-integrity signals**, not just
latency/errors — exactly because these systems fail on *correctness*, which normal monitoring misses (m10/
m28).

---

### Q17. (Both) "Have you worked with [caching / async / idempotency / sharding]?" — answer for each.
**A.** **Caching:** "yes — Redis cache-aside for hot reference data (DCE) and snapshot-table precompute
(Questimate), m06." **Async:** "yes — Kafka for claim ingestion (DCE) and Celery Beat pipelines (Questimate),
m08." **Idempotency:** "yes — reprocessing claims must be idempotent so a replay doesn't double-adjudicate,
and a retried trade order mustn't double-execute, m08/m03." **Sharding-awareness:** "Mongo replica
sets/sharding + Redis Cluster slots in DCE; Questimate is read-heavy so I'd add replicas before sharding,
m05." **Turn every concept into a 'yes, on [system] I…'.**

---

### Q18. (Meta) When do you bring up DCE vs Questimate?
**A.** **Match the system to the role + the question.** **ML / data / correctness / healthcare / platform**
roles → **lead with DCE** (1M/day ML pipeline, rules+ML, suppression, scale + cost story). **Backend /
architecture / fintech / from-scratch** roles → **lead with Questimate** (4-service mesh, Redis-JWT,
precompute, BDT, full ownership). For an open "tell me about a system you built," pick the one that best
shows the **dominant skill the role wants**, and keep the other ready to pivot to. And for **AI/ML/LLM**
roles, bridge to **InsightDesk** (RAG/agents/eval/MLOps) on top of the production credibility.

---

### Q19. (Meta) An interviewer challenges a decision you made. How do you respond?
**A.** With **curiosity, not defense** (the m01 green flag): *"Good point — for the scale/constraints we had,
[my choice] gave [benefit] at [cost]; if [their condition] held, I'd reconsider toward [alternative]."* E.g.
"Why REST not gRPC?" → "Simplicity + debuggability for coarse-grained, non-latency-critical calls; I'd move
a hot path to gRPC if it became a bottleneck." Showing I'd **adjust** + can **argue both sides of my own
decision** is more senior than defending it rigidly. (And being honest about the **known prod-hygiene items**
in Questimate — DEBUG/SECRET_KEY — as "things to harden" shows maturity, not weakness.)

---

### Q20. (Meta) Summarize how you'd open a "tell me about a system you've built" answer.
**A.** *"I'll give you DCE — pick the depth. **[90-second pitch]:** an AI pipeline auto-adjudicating ~1M US
healthcare claims a day with DENY/BYPASS/ROUTE/UPDATE; a **rules+ML hybrid** with exclusion rules, ML + rule
models, and a **suppression safety net**; Kafka/Mongo/Redis/Snowflake/EKS; we cut cost ~$675K→$50K/month and
the team 20→4 while holding throughput. The thing I'm proudest of is making **correctness the #1 NFR** — a
wrong auto-deny has real consequences, so the suppression layer guards the output and everything's audited.
**Happy to go deep on the suppression layer, the feature pipeline, or the migrations.**"* — pitch + number +
the proud decision + a thread to pull. **Then I drive the m01 framework on whichever thread they pick.**
