# m34 — Exercises (LLM Evaluation & Safety)

> Do these on paper / out loud before checking `solutions/m34_llm_eval_safety/`. LLM-as-judge, prompt
> injection, guardrails, and regression gates are the core — and you built the eval half.

---

## Exercise 1 — Why eval is hard
1. Name three reasons LLM eval inverts traditional ML eval.
2. Why don't accuracy / BLEU work for open-ended generation?
3. What three approaches do you use instead?

---

## Exercise 2 — LLM-as-judge
1. What is it and why is it dominant?
2. Name three biases and a mitigation for each.
3. What's the one overarching mitigation (the calibration step)?

---

## Exercise 3 — RAG eval
1. Name the four RAG metrics.
2. Which is usually the bottleneck?
3. Low faithfulness vs low context recall — which half do you fix in each case?

---

## Exercise 4 — Regression gates
1. What is a regression gate and why essential for LLMs?
2. Cite your own example of a gate catching a regression.
3. A prompt change improves one test case — do you ship it? What do you do?

---

## Exercise 5 — Guardrails
1. Input vs output guardrails — name three of each.
2. Where do they run (which module), and why there?
3. Which of your DCE components is an output guardrail?

---

## Exercise 6 — Prompt injection
1. Direct vs indirect — define each.
2. Why is indirect scarier, especially for agents?
3. Is it solved? What's the honest defense strategy?

---

## Exercise 7 — Blast radius
1. Why "limit the blast radius" rather than "prevent every injection"?
2. Name two blast-radius controls (tie to m32).
3. How does treating retrieved/tool content as "data not instructions" help?

---

## Exercise 8 — PII/PHI
1. Where do you detect/redact sensitive data?
2. Why does self-hosting matter for PHI (tie to m29)?
3. Why is this non-negotiable in your domain?

---

## Exercise 9 — Production monitoring
1. Why do LLM systems "fail silently," and what follows?
2. Name the three layers of eval+safety (offline/online/runtime).
3. What online signals do you collect?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Map your **InsightDesk eval** (faithfulness/recall, LLM-as-judge+biases, CI gate that failed a config)
   to §1–§3 and §6.
2. How is **DCE suppression + audit** an output guardrail + logging layer?
3. Design eval+safety for a healthcare-policy RAG assistant (eval set, metrics, guardrails, PHI,
   injection, gate).

---

When done, check `solutions/m34_llm_eval_safety/README.md`, then say **"Module 35"** (Part 5 — Capstone
begins).
