# m33 — Exercises (LLM Gateway / Platform)

> Do these on paper / out loud before checking `solutions/m33_llm_gateway/`. Routing, semantic caching,
> cost control, and the gateway-as-platform are the core.

---

## Exercise 1 — Why a gateway
1. What cross-cutting concerns does an LLM gateway centralize?
2. Which two earlier modules' patterns is it (gateway + platform)?
3. Why not let each app handle these itself?

---

## Exercise 2 — Semantic caching
1. Why does exact-match caching rarely hit for LLM queries?
2. How does semantic caching work (step by step)?
3. What's the false-hit risk, and how do you manage the threshold?
4. Give two query types you would NOT semantic-cache.

---

## Exercise 3 — Model routing
1. Why is routing the dominant cost lever (give the rough cost spread)?
2. Describe cost-based routing (cheap-first/escalate).
3. Name two other routing dimensions besides cost.

---

## Exercise 4 — Fallbacks & reliability
1. A provider is rate-limiting you. What does the gateway do?
2. Sketch a fallback chain and the resilience patterns involved (m10).
3. Why is "a cheaper model's answer beats an error" the right framing?

---

## Exercise 5 — Cost control
1. How does the gateway give org-wide cost control?
2. What do you track and enforce?
3. Connect this to your own per-call cost tracker + $5 budget.

---

## Exercise 6 — Rate limiting
1. Per what do you rate-limit, and why token-based not just request-based?
2. Why must the gateway respect the *upstream* provider's limits?

---

## Exercise 7 — SPOF & observability
1. The gateway is on every call — what's the risk and how do you mitigate it?
2. What do you log per call, and for which four purposes?

---

## Exercise 8 — Gateway vs API gateway
1. What does an LLM gateway add beyond a normal API gateway (m03)?
2. How do guardrails (m34) relate to the gateway?
3. How would you safely roll out a new model via the gateway?

---

## Exercise 9 — Build vs buy + prompts
1. Build vs buy — when each? Name two real gateways.
2. What does prompt management at the gateway give you?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map your **per-call cost tracker, $5 budget, "embeddings local / generation API"** to gateway features.
2. How is **semantic caching** your m06 caching + embeddings combined?
3. Design an org LLM gateway for a company running a chatbot (m31), RAG (m30), and agents (m32) — name the
   six things each call passes through.

---

When done, check `solutions/m33_llm_gateway/README.md`, then say **"Module 34"**.
