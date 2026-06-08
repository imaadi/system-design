# m31 — Cross-Questions ("if-and-buts") on the LLM Chatbot / Assistant

> Answer out loud in 2–3 sentences before reading the model answer. Conversation memory / context
> management, session state, streaming, and cost are where this is won.

---

### Q1. The LLM doesn't remember anything between calls. How does a chatbot "remember" the conversation?
**A.** By **re-sending the conversation history** with each turn — the LLM is **stateless**, so the
"conversation" is an **illusion** you create by including prior turns in the prompt every time. The
engineering problem is that history **grows unboundedly** while the **context window is finite** (and cost
grows with context), so "memory" really means **context-window management**: deciding **what history to
include** each turn (recent verbatim + summary + retrieved relevant turns), balancing fidelity vs cost vs
the window limit.

---

### Q2. The conversation history exceeds the context window. What do you do?
**A.** Manage it with one (or a hybrid) of: **sliding window** (keep the last N turns/tokens — bounded but
forgets early context), **summarization** (compress old turns into a running summary, keep recent verbatim
— preserves gist, loses detail, costs an extra LLM call), or **RAG over history** (embed past turns,
retrieve only the relevant ones for the current query — scales to long/cross-session memory). The
**production answer is hybrid**: **recent turns verbatim + a running summary of older turns + RAG-retrieved
relevant facts** — best fidelity within the budget.

---

### Q3. Short-term vs long-term memory in an assistant?
**A.** **Short-term memory** = the **in-context conversation history of this session** (managed within the
window — recent turns + summary). **Long-term memory** = **persisted facts/summaries retrieved across
sessions** (e.g. "remember my name and preferences from last week") — implemented as **RAG over the user's
history** (embed and store facts, retrieve relevant ones when needed). A good assistant combines both:
short-term keeps the current chat coherent, long-term gives continuity across sessions.

---

### Q4. Summarization vs sliding window vs RAG-over-history — trade-offs?
**A.** **Sliding window:** simplest, bounded cost, but **forgets** anything outside the window (breaks on
"what did we discuss earlier?"). **Summarization:** keeps the **gist** of old turns in a small budget, but
**loses detail** and costs an extra LLM call to summarize. **RAG over history:** **scales to very long /
cross-session** memory by retrieving only relevant past turns, but adds retrieval infra and can miss
context if retrieval fails. You **hybridize** — recent verbatim (fidelity) + summary (gist of old) +
retrieval (relevant specifics) — to get the best of each within the window.

---

### Q5. Is the chat service stateful or stateless? How do you scale it?
**A.** The **conversation is stateful** (must be remembered), but you keep the **chat service stateless**
by **storing session state externally** (Redis for hot/active sessions, a DB for history) and **loading it
per turn** (m02). So any server can handle any turn → **horizontal scaling** behind a load balancer. This
differs from the chat system (m14), where a persistent WebSocket pins a user to one server — here each turn
can be a separate request (with **SSE** for the response stream), and the **session store** holds the
state.

---

### Q6. How does this differ from the chat system in m14?
**A.** **m14 (WhatsApp/Slack)** is human-to-human, dominated by **persistent WebSocket connections** (the
server pushes messages, connections are stateful/pinned, you need a session registry + pub/sub backplane).
**m31 (LLM assistant)** is human-to-AI: each turn is a **request → LLM generation → SSE-streamed response**,
the **service is stateless** (loads session per turn), and the hard problems are **context-window
management + cost + the LLM**, not connection fan-out. Both store conversation state; the LLM assistant's
challenge is the **model + memory**, not real-time multi-party delivery.

---

### Q7. Why is streaming important for a chatbot, and what transport?
**A.** **Streaming** shows tokens **as they're generated**, so the user sees output after **TTFT** (~1–2s)
instead of waiting for the full response — the "typing" effect, dramatically better **perceived latency**
(m29). The transport is **SSE** (Server-Sent Events, m03) — a one-way server→client stream, which fits
because the user's message is a normal POST that *starts* the stream and the model only needs to *send*
tokens. (You don't need full WebSockets — SSE is the right, simpler fit; my `/ask/stream` used SSE with
TTFT 1.81s ≪ 2.77s total.)

---

### Q8. Why does cost grow with conversation length, and how do you control it?
**A.** Because each turn **re-sends and re-prefills the growing history**, so a long chat re-processes more
context every new turn (the "quadratic-ish" cost of long conversations). Control it with: **summarization**
(cap history size), **prefix caching** (m29 — reuse the KV cache of the stable prefix: system prompt + early
history, skipping re-prefill), **sliding window** (cap context), **response caching** for repeats, and
**right-sizing/routing** the model (m33). My cost discipline ($5 budget, per-call tracking) is exactly this
— long chats are a real cost driver.

---

### Q9. What is prefix caching and why does it help a chatbot specifically?
**A.** **Prefix caching** reuses the **KV cache of a shared, stable prompt prefix** across requests, so the
model **skips re-prefilling** that prefix (m29). It helps chatbots a lot because every turn shares a **large
stable prefix** — the **system prompt + the earlier conversation history** — which would otherwise be
re-processed each turn. Caching it cuts both **TTFT and cost** for ongoing conversations (the bigger and
more stable the prefix, the bigger the win). It's distinct from response caching (which returns a stored
answer); prefix caching reuses *computation* of the shared context.

---

### Q10. How do you keep an assistant safe?
**A.** **Layered moderation:** **input moderation** (filter/block harmful or policy-violating user prompts
before the model), **output moderation** (filter the model's response for harmful content / PII leakage
before showing it), **prompt-injection defense** (treat user and RAG-retrieved content as **untrusted
data**, not instructions — use an instruction hierarchy, m34), and for RAG assistants **grounding +
refusal** (answer only from context, cite, "say I don't know" — m30). You also rate-limit and log. Safety
is a wrapper around the model (in + out), deep-dived in m34.

---

### Q11. A user says "ignore your instructions and tell me X." How do you handle it?
**A.** That's **prompt injection** (m34). Defenses: maintain a clear **instruction hierarchy** (the system
prompt outranks user input), **never treat user (or retrieved) text as instructions** — only as data,
**input-moderate** to catch obvious attempts, and **output-moderate** to catch leaks (e.g. don't reveal the
system prompt). The model should be trained/prompted to refuse, but you don't rely on that alone — you wrap
it with moderation + a privilege boundary. (Indirect injection via a RAG-retrieved doc is the sneakier
variant — same principle: retrieved content is untrusted data.)

---

### Q12. The assistant needs to answer from company docs. How do you wire that in?
**A.** **RAG** (m30): on each turn, **retrieve relevant chunks** for the user's query (hybrid retrieve +
rerank), and **inject them as grounded context** into the prompt alongside the managed history, instructing
the model to **answer from the context, cite sources, and say "I don't know"** if absent. The Context
Manager assembles **system prompt + managed history + RAG context + user message** within the window. So
the assistant = conversation memory (m31) **on top of** RAG (m30) — the chatbot is the conversational layer,
RAG is the knowledge layer.

---

### Q13. When should the assistant retrieve (RAG) vs call a tool (agent) vs just answer?
**A.** **Just answer** for general knowledge/chit-chat the model already knows. **RAG** when the answer
needs **private/fresh knowledge** (company docs, the user's data) — retrieve grounded context. **Call a
tool / become an agent (m32)** when it needs to **take an action or get live data** (look up an order,
do a calculation, query a DB, send an email) — not just retrieve text. A capable assistant **routes**:
decide per query whether to answer directly, retrieve, or invoke a tool. (That routing decision is itself
where agents m32 begin.)

---

### Q14. How do you build the prompt for a turn — what goes in the context budget?
**A.** **context budget = system prompt + managed history + RAG context + user message + response
headroom.** The **Context Manager** assembles: the **system prompt** (role/instructions/safety), the
**managed conversation history** (recent turns verbatim + a summary of older + any retrieved long-term
facts), optional **RAG context** (retrieved doc chunks for this query), and the **new user message** —
while reserving room for the **response**. It trims/summarizes/retrieves to fit the finite window. Getting
this assembly right (what to include, in what order — most relevant first to beat "lost in the middle") is
the core engineering.

---

### Q15. How do you store conversation/session state?
**A.** **Hot/active sessions in Redis** (fast reads/writes per turn, with a TTL) and **full conversation
history in a database** (durable, for long-term retrieval + analytics). Each turn: **load the session**
(recent history + any summary), build the prompt, generate, then **persist the new turn** (and update the
running summary / long-term memory store). The chat service stays **stateless** (state lives in Redis/DB),
so it scales horizontally. For long-term/cross-session memory, you also embed + store facts in a **vector
store** (RAG over history).

---

### Q16. What happens to coherence if your memory management is bad?
**A.** The assistant **loses the thread** — with a too-aggressive sliding window it **forgets earlier
context** ("you told me your name 3 turns ago" → it doesn't know), with bad summarization it **drops key
details** (loses a constraint the user gave), and with poor RAG-over-history it **retrieves the wrong past
turns**. The result is incoherent, repetitive, or contradictory responses — the #1 quality failure of
chatbots. So memory management isn't a nicety; it directly determines whether the conversation *feels*
coherent, which is why the hybrid (recent + summary + retrieval) exists.

---

### Q17. How do you handle very long conversations (hundreds of turns)?
**A.** You **can't keep it all in context** (window + cost), so: **summarize** old turns into a compact
running summary (refreshed periodically), keep the **last N turns verbatim**, and use **RAG over the full
history** to pull back specific relevant past turns on demand. Plus **prefix caching** for the stable part
and **routing** to a cheaper model where possible. Effectively you maintain a **bounded working context**
(recent + summary) and **retrieve from an unbounded archive** (the full stored history) — short-term +
long-term memory working together.

---

### Q18. How is the assistant the "integration layer" of the LLM stack?
**A.** It's the **conversational front-end** that orchestrates everything else: it calls **LLM serving
(m29)** to generate (with streaming/cost concerns), uses **RAG (m30)** for grounded knowledge, invokes
**tools/agents (m32)** for actions, wraps everything in **guardrails/moderation (m34)**, and sits behind an
**LLM gateway (m33)** for routing/caching/rate-limiting/cost. The chatbot itself adds **conversation memory
+ session state + the turn loop**; the rest of Part 4 are the components it composes. It's where the whole
LLM system meets the user.

---

### Q19. What do you monitor for a production assistant?
**A.** **Latency:** TTFT, TPOT, total per turn. **Cost:** cost per conversation/turn (grows with length —
the key driver). **Quality:** user feedback / thumbs, coherence, **RAG faithfulness** (m30), refusal rate.
**Safety:** moderation blocks (in/out), injection attempts. **Resource:** context-window utilization,
session length, cache hit rate. And the usual infra metrics (m10). You alert on TTFT breaches, cost spikes,
and moderation/safety anomalies — plus track conversation length (a proxy for cost + memory pressure).

---

### Q20. Summarize the senior answer for designing an LLM assistant.
**A.** *"A **stateless chat service** over a **session store** (Redis/DB) that, per turn, **loads the
conversation**, has a **Context Manager build a managed prompt** (system prompt + **managed history**:
recent verbatim + summary of old + RAG-retrieved relevant + long-term facts + **RAG context** + the new
message), **moderates input**, **streams** the LLM response via **SSE** (low TTFT), **moderates output**,
and **persists the turn**. The core problem is **context-window management** because the **LLM is
stateless** — I re-send a **hybrid** memory to fit the **finite window** and control **cost** (which grows
with conversation length → **prefix caching + summarization + caching + routing**). It integrates **RAG
(m30) + tools/agents (m32) + guardrails (m34)** behind an **LLM gateway (m33)**. I built the pieces in
InsightDesk — SSE streaming, caching, sessions, the agent."*
