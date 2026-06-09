# m34 — Cross-Questions ("if-and-buts") on LLM Evaluation & Safety

> Answer out loud in 2–3 sentences before reading the model answer. LLM-as-judge, prompt injection,
> guardrails, and regression gates are where this is won — and you built the eval half.

---

### Q1. Why is evaluating an LLM harder than evaluating a normal ML model?
**A.** Because LLM outputs are **open-ended and non-deterministic with no single ground truth** — many
answers are correct in different words, and the same prompt gives different outputs — so **accuracy /
exact-match / BLEU-ROUGE** (which compare to a fixed reference) are **weak or useless**. Quality is also
**subjective and multi-dimensional** (correctness, relevance, coherence, safety, format). So you engineer
measurability via **task-specific proxies** (RAG faithfulness/recall), **LLM-as-judge**, and **human eval** —
it inverts the clean ground-truth eval of traditional ML.

---

### Q2. What is LLM-as-judge and why is it the dominant approach?
**A.** Using a **strong LLM to score outputs** against criteria (a rubric/scale, or pairwise A-vs-B) — e.g.
"is this answer faithful to the context? rate 1–5." It's dominant because it's **scalable and cheap vs
human eval**, **correlates reasonably with human judgment** (validated by MT-Bench/Chatbot Arena), and
works for **open-ended** outputs where references fail. But it's **a model grading a model**, so it has
**biases** (§3) you must mitigate and **calibrate against human labels** before trusting it.

---

### Q3. What biases does LLM-as-judge have, and how do you mitigate them?
**A.** **Position bias** (prefers the first/a fixed option in pairwise → **randomize/swap positions and
average**), **verbosity bias** (prefers longer answers → **control for length / instruct to ignore it**),
**self-preference bias** (prefers its own style/model family → **use a different/stronger judge or an
ensemble**), and **inconsistency** (sensitive to phrasing/temperature → **clear rubric, low temperature,
multiple samples**). The overarching mitigation: **calibrate the judge against human labels** — verify it
agrees with humans on a sample before relying on it. (My m09 lesson.)

---

### Q4. How do you evaluate a RAG system specifically?
**A.** With **task-specific metrics** (RAGAS-style, which I implemented): **faithfulness** (answer grounded
in context — anti-hallucination), **answer relevancy** (addresses the question), **context precision**
(retrieved chunks relevant), and **context recall** (retrieval got the needed chunks — usually the
bottleneck). Scored by **LLM-as-judge** on a **fixed eval set**, **gated in CI**. And you **decompose
failures**: low faithfulness → generation/prompt; low context recall → retrieval/chunking (m30). Plus
online: A/B + user feedback. (My numbers: faithfulness ~0.96, context recall ~0.72.)

---

### Q5. What is a regression gate and why is it essential for LLM systems?
**A.** A **CI check that runs a fixed eval set on every change** (prompt, model, RAG config) and **blocks
the deploy if quality drops below a threshold** — so you **never ship a regression**. It's essential
because LLM systems **fail silently** (fluent-but-worse output, no error) and changes are easy to make
(tweak a prompt) with non-obvious quality impact — so without a gate you'd silently degrade. In my work a
**degraded config FAILED the faithfulness gate (0.83 < 0.85) → exit 1** — the gate caught it. It's the m22/
m28/m30 discipline applied to LLMs.

---

### Q6. What are guardrails, and what's the input vs output split?
**A.** Guardrails are the **safety wrapper around every LLM call** (defense in depth, applied at the
gateway m33). **Input guardrails** (before the model): **moderation** (block harmful prompts), **PII/PHI
detection/redaction**, **prompt-injection detection**, **topic/scope restriction**. **Output guardrails**
(before returning): **moderation** (filter harmful output), **PII redaction** (don't leak), **hallucination/
faithfulness check** (refuse if ungrounded), **format validation** (valid JSON/schema). My DCE
**suppression layer** — blocking wrong/unwanted outputs at the exit — is exactly an output guardrail.

---

### Q7. What is prompt injection, direct vs indirect?
**A.** **Prompt injection** = malicious instructions that **override the system prompt / intended
behavior**. **Direct:** the **user** types "ignore your instructions and do X / reveal the system prompt."
**Indirect (the scary one):** a **retrieved document, tool output, or web page** contains hidden
instructions the model follows — so the attack comes from **your own data or a third party** (RAG/agents).
Indirect is especially dangerous for **agents that take actions** (m32) — an injected instruction could
exfiltrate data or send money. It's the **#1 LLM security risk**.

---

### Q8. Is prompt injection solved? How do you defend against it?
**A.** **No — it's not fully solved** (an open research problem with no complete defense), so the honest
answer is **defense in depth + limiting the blast radius**. Defenses: **treat all user/retrieved/tool
content as untrusted *data*, never instructions**; maintain an **instruction hierarchy** (system > user >
retrieved); **input/output filtering** (detect injection attempts, block leaked secrets); **sandboxing**
(code execution); and **delimiting** untrusted context. Crucially, **least-privilege tools + human approval
for dangerous actions** (m32) so even a successful injection **can't cause real harm**. You manage the risk;
you don't eliminate it.

---

### Q9. Why is indirect prompt injection scarier than direct?
**A.** Because the user doesn't have to be malicious — the attack is **embedded in content the system
ingests**: a **RAG-retrieved document, a tool's output, a scraped web page, an email** the agent reads. So
a **benign user** can trigger it just by asking about poisoned content, and it's **invisible** (hidden in
data you trusted). For an **agent that acts** (m32), the injected instruction can drive **real actions**
(send the database to an attacker). It widens the attack surface to all your data sources and turns
"untrusted input" into "untrusted *everything the model reads*" — which is why you treat retrieved/tool
content as untrusted data.

---

### Q10. How do you handle PII/PHI in an LLM system?
**A.** **Detect and redact** sensitive data at the boundaries: **input** (redact PII/PHI before it reaches
the model — esp. for external API providers where data leaves your boundary, m29) and **output** (don't
leak sensitive data in responses). Use **PII detection** (Presidio, regex+ML), **least-data** principle
(don't send what you don't need), **self-host** for the most sensitive data (data stays in-house, m29),
**encryption + access controls + audit trails**, and **compliance** (HIPAA/GDPR). In my **healthcare**
world this is mandatory — claims contain PHI, so detection/redaction + audit is a hard requirement, not a
nicety.

---

### Q11. How do you evaluate quality once in production (online)?
**A.** You can't A/B everything, so you combine: **user feedback** (thumbs up/down, edits, regenerations,
copy events — implicit signals), **A/B tests** on quality + business metrics (m22) for important changes,
**sampling + LLM-judge/human review** of live outputs, and **quality-drift monitoring**. Because LLMs **fail
silently** (m22/m28 — fluent-but-wrong/unsafe with no error), normal latency/error monitoring won't catch a
quality regression, so you **must** instrument quality + safety signals explicitly. Online eval catches
what offline (a proxy) misses — the real-world distribution.

---

### Q12. Why do LLM systems "fail silently" and what follows?
**A.** Because a degraded LLM returns **fluent, confident, well-formatted output that's simply wrong or
unsafe** — no exception, no 500, the service looks "healthy." So normal monitoring (latency, errors) sees
nothing wrong. What follows: you **must evaluate and monitor quality + safety explicitly** (eval sets, CI
gates, user feedback, quality-drift monitoring, guardrails) — eval isn't optional for LLMs, it's the
**only** way to detect the failure mode. It's the m22 silent-degradation theme at its most acute (LLM
output is the most fluent-but-wrong of all ML).

---

### Q13. When do you use human eval vs LLM-as-judge?
**A.** **Human eval is the gold standard** — most accurate, the **calibration target** — but **expensive +
slow**, so you use it **sparingly**: to **validate/calibrate your LLM-judge** (confirm it agrees with
humans before trusting it), for **high-stakes/subtle** decisions, and to **build a labeled eval set**. You
use **LLM-as-judge** for the **scalable, frequent** evaluation (every CI run, large eval sets) once it's
calibrated. The pattern: **humans to establish ground truth + calibrate; LLM-judge to scale** — never
trust the automated judge blindly without human validation.

---

### Q14. How do you build an eval set, and what makes a good one?
**A.** Curate a **fixed set of representative (input, criteria/expected) examples** — covering common
cases, **edge cases, adversarial inputs, and known failure modes** — with **clear criteria** (a rubric for
the judge or expected facts). A good eval set is **representative** (matches production distribution),
**discriminative** (distinguishes better from worse systems), **stable** (so scores are comparable across
changes), and **maintained** (add new failures as they appear in production — a flywheel). You run it on
**every change** and **gate** on it. Building the eval set *is* much of the work — "any production RAG has
a recall metric" means someone built the eval set to measure it.

---

### Q15. A prompt change improved one test case but you're not sure overall. What do you do?
**A.** **Run the full eval set** (don't judge on one anecdote) and compare aggregate metrics against the
current version via the **regression gate** — does it improve the metrics overall **without regressing**
others? LLM changes are notoriously **non-local** (a prompt tweak helps some cases, hurts others), so
**single-example reasoning is misleading**. If offline improves, **canary/A-B online** on real traffic +
user feedback (m22) before full rollout. This is exactly why you have a fixed eval set + gate — to make
prompt/model changes **measurable and safe** rather than vibes-based.

---

### Q16. How do guardrails relate to the gateway (m33) and where do they run?
**A.** The **gateway (m33) is the central enforcement point** — all LLM traffic flows through it, so it's
where you apply **input guardrails (moderation/PII/injection-detection) before the model and output
guardrails (moderation/PII-redaction/faithfulness/format) before returning** — **once, consistently, for
every app** (chatbot/RAG/agents). m34 defines **what** the guardrails are (the safety/eval logic); m33 is a
primary place they **run**. You can also apply guardrails in-app, but the gateway gives you **org-wide,
consistent** enforcement (defense in depth across all LLM usage).

---

### Q17. How do you evaluate an agent (m32), which is multi-step?
**A.** Harder than single-shot — you evaluate at multiple levels (m32): **end-to-end task success** (did it
achieve the goal? — verifiable end state or human/LLM-judged), **per-step correctness** (right tool/args/
intermediate result), **efficiency** (steps/cost/latency), and **safety** (no harmful actions, injection
resistance). You build **task eval sets** with checkable outcomes, use **LLM-as-judge** for fuzzy steps,
and **trace every run** (observability) to debug trajectories. Evaluating **trajectories**, not just final
answers, is the agent-specific challenge — and why agents are the hardest LLM systems to eval reliably.

---

### Q18. What's the relationship between eval and the rest of the LLM stack?
**A.** Eval + safety is the **quality/safety layer wrapping everything**: it measures the **chatbot (m31)**,
**RAG (m30)** (faithfulness/recall), and **agents (m32)** (trajectories), **gates changes in CI**, **runs
guardrails at the gateway (m33)**, monitors **serving (m29)** output quality, and feeds the **MLOps loop
(m28)** (drift → retrain/iterate). Without it, you ship LLM systems blind (silent failure). It's the
**non-negotiable foundation** that makes everything else *production* rather than *demo* — which is exactly
why "any production RAG has a recall metric" and why I built eval from scratch.

---

### Q19. What's the honest take on LLM safety in an interview?
**A.** *"LLM safety is **defense in depth, not a solved problem.** I wrap every call in **input + output
guardrails** (moderation, PII/PHI redaction, faithfulness/refusal, format validation), I **evaluate +
gate** quality in CI and **monitor** for silent drift, and I treat **prompt injection** — especially
**indirect** via retrieved docs/tools — as **unsolved**, so I **limit the blast radius** (least-privilege
tools, human approval for dangerous actions) rather than assuming I can prevent every injection. The
mindset: assume the model can be wrong or manipulated, and **engineer the system so that's safe** — exactly
the correctness-first instinct from my DCE suppression + audit work."*

---

### Q20. Summarize the senior answer for LLM eval & safety.
**A.** *"**Eval is hard** (open-ended, non-deterministic, no ground truth) → **task proxies** (RAG:
faithfulness/recall), **LLM-as-judge** (scalable, but biased: position/verbosity/self-preference →
randomize/control-length/different-judge/rubric/**calibrate vs humans**), and **human eval** (gold
standard), on a **fixed eval set gated in CI** (don't ship a regression — my faithfulness gate caught a
degraded config). **Safety = guardrails** on every call (input: moderation/PII/injection-detection; output:
moderation/PII-redaction/**faithfulness-refusal**/format) at the gateway (m33) — my DCE suppression layer is
an output guardrail. **Prompt injection** (esp. **indirect**) is the **#1 unsolved** risk → defend in depth
+ **limit the blast radius** (least-privilege + human approval, m32). Production = **offline eval + CI gate,
online A/B + user feedback + quality-drift monitoring** (LLMs **fail silently**), and runtime guardrails —
the three-layer quality+safety wrapper. I built the eval half from scratch in InsightDesk."*
