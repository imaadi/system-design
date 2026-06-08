# m31 — Solutions (LLM Chatbot / Assistant)

> Check the *reasoning*. Recurring lessons: the LLM is stateless → "memory" = context management; hybrid
> memory (recent + summary + retrieved); stateless service + session store; stream for TTFT; cost grows
> with length.

---

## Exercise 1 — The stateless LLM
1. By **re-sending the conversation history** in the prompt each turn — the LLM has no memory, so you
   reconstruct context every call.
2. History **grows unboundedly** while the **context window is finite** (and cost grows with context) — so
   you can't just keep appending forever.
3. Because "remembering" reduces to **deciding what history to include in the finite window** each turn
   (recent + summary + retrieved), balancing fidelity vs cost vs the limit — that's context-window
   management, not literal memory.

---

## Exercise 2 — Memory strategies
1. **Full history** (best fidelity, hits the limit + cost grows); **sliding window** (bounded, but
   **forgets** early context); **summarization** (compress old turns — gist kept, **detail lost**, extra
   LLM call); **RAG over history** (retrieve relevant past turns — **scales**, but needs retrieval infra +
   can miss).
2. **Hybrid:** **recent turns verbatim + a running summary of older turns + RAG-retrieved relevant facts**
   — best fidelity within the budget.
3. **Short-term** = in-context history of **this session** (recent + summary). **Long-term** = **persisted
   facts/summaries retrieved across sessions** (RAG over the user's history) — continuity beyond one chat.

---

## Exercise 3 — Session state
1. **Stateless service, stateful conversation** — the **state lives in an external store** (Redis for hot
   sessions, a DB for history).
2. The service **loads the session per turn** (m02), so **any server handles any turn** → horizontal scale
   behind a load balancer; the session store holds the state.
3. **m14** uses **persistent WebSocket connections** (pinned, stateful, needs a session registry + pub/sub
   backplane for multi-party real-time delivery); **m31** is request-per-turn with **SSE** for the
   response, a **stateless service + session store** — the hard problem is the **LLM + memory**, not
   connection fan-out.

---

## Exercise 4 — Streaming
1. **Stream** so the user sees output after **TTFT** (~1–2s) instead of waiting for the whole response — it
   improves **perceived latency** (the "typing" effect).
2. **SSE** (one-way server→client) — the user's message is a normal POST that *starts* the stream, and the
   model only needs to *send* tokens, so full bidirectional WebSockets are unnecessary.
3. My `/ask/stream` measured **TTFT 1.81s ≪ 2.77s total** — the user saw output at 1.81s.

---

## Exercise 5 — Cost
1. Each turn **re-sends and re-prefills the growing history**, so a long chat re-processes more context per
   new turn (the "quadratic-ish" cost of long conversations).
2. **Summarization** (cap history), **prefix caching** (reuse the stable prefix's KV cache), **response
   caching** (repeats), **right-size/route the model** (m33) — and cap context (sliding window).
3. **Prefix caching** reuses the **KV cache of the shared, stable prefix** (system prompt + earlier
   history) across turns → **skips re-prefilling** it. It helps chatbots especially because every turn
   shares a **large stable prefix**, so caching it cuts TTFT + cost.

---

## Exercise 6 — Building the prompt
1. **context budget = system prompt + managed history + RAG context + user message + response headroom**
   (managed history = recent verbatim + summary of old + retrieved long-term facts).
2. Because of **"lost in the middle"** (m30) — LLMs under-attend to the middle of a long context, so you
   put the **most relevant content first** (and last) so it isn't buried.
3. The **Context Manager** assembles + trims/summarizes/retrieves the history and context to **fit the
   finite window** and order it well — the core per-turn engineering.

---

## Exercise 7 — Safety
1. **Input moderation** (filter/block harmful user prompts before the model) and **output moderation**
   (filter the model's response for harmful content / PII before showing it).
2. **Prompt injection** — defend by treating user/retrieved text as **untrusted data, not instructions**,
   enforce an **instruction hierarchy** (system prompt outranks user input), and input/output-moderate
   (m34).
3. **Grounding + refusal:** answer **only from the retrieved context**, **cite sources**, and **say "I
   don't know"** if the answer isn't there (m30) — prevents hallucination.

---

## Exercise 8 — Routing
- **Just answer:** general knowledge / chit-chat the model already knows ("what's the capital of France?").
- **Retrieve (RAG):** needs **private/fresh knowledge** ("what's our refund policy?" → retrieve company
  docs).
- **Call a tool (agent, m32):** needs an **action or live data** ("what's the status of my order #123?" →
  query the orders DB; "book a meeting" → call the calendar API). A capable assistant **routes per query**.

---

## Exercise 9 — Long conversations & coherence
1. Keep a **bounded working context** — **last N turns verbatim + a running summary** of older turns — and
   **RAG over the full stored history** to pull back specific relevant past turns on demand; plus **prefix
   caching** + routing for cost.
2. Bad memory management → the assistant **loses the thread**: forgets earlier context (aggressive sliding
   window), drops key details (bad summarization), or retrieves wrong past turns → **incoherent/repetitive/
   contradictory** responses (the #1 chatbot quality failure).
3. **Short-term** keeps the current chat coherent (recent + summary in-context); **long-term** retrieves
   the returning user's **persisted facts/preferences** ("your name, your plan, last issue") via RAG over
   their history — together giving in-session coherence + cross-session continuity.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **InsightDesk → m31:** **`/ask/stream` (SSE)** = streaming + TTFT (§4); **response caching** (929×, $0)
   = cost (§5); **Redis** = the session store (§4); the **multi-tool agent** = tool integration/routing
   (§7, m32). You built the conversational pieces.
2. **RAG (m30) as the knowledge layer:** on each turn the assistant **retrieves relevant chunks** (hybrid +
   rerank) and injects them as **grounded context** into the managed prompt (answer-from-context + cite +
   refuse) — the chatbot adds conversation memory *on top of* RAG.
3. **Customer-support assistant over company docs:** **stateless chat service + Redis/DB sessions**; per
   turn **load session → Context Manager builds prompt** (system prompt + **recent turns + running summary
   + RAG-retrieved policy chunks** + user message) → **input-moderate → stream the LLM via SSE → output-
   moderate → persist turn**; **RAG (m30)** for grounded, **cited** answers with **refusal** when unknown;
   **long-term memory** (the customer's past tickets) via RAG over history; **cost** controlled via
   summarization + prefix caching + routing (m33); **tools (m32)** for actions like "check order status";
   **guardrails** (PII/prompt-injection, m34). Monitor TTFT, cost/conversation, faithfulness, moderation.
