# Module 32 — Design Agentic Systems ⭐⭐ 🏭

> An **agent** doesn't just answer — it **acts**: it uses **tools** to take **multi-step actions** toward a
> goal, with the **LLM as the reasoning engine** deciding *what to do next* and **tools as the hands** that
> do it. The core idea is **the agent loop** (Reason → Act → Observe → repeat). The hard truth: **agents
> are powerful but *unreliable*** — they loop, hallucinate tool calls, take wrong/destructive actions, and
> **compound errors** — so the real engineering is **control + reliability**. You built this in InsightDesk
> (a **multi-tool LangGraph agent**, function calling, the lesson that **tool descriptions decide
> routing**).

> **Format (Part 4 deep-dive):** worked design; **tool/function calling**, the **agent loop**, **reliability/
> control**, and **multi-agent** are the deep-dives.

---

## 1. What makes it an "agent" + requirements
An agent = **LLM (reasoning) + tools (actions) + a loop (control) + memory (state).** It **decides actions,
executes them, observes results, and iterates** until the goal is met — vs a chatbot that just responds.
- **Functional:** accomplish a **multi-step task using tools** (e.g. "find my overdue invoices and email
  the clients" → query DB → draft emails → send).
- **Non-functional:** **reliability** (the loop can fail/runaway — *the* hard one), **latency** (multi-step
  = slow), **cost** (multi-step = many LLM calls = expensive), **safety** (tools can do real damage),
  **observability** (debug a multi-step trace).

> **The defining insight:** the **LLM doesn't do the work — it decides what to do** (which tool, what
> args); the **tools do it.** Power comes from the **loop**; so does the **unreliability** — which is why
> the design is mostly about **controlling the loop.**

---

## 2. Deep Dive: tool / function calling ⭐ (the foundation)
**Function calling** is how the LLM acts: you give it a set of **tools** (functions/APIs) with **names +
descriptions + argument schemas**; the LLM outputs a **structured request** to call a tool with arguments
(JSON); your system **executes the tool** and returns the **result** to the LLM, which decides the next
step.
```
  LLM ──"call get_invoices(status='overdue')"──▶ your system executes the tool ──result──▶ LLM (next step)
```
- **Tool descriptions are the API contract for the LLM** ⭐ — the LLM picks which tool by its **name +
  description**, so **vague descriptions → wrong tool selection.** *(Your measured lesson: vague tool
  descriptions routed **3/6** correctly vs good descriptions **6/6**.)* Agent "prompt engineering" is
  largely **tool-description engineering**.
- **Structured output (JSON mode):** the tool call must be **valid structured output** (correct tool, valid
  args) — you validate it (and handle the LLM hallucinating a non-existent tool / bad args).

---

## 3. Deep Dive: the agent loop ⭐ (ReAct)
The classic pattern is **ReAct (Reason + Act):**
```
  loop:
    REASON  — LLM thinks: what's the next step toward the goal?
    ACT     — LLM calls a tool (function call)
    OBSERVE — system executes the tool, returns the result to the LLM
  until the LLM decides the goal is achieved (or a limit is hit)
```
- The LLM **uses each tool result to inform the next step** — it's a **loop, not a single call**, which is
  the source of both **capability** (multi-step reasoning + action) and **unreliability** (§5).
- **Planning variants:** **ReAct** (interleave reason/act, decide each step) vs **plan-then-execute**
  (decompose the goal into a plan up front, then execute steps) — plan-execute is more structured/
  predictable; ReAct is more adaptive. **Reflection** (the agent critiques + retries its own output)
  improves quality.
- *(Your InsightDesk `run_agent` loop + multi-tool routing — stats(pandas) / sentiment-model(joblib) /
  RAG — is exactly this.)*

---

## 4. Deep Dive: orchestration & multi-agent ⭐
**Orchestration** = how you structure the agent's control flow:
- **Single agent loop** (ReAct) — simplest; one LLM + tools in a loop. Often enough.
- **Plan-execute** — a planner decomposes, an executor runs steps. More predictable.
- **Graph-based (LangGraph `StateGraph`)** — model the agent as a **graph of nodes (steps) + edges
  (transitions)** with explicit state → controllable, inspectable, supports cycles/branches. *(Your
  LangGraph StateGraph work.)*

**Multi-agent** ⭐ — multiple **specialized agents** collaborating:
- **Orchestrator + workers** (a manager delegates sub-tasks to specialist agents), **hierarchical**, or
  **debate/critique** (agents review each other).
- **When worth it:** tasks that genuinely **decompose into specialized roles** (researcher + coder +
  reviewer). **When NOT:** most tasks — multi-agent adds **coordination overhead + more failure modes +
  more cost**, so **a single well-tooled agent is often better.** *Don't over-engineer into multi-agent.*

> **Say this:** *"I'd start with a **single ReAct agent** (LLM + well-described tools + a loop), structured
> as a **graph (LangGraph) for controllability**. I'd only go **multi-agent** when the task truly
> decomposes into specialist roles — it adds coordination cost and failure modes, so it's not a default."*

---

## 5. Deep Dive: reliability & control ⭐⭐ (THE hard part — why agent demos don't ship)
Agents are **unreliable**: they **loop forever**, **hallucinate tool calls** (call non-existent tools / bad
args), take **wrong or destructive actions**, and **compound errors** (a wrong early step derails the rest).
The engineering is **controlling the loop:**
- **Loop limits / max iterations + timeouts:** cap steps so a runaway agent halts (never an unbounded
  loop).
- **Validation:** validate every tool call (valid tool? valid args? — JSON-schema check) and tool result
  before proceeding.
- **Human-in-the-loop approval** ⭐: for **dangerous/irreversible actions** (send money, delete data, email
  customers), require **human approval** — the agent proposes, a human confirms (like m21 payments / m27
  fraud review). **Least-privilege tools** (the agent can only do what it's explicitly allowed).
- **Cost ceilings:** cap total LLM calls / spend per task (multi-step cost compounds — §6).
- **Retries / fallbacks / guardrails:** retry a failed tool, fall back gracefully, and wrap with guardrails
  (m34 — and prompt injection: a tool result / retrieved doc could hijack the agent).

> **Error compounding (the key math) ⭐:** each step has some success probability, and they **multiply** — a
> **10-step** task at **95%/step** reliability = **0.95¹⁰ ≈ 60%** end-to-end success. So **long agent
> chains are fragile** → keep them **short**, **validate each step**, add **checkpoints/human approval**,
> and prefer **deterministic tools** over LLM steps where possible. *"Agents are powerful but unreliable;
> the work is making them reliable enough to ship."*

---

## 6. Cost, latency & the architecture
- **Cost + latency compound:** each step is an **LLM call**, so a 10-step agent ≈ **10× the calls** = 10×
  cost + latency. Mitigations: **cheaper models for sub-steps** (route — m33), **cap steps**, **parallelize
  independent tool calls**, **cache**, and **prefer deterministic tools** (don't use an LLM step for what a
  function can do). Agents are **expensive + slow** — budget for it.
- **Architecture:** **LLM (reasoning)** + **tool registry (functions/APIs with schemas)** + **memory/state**
  (working state across steps + conversation, m31) + **orchestrator (the control loop / graph)** +
  **guardrails (m34)** + **observability (traces)**. Tools may include **RAG (m30)**, DB queries,
  calculators, external APIs.
- **MCP (Model Context Protocol):** an emerging **standard for connecting LLMs to tools/data sources** —
  standardizes the tool interface (your MCP note).

**Wrap-up:** *"An agent = **LLM (reasoning) + tools (actions) + a control loop + state.** Via **function
calling**, the LLM emits structured tool calls (and **tool descriptions decide routing** — vague → wrong
tool, my 3/6-vs-6/6 lesson), executed in a **ReAct loop** (Reason → Act → Observe → repeat), often
structured as a **graph (LangGraph)** for control. The hard part is **reliability**: agents loop,
hallucinate calls, and **compound errors** (95%/step over 10 steps ≈ 60% success), so I add **loop limits +
timeouts, validation, human approval for dangerous actions, least-privilege tools, cost ceilings, and
guardrails (m34)**, keep chains **short**, and prefer **deterministic tools**. **Cost/latency compound**
(many LLM calls → route to cheaper models, parallelize, cache). I'd use a **single well-tooled agent** by
default and **multi-agent only when the task truly decomposes** into specialist roles. I built this — a
multi-tool LangGraph agent — in InsightDesk."*

---

## State-of-the-art & real-world notes 📚
- **ReAct (Yao et al.)** — the reason+act loop. **Function/tool calling** (OpenAI/Anthropic) — the action
  mechanism. **LangGraph** (graph-based agent orchestration), **LangChain agents**, **AutoGen / CrewAI**
  (multi-agent), **AutoGPT** (the early autonomous-agent demo — and its unreliability lessons).
- **MCP (Model Context Protocol, Anthropic)** — standardizing tool/data connections for LLMs.
- **Reflection / self-critique**, **plan-execute**, **tool use + sandboxing** — the techniques making
  agents more reliable. The industry consensus: **reliability + control are the gating problem**, not
  capability.

---

## From your systems 🏭 (you built an agent)
- **InsightDesk multi-tool agent = this module:** a **LangGraph `StateGraph`** agent with a **`run_agent`
  tool-call loop**, **function-calling protocol**, **JSON-mode structured output**, and **multi-tool
  routing** (stats(pandas) / sentiment-model(joblib) / RAG) — §2–§4 hands-on. And the **MCP concept**.
- **Tool descriptions decide routing (your measured lesson):** **vague descriptions 3/6 vs good
  descriptions 6/6** — exactly §2's "tool-description engineering."
- **LLM + calculator tool** (your m01 AI note): "function calling = agents" — the foundational insight.
- **RAG as a tool:** your agent could call RAG (m30) — the integration of m30/m31/m32.
- **Cost awareness:** multi-step cost (each call tracked) = §6's compounding cost concern.

---

## Key concepts (interview-ready)
- **Agent = LLM (reasoning) + tools (actions) + loop (control) + state.** The LLM **decides what to do**;
  tools **do it**. Power + unreliability both come from the **loop**.
- **Function/tool calling:** the LLM emits a **structured tool call** (JSON, validated); **tool
  descriptions are the contract** — vague → wrong routing (3/6 vs 6/6).
- **The agent loop (ReAct):** **Reason → Act → Observe → repeat** until done. Variants: **plan-execute**
  (structured), **reflection** (self-critique). Orchestrate as a **graph (LangGraph)** for control.
- **Reliability is THE hard part:** agents loop, hallucinate calls, take wrong actions, and **compound
  errors** (0.95¹⁰ ≈ 60%) → **loop limits + timeouts, validation, human-approval for dangerous actions,
  least-privilege tools, cost ceilings, retries/guardrails (m34)**; keep chains short; prefer deterministic
  tools.
- **Cost + latency compound** (many LLM calls) → route to **cheaper models for sub-steps (m33)**,
  parallelize independent calls, cache, cap steps.
- **Single vs multi-agent:** default to a **single well-tooled agent**; go **multi-agent only when the task
  truly decomposes** into specialist roles (coordination cost + more failure modes). **MCP** standardizes
  tool/data connections.

---

## Go deeper (reading)
- **ReAct (Yao et al.)** and **Toolformer** — the foundational reason+act / tool-use papers.
- **OpenAI/Anthropic function-calling docs**; **LangGraph** (graph agents) + **LangChain agents**;
  **AutoGen / CrewAI** (multi-agent).
- **Anthropic "Building effective agents"** — the practical "keep it simple, single agent first,
  reliability over cleverness" guidance.
- **MCP (Model Context Protocol)** docs — standard tool/data connections.
- **Your own AI/ML notes (`../ai_ml_llm/` m08 agents, m01 LLM+tool)** — you built the multi-tool LangGraph
  agent; revisit it. Plus **m30** (RAG-as-a-tool), **m33** (routing/cost), **m34** (guardrails/injection).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m32_agentic_systems/`](../../solutions/m32_agentic_systems/README.md).

When you've done the exercises, say **"Module 33"** to design an *LLM Gateway / Platform* — model routing,
semantic caching, rate limiting/quotas, fallbacks, cost control, and observability (the shared
infrastructure all your LLM apps run behind).
