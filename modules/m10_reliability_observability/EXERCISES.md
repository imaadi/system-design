# m10 — Exercises (Reliability, Observability & Operations)

> Do these on paper / out loud before checking `solutions/m10_reliability_observability/`. Goal: define
> reliability precisely, apply the resilience patterns, and reason about observability + safe ops.

---

## Exercise 1 — SLI / SLO / SLA / error budget
1. For a checkout API, propose one SLI, a matching SLO, and how the SLA might differ.
2. Your SLO is 99.95% monthly. How much downtime/error is that per 30-day month (the error budget)?
3. You're 3 weeks in and have already burned 90% of the budget. What should the team do, and what if
   you'd burned only 10%?

---

## Exercise 2 — Golden signals
1. Name the four golden signals and what each tells you.
2. Your average latency looks great but customers are complaining. Give two reasons the average is lying
   and what you'd look at instead.
3. What does "saturation" mean and why is it a *leading* indicator of trouble?

---

## Exercise 3 — The missing timeout
Service A calls payment-service B synchronously with **no timeout**. B starts taking 30s per call.
1. Walk through what happens to A (use Little's Law).
2. Why can one slow dependency take down a service that isn't itself broken?
3. What's the fix, and what else would you add around the call beyond a timeout?

---

## Exercise 4 — Retry storm
A downstream service hiccups; all clients retry immediately and it gets *worse*, staying down even after
the original blip passes.
1. Name this failure pattern.
2. Why does naive retrying amplify it?
3. Design the retry policy properly (name four elements).
4. Which operations are safe to retry, and which errors?

---

## Exercise 5 — Circuit breaker
1. Draw/describe the three states and the transitions between them.
2. What problem does it solve that retries+backoff alone don't?
3. What should happen to user requests while the breaker is **open**?

---

## Exercise 6 — Contain the blast radius
Your service depends on: a critical payments service, a non-critical recommendations service, and a
search service. Recommendations starts hanging.
1. Without isolation, why might the *whole* service go down because of *recommendations*?
2. How does a **bulkhead** prevent that?
3. What **graceful degradation** would you apply to the recommendations feature specifically?

---

## Exercise 7 — The three pillars
For each question, say which pillar (metrics / logs / traces) you'd reach for first:
1. "Is the error rate above our SLO right now?"
2. "For this one slow request, which of our 8 services ate the 2 seconds?"
3. "What was the exact exception and stack for the failed request id abc-123?"
4. Then: explain how the three are used *together* in one investigation, and what ID ties them.

---

## Exercise 8 — Alerting
1. Your team is drowning in pages and starting to ignore them. Diagnose the likely cause and the fix.
2. Should you page on "CPU > 90%"? Why or why not? What *would* you page on?
3. What is an error-budget burn-rate alert and why is it good?

---

## Exercise 9 — Safe deploys & incident response
1. Most of your outages happen right after deploys. Name three deployment practices that reduce this.
2. Canary vs blue-green vs rolling — one-line trade-off of each.
3. A new deploy is causing errors *right now*. What's your first action, and why is it not "find the
   root cause"?
4. Why are postmortems "blameless"?

---

## Exercise 10 — Apply to your own systems 🏭
1. **DCE suppression layer:** frame it as reliability/graceful-degradation — what failure does it
   prevent, and which NFR does it prioritize over raw uptime?
2. **Ops tooling:** map `claims_count`/`exclusions_count`, `reprocess.py`/`aws_batch_reprocessing`, and
   `compare_results.py` to observability / mitigation / safe-deploy-validation.
3. **ML monitoring:** your drift alerts (DATA_DRIFT / POSRATE_SHIFT / OVER_BUDGET) — which are symptom
   alerts, and what does OVER_BUDGET show about treating *cost* as an SLO?
4. **Running 1M claims/day with a team of 4:** what reliability/ops practices make that possible?

---

When done, check `solutions/m10_reliability_observability/README.md`, then say **"Module 11"** to start
Part 2 (the URL Shortener).
