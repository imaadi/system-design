# m35 — Exercises (Rehearse Your Systems)

> These are **rehearsal drills** — do them **out loud, with a timer, recording yourself**. The goal is
> fluency: you should be able to pitch, walk through, and deep-dive your systems without notes. (Play back
> the recording — silence + un-quantified claims are obvious.)

---

## Exercise 1 — The 90-second pitch (×2)
1. Deliver the **DCE** 90-second pitch from memory, out loud, in ≤ 90s. Hit: what it does, the rules+ML
   hybrid, the suppression net, the stack, **one number**, the proud decision, and a thread to pull.
2. Same for **Questimate**.
3. Record both. Play back: any silence? Did you quote a number? Did you offer a thread?

---

## Exercise 2 — Drive the framework on your own system
Pick **DCE**. Run the full m01 framework on it as if it were the interview question, ≤ 6 min:
requirements/NFRs → estimation (~1M/day) → data model + storage choice → high-level pipeline → one
deep-dive (3 levels) → bottlenecks/evolution. Then do **Questimate**.

---

## Exercise 3 — One deep-dive, three levels
For each, go **component → mechanism → failure/trade-off**:
1. DCE suppression layer.
2. Questimate Redis-backed JWT.
3. Questimate snapshot precompute.
4. The DCE feature/preprocessing pipeline.

---

## Exercise 4 — Name the pattern
For each piece of your work, **name the curriculum concept + module**:
1. Suppression rules at the exit.
2. Merging Redis reference data into a claim at scoring.
3. Precomputing risk into snapshot tables.
4. `compare_results.py` parity validation.
5. Reprocessing scripts.
6. Redis-backed JWT.
7. 4 services talking over REST with env-injected URLs.

---

## Exercise 5 — Turn "have you done X?" into "yes, on [system] I…"
Answer each in one sentence with a system + a specific:
caching · async messaging · idempotency · monitoring · safe deploys · feature engineering · cost control ·
reconciliation · RAG · agents · model serving.

---

## Exercise 6 — The numbers drill
Without looking, recall: DCE claims/day, the cost-reduction numbers (run cost, ETL, infra), the team
reduction, Questimate's service/model/API counts, the InsightDesk RAG faithfulness/recall. Then weave **one**
into a sentence naturally.

---

## Exercise 7 — Hardest part / bug fixed
Tell a **specific, 60–90s** "hardest technical problem" story for:
1. DCE (the correctness safety net, or a migration parity issue).
2. Questimate (the BDT pricer).
Each must have: the problem, what made it hard, what you did, the outcome.

---

## Exercise 8 — "What would you do differently?"
Give a credible, senior answer for each (shows you see the maturity gap, not that the work was bad):
1. DCE.
2. Questimate (incl. the honest prod-hygiene framing).

---

## Exercise 9 — Handle the challenge
For each pushback, respond with curiosity + a both-sides answer:
1. "Why REST and not gRPC between your services?"
2. "Why not just one big database instead of Mongo + Redis + Snowflake?"
3. "Isn't a single Postgres primary a risk?"
4. "Your snapshot tables are stale — isn't that a problem?"

---

## Exercise 10 — Role-matching + the bridge
1. For each role, say which system you'd **lead with** and why: (a) ML Engineer at a health-tech, (b)
   Backend Engineer at a fintech, (c) AI/LLM Engineer at a startup, (d) Platform/MLOps role.
2. For (c) and (d), how do you **bridge** to InsightDesk (RAG/agents/eval/MLOps)?

---

When done, check the **story bank** in `solutions/m35_your_systems_case_studies/README.md`, then say
**"Module 36"** for the final module.
