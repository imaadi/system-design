# m34 — Solutions (LLM Evaluation & Safety)

> Check the *reasoning*. Recurring lessons: no ground truth → LLM-as-judge (biased, calibrate) + proxies +
> human eval + CI gate; guardrails wrap every call; prompt injection unsolved → limit blast radius.

---

## Exercise 1 — Why eval is hard
1. **No single ground truth** (many correct answers), **non-deterministic** (same prompt → different
   outputs), and **subjective + multi-dimensional** quality (correctness/relevance/coherence/safety/format).
2. Because they **compare to a fixed reference** — open-ended generation has **valid paraphrases** that
   exact-match/BLEU penalize, so they don't capture correctness for free-form output.
3. **Task-specific proxies** (RAG: faithfulness/recall), **LLM-as-judge**, and **human eval** (on a fixed
   eval set, gated in CI).

---

## Exercise 2 — LLM-as-judge
1. A **strong LLM scores outputs** against criteria (rubric/scale or pairwise); dominant because it's
   **scalable + cheap vs humans**, **correlates with human judgment**, and works for **open-ended** output.
2. **Position bias** (randomize/swap positions + average), **verbosity bias** (control for length / ignore
   it), **self-preference bias** (use a different/stronger judge or ensemble). (Also inconsistency → rubric
   + low temp + multiple samples.)
3. **Calibrate against human labels** — verify the judge agrees with humans on a sample before trusting it.

---

## Exercise 3 — RAG eval
1. **Faithfulness, answer relevancy, context precision, context recall.**
2. **Context recall** (did retrieval get the needed chunks) — retrieval is the bottleneck ("most
   hallucinations are retrieval misses").
3. **Low faithfulness → fix generation/prompt** (context was there, model ignored/hallucinated); **low
   context recall → fix retrieval/chunking** (the right chunks weren't retrieved).

---

## Exercise 4 — Regression gates
1. A **CI check running a fixed eval set on every change**, **blocking the deploy if quality drops below a
   threshold** — essential because LLMs **fail silently** and changes (esp. prompts) have non-obvious
   quality impact, so without a gate you silently degrade.
2. My InsightDesk gate: a **degraded config FAILED faithfulness 0.83 < 0.85 → exit 1** — caught before
   shipping.
3. **Don't ship on one anecdote** — run the **full eval set** (LLM changes are non-local: a tweak helps
   some cases, hurts others), confirm aggregate improvement **without regressions** via the gate, then
   **canary/A-B online** before full rollout.

---

## Exercise 5 — Guardrails
1. **Input:** moderation (block harmful prompts), PII/PHI detection/redaction, prompt-injection detection,
   topic/scope restriction. **Output:** moderation (filter harmful), PII redaction, faithfulness/refusal
   check, format validation. (Three of each.)
2. **At the gateway (m33)** — all LLM traffic flows through it, so guardrails are applied **once,
   consistently, for every app** (defense in depth, org-wide).
3. **DCE's suppression layer** — it blocks wrong/unwanted outputs at the exit = an **output guardrail**
   (and the Mongo audit trail = logging).

---

## Exercise 6 — Prompt injection
1. **Direct:** the **user** injects "ignore your instructions and do X." **Indirect:** a **retrieved doc /
   tool output / web page** contains hidden instructions the model follows.
2. **Indirect is scarier** because the attack is in **content the system ingests** (your own data / a third
   party) — a benign user triggers it, it's invisible, and for an **agent that acts** (m32) it can drive
   **real harmful actions** (exfiltrate data, send money).
3. **Not solved** (open problem, no complete defense) → **defend in depth + limit the blast radius**: treat
   external content as **untrusted data**, instruction hierarchy, input/output filtering, sandboxing, and
   **least-privilege + human approval** so a successful injection can't do damage.

---

## Exercise 7 — Blast radius
1. Because **you can't prevent every injection** (it's unsolved), so you assume one will succeed and design
   so it **can't cause real harm** — risk management over a (false) promise of prevention.
2. **Least-privilege tools** (the agent can only do what's explicitly granted) and **human approval for
   dangerous/irreversible actions** (m32) — so even a hijacked agent can't send money/delete data
   unilaterally.
3. If retrieved/tool content is **data, not instructions**, the model **won't execute** an embedded "do X"
   command — it treats it as text to reason *about*, not a directive to follow, neutralizing the injection
   vector (combined with an instruction hierarchy).

---

## Exercise 8 — PII/PHI
1. At the **boundaries**: **input** (redact before the model — esp. external API providers where data
   leaves your boundary) and **output** (don't leak in responses) — via PII detection (Presidio/regex+ML).
2. **Self-hosting keeps PHI in-house** (data never leaves your boundary to a third-party API, m29) —
   essential when compliance (HIPAA) forbids sending sensitive data externally.
3. Because **healthcare claims contain PHI** — detection/redaction + encryption + access controls + audit
   are **HIPAA requirements** (legal/financial risk), not nice-to-haves, in my domain.

---

## Exercise 9 — Production monitoring
1. LLMs return **fluent, confident, well-formatted output that's wrong/unsafe — with no error** — so the
   service looks healthy and normal latency/error monitoring misses it. It follows that you **must
   explicitly evaluate + monitor quality/safety** (it's the only way to detect the failure).
2. **Offline eval + CI regression gate** (catch regressions pre-deploy), **online (A/B + user feedback +
   quality-drift monitoring)** (real-world quality), and **runtime guardrails** (input/output safety live).
3. **User feedback** (thumbs up/down, edits, regenerations, copies), **A/B metrics**, **sampled LLM-judge/
   human review** of live outputs, and **quality-drift** signals.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **InsightDesk eval → m34:** **faithfulness ~0.96 / answer_relevancy ~0.89 / context_precision ~0.87 /
   context_recall ~0.72** = task-specific RAG metrics (§2/§4-eval); **LLM-as-judge + its biases** = §3; the
   **CI gate** (degraded config **FAILED faithfulness 0.83<0.85 → exit 1**) = §6 regression gate; the
   from-scratch tracer = logging→eval.
2. **DCE suppression = output guardrail** (blocks wrong/unwanted outputs at the exit) + **Mongo audit trail
   = logging** (for eval, debugging, compliance) — the §4 + §6 layer in production, in a non-LLM pipeline.
3. **Healthcare-policy RAG assistant eval+safety:** build a **fixed eval set** of (policy question →
   expected answer/clause) incl. edge cases + adversarial/injection examples; metrics = **faithfulness +
   context recall** (RAGAS, LLM-judge **calibrated vs human** review by a compliance expert), **gated in
   CI** (don't ship a regression); **guardrails** = input **PHI redaction** + injection detection, output
   moderation + **faithfulness/refusal** ("answer only from policy, cite the clause, say 'not covered/
   unknown'", m30) + format validation; **prompt injection** — treat retrieved policy docs as **untrusted
   data**, least-privilege (read-only retrieval, no actions), human review for anything consequential;
   **PHI** detected/redacted + audited (HIPAA); **online** = user feedback + quality-drift monitoring +
   sampled human review. Three layers (offline gate + online + runtime guardrails) — exactly the eval-from-
   scratch + suppression/audit discipline I've built, applied to my own regulated domain.
