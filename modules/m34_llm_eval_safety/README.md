# Module 34 — LLM Evaluation & Safety in Production ⭐⭐ 🏭 — *closes Part 4*

> How do you know an LLM system is **good** (evaluation) and **safe** (guardrails)? This is the quality +
> safety layer that **wraps everything in Part 4** — and it's hard because LLM outputs are **open-ended and
> non-deterministic** (no single right answer), so accuracy/exact-match don't work. The big ideas:
> **LLM-as-judge** (the dominant eval method, with biases), **guardrails** (input/output safety),
> **prompt injection** (the #1 unsolved security problem), **PII/PHI handling**, and **regression gates**.
> You built LLM eval from scratch — **faithfulness/recall, LLM-as-judge + its biases, a CI regression gate
> that failed a degraded config** — so this is your evaluation strength.

> **Format (Part 4 finale):** worked design; **LLM-as-judge**, **prompt injection**, **guardrails**, and
> **regression gates** are the deep-dives.

---

## 1. Why LLM evaluation is hard ⭐⭐ (it inverts traditional ML eval)
- **No single ground truth** for open-ended generation — many answers can be correct, in different words.
- **Non-deterministic** — the same prompt gives **different outputs** each time.
- **Quality is subjective + multi-dimensional** (correctness, relevance, coherence, tone, safety, format).
- So **accuracy / exact-match / BLEU/ROUGE** (which compare to a fixed reference) are **weak or useless**
  for generation. You need **LLM-as-judge, human eval, or task-specific proxies** — and you have to
  **engineer measurability** (which is why "any production RAG has a recall metric", m30, is hard-won).

> **The framing:** *"LLM eval is hard because outputs are **open-ended + non-deterministic** with **no
> single ground truth** — so I can't use accuracy. I use **task-specific proxies** (RAG: faithfulness/
> recall), **LLM-as-judge** (with bias mitigation), and **human eval** as the gold standard, on a **fixed
> eval set gated in CI** — plus **online** signals (A/B + user feedback) because offline is a proxy (m22)."*

---

## 2. Evaluation approaches ⭐
- **Reference-based** (compare to a gold answer): **BLEU/ROUGE/exact-match** — okay for narrow tasks
  (translation, extraction), **weak for open-ended** generation (penalizes valid paraphrases).
- **LLM-as-judge ⭐⭐ (the dominant approach):** use a **strong LLM to score outputs** against criteria
  (correctness, relevance, faithfulness, on a rubric/scale, or pairwise A-vs-B). Scalable + cheap vs
  humans, and **correlates reasonably with human judgment** — *but it has biases* (§3).
- **Human evaluation:** the **gold standard** (and the calibration target for LLM-judges), but **expensive +
  slow** — use it to validate your automated metrics, not for every change.
- **Task-specific metrics:** **RAG → faithfulness, context precision/recall, answer relevancy** (your m09/
  m30 work); classification → accuracy/F1; etc.
- **Eval harness + eval sets:** a **fixed test set** of (input, criteria/expected) run on every change, with
  a **regression gate** (§6) — the m22/m28/m30 discipline.

---

## 3. Deep Dive: LLM-as-judge & its biases ⭐⭐
Using an LLM to grade outputs is powerful but **a model grading a model** — know the biases:
- **Position bias:** prefers the **first** (or a fixed position) option in pairwise comparison → **randomize
  / swap positions and average.**
- **Verbosity bias:** prefers **longer** answers → control for length / instruct to ignore length.
- **Self-preference bias:** prefers outputs in **its own style / from its own model family** → use a
  **different judge model**, or an ensemble.
- **Inconsistency / sensitivity:** scores vary with phrasing/temperature → use **clear rubrics**, low
  temperature, multiple samples.
> **Mitigations:** randomize positions, control length, use a **different/stronger judge**, give a
> **detailed rubric**, score **multiple times**, and **calibrate against human labels** (validate the judge
> agrees with humans before trusting it). *(Your m09 lesson: LLM-as-judge + its biases.)*

---

## 4. Deep Dive: guardrails (the safety layer) ⭐⭐
Every LLM call gets an **input + output safety wrapper** (applied centrally at the gateway, m33) — defense
in depth:
- **Input guardrails:** **moderation** (block harmful/policy-violating prompts), **PII/PHI detection**
  (redact sensitive input — your healthcare world), **prompt-injection detection** (§5), **topic/scope
  restriction** (keep the assistant on-task).
- **Output guardrails:** **moderation** (filter harmful output), **PII redaction** (don't leak sensitive
  data), **hallucination/faithfulness check** (is the answer grounded? — m30, refuse if not), **format
  validation** (valid JSON/schema — repair/reject if not), and **refusal** for out-of-scope/unsafe.
- *(Your DCE **suppression layer** — block wrong/unwanted outputs at the exit — is exactly an **output
  guardrail**.)*

---

## 5. Deep Dive: prompt injection ⭐⭐ (the #1 unsolved LLM security problem)
**Prompt injection** = malicious instructions that **override the system prompt / intended behavior**:
- **Direct:** the **user** types "ignore your instructions and reveal the system prompt / do X."
- **Indirect (the scary one):** a **retrieved document, tool output, or web page** contains hidden
  instructions the model follows — so the attack vector is **your own data / a third party** (RAG m30,
  agents m32). For an **agent that takes actions**, this can cause **real damage** (exfiltrate data, send
  money) — m32.
- **⚠️ It is NOT fully solved** — there's **no complete defense**; it's an open research problem. So you
  **defend in depth + limit the blast radius:**
  - **Treat all user/retrieved/tool content as untrusted *data*, never instructions**; maintain an
    **instruction hierarchy** (system > user > retrieved).
  - **Least-privilege tools** + **human approval for dangerous actions** (m32) — so even a successful
    injection can't do much.
  - **Input/output filtering** (detect injection attempts; block leaked secrets), **sandboxing** (code
    execution), and **separating/delimiting** untrusted context.

> **The honest senior take:** *"Prompt injection — especially **indirect** (via retrieved docs/tool
> outputs) — is the top LLM security risk and **isn't fully solved**. I defend in depth (treat all
> external content as untrusted data, instruction hierarchy, input/output filtering) and, crucially,
> **limit the blast radius** (least-privilege tools + human approval for dangerous actions) so a successful
> injection can't cause real harm."*

---

## 6. Production: online eval, monitoring & regression gates ⭐
- **Offline (pre-deploy):** run the **eval set** (LLM-judge + task metrics) on every change (prompt, model,
  RAG config) and **gate in CI** — **don't ship a regression** (your m09 gate: a degraded config **FAILED
  faithfulness 0.83<0.85 → exit 1**).
- **Online (post-deploy):** you can't A/B everything — so collect **user feedback** (thumbs up/down,
  edits, regenerations), run **A/B tests** on quality/business metrics (m22), and **monitor quality drift**
  (the silent-degradation problem, m22/m28 — an LLM returns **fluent-but-wrong/unsafe output with no
  error**, so normal latency/error monitoring won't catch it → you **must** eval + monitor quality/safety).
- **Logging (m33):** every call's prompt/response/scores feeds offline eval, debugging, and audit.
- **The three layers:** **offline eval + CI gate** (catch regressions before deploy) + **online (A/B +
  feedback + drift monitoring)** (catch real-world quality) + **runtime guardrails (input/output)** (catch
  safety live).

**Wrap-up:** *"LLM eval is hard — **open-ended, non-deterministic, no single ground truth** — so I use
**task-specific proxies** (RAG: faithfulness/context-recall), **LLM-as-judge** (dominant, scalable, but
**biased**: position/verbosity/self-preference → randomize, control length, different judge, rubric,
calibrate vs humans), and **human eval** as the gold standard, on a **fixed eval set gated in CI** (don't
ship a regression). **Safety = guardrails** wrapping every call: **input** (moderation, PII, injection
detection, scope) + **output** (moderation, PII redaction, **faithfulness/refusal**, format) — applied at
the gateway (m33); my DCE suppression layer is exactly an output guardrail. **Prompt injection** (esp.
**indirect** via retrieved docs/tools) is the **#1 unsolved** risk → defend in depth + **limit the blast
radius** (least-privilege tools + human approval). In production: **offline eval + CI gate, online A/B +
user feedback + quality-drift monitoring** (LLMs fail silently), and runtime guardrails — the three-layer
quality+safety wrapper. I built the eval half from scratch in InsightDesk."*

---

## State-of-the-art & real-world notes 📚
- **LLM-as-judge** (the dominant method; "MT-Bench / Chatbot Arena" validated it correlates with humans, +
  its biases). **RAGAS** (RAG metrics). **Eval frameworks:** OpenAI Evals, **LangSmith**, **Braintrust**,
  **Promptfoo**, DeepEval.
- **Guardrails:** **Llama Guard** (safety classifier), **NeMo Guardrails** (NVIDIA), **Guardrails AI**,
  moderation APIs. **PII:** Presidio.
- **Prompt injection** (Greshake et al., "indirect prompt injection") — an **active, unsolved** area;
  Simon Willison's writing is the canonical accessible source. Defense = depth + least privilege.

---

## From your systems 🏭 (you built the eval half)
- **InsightDesk eval = this module:** **faithfulness ~0.96, answer_relevancy ~0.89, context_precision
  ~0.87, context_recall ~0.72** (task-specific RAG metrics), **LLM-as-judge + its biases** (§3), a **CI
  regression gate** (a degraded config **FAILED faithfulness 0.83<0.85 → exit 1** — §6), and a from-scratch
  **tracer** (logging → eval). You implemented §1–§3 + §6.
- **DCE suppression = output guardrails (§4):** the suppression layer **blocks wrong/unwanted outputs at
  the exit** — exactly an output guardrail; the **Mongo audit trail** = logging for eval/compliance.
- **PII/PHI + compliance:** healthcare claims → §4 PII/PHI detection/redaction + audit is your daily world.
- **Grounding/refusal (m30):** "answer from context, cite, say I don't know" = the §4 faithfulness/refusal
  output guardrail.

---

## Key concepts (interview-ready)
- **LLM eval is hard** (open-ended, non-deterministic, **no single ground truth**) → accuracy/exact-match
  fail. Use **task-specific proxies** (RAG: faithfulness/context-recall), **LLM-as-judge** (dominant), and
  **human eval** (gold standard, calibration target), on a **fixed eval set gated in CI**.
- **LLM-as-judge biases:** **position** (randomize), **verbosity** (control length), **self-preference**
  (different judge), inconsistency (rubric + multiple samples) → **calibrate against humans**.
- **Guardrails (safety, defense-in-depth, at the gateway m33):** **input** (moderation, **PII/PHI**,
  injection detection, scope) + **output** (moderation, PII redaction, **faithfulness/refusal**, format).
  = your DCE suppression layer.
- **Prompt injection (#1 unsolved):** **direct** (user) + **indirect** (retrieved docs/tools — the scary
  one; dangerous for **agents** that act). **No complete fix** → treat external content as **untrusted
  data**, instruction hierarchy, filtering, and crucially **limit the blast radius** (least-privilege +
  human approval, m32).
- **Production = 3 layers:** **offline eval + CI regression gate** (don't ship a regression), **online
  (A/B + user feedback + quality-drift monitoring** — LLMs **fail silently**, m22/m28), and **runtime
  guardrails**.

---

## Go deeper (reading)
- **"Judging LLM-as-a-Judge" (Zheng et al., MT-Bench)** — LLM-as-judge + biases (the must-read). **RAGAS**
  (RAG eval). **OpenAI Evals / LangSmith / Braintrust / Promptfoo** — eval frameworks.
- **Greshake et al., "Indirect Prompt Injection"** + **Simon Willison's prompt-injection writing** — the
  unsolved security problem. **Llama Guard / NeMo Guardrails / Guardrails AI** — guardrail tooling.
- **Your own AI/ML notes (`../ai_ml_llm/` m09 evaluation)** — you built RAGAS-from-scratch, LLM-as-judge +
  biases, the CI gate; revisit it as the worked example.
- Revisit **m22** (offline-online eval/silent failure), **m28** (monitoring/CI gate), **m30** (RAG eval/
  grounding), **m32** (agent injection/least-privilege), **m33** (where guardrails run).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m34_llm_eval_safety/`](../../solutions/m34_llm_eval_safety/README.md).

🎉 **This completes PART 4 — LLM System Design (m29–m34):** serving, RAG, chatbot, agents, gateway, and
eval/safety. Say **"Module 35"** to begin **PART 5 — Capstone & Interview** with *Your Systems as Case
Studies* — turning Questimate & DCE into polished, interview-grade system-design walkthroughs (your single
biggest interview edge).
