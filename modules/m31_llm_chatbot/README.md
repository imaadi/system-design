# Module 31 — Design an LLM Chatbot / Assistant ⭐⭐ 🏭

> A conversational AI assistant — **multi-turn**, **remembers context**, **streams** responses, stays
> **safe**, and may **retrieve (RAG, m30)** or **call tools (agents, m32)**. The make-or-break insight:
> **the LLM is *stateless* per call — it has no memory.** The "conversation" is an **illusion you create by
> re-sending the history each turn**, so the central engineering problem is **context-window management**
> (the window is finite + costs grow with it, but conversations grow unboundedly). You built the pieces in
> InsightDesk (`/ask`, **`/ask/stream` SSE streaming**, caching, the agent).

> **Format (Part 4 deep-dive):** worked design; **conversation memory / context management**, **session
> state**, **streaming**, and **moderation** are the deep-dives.

---

## 1. Requirements
**Functional:** **multi-turn conversation** that **remembers context** (within a session, ideally across
sessions), **streams** responses, and may use **RAG/tools**.
**Non-functional:** **low latency / TTFT** (m29 — stream it), **scale** (many concurrent conversations),
**cost** (grows with conversation length — §5), **safety/moderation** (input + output, m34), and **session
state** (the conversation is stateful).

> **The defining insight:** the LLM is **stateless** — so "memory" = **deciding what conversation history
> to fit in the finite context window each turn**, balancing **completeness vs cost vs the window limit.**

---

## 2. Architecture
```
  client ──(SSE stream)── Chat Service (stateless) ──▶ Context Manager (build the prompt)
                              │ load/save session            │ system prompt + history (managed) + RAG context
                              ▼                              ▼
                        Session/Memory store (Redis/DB)   LLM (m29, streaming) ──▶ tokens ──▶ client
                              ▲                              │
                              └─ persist turn ◀──────────────┘   (+ moderation in/out, + RAG m30 / tools m32)
```
- Each turn: **load the conversation/session** (from Redis/DB) → **Context Manager builds the prompt**
  (system prompt + **managed history** + optional **RAG context** + the new user message) → **moderate
  input** → **LLM generates (streamed)** → **moderate output** → **persist the turn**.
- **The chat service is stateless** (m02): it **loads session state per turn**, so any server handles any
  turn → easy horizontal scaling. *(Unlike m14's persistent WebSocket — here each turn can be a separate
  request, with SSE for the response stream.)*

---

## 3. Deep Dive: conversation memory & context-window management ⭐⭐ (the core)
The LLM has **no memory between calls** — you **re-send the conversation history** each turn so it "sees"
the context. But history **grows every turn**, and the **context window is finite** (and **cost grows with
context** — m29). So you must **manage what history goes into the window**:
- **Full history** (until the limit): simplest, best fidelity — but hits the window + cost grows fast.
- **Sliding window** (last N turns / tokens): drop the oldest — simple, bounded — but **forgets early
  context**.
- **Summarization** ⭐: **compress old turns into a running summary**, keep recent turns verbatim →
  preserves gist within a small budget — but summarization **loses detail** and costs an extra LLM call.
- **RAG over history** ⭐: **embed + store past turns**, and **retrieve only the relevant ones** for the
  current query (the m30 funnel applied to conversation history) → scales to very long/cross-session memory.
- **Hybrid (the production answer):** **recent turns verbatim + a running summary of older turns +
  retrieved relevant facts** — best fidelity within the budget.

```
  context budget = system prompt + [managed history] + [RAG context] + user msg + response headroom
  managed history = recent turns (verbatim) + summary (old turns) + retrieved relevant past turns
```

> **Short-term vs long-term memory:** **short-term** = the in-context history of **this session** (managed
> above); **long-term** = **persisted facts/summaries retrieved across sessions** (RAG over the user's
> history — "remember my name/preferences from last week"). A good assistant combines both.

> **Say this:** *"The LLM is stateless, so 'memory' is **context management** — I re-send a **managed**
> history each turn: **recent turns verbatim + a running summary of older turns + RAG-retrieved relevant
> past turns**, fitting the **finite window** and controlling **cost** (which grows with context). Sliding
> window forgets, summarization loses detail, RAG-over-history scales — so I hybridize."*

---

## 4. Deep Dive: session state & streaming
- **Session state (statefulness done right):** the conversation **must be stored** (Redis for hot/active
  sessions, a DB for history) — but the **chat service stays stateless** by **loading the session per
  turn** (m02). This decouples it from m14's persistent-connection model: here each turn is a request
  (with SSE for the response), so you scale the service horizontally and the **session store** holds the
  state. (You may use a WebSocket for a live bidirectional UI, but the *server* stays stateless via the
  session store.)
- **Streaming (SSE, m03/m29):** stream tokens as generated → the user sees output after **TTFT** (~1–2s,
  your measured lesson) instead of waiting for the full response → the "typing" effect, far better
  perceived latency. SSE (one-way server→client) is the right transport — the user's message is a normal
  POST that *starts* the stream.

---

## 5. Deep Dive: cost & the growing-conversation problem ⭐
Every turn **re-sends (and re-prefills) the growing history**, so **cost grows with conversation length**
(a 50-turn chat re-processes ~50 turns of context each new turn — the "quadratic-ish" cost of long chats):
- **Summarize** old turns (cap the history size — §3).
- **Prefix caching** (m29): reuse the **KV cache of the stable prefix** (system prompt + early history)
  across turns → skip re-prefilling it → big cost/latency saving for long, stable-prefix chats.
- **Cache** identical responses (your `/ask` 929×, $0), **right-size the model** (route — m33), and **cap
  context** (sliding window).
- *(Your cost discipline — $5 budget, per-call tracking — is exactly this concern.)*

---

## 6. Deep Dive: safety / moderation (input + output)
- **Input moderation:** filter/block harmful or policy-violating **user prompts** before they reach the
  model (a moderation classifier/API).
- **Output moderation:** filter the **model's response** before showing it (harmful content, PII leakage).
- **Prompt injection** (m34): a user (or a RAG-retrieved doc) tries to **override the system prompt**
  ("ignore your instructions…") → treat user/retrieved content as **untrusted data**, use an instruction
  hierarchy, and filter.
- **Refusal & grounding:** for a RAG assistant, **answer from context + cite + say "I don't know"** (m30) —
  anti-hallucination. (Deep-dived in m34.)

---

## 7. The assistant as the integration layer + bottlenecks
The chatbot is the **conversational front-end** that orchestrates the LLM stack:
- **RAG (m30)** for grounded knowledge, **tools/agents (m32)** for actions, **serving (m29)** for
  generation, **guardrails (m34)** for safety, behind an **LLM gateway (m33)** for routing/caching/cost.
- **Bottlenecks:** the **context window** (the hard limit → memory management), **cost** (growing history →
  summarize/prefix-cache/route), **latency** (TTFT → stream + smaller model), **memory-management quality**
  (summarization loses info; bad history selection breaks coherence), and **safety** (moderation +
  injection).
- **Observability (m10):** TTFT/TPOT, cost per conversation, context-window utilization, moderation
  blocks, session length, user satisfaction.

**Wrap-up:** *"An assistant is a **stateless chat service** over a **session store** (Redis/DB) that, each
turn, **loads the conversation**, **builds a managed prompt** (system prompt + **managed history** + **RAG
context** + the new message), **moderates input**, **streams** the LLM response (SSE — low TTFT),
**moderates output**, and **persists the turn**. The core problem is **context-window management** — the
LLM is **stateless**, so I re-send a **hybrid** history (**recent verbatim + summary of old + RAG-retrieved
relevant** turns) to fit the **finite window** and control **cost** (which grows with conversation length →
**prefix caching** + summarization + caching). It integrates **RAG (m30)**, **tools/agents (m32)**, and
**guardrails (m34)** behind an **LLM gateway (m33)**. I built the pieces — `/ask/stream` SSE, caching,
sessions, the agent — in InsightDesk."*

---

## State-of-the-art & real-world notes 📚
- **ChatGPT / Claude / Gemini assistants:** managed conversation memory (recent + summary + retrieved
  long-term memory), streaming, tool use, moderation. **LangChain/LlamaIndex memory** modules (buffer,
  summary, vector-retrieval memory) productize §3.
- **Long-term memory** (cross-session — "memory" features) = RAG over a user's history/facts. **Prefix/
  prompt caching** (Anthropic/OpenAI) cuts the cost of long stable prefixes (§5).
- **Moderation APIs** (OpenAI moderation, Llama Guard) for input/output safety (§6/m34).

---

## From your systems 🏭
- **You built the conversational pieces:** InsightDesk's **`/ask` and `/ask/stream` (SSE streaming)** =
  §4 streaming + TTFT (1.81s); **response caching** (929×, $0) = §5 cost; the **multi-tool agent** =
  §7 tool integration (m32); **Redis** = §4 session store.
- **RAG-as-knowledge:** your InsightDesk RAG (m30) is the assistant's grounding layer (answer from context,
  cite, refuse).
- **Cost discipline:** $5 budget + per-call cost tracking = §5's growing-conversation cost concern.
- **Stateless service + Redis state:** your FastAPI stateless app + Redis sessions = §4 (state in Redis,
  service stateless) — the m02 pattern.

---

## Key concepts (interview-ready)
- **The LLM is stateless per call** → "memory" is **context-window management**: re-send a **managed**
  history each turn within the **finite window** (and **cost grows with context**).
- **Memory strategies:** full history (limit-bound), **sliding window** (forgets), **summarization**
  (compress old turns, loses detail), **RAG over history** (retrieve relevant past turns, scales) →
  **hybrid** (recent verbatim + summary + retrieved). **Short-term (in-context) vs long-term (persisted +
  retrieved across sessions).**
- **Session state:** store the conversation (Redis/DB), keep the **chat service stateless** (load per turn,
  m02) → horizontal scale. **Streaming (SSE)** for TTFT/perceived latency (m03/m29).
- **Cost** grows with conversation length (re-prefill the history each turn) → **prefix caching** (m29),
  **summarization**, **response caching**, **right-size/route** (m33), cap context.
- **Safety:** **input + output moderation**, **prompt injection** defense (untrusted data, m34), grounding/
  refusal for RAG assistants.
- **The integration layer:** orchestrates **RAG (m30) + tools/agents (m32) + serving (m29) + guardrails
  (m34)** behind an **LLM gateway (m33)**.

---

## Go deeper (reading)
- **LangChain / LlamaIndex memory** docs (buffer / summary / vector-retrieval memory) — §3 patterns.
- **Anthropic/OpenAI prompt (prefix) caching** docs — §5 cost.
- **OpenAI moderation / Llama Guard** — §6 safety (and m34).
- **Your own AI/ML notes (`../ai_ml_llm/` m07 RAG, m08 agents, m11 streaming/serving)** — the conversational
  pieces you built.
- Revisit **m29** (serving/streaming/cost), **m30** (RAG grounding), **m32** (tools/agents), **m33** (LLM
  gateway), **m34** (guardrails/moderation/injection), **m14** (the persistent-connection contrast).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m31_llm_chatbot/`](../../solutions/m31_llm_chatbot/README.md).

When you've done the exercises, say **"Module 32"** to design *Agentic Systems* — tool/function calling,
orchestration, planning, multi-agent, and reliability/loop-control (the assistant that *acts*, not just
answers — your LangGraph agent work).
