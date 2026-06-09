# m33 — Cross-Questions ("if-and-buts") on the LLM Gateway / Platform

> Answer out loud in 2–3 sentences before reading the model answer. Routing, semantic caching, cost
> control, and the gateway-as-platform are where this is won.

---

### Q1. What is an LLM gateway and why build one?
**A.** A **centralized proxy that all LLM traffic flows through**, providing **routing, semantic caching,
rate-limiting/quotas, fallbacks, cost control, observability, and guardrails** across multiple models/
providers. You build it because, as LLM use spreads, you **don't want every app to reimplement** provider
integration, cost tracking, caching, fallbacks, and safety — you **centralize** them (the **m03 API gateway
+ m28 platform** pattern for LLMs). It gives the org **one place** to control LLM spend, reliability, and
policy, and keeps apps simple.

---

### Q2. What is semantic caching and how does it differ from normal caching?
**A.** **Normal caching is exact-match** (same key → cached value), but **LLM queries are natural language**
— "What's the capital of France?" and "France's capital?" are different strings but the **same question**,
so exact-match rarely hits. **Semantic caching embeds the query and does a vector similarity search over
cached (question→answer) pairs**; if a **semantically-similar** past query exists (above a threshold), it
**returns the cached answer**, skipping the LLM. It captures paraphrases/repeats that exact-match misses —
big cost + latency savings. (It's m06 caching + embeddings/vector search combined.)

---

### Q3. What's the risk with semantic caching and how do you manage it?
**A.** **False hits** — two queries that are **semantically similar but actually different** ("flights *to*
Paris" vs "flights *from* Paris", or "weather today" vs "weather tomorrow") can match, returning a **wrong
cached answer**. You manage it by **tuning the similarity threshold** (too loose → wrong answers; too tight
→ few cache hits), **not semantic-caching where exactness or freshness is critical** (personalized, time-
sensitive, or precise factual queries), and monitoring cache-hit correctness. It's a precision/recall trade
on the cache itself — the convenience of meaning-based caching against the risk of a confidently-wrong hit.

---

### Q4. What is model routing and why is it the dominant cost lever?
**A.** **Routing each request to the right model** — and since LLM cost/quality varies **~100× across
models**, routing **easy queries to a cheap/small model and hard ones to an expensive one** (escalate only
when needed) is the **biggest cost lever**. A **router** (a heuristic or an **ML classifier predicting
difficulty**) decides per request; you also route by **capability** (code/vision/long-context) and
**provider/load** (spread across providers, fall back). It centralizes the m29/m32 "use the smallest
sufficient model" decision — a small per-query routing win is enormous money at scale.

---

### Q5. How would you implement cost-based routing?
**A.** Classify each query's **difficulty/requirements** (a fast heuristic — length/keywords — or a small
**ML classifier**), then **route cheap-model-first and escalate**: send it to a small/cheap model, and if
the task is hard (or the cheap model's output fails a quality/confidence check) **escalate to a bigger
model**. You also factor **capability** (does it need code/vision/long-context?) and **cost ceilings**.
The router itself is cheap (it must not add much latency/cost), and you **A/B-test** routing policies on
cost vs quality. The goal: **minimize cost while meeting quality** per query.

---

### Q6. How do you handle a provider outage or rate-limit?
**A.** **Fallback chains + retries** (m10): if provider A is **down / rate-limiting / timing out**,
**fall back to provider B**, then to a **cheaper model**, with **retries + exponential backoff** and a
**circuit breaker** (stop hammering a dead provider). The gateway treats providers as **flaky external
dependencies** and **degrades gracefully** — a cheaper model's answer beats an error. This keeps your LLM
apps up despite provider unreliability, and the gateway is the one place that knows the fallback policy
(apps don't each reimplement it).

---

### Q7. How does the gateway control LLM cost across an org?
**A.** It's the **one place all LLM spend flows through**, so it **logs tokens + cost per call**,
**attributes cost to teams/apps/users**, and enforces **budgets/quotas** (alert and/or cut off when a team
exceeds budget — my OVER_BUDGET instinct). Plus **cost-based routing** (cheap models, §3) and **semantic
caching** (skip paid calls). Without a gateway, cost is scattered and uncontrolled; with it, you get
**org-wide visibility + enforcement** of the dominant LLM concern. It's the financial control plane for
LLMs (echoing the cost-as-SLO idea from m28).

---

### Q8. Isn't the gateway a single point of failure and a bottleneck?
**A.** Yes — it's **on every LLM call** — so you treat it like any critical edge component (the m03/m07
API-gateway lesson): **run it redundantly** (multiple instances behind a load balancer with health
checks), keep it **thin and stateless** (just routing/caching/policy — no business logic), scale it
**horizontally**, and make it **fast** (minimal added latency). The cross-cutting value (centralized cost/
reliability/policy) outweighs the SPOF risk **if** you engineer it as HA infrastructure — exactly how you'd
treat an API gateway.

---

### Q9. What do you log/observe at the gateway, and why?
**A.** **Every call:** prompt, response, model used, **tokens, cost, latency**, cache-hit/miss, which
**fallback** fired, and the user/team. Why: **debugging** (LLMs fail in weird, non-deterministic ways — you
need the actual prompt/response), **cost attribution** (per team/app), **eval** (feed traces to m34's eval
harness — measure quality/faithfulness offline), and **audit/compliance** (especially in regulated domains
— my healthcare world). The gateway is the natural **LLM-ops/observability** layer — it sees all traffic,
so it's where you instrument everything (my per-call cost tracker, centralized).

---

### Q10. How is an LLM gateway different from a normal API gateway (m03)?
**A.** It's the **same pattern** (centralized entry, cross-cutting concerns, thin + HA) **plus LLM-specific
features**: **model routing** (across providers/models by cost/capability — normal gateways route by path,
not by predicted query difficulty), **semantic caching** (cache by meaning, not exact key), **token/cost
tracking + budgets** (the dominant LLM concern), **prompt management**, and **LLM guardrails** (moderation/
PII/prompt-injection, m34). So it inherits the API-gateway role (auth/rate-limit/observability) and adds the
**LLM control plane** (route/cache/cost/guardrail across models).

---

### Q11. How do you do rate limiting in an LLM gateway, and what's special about it?
**A.** **Per user/team/app** rate limits (token-bucket, m12) returning **429 + Retry-After**, with tiers per
plan. What's special: you often limit on **tokens, not just requests** (a single request can be huge —
a 100K-token prompt — so token-based quotas reflect true cost/load), and you must **respect the upstream
providers' rate limits** (the gateway shouldn't exceed OpenAI/Anthropic limits → it rate-limits + queues +
spreads across providers). So it's m12 rate-limiting, but **token-aware** and **protecting both your users
and the upstream providers**.

---

### Q12. When should you NOT use semantic caching?
**A.** When **exactness or freshness is critical**: **personalized** answers (depend on the user/context, so
another user's cached answer is wrong), **time-sensitive** queries ("what's the price *now*", "today's
news"), **precise factual/legal/medical** answers where a near-match could be subtly wrong, and **stateful
conversation turns** (the answer depends on prior context). In those cases a semantic false-hit returns a
**confidently-wrong** answer — worse than a cache miss. You'd use exact-match caching (or none) there, and
reserve semantic caching for **stable, general, paraphrase-heavy** queries (FAQ-like).

---

### Q13. How does the gateway relate to the rest of Part 4?
**A.** It's the **shared infrastructure all the LLM apps run behind**: the **chatbot (m31)**, **RAG (m30)**,
and **agents (m32)** all make LLM calls **through the gateway**, which handles **serving concerns (m29 —
the gateway may front self-hosted vLLM and/or provider APIs)**, **routing/caching/cost/fallbacks**, and
**guardrails (m34)**. It's the **control plane** for LLM traffic — the LLM analog of the m28 ML platform
(which served traditional models). Build it once; every LLM application benefits (cost control, reliability,
observability, policy) without reimplementing.

---

### Q14. How do you safely roll out a new model through the gateway?
**A.** Since the gateway controls **routing**, you do **safe rollout there** (the m24/m10 patterns for
LLMs): **shadow** (send traffic to the new model, log/compare, don't serve it), **canary** (route a small %
of real traffic to it, watch cost/latency/quality), then **A/B test** (split + measure quality + cost +
the business metric) before full rollout — with **instant rollback** (flip the routing config back). The
gateway is the natural place because it already decides which model gets each request — model rollout is
just a routing-policy change, gated on metrics.

---

### Q15. How do you manage prompts in a gateway?
**A.** **Centralize versioned prompts/templates** in a **prompt registry** so you can **change a system
prompt or template without redeploying every app** (apps reference a prompt by name/version; the gateway
injects the current version). This gives **versioning** (roll back a bad prompt), **A/B testing** of
prompts, **consistency** (all apps use the approved prompt), and **governance** (review/approve prompt
changes — important in regulated domains). It's the prompt analog of a config service / the m28 registry —
decouple prompt changes from app deploys.

---

### Q16. A team is blowing the LLM budget. What does the gateway let you do?
**A.** Because all spend flows through it: **see exactly which team/app/user is spending** (cost
attribution from the logs), **enforce a budget/quota** (alert at thresholds, then **throttle or cut off**
when exceeded — the OVER_BUDGET pattern), **route their easy queries to cheaper models** (cost-based
routing), and **enable semantic caching** to skip repeated paid calls. Without the gateway you'd have no
visibility or control; with it, cost is a **governed, attributable, enforceable** resource — the dominant
LLM operational concern, centrally managed.

---

### Q17. What's the latency cost of a gateway, and how do you minimize it?
**A.** It adds a **hop** (a few ms) plus the **cache lookup + routing decision** on every call — usually
**negligible vs the LLM call itself** (hundreds of ms to seconds), so the overhead is small relative to the
benefit. To minimize it: keep the gateway **thin** (no heavy logic), make the **cache lookup fast** (vector
search must be quick — m18), make the **router cheap** (heuristic or a tiny model), and run it **co-located/
HA** (m07). And it often **reduces** net latency (a semantic cache hit returns instantly, skipping the LLM
entirely). The gateway's overhead is dwarfed by the LLM, so it's worth it.

---

### Q18. How do guardrails fit into the gateway (vs m34)?
**A.** The gateway is a natural **enforcement point** for guardrails (m34): **input moderation/PII
redaction before the model**, **output moderation before returning**, and **prompt-injection** defenses —
applied **centrally** so every app gets them without reimplementing. m34 is the **deep-dive on what the
guardrails are** (eval, injection, PII, moderation); the gateway is **where many of them run** (the choke
point all traffic passes through). So they're complementary: m34 = the safety/quality logic, m33 = a primary
place to apply it consistently across all LLM apps.

---

### Q19. Build vs buy for an LLM gateway?
**A.** Often **buy/adopt** — mature options exist: **LiteLLM** (multi-provider proxy + routing + cost),
**Portkey**, **Cloudflare AI Gateway**, **Helicone** (observability + caching), **Kong AI Gateway** — they
give routing/caching/fallback/cost/observability out of the box. **Build** only for unusual needs (custom
routing logic, strict data-residency, deep integration). The decision mirrors m24's build-vs-buy: use a
proven gateway for the standard cross-cutting concerns; don't reinvent the LLM control plane unless you have
a specific reason. The org-level value (cost control, reliability, observability) is the same either way.

---

### Q20. Summarize the LLM gateway as a senior answer.
**A.** *"It's an **API gateway for LLMs** (m03 + m28 platform, for LLMs): every LLM call from every app
(chatbot/RAG/agents) flows through it for **semantic caching** (embed query → similarity-search cached Q/A →
return on a close hit; tune the threshold to avoid false hits — the LLM-specific caching innovation),
**rate-limiting/quotas** (token-aware, per team, protect providers + control cost), **input/output
guardrails** (moderation/PII, m34), **model routing** (cheap-first/escalate + capability + provider — the
~100× cost lever), **provider calls with a fallback chain + retries** (resilience), and **cost tracking +
budgets + full observability** (log tokens/cost/latency per call). It's **thin + HA** (a SPOF on every
call). It centralizes **m03/m06/m10/m12/m28/m29** for LLMs — exactly the manual cost-tracker + budget +
cheap-local-vs-API discipline I used, productized into the LLM control plane."*
