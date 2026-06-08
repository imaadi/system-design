# m31 — Exercises (LLM Chatbot / Assistant)

> Do these on paper / out loud before checking `solutions/m31_llm_chatbot/`. Conversation memory / context
> management, session state, streaming, and cost are the core.

---

## Exercise 1 — The stateless LLM
1. The LLM has no memory. How does the chatbot "remember" the conversation?
2. What's the central engineering problem this creates?
3. Why is "memory" really "context-window management"?

---

## Exercise 2 — Memory strategies
1. Name four ways to manage growing history and the trade-off of each.
2. What's the production (hybrid) approach?
3. Short-term vs long-term memory — define each and how each is implemented.

---

## Exercise 3 — Session state
1. Is the chat service stateful or stateless? Where does the state live?
2. How do you scale the service horizontally given conversations are stateful?
3. How does this differ from the m14 chat system?

---

## Exercise 4 — Streaming
1. Why stream, and which metric does it improve?
2. What transport, and why not WebSockets?
3. Cite your own TTFT numbers.

---

## Exercise 5 — Cost
1. Why does cost grow with conversation length?
2. List four cost levers.
3. What is prefix caching and why does it help a chatbot specifically?

---

## Exercise 6 — Building the prompt
1. What goes into the context budget for a turn?
2. Why order context "most relevant first"?
3. What does the Context Manager do?

---

## Exercise 7 — Safety
1. Name the two moderation points and what each filters.
2. A user (or a retrieved doc) says "ignore your instructions." What is this and how do you defend?
3. For a RAG assistant, what grounding behaviors prevent hallucination?

---

## Exercise 8 — Routing
When should the assistant: just answer / retrieve (RAG) / call a tool (agent)? Give an example of each.

---

## Exercise 9 — Long conversations & coherence
1. How do you handle a 300-turn conversation?
2. What breaks if memory management is bad?
3. How do short-term + long-term memory work together for a returning user?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map InsightDesk's **`/ask/stream` (SSE), caching, Redis sessions, the agent** to this module.
2. How does your **RAG** (m30) plug into the assistant as the knowledge layer?
3. Design a customer-support assistant over company docs end-to-end (memory, RAG, session, streaming,
   cost, safety).

---

When done, check `solutions/m31_llm_chatbot/README.md`, then say **"Module 32"**.
