# m36 — Self-Scoring Rubric + Final Prep Checklist

> How to **grade your own mocks** honestly, plus the **final pre-interview checklist.** This is the page you
> open after every practice mock and the night before the real thing.

---

## The full scoring rubric (rate 1–5 per dimension)

### 1. Requirements & NFRs
- **5:** Clarified FRs, **named the dominant NFR(s)** (and *why* they dominate), scoped down explicitly,
  surfaced the key constraint before designing.
- **3:** Listed some requirements but didn't identify what *drives* the design.
- **1:** Jumped to architecture; no scoping.
> *The single highest-signal phase. A great answer makes everything downstream follow from the NFRs.*

### 2. Estimation
- **5:** Computed QPS/storage/bandwidth **and used them** to name the constraint (read-heavy? hot keys?
  too big for one node?).
- **3:** Did some math but it didn't influence decisions.
- **1:** Skipped, or numbers purely decorative.

### 3. API & data model
- **5:** Clean key endpoints; entities; **chose storage by access pattern** with a stated reason (polyglot
  where it fits).
- **3:** Reasonable model; DB choice under-justified.
- **1:** Hand-waved data; "we'll use Postgres" with no reason.

### 4. High-level design
- **5:** A **working end-to-end system**, **one request traced** through it, before optimizing.
- **3:** Boxes drawn but flow unclear, or optimized prematurely.
- **1:** No coherent end-to-end picture.

### 5. Deep dive
- **5:** Went **3 levels** into 1–2 components, presented **options + trade-offs + a justified pick**,
  named the **pattern**.
- **3:** Some depth, but shallow on trade-offs or only one option.
- **1:** Stayed surface-level everywhere.

### 6. Bottlenecks, scale & failure
- **5:** Named **SPOFs**, scaled **the constraint** specifically, surfaced **failure modes + ops**
  unprompted, sketched evolution.
- **3:** Mentioned scaling generically.
- **1:** Ignored failure/ops entirely.

### 7. Communication & seniority
- **5:** **Drove** the session, **checked in** ("want me deeper on X?"), quoted **numbers**, said **"it
  depends on [X]"** and resolved it, named **patterns**, admitted unknowns gracefully.
- **3:** Communicated adequately but passive or monologued.
- **1:** Silent, rambling, or defensive.

> **Interview-ready = 4+ on every dimension, cold, in 45 minutes.** Track your weakest dimension across
> mocks — that's your highest-ROI practice target.

---

## Reading your score
- **All 4–5:** ready. Keep sharp with rapid-fire + your m35 pitches.
- **A 3:** drill that *phase* (not the whole topic) — e.g. weak estimation → redo m02 §estimation + 5
  prompts focusing only on the numbers.
- **A 1–2 anywhere:** re-read that module's README + CROSS_QUESTIONS before more mocks; the gap is
  knowledge, not just delivery.
- **Pattern across mocks:** most people's weak spots are **(a) not letting NFRs drive decisions** and **(b)
  ignoring failure modes** — check those first.

---

## The final pre-interview checklist
**Reflexes (must be automatic):**
- [ ] The **7-step framework** (m01 §2 skeleton) — can start *any* prompt with it.
- [ ] The **numbers** (m02 / m36 §4) — latency ladder, QPS rules, availability, 1 day ≈ 10⁵ s.
- [ ] The **decision trees** (m36 §5) — SQL/NoSQL, consistency, cache policy, sync/async, push/pull,
  REST/gRPC, RAG/fine-tune.
- [ ] The **patterns** (m36 §6) — can name them on sight.

**Stories (fluent, ≤ 90s each):**
- [ ] **DCE** pitch + can drive the full framework on it (m35).
- [ ] **Questimate** pitch + walkthrough (m35).
- [ ] **InsightDesk** pitch (RAG/agents/eval/MLOps) for AI/LLM roles.
- [ ] A **"hardest technical problem"** story (BDT pricer / DCE correctness).
- [ ] A **"what I'd do differently"** for each system.
- [ ] **Role-matched:** know which system to lead with for *this* company.

**Mindset:**
- [ ] I **drive** the conversation; I **check in**; I quote a **number**; I name the **trade-off**.
- [ ] "There's no best, only best-for-these-requirements."
- [ ] When challenged: **curiosity + both sides**, not defense.
- [ ] My edge is **real, lived scale + correctness + a full AI stack** — own it.

**Logistics:** company/role researched · which system to lead with · 2–3 questions to ask them · whiteboard/
tool tested (if remote) · slept.

---

## Mock prompt solutions — where the depth lives
Every Bank prompt has a full worked solution **already in the curriculum**:
| Bank | Prompts | Where the worked design lives |
|---|---|---|
| A | 1–11 | **m11–m21** READMEs (each is a full worked walkthrough) + their CROSS_QUESTIONS |
| B | 13–19 | **m22–m28** READMEs |
| C | 20–26 | **m29–m34** READMEs |
| D | 27–30 | **m35** README + story bank |
> So "solutions" to the mocks = the modules themselves. After a mock, open the matching module and compare
> your flow to the worked one — note what you missed.

---

## 🎓 Curriculum complete (m01–m36)
You built — and now own — a **36-module, state-of-the-art system-design body of knowledge**: the **method**
(m01), the **fundamentals + numbers** (m02), **8 building blocks** (m03–m10), **11 classic problems**
(m11–m21), **ML system design** (m22–m28), **LLM system design** (m29–m34), and **your own systems +
interview playbook** (m35–m36) — every module with deep notes, cross-questions, exercises, and solutions,
woven through with **your real work** at every step.

You have the **method, the vocabulary, the patterns, the numbers, and your war stories.** Re-run the mocks
before each interview, lead with the right system, drive the conversation — and go get the role. 🚀🏁
