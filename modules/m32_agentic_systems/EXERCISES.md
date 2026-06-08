# m32 — Exercises (Agentic Systems)

> Do these on paper / out loud before checking `solutions/m32_agentic_systems/`. Tool calling, the loop,
> reliability/control, and multi-agent are the core — and you built a LangGraph agent.

---

## Exercise 1 — What's an agent
1. Agent vs chatbot — the defining difference.
2. Name the four parts of an agent.
3. What does the LLM actually do vs what the tools do?

---

## Exercise 2 — Tool calling
1. Walk through one function-calling round trip.
2. Why do tool descriptions matter? Cite your own routing numbers.
3. What can go wrong with the structured tool call, and how do you guard it?

---

## Exercise 3 — The agent loop
1. Describe the ReAct loop's three repeating steps.
2. ReAct vs plan-then-execute — when each?
3. Why is "a loop, not a single call" the source of both power and unreliability?

---

## Exercise 4 — Reliability (the hard part)
1. List four ways agents fail.
2. Compute the end-to-end success of a 10-step agent at 95%/step. What does this imply?
3. List five controls you'd add.

---

## Exercise 5 — Dangerous actions
1. Which actions need human approval, and what's the propose-confirm pattern?
2. What is least-privilege for tools?
3. How does prompt injection become more dangerous for an agent than a chatbot?

---

## Exercise 6 — Multi-agent
1. When is multi-agent worth it vs a single agent?
2. What does multi-agent cost you?
3. Why is "default to a single well-tooled agent" the senior stance?

---

## Exercise 7 — Cost & latency
1. Why do cost and latency compound in agents?
2. List four ways to control them.
3. Why prefer a deterministic tool over an LLM step where possible?

---

## Exercise 8 — Orchestration & debugging
1. Why structure an agent as a graph (LangGraph)?
2. Your agent calls the wrong tool repeatedly — debug it (what do you check first?).
3. How do you evaluate a multi-step agent?

---

## Exercise 9 — Composition
1. How do an agent, RAG (m30), and a chatbot (m31) compose? When does the assistant route to each?
2. Name three tools a real agent might have and the controls on the dangerous ones.

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map your **InsightDesk LangGraph multi-tool agent** (run_agent loop, function calling, JSON mode,
   stats/sentiment/RAG routing, StateGraph, MCP) to this module.
2. What did your **tool-description experiment (3/6 vs 6/6)** teach, and which §?
3. Design a "resolve overdue invoices" agent end-to-end with the reliability/safety controls.

---

When done, check `solutions/m32_agentic_systems/README.md`, then say **"Module 33"**.
