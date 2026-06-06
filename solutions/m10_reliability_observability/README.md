# m10 — Solutions (Reliability, Observability & Operations)

> Check the *reasoning*. Recurring lessons: define reliability with SLOs/error budgets; a missing
> timeout is an outage; naive retries amplify; combine patterns to contain blast radius; alert on
> symptoms; deploys cause outages.

---

## Exercise 1 — SLI / SLO / SLA / error budget
1. **SLI:** % of checkout requests that succeed (non-5xx) and return < 500 ms. **SLO:** 99.9% of
   checkouts succeed within 500 ms over 30 days (internal target). **SLA:** maybe "99.5% monthly or we
   credit you" — **looser** than the SLO, with penalties, so you catch problems internally before the
   contract breaks.
2. 99.95% → allowed failure = 0.05%. 30 days ≈ 43,200 min × 0.0005 ≈ **~21.6 min/month** of error budget.
3. **90% burned at week 3:** stop the bleeding — **freeze risky features, focus on reliability**, slow
   deploys, until you're back in budget. **Only 10% burned:** you have headroom — **ship faster**, take
   more risk, run experiments. The budget turns reliability into a managed velocity dial.

---

## Exercise 2 — Golden signals
1. **Latency** (how slow — percentiles), **Traffic** (demand/QPS), **Errors** (failure rate),
   **Saturation** (how full — CPU/mem/queue/pool, nearness to a limit).
2. Average lies because **(a)** it hides the **tail** — the slow 1% (p99) that users feel is buried under
   fast requests; **(b)** **fast errors** drag it down (a flood of quick 500s looks "fast"). Look at
   **percentiles (p99/p999)** and **split success vs error latency**, paired with the **error rate**.
3. **Saturation** = how close a resource is to its limit (queue depth, pool usage, CPU). It's *leading*
   because performance degrades **before** you hit 100% — a filling queue / rising pool usage predicts an
   imminent latency cliff and lets you act before users see errors.

---

## Exercise 3 — The missing timeout
1. Every call to B now blocks for 30s. A's worker threads/connections fill up waiting on B; by **Little's
   Law**, with latency W jumping to 30s and a fixed pool of N, A's max throughput λ = N/W **collapses** —
   new requests queue, A's latency climbs, A starts timing out *its* callers.
2. Because A's **resources (threads/conns) are exhausted waiting on B** — A isn't broken, but it has no
   capacity left to serve *anything*, so it goes down too. One slow leaf starves the whole chain.
3. **Fix:** set an aggressive **timeout** on the B call so threads are freed quickly. **Also add:**
   **retries with backoff+jitter** (for transient B failures), a **circuit breaker** (stop calling B when
   it's clearly down → fail fast), a **bulkhead** (a dedicated pool for B so it can't exhaust A's other
   threads), and a **graceful fallback** (e.g. queue the payment / show "processing").

---

## Exercise 4 — Retry storm
1. **Retry storm → metastable failure.**
2. Naive retries **add load to an already-overloaded service**: slow/failing → everyone retries
   immediately → even more load → fails harder → more retries. The extra retry load keeps it pinned down
   **even after the original blip clears** (metastable — it won't self-recover).
3. **Proper policy:** (a) **exponential backoff** (1s,2s,4s…), (b) **jitter** (randomize waits so clients
   don't sync), (c) a **retry budget/cap** (e.g. retries ≤10% of traffic, max N attempts), (d) a
   **circuit breaker** to stop retrying a dead dependency entirely.
4. Retry only **idempotent** operations (m08) and only **retryable** errors (timeouts, 503, connection
   resets) — **not** 4xx like 400/422 (won't fix on retry) or non-idempotent writes without an
   idempotency key.

---

## Exercise 5 — Circuit breaker
1. **Closed** (normal: requests flow, count failures) → threshold exceeded → **Open** (fail fast /
   fallback without calling the dependency) → after cooldown → **Half-open** (allow a few trials) →
   success → **Closed**; failure → back to **Open**.
2. Retries+backoff still *call* the failing dependency (just less often) and still consume your threads
   waiting; the **circuit breaker stops calling it entirely** while it's down — protecting **your own
   resources** and giving the dependency room to recover. It prevents the retry storm rather than just
   slowing it.
3. While **open**, user requests **fail fast** (immediately) or get a **fallback** (cached/default/
   degraded response) — they do **not** wait on or hit the dead dependency.

---

## Exercise 6 — Contain the blast radius
1. Without isolation, all dependencies **share the same thread/connection pool**. Recommendations hangs →
   its calls occupy threads → the pool **exhausts** (Little's Law) → no threads left to serve
   **payments** or **search** either → the whole service dies because of a **non-critical** dependency.
2. A **bulkhead** gives recommendations its **own isolated pool**; when it hangs, only *that* pool fills,
   leaving payments/search threads untouched → only the recs feature degrades, not the service.
3. **Graceful degradation for recs:** on timeout/breaker-open, return a **default/popular list** (or a
   cached previous result, or simply **omit** the recommendations section) and render the rest of the
   page normally — better than failing the whole request for a non-essential feature.

---

## Exercise 7 — The three pillars
1. **Metrics** — aggregate error rate vs SLO is a time-series question.
2. **Traces** — a single request's latency across services is exactly what distributed tracing/spans
   show.
3. **Logs** — the exact exception + stack for one request id is detailed event data.
4. **Together:** a **metric** alert fires ("error rate breached SLO") → you open a **trace** for a
   failing/slow request to **localize** which service/span is responsible → you pull that service's
   **logs** (filtered by the same id) for the **root cause**. The **trace/correlation ID** (propagated
   through every hop, m08) is what ties metrics-triggered investigation → trace → logs together.

---

## Exercise 8 — Alerting
1. **Alert fatigue** from too many noisy/non-actionable pages (often paging on *causes*). Fix: **page
   only on actionable, user-facing symptoms** (SLO violations / error-budget burn), route the rest to
   dashboards/tickets, add severity tiers, and tie each page to a runbook.
2. **No** — high CPU can be perfectly healthy (efficient) and can also miss real problems; it causes false
   pages → fatigue. **Page on symptoms users feel**: error rate up, p99 over target, error-budget burning
   fast. Use CPU as **diagnostic context** on a dashboard, not a pager.
3. An **error-budget burn-rate alert** pages when you're **consuming the error budget too fast** (e.g.
   "at this rate the monthly budget is gone in an hour"). It's good because it fires on **real,
   user-impacting degradation** proportional to severity, while ignoring trivial blips — catching
   genuine problems without noise.

---

## Exercise 9 — Safe deploys & incident response
1. **Canary** (small % first, watch SLOs), **feature flags** (decouple deploy/release + instant
   kill-switch), **fast rehearsed rollback**, **backward-compatible changes** (expand-contract) — any
   three.
2. **Rolling:** gradual instance replacement, zero-downtime, but old+new coexist and rollback is slower.
   **Blue-green:** two full envs, instant switch + instant rollback, but double infra during the cutover.
   **Canary:** small-% traffic to new version with metric-watching, smallest blast radius, needs good
   metrics/automation.
3. **First action: mitigate — roll back the deploy** (or flip the feature flag off). Not root-cause,
   because **users are hurting now**; you stop the bleeding first, then investigate calmly afterward. "Roll
   back now, debug later."
4. **Blameless** because people inevitably make mistakes; blaming them makes everyone **hide information**
   and stop reporting near-misses, destroying the learning loop. The useful question is "**why did the
   system let this happen?**" — you fix guardrails/process so the next person can't cause the same outage.

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **Suppression layer:** a **correctness safety net / guard** at the pipeline exit that **prevents
   incorrect recommendations** from reaching downstream adjudication. It prioritizes **reliability/
   correctness** (a wrong auto-deny is costly) over raw throughput/uptime — i.e. it's better to **suppress
   (degrade to a pend/no-action)** than to emit a wrong decision. Reliability-as-correctness, not just
   "is it up."
2. **Mapping:** `claims_count`/`exclusions_count` = **observability/metrics** (daily volume + exclusion
   tracking → spot anomalies); `reprocess.py`/`aws_batch_reprocessing` = **mitigation/recovery** (replay
   from the log to fix/redo affected claims, m08); `compare_results.py` = **safe-deploy/migration
   validation** (parity check old vs new env — canary-style verification before cutover, §8).
3. **ML monitoring:** **DATA_DRIFT** and **POSRATE_SHIFT** are **symptom alerts** (the model's inputs/
   outputs are deviating → quality at risk), and **OVER_BUDGET** treats **cost as an SLO** — you set a
   budget and alert when burn exceeds it, exactly like an error budget but for spend (a great example that
   "reliability" includes cost/efficiency objectives, not just uptime).
4. **Running 1M/day with a team of 4** requires: heavy **automation** (reprocessing, reporting, compare
   scripts), good **monitoring** (counts, exclusions, drift), **safe deploys** (parity-validated
   migrations, rollback), **idempotent reprocessing** (recover without fear), and the **suppression
   safety net** so failures degrade safely — i.e. exactly the §1–§10 toolkit, which is why a small team
   can operate that scale.
