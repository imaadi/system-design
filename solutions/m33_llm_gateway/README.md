# m33 — Solutions (LLM Gateway / Platform)

> Check the *reasoning*. Recurring lessons: gateway = m03+m28 for LLMs; semantic caching = cache by meaning
> (tune threshold); routing = the ~100× cost lever; thin + HA.

---

## Exercise 1 — Why a gateway
1. **Routing, semantic caching, rate-limiting/quotas, fallbacks, cost tracking/budgets, observability, and
   guardrails** — across multiple models/providers.
2. The **m03 API gateway** (centralized entry, cross-cutting concerns, thin + HA) **+ m28 platform** (build
   once, all apps reuse) — for LLMs.
3. Because every app **reimplementing** provider integration, cost tracking, caching, fallbacks, and safety
   is wasteful, inconsistent, and gives **no org-wide control** over LLM spend/reliability/policy —
   centralizing fixes all of that.

---

## Exercise 2 — Semantic caching
1. LLM queries are **natural language**, so the same question has many phrasings ("capital of France?" vs
   "France's capital?") → **exact-match keys rarely match**, so the cache almost never hits.
2. **Embed the query** → **vector similarity search** over cached (question→answer) pairs → if a cached
   query is **similar enough** (above threshold), **return its answer** (skip the LLM); else call the LLM
   and **cache the new Q/A**.
3. **False hits** — semantically-similar-but-different queries ("flights *to* vs *from* Paris") return a
   **wrong** answer → **tune the similarity threshold** (loose → wrong answers; tight → few hits) and
   monitor correctness.
4. **Personalized** answers (depend on the user) and **time-sensitive** queries ("price *now*", "today's
   news") — also precise factual/legal/medical and stateful conversation turns.

---

## Exercise 3 — Model routing
1. **Model cost/quality varies ~100×**, so routing each query to the cheapest sufficient model is the
   biggest cost lever — a small per-query saving is enormous at scale.
2. **Cost-based (cheap-first/escalate):** classify the query's difficulty (heuristic or ML classifier),
   send it to a **small/cheap model**, and **escalate to a bigger model** only if the task is hard / the
   cheap output fails a quality check.
3. **Capability-based** (code → code model, vision → multimodal, long-context → long-context model) and
   **provider/load-based** (spread across providers, fall back, A/B new models).

---

## Exercise 4 — Fallbacks & reliability
1. **Fall back to another provider/model + retry with backoff + circuit-break** the rate-limited provider —
   keeping requests flowing despite provider A's limit.
2. **Chain:** provider A → provider B → a cheaper model; with **retries + exponential backoff** (m10),
   **circuit breakers** (stop hammering a dead provider), and **timeouts** — degrade gracefully.
3. Because providers are **flaky external dependencies**, and for most use cases a slightly-worse answer
   from a fallback model is **far better than failing the request** — availability over perfect model
   choice (m10 graceful degradation).

---

## Exercise 5 — Cost control
1. Because **all LLM spend flows through it**, the gateway is the **one place** to see and control cost —
   org-wide visibility + enforcement (scattered, uncontrolled otherwise).
2. **Track** tokens + cost per call, **attribute** to teams/apps/users; **enforce** budgets/quotas (alert
   at thresholds, then **throttle/cut off** over budget). Plus cost-based routing + semantic caching to cut
   spend.
3. My **per-call cost tracker** = the gateway's per-call cost logging/observability; my **$5 budget** = a
   budget/quota; the gateway **centralizes + automates** what I did by hand on one app, across the whole org.

---

## Exercise 6 — Rate limiting
1. **Per user/team/app** (token-bucket, m12, with **429 + Retry-After**, tiered by plan). **Token-based**
   because a single request can be huge (a 100K-token prompt = far more cost/load than a tiny one), so
   token quotas reflect **true cost/load** better than request counts.
2. Because exceeding the **upstream provider's rate limits** gets you throttled/blocked for *everyone* — so
   the gateway must **rate-limit + queue + spread across providers** to stay within provider limits while
   serving its users (protecting both sides).

---

## Exercise 7 — SPOF & observability
1. **Risk:** it's on **every LLM call** → a SPOF + potential bottleneck. **Mitigate:** run it **redundantly
   behind a load balancer** (HA, m07), keep it **thin + stateless** (just routing/caching/policy), scale
   **horizontally**, and make it **fast** — the m03/m07 API-gateway discipline.
2. **Per call:** prompt, response, model, **tokens, cost, latency**, cache-hit, fallback fired, user/team —
   for **(1) debugging** (LLMs fail weirdly), **(2) cost attribution**, **(3) eval** (traces → m34), and
   **(4) audit/compliance**.

---

## Exercise 8 — Gateway vs API gateway
1. **Model routing** (across models/providers by cost/capability/difficulty — not just by path), **semantic
   caching** (cache by meaning), **token/cost tracking + budgets**, **prompt management**, and **LLM
   guardrails** (moderation/PII/prompt-injection) — on top of the normal gateway role (auth/rate-limit/
   observability).
2. The gateway is a **central enforcement point** for guardrails (m34): **input moderation/PII before the
   model, output moderation before returning, prompt-injection defenses** — applied once for all apps; m34
   is the *logic*, the gateway is *where it runs*.
3. **Safe rollout via routing:** **shadow** (new model gets traffic, logged not served) → **canary**
   (small % of real traffic) → **A/B test** (quality + cost + business metric) → full, with **instant
   rollback** (flip routing config). The gateway already decides which model handles each request, so model
   rollout is a routing-policy change gated on metrics (m24/m10).

---

## Exercise 9 — Build vs buy + prompts
1. **Buy/adopt** for the standard cross-cutting concerns (proven options: **LiteLLM, Portkey, Cloudflare AI
   Gateway, Helicone, Kong AI Gateway**); **build** only for unusual needs (custom routing, strict data
   residency, deep integration) — same build-vs-buy logic as m24.
2. **Prompt management** = a **versioned prompt registry**: change a system prompt/template **without
   redeploying every app** (apps reference by name/version), enabling **versioning + rollback, A/B testing,
   consistency, and governance** (review/approve changes — important in regulated domains).

---

## Exercise 10 — Your-systems tie-in 🏭
1. **Per-call cost tracker → §6 observability + cost tracking**; **$5 budget → §5 budgets/quotas**;
   **"embeddings local/free, only generation calls the API" → §3 cost-based routing** (cheap/local for
   easy/cheap work, the expensive API only when needed). You hand-built the gateway's core jobs on one app.
2. **Semantic caching = m06 caching + your embeddings/vector search:** you **embed the query** (your
   embeddings work) and do a **similarity search over cached answers** (your vector DB/ANN) instead of an
   exact-key lookup — combining both halves you already built into the LLM-specific cache.
3. **Org LLM gateway** for chatbot (m31) + RAG (m30) + agents (m32): every call passes through **(1)
   semantic cache check** → **(2) rate-limit/quota** (per team, token-aware) → **(3) input guardrails**
   (moderation/PII, m34) → **(4) model routing** (cheap-first/escalate + capability + provider) → **(5)
   provider call with fallback chain + retries** → **(6) output guardrails + cache the result + log
   tokens/cost/latency**. Thin + HA, with central cost attribution/budgets, prompt registry, and
   observability feeding eval (m34). One control plane; all three apps benefit.
