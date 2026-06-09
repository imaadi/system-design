# Module 36 — Mock Interview Drills + Cheat-Sheets + Rapid-Fire 🏁 — *the final module*

> The exam-prep layer that **ties all 36 modules together**: how the interview actually runs, the **timing
> template**, a **full worked mock**, **one-page cheat-sheets** (numbers, patterns, decision trees), the
> **red/green flags**, and a **2-week prep plan**. Use this as your **go-to reference the night before any
> interview.** You've done the deep work (m01–m35); this makes it *deliverable under pressure*.

> **Format (capstone):** the playbook + cheat-sheets (here), **rapid-fire recall** across the whole
> curriculum (CROSS_QUESTIONS), a **mock-interview prompt bank + rubric** (EXERCISES), and **self-scoring +
> a final checklist** (solutions).

---

## 1. How a system-design interview actually runs (45–60 min)
| Phase | Time | What you do | The trap to avoid |
|---|---|---|---|
| **Requirements + scope** | 5–8 min | Clarify FRs, **NFRs (the drivers)**, scope down. Ask, don't assume. | Jumping to architecture before scoping |
| **Estimation** | 3–5 min | QPS, storage, bandwidth → derive the **constraint** (read-heavy? hot keys?) | Numbers for show; not using them |
| **API + data model** | 5 min | Key endpoints; entities; **storage choice by access pattern** | Picking a DB with no reason |
| **High-level design** | 8–10 min | Boxes + data flow, end-to-end one request. Get a working system *first*. | Over-engineering before it works |
| **Deep dives** | 10–15 min | Go 3 levels into 1–2 components the interviewer probes; **trade-offs** | Staying shallow everywhere |
| **Bottlenecks + scale** | 5–8 min | SPOFs, scaling the constraint, failure modes, evolution | Ignoring failure/ops |
| **Wrap-up** | 2 min | Summarize design + the key trade-offs you made | Trailing off |

> **What they're scoring (the rubric):** **structured approach** (not chaos), **NFR-driven decisions**
> (every choice tied to a requirement), **trade-off awareness** (options + why), **depth on demand** (3
> levels), **communication** (drive + check in), and **seniority signals** (numbers, failure modes, "it
> depends on X"). It's a **conversation you drive**, not a quiz.

---

## 2. The universal answer skeleton (works for ANY prompt)
```
1. "Let me clarify requirements." → FRs (what it does) + NFRs (scale, latency, consistency,
   availability — which DOMINATES?) + scope down.        ← m01
2. "Let me estimate." → QPS (read vs write), storage, bandwidth → name the CONSTRAINT.   ← m02
3. "APIs + data model." → key endpoints; entities; STORAGE by access pattern.   ← m04
4. "High-level." → draw boxes, trace ONE request end-to-end. Working system first.   ← m01
5. "Deep dive." → the interesting component, 3 levels, OPTIONS + trade-off + pick.   ← all
6. "Bottlenecks + scale." → SPOFs, scale the constraint, failure modes, evolution.   ← m10
7. "Wrap." → recap design + the 2–3 key trade-offs.
```
**The senior tells:** quote a **number**; say **"it depends on [X]"** and resolve it; name the **trade-off**
on every choice; surface a **failure mode** unprompted; map to a **known pattern** ("that's a materialized
view / rules+ML hybrid / CDC").

---

## 3. A full worked mini-mock (condensed) — "Design a URL shortener"
> *(The flow, not the full essay — m11 has the depth. This shows the rhythm.)*
1. **Reqs:** shorten URL → short code; redirect; analytics(?). **NFRs: read-heavy (100:1), redirect
   latency-critical (<50ms), high availability** (a dead shortener breaks every link). Scope out custom
   domains.
2. **Estimate:** 100M new URLs/mo ≈ **40 writes/s**, **4K redirects/s** peak; 5 yr ≈ 6B rows ≈ ~½ TB →
   *the constraint is read latency + availability, not storage.*
3. **API/data:** `POST /urls`, `GET /{code}`; table `(code PK, long_url, created)`. **KV/indexed lookup**
   by code (point reads) → KV store or indexed SQL.
4. **High-level:** client → LB → app → (cache → DB); writes generate a **code** (base62 of a counter / KGS)
   → store; reads **cache-aside** the code→URL (read-heavy → cache hit-rate is everything) → 301/302.
5. **Deep dive — code generation:** options: **hash+collision-check** vs **counter→base62** vs **pre-gen
   key service (KGS)**. Pick **KGS/counter** (no collision checks, sequential is fine since codes are
   opaque); trade-off: counter is a write hotspot → **range-allocate** blocks per server (m05).
6. **Bottlenecks:** cache is the SPOF for latency → replicate + high hit rate; DB read scale → replicas
   (read-heavy, m05); the counter → range allocation; analytics → **async** via a queue (m08, don't block
   redirects).
7. **Wrap:** "read-heavy + latency-critical → cache-fronted KV with pre-generated keys; analytics async;
   scale reads with replicas. Key trade-off: 301 (cacheable, faster) vs 302 (keeps control/analytics)."

> Notice: **every decision cited an NFR**, quoted a **number**, gave **options + a pick**, and named
> **failure modes**. That's the whole game. (Replay this rhythm on any m11–m21 / m25–m34 prompt.)

---

## 4. Cheat-sheet A — the numbers (from m02; quote these)
- **Latency ladder:** L1 ~1ns · RAM ~100ns · SSD ~100µs · **datacenter RTT ~0.5ms** · disk seek ~10ms ·
  **cross-continent RTT ~150ms**. (Memory ≈ 10⁵× faster than disk; network dominates.)
- **Throughput rules of thumb:** one server ~**10K** simple req/s; a DB ~**1–10K** writes/s/node, reads
  scale with replicas/cache; Redis ~**100K+** ops/s; Kafka ~**millions** msgs/s.
- **Availability:** 99.9% = **8.7h/yr** down · 99.99% = 52min · 99.999% = 5min. Each 9 ≈ 10× cost.
- **Estimation shortcuts:** 1 day ≈ **10⁵ s**; "X per day" ÷ 10⁵ = avg/s; **peak ≈ 2–10× avg**;
  read:write often **100:1**; **80/20** hot keys. A UUID ~16B, a tweet ~300B, an embedding (768×4B) ~3KB.
- **Capacity:** QPS × payload = bandwidth; rows × row-size × replication × years = storage; **+ overhead,
  round up.**

---

## 5. Cheat-sheet B — the decision trees (the "it depends on X")
- **SQL vs NoSQL:** relations/transactions/ad-hoc queries → **SQL**; massive scale/flexible schema/known
  access pattern → **NoSQL**; *polyglot* by access pattern (m04).
- **Consistency:** money/inventory/auth-revoke → **strong** (CP); feeds/likes/analytics → **eventual** (AP).
  Choose **per feature** (m02/m09).
- **SQL scaling order:** index → cache → **read replicas** → vertical → **shard** (last). (m05)
- **Cache write policy:** read-heavy → **cache-aside**; write-heavy + read-after-write → write-through;
  tolerate loss for speed → write-back. Invalidation = TTL + event. (m06)
- **Sync vs async:** user needs the result now → **sync**; slow/retryable/spiky/fan-out → **async queue**
  (m08).
- **Push vs pull (feeds):** most users → **fan-out-on-write (push)**; celebrities → **pull**; **hybrid**
  (m13).
- **REST vs gRPC vs GraphQL:** public/simple → REST; internal/low-latency → **gRPC**; client-shaped
  aggregation → GraphQL (m03).
- **RAG vs fine-tune vs long-context:** knowledge → **RAG**; behavior → fine-tune; bounded+affordable →
  long-context. Order: prompt → RAG → fine-tune (m30).
- **LLM build vs buy:** quality/speed/low-volume → **API**; scale/privacy/cost → **self-host**; **hybrid +
  gateway** (m29/m33).

---

## 6. Cheat-sheet C — the reusable patterns (name them)
**load balancing · caching (cache-aside/CDN) · replication (HA + read scale) · sharding (write scale) ·
async queue / event-driven (decouple + burst) · CDC / outbox · idempotency (effectively-once) · materialized
view / precompute · CQRS · rate limiting (token bucket) · circuit breaker / retry+backoff · bulkhead ·
consistent hashing · quorum (R+W>N) · WAL / log + replay · saga (distributed txn) · retrieve-then-rerank
funnel · rules+ML hybrid + guardrails · feature store · eval + CI gate · semantic cache + model routing.**
> In the interview, **naming the pattern** ("this is just a materialized view," "that's the retrieve-then-
> rerank funnel," "we'd use an outbox for exactly-once") signals you've seen it before — that's seniority.

---

## 7. Red flags vs green flags (self-audit)
| 🟢 Green (do) | 🔴 Red (avoid) |
|---|---|
| Clarify NFRs first; tie every choice to one | Jump straight to boxes |
| Quote numbers; use the estimate | Numbers as decoration |
| "Option A vs B; I'd pick A because [NFR]" | "We'll use Kafka" with no why |
| Get a working system, *then* optimize | Over-engineer (sharding for 100 users) |
| Surface SPOFs + failure modes unprompted | Ignore failure/ops |
| Drive + check in ("want me to go deeper on X?") | Monologue or go silent |
| "It depends on [X]" → resolve it | One true answer / dogma |
| Admit unknowns, reason from principles | Bluff |

---

## 8. The 2-week prep plan (how to use m01–m36)
- **Days 1–2:** Re-read **m01 (framework)** + **m02 (fundamentals + numbers)** until the **answer skeleton
  (§2) + numbers (§4) are reflex.**
- **Days 3–6:** Drill **classic problems (m11–m21)** — one/day, timed, out loud; check CROSS_QUESTIONS.
- **Days 7–9:** Drill **ML SD (m22–m28)** — framework + recsys/ranking/fraud/MLOps.
- **Days 10–12:** Drill **LLM SD (m29–m34)** — serving/RAG/agents/gateway/eval.
- **Days 13–14:** **Rehearse YOUR systems (m35)** — pitches + walkthroughs until fluent; then **timed full
  mocks** (EXERCISES here) and re-read these cheat-sheets.
- **Daily (15 min):** rapid-fire recall (this module's CROSS_QUESTIONS).
- **Night before:** §2 skeleton + §4 numbers + §5 decision trees + your m35 pitches. Sleep.

---

## 9. Final mindset
- **You drive the conversation** — it's collaborative, not a test you pass by reciting.
- **You're not memorizing designs; you have a method** (m01) + **vocabulary** (m02–m34) + **lived systems**
  (m35). Apply them to *any* prompt, including ones you've never seen.
- **There is no "best" — only "best for these requirements."** Saying that *is* the senior answer.
- **Your edge is real:** most candidates imagine scale; you've **operated 1M claims/day** and **built a
  4-service platform + a full RAG/agent/MLOps stack.** Walk in and own it.

---

## 🎓 You've completed the 36-module curriculum
**Part 0** (framework + fundamentals) → **Part 1** (8 building blocks) → **Part 2** (11 classic problems) →
**Part 3** (ML system design) → **Part 4** (LLM system design) → **Part 5** (your systems + this playbook).
You now have the **method, the vocabulary, the patterns, the numbers, and your own war stories.** Go get
the role. 🚀

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md) (rapid-fire recall across the whole curriculum), then
self-administer the **mock-interview prompt bank** in [`EXERCISES.md`](EXERCISES.md) and score yourself with
the rubric + final checklist in
[`solutions/m36_mock_interview_drills/`](../../solutions/m36_mock_interview_drills/README.md).

This is the **last module** — the curriculum is complete. Revisit any module anytime; re-run the mocks
before each interview. 🏁
