# Module 33 — Design an LLM Gateway / Platform ⭐⭐ 🏭

> A **centralized gateway** that all LLM traffic flows through — providing **model routing**, **semantic
> caching**, **rate limiting/quotas**, **fallbacks**, **cost control**, **observability**, and
> **guardrails** across multiple models/providers. It's the **m03 API gateway + m28 platform**, specialized
> for LLMs: build the cross-cutting concerns **once** so every LLM app (chatbot m31, RAG m30, agents m32)
> doesn't reimplement them. You did the manual version — **per-call cost tracking, a $5 budget,
> "embeddings local / generation calls the API"** — which are exactly a gateway's jobs.

> **Format (Part 4 deep-dive):** worked design; **model routing**, **semantic caching**, **cost control/
> fallbacks**, and **the gateway-as-platform** are the deep-dives.

---

## 1. Why an LLM gateway (the platform argument)
As LLM use spreads across an org, you **don't want every app to reimplement** provider integration, cost
tracking, rate limiting, caching, fallbacks, and guardrails. So you **centralize** them in a gateway every
app calls — the **m03 API gateway + m28 platform** pattern for LLMs:
- **Functional:** route to models, cache, rate-limit, track cost, fall back, observe, guardrail.
- **Non-functional:** **low overhead** (it's on every LLM call — keep it thin/fast), **high availability**
  (a SPOF if all traffic flows through it → make it HA, m07), **multi-provider** (OpenAI/Anthropic/self-
  hosted), **cost control** (the dominant LLM concern), **observability** (LLMs fail weirdly — log
  everything).

> **The framing:** *"It's an **API gateway for LLMs** — centralize routing, caching, rate-limiting, cost,
> fallbacks, observability, and guardrails so apps stay simple and the org gets one place to control LLM
> spend, reliability, and policy. Build once, every app uses it."*

---

## 2. The architecture
```
  apps (chatbot/RAG/agents) ─▶ LLM GATEWAY ─┬─ ① semantic CACHE check ──HIT──▶ return cached answer
                                            ├─ ② RATE LIMIT / QUOTA check (per user/team/app)
                                            ├─ ③ guardrails (input moderation, PII — m34)
                                            ├─ ④ ROUTE → pick model/provider (cost/quality/capability)
                                            ├─ ⑤ call provider (with FALLBACK chain + retries)
                                            └─ ⑥ guardrails (output) → cache it → LOG (tokens/cost/latency)
   side: cost tracking + budgets · observability/traces · prompt registry · provider keys
```
Every call: **cache check → rate-limit → input-guardrails → route → provider (with fallback) → output-
guardrails → cache + log.** The gateway is **thin plumbing** (no business logic) and **HA**.

---

## 3. Deep Dive: model routing ⭐ (the cost lever)
LLM cost/quality varies **~100× across models**, so you **route each request to the right model:**
- **Cost-based (the big one):** route **easy queries to a cheap/small model**, **hard ones to an expensive
  one** — escalate only when needed. A **router** (a heuristic, or an **ML classifier** that predicts
  difficulty) decides per request. *(Your "embeddings local/free, generation calls the API" instinct,
  generalized.)*
- **Capability-based:** route by what the task needs (code → a code model; vision → a multimodal model;
  long context → a long-context model).
- **Provider/load-based:** spread across providers (avoid one's rate limits/outages), or A/B/canary new
  models (m24-style safe rollout for LLMs).

> **The senior line:** *"Routing is the **dominant cost lever** — models vary ~100× in price, so I route
> **cheap-model-first and escalate** (a router predicts difficulty), route by **capability**, and spread/
> fall back across **providers**. It centralizes the m29/m32 'use the smallest sufficient model' decision."*

---

## 4. Deep Dive: semantic caching ⭐⭐ (the LLM-specific innovation)
Traditional caching is **exact-match** (same key → cached value), but **LLM queries are natural language** —
*"What's the capital of France?"* and *"France's capital?"* are **different strings but the same
question**, so exact-match caching rarely hits. **Semantic caching:**
- **Embed the query**, do a **vector similarity search** over cached (question → answer) pairs; if a
  **semantically-similar** past query exists (similarity above a threshold), **return its cached answer** —
  skipping the LLM call entirely.
```
  query → embed → similarity search over cached Q/A → close enough? → return cached answer (skip LLM)
                                                     → else → call LLM, then cache the new Q/A
```
- **Huge savings** on paraphrased/repeated queries (cost — m29 — + latency), beyond exact-match caching.
- **The risk ⭐:** **false hits** — two queries that are *semantically similar but actually different*
  ("flights to Paris" vs "flights from Paris") → returning a **wrong cached answer.** So you **tune the
  similarity threshold** (too loose → wrong answers; too tight → few hits) and don't semantic-cache where
  exactness/freshness is critical. *(It's your m06 caching + your embeddings/vector-search, combined.)*

---

## 5. Deep Dive: cost control, rate limiting & fallbacks
- **Cost control + tracking (the dominant LLM concern, m29):** **log tokens + cost per call**, **attribute
  cost to teams/apps/users**, enforce **budgets/quotas** (alert/cut off when a team exceeds budget — your
  **OVER_BUDGET** instinct). The gateway is the **one place to see and control LLM spend** org-wide.
- **Rate limiting / quotas (m12):** per **user/team/app** — protect provider rate limits **and** control
  cost (token-bucket, m12), with **429 + Retry-After**. Different tiers per plan.
- **Fallbacks & retries (m10):** providers **go down / rate-limit / time out** (flaky external dependency)
  → a **fallback chain** (provider A → B → a cheaper model) + **retries with backoff** + **circuit
  breakers** keeps you up. Degrade gracefully (a cheaper model's answer beats an error).
- **Prompt management:** centralize **versioned prompts/templates** (change a system prompt without
  redeploying every app) — and **guardrails (m34)** at the gateway (input/output moderation, PII).

---

## 6. Observability + bottlenecks
- **Observability (critical for LLMs):** **log every call** — prompt, response, model, **tokens, cost,
  latency, cache-hit, which fallback fired** — for **debugging** (LLMs fail weirdly), **cost attribution**,
  **eval** (feed traces to m34 eval), and **audit** (your healthcare world). The gateway is the **"LLM ops"
  layer**. *(Your per-call cost tracker is exactly this, centralized.)*
- **Bottlenecks/SPOF:** the gateway is **on every LLM call** → it's a **SPOF + potential bottleneck** →
  make it **HA (redundant, behind an LB, m07)** and **thin** (no business logic — just routing/caching/
  policy), exactly the m03 API-gateway lesson. **Semantic-cache correctness** (false hits → tune
  threshold), **routing quality**, and **multi-provider complexity** are the other risks.
- **Metrics (m10/m28):** cache hit rate, cost/team, p99 latency (gateway overhead), fallback rate, rate-
  limit rejections, routing distribution.

**Wrap-up:** *"An **API gateway for LLMs** — every app's LLM calls flow through it for **① semantic cache
check** (embed the query, similarity-search cached Q/A → return on a close hit; tune the threshold to avoid
false hits — the LLM-specific caching innovation), **② rate-limiting/quotas** (per team, protect providers
+ control cost), **③ guardrails** (moderation/PII, m34), **④ model routing** (cheap-first/escalate by
difficulty + capability + provider — the ~100× cost lever), **⑤ provider call with a fallback chain +
retries** (resilience for the flaky external dependency), and **⑥ output guardrails + caching +
observability** (log tokens/cost/latency per call for cost attribution, debugging, eval, audit). It's
**thin + HA** (a SPOF on every call). Centralizes m03/m06/m10/m12/m28/m29 for LLMs — exactly my manual
cost-tracker + budget + 'cheap-local-vs-API' discipline, productized."*

---

## State-of-the-art & real-world notes 📚
- **LLM gateways:** **LiteLLM** (multi-provider proxy + routing + cost), **Portkey**, **Cloudflare AI
  Gateway**, **Helicone** (observability + caching), **Kong AI Gateway**. **Semantic caching:** **GPTCache**.
- **Routing:** model routers / "LLM router" projects (route by predicted difficulty/cost); A/B model
  testing. **Cost/observability:** the "LLMOps" layer (Langfuse, Helicone, W&B) — log/trace every call.
- It's the convergence of **API gateway (m03) + caching (m06) + rate limiting (m12) + resilience (m10) +
  platform (m28) + LLM cost (m29)** — the LLM-ops control plane.

---

## From your systems 🏭
- **You hand-built the gateway's jobs:** a **per-call cost tracker** (= §6 observability/cost), a **$5
  budget** (= §5 budgets/quotas), and **"embeddings local/free, only generation calls the API"** (= §3
  cost-based routing). The gateway **centralizes + automates** exactly what you did manually.
- **API gateway experience (m03):** Questimate **nginx + JWT at the edge** is the gateway pattern this
  applies to LLMs (thin, HA, cross-cutting concerns at one entry point).
- **Semantic caching = your m06 caching + embeddings/vector search** — you have both halves; the gateway
  combines them (embed the query, similarity-search cached answers).
- **Fallbacks + multiple models:** you used **gpt-4o-mini (API) + Ollama (local)** — a natural **routing/
  fallback** setup.

---

## Key concepts (interview-ready)
- **An LLM gateway = the m03 API gateway + m28 platform for LLMs:** centralize **routing, caching, rate-
  limiting, cost, fallbacks, observability, guardrails** so apps stay simple. **Thin + HA** (it's on every
  call — a SPOF).
- **Model routing (the cost lever):** models vary **~100× in cost** → **cheap-model-first/escalate by
  difficulty** (heuristic or ML router), **capability-based**, **provider/load-based**, A/B. The
  centralized m29/m32 "smallest sufficient model" decision.
- **Semantic caching (the LLM innovation):** cache by **meaning** — embed the query, **similarity-search
  cached Q/A**, return on a close hit (skip the LLM). Big savings on paraphrases; **risk = false hits** →
  **tune the similarity threshold**; don't use where exactness/freshness is critical. (= caching m06 +
  embeddings.)
- **Cost control** (token/cost logging + **per-team budgets/attribution** — the dominant LLM concern),
  **rate limiting/quotas** (per user/team, m12), **fallback chains + retries + circuit breakers** (flaky
  providers, m10), **prompt management** (versioned), **guardrails** (m34).
- **Observability** (log every call — prompt/response/tokens/cost/latency/cache/fallback) for debugging,
  cost attribution, eval, audit — the **LLM-ops** layer.

---

## Go deeper (reading)
- **LiteLLM / Portkey / Cloudflare AI Gateway / Helicone / Kong AI Gateway** docs — production LLM gateways
  (routing, caching, cost, fallback, observability).
- **GPTCache** — semantic caching. **Langfuse / Helicone / W&B** — LLM observability/tracing.
- **LLM router** projects (route by predicted difficulty/cost) and "RouteLLM"-style research.
- Revisit **m03** (API gateway), **m06** (caching — semantic caching builds on it), **m10** (fallbacks/
  circuit breakers), **m12** (rate limiting), **m28** (platform), **m29** (cost/routing), **m34**
  (guardrails). The gateway is all of these for LLMs.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m33_llm_gateway/`](../../solutions/m33_llm_gateway/README.md).

When you've done the exercises, say **"Module 34"** to design *LLM Evaluation & Safety in Production* —
eval harnesses, LLM-as-judge, guardrails, **prompt injection**, PII/PHI handling, and regression gates
(the safety/quality layer that wraps everything in Part 4).
