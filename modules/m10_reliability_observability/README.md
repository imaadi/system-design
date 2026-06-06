# Module 10 — Reliability, Observability & Operations ⭐⭐

> m09 proved things *will* fail. This module is how you **stay up anyway** and **know what's
> happening**. It's the "day 2" skill set most candidates skip and seniors live in: defining
> reliability with **SLOs + error budgets**, the **resilience patterns** (timeouts, retries+backoff,
> circuit breakers, bulkheads, graceful degradation, load shedding) that stop one failure from sinking
> the ship, the **three pillars of observability** (metrics, logs, traces), and safe **deploys/incident
> response**. This is your operational wheelhouse — DCE at 1M claims/day with a team of 4, suppression
> safety nets, reprocessing tooling, and ML drift monitoring.

Read in passes: **§1–§5** (SLOs, signals, resilience patterns, cascading failure) and **§6–§10**
(observability, alerting, safe deploys, chaos, incident response).

---

## 1. The mindset: "everything fails, all the time" ⭐

Werner Vogels's line. At scale, **something is always broken** — a disk, a node, a network link, a
dependency, a deploy. Reliability is **not** "prevent all failure" (impossible); it's **"contain
failure so users barely notice."** Two consequences:
- **Design for failure**, not for the happy path: assume every dependency can be slow, down, or wrong,
  and decide *now* what happens then.
- **Reliability is a property of the *whole system*, including humans and process** — your deploys,
  monitoring, on-call, and runbooks are as much "the system" as the code.

> The shift from junior to senior here: junior asks "how do I make it work?"; senior asks "how does it
> **degrade** when (not if) a dependency fails, and how would I **know**?"

---

## 2. SLA / SLO / SLI & error budgets ⭐⭐ (define reliability precisely)

You can't manage reliability without measuring it. Three terms (don't mix them up):
- **SLI (Indicator):** the **measurement** — a concrete metric of service health. E.g. "% of requests
  served < 200 ms," "% of requests that succeed (non-5xx)," "uptime %."
- **SLO (Objective):** the **internal target** for an SLI. E.g. "99.9% of requests succeed over 30
  days," "p99 latency < 300 ms." *This is what you design and alert against.*
- **SLA (Agreement):** the **external contract** with customers, with **penalties** if breached (refunds/
  credits). Usually **looser** than the SLO (you set the internal SLO stricter so you fix things before
  the contract breaks).

> **SLI ⊆ SLO ⊆ SLA** in strictness: measure (SLI) → target stricter internally (SLO) → promise looser
> externally (SLA). Mixing these up is a classic tell.

### Error budgets — the killer concept ⭐
An SLO of **99.9%** means you're **allowed to fail 0.1%** — that's the **error budget**: ~**43
min/month** of downtime/errors you can "spend." Reframes reliability as a **budget that buys velocity**:
- **Budget remaining?** → ship faster, take risks, run experiments. Reliability is "good enough."
- **Budget exhausted?** → **freeze features, stabilize**, focus engineering on reliability until you're
  back in budget.

> **Think differently — more nines is NOT always better.** Each extra nine costs ~10× more (m02), and
> chasing 100% uptime **kills velocity** (you ship nothing for fear of breaking it). The error budget
> makes reliability a **deliberate, managed trade-off against feature speed**, owned jointly by dev and
> ops — not "always maximize uptime." Saying "I'd set an SLO and manage to an error budget" signals real
> SRE maturity.

---

## 3. The signals worth measuring: Golden Signals / RED / USE ⭐

Don't measure everything; measure the few signals that reveal health. Three popular, overlapping frames:
- **Google's Four Golden Signals:** **Latency** (how slow — use percentiles, m02), **Traffic** (how
  much demand — QPS), **Errors** (rate of failures), **Saturation** (how full — CPU/mem/queue/connection
  pool, i.e. how close to a limit).
- **RED (for request-driven services):** **Rate** (requests/sec), **Errors** (failed/sec), **Duration**
  (latency distribution). Simple and effective for APIs.
- **USE (for resources):** **Utilization**, **Saturation**, **Errors** — for CPUs, disks, queues.

> **Latency is bimodal — split success from failure.** Measure the latency of **successful** vs
> **failed** requests separately; a flood of fast errors can make average latency *look great* while the
> service is on fire. Always pair latency with error rate, and always use **percentiles** (p50/p99/p999),
> never averages (m02 tail latency).

---

## 4. Resilience patterns — contain the failure ⭐⭐ (the core toolkit)

How you stop one failing/slow dependency from taking down the whole system. Know all of these by name.

### Timeouts (the most basic, most forgotten)
**Never wait forever.** Every network call needs a timeout. Why it's critical: a slow dependency without
a timeout **ties up your threads/connections**, and via **Little's Law (m02)** that **collapses your
throughput** — one slow downstream service can exhaust your pool and take *you* down. A missing timeout
is a **latent outage**.

### Retries with exponential backoff + jitter ⭐
Transient failures (a blip, a brief overload) often succeed on retry — but **naive retries are
dangerous**:
- **Retry immediately/aggressively** → you **hammer a struggling service**, adding load when it's
  already failing → **retry storm** (a *metastable* failure: the system stays collapsed even after the
  original trigger passes, because retries keep the load high).
- **Fix:** **exponential backoff** (wait 1s, 2s, 4s…) so you back off as failures persist, **+ jitter**
  (randomize the wait) so all clients don't retry in lockstep (a synchronized thundering herd, m06).
- **Plus:** a **retry budget / cap** (limit total retries, e.g. ≤10% of traffic), only retry
  **idempotent** operations (m08), and only retry **retryable** errors (a 400 won't fix itself; a 503
  might).

```
  naive:  fail → retry now → fail → retry now ...  → retry STORM (amplifies the outage)
  good:   fail → wait 1s±j → wait 2s±j → wait 4s±j ... (backoff + jitter + cap + circuit breaker)
```

### Circuit breaker ⭐ (stop hammering a dead dependency)
Wraps a dependency and **trips** when it's failing, so you **fail fast** instead of piling up. Three
states:
- **Closed** (normal): requests flow; count failures.
- **Open** (tripped): too many failures → **immediately fail/fallback without calling** the dependency
  (give it time to recover; protect your own threads). After a cooldown →
- **Half-open:** allow a few trial requests; if they succeed → **close** (recovered); if they fail →
  back to **open**.

> The circuit breaker is the antidote to the retry storm — it converts "keep hammering a dead service +
> tie up our resources" into "fail fast, shed the dependency, recover gracefully." Pair retries +
> timeouts + circuit breaker together.

### Bulkheads (isolation — contain the blast radius)
Named after a ship's watertight compartments: **isolate resources** so one failing part can't sink the
whole. E.g. **separate thread pools / connection pools per dependency** — if dependency X hangs, it
exhausts only *its* pool, not the threads serving everything else. Also: isolate tenants, isolate
critical vs non-critical paths.

### Graceful degradation / fallbacks
When a dependency is down, **serve a reduced experience instead of an error.** Examples: show **cached/
stale** data (m06), hide a non-essential widget, return a default recommendation, **queue** the write to
process later (m08). Decide per-feature what "degraded but useful" looks like.

### Load shedding & rate limiting
When **overwhelmed**, **deliberately drop/reject some requests** (esp. low-priority) to keep the core
healthy for the rest — better to serve 90% well than fail 100%. Return **429/503 + Retry-After** (m03)
so clients back off. **Rate limiting** (m12) caps load proactively; **load shedding** sheds reactively
under stress. (This is the controlled version of the backpressure from m08.)

---

## 5. Cascading failure & blast radius ⭐ (why the patterns combine)

The patterns above exist because failures **cascade**: service A depends on B; B slows; A's requests to
B pile up (no timeout) → A's threads exhaust (Little's Law) → A becomes unresponsive → A's *callers*
pile up → the whole chain collapses from one slow leaf. **Tail latency amplification** (m02) makes it
worse — a fan-out request is as slow as its slowest dependency.

You **contain the blast radius** by combining: **timeouts** (don't wait on B), **circuit breakers**
(stop calling a sick B), **bulkheads** (B's failure exhausts only B's pool), **graceful degradation**
(serve without B), and **load shedding** (drop excess so the core survives). The goal: **a failure in
one component degrades *one feature*, not the whole system.**

> **Say this:** *"I assume every dependency can fail or slow down, so I wrap each call in a timeout +
> retry-with-backoff-and-jitter + circuit breaker, isolate it with a bulkhead, and define a graceful
> fallback — so one bad dependency degrades a single feature instead of cascading into a full outage."*
> That sentence is a senior reliability answer in one breath.

---

## 6. Observability — the three pillars ⭐⭐ ("you can't fix what you can't see")

**Observability** = the ability to understand a system's internal state from its outputs — *especially
to answer questions you didn't anticipate.* Built on three pillars:

- **Metrics:** numeric **time-series** (QPS, p99 latency, error rate, CPU, queue depth, cache hit
  ratio). Cheap, aggregable, great for **dashboards + alerting + trends**. Low cardinality; answer "how
  much / how fast / how many." (Prometheus, Grafana, Datadog.)
- **Logs:** discrete, timestamped **event records**. Use **structured logs** (JSON key-values, not free
  text) so they're queryable, with **levels** (DEBUG/INFO/WARN/ERROR) and **correlation/trace IDs** to
  tie a request's logs together. Centralize them (ELK/Loki/Splunk). Answer "what exactly happened in
  this event." High detail, higher cost/volume.
- **Traces (distributed tracing):** follow **one request across all services** as a tree of **spans**
  (each span = one operation with timing), stitched by a **trace ID** propagated through every hop.
  **Essential for microservices/async** (m08) where there's no single call stack — shows you *where* the
  latency/error is across services. (OpenTelemetry is the standard instrumentation; Jaeger/Tempo/Zipkin
  to view.)

```
  Metrics:  "p99 latency spiked to 2s at 14:03"          (WHAT/aggregate — alerts you)
  Traces:   "for this slow request, 1.8s was in service-C's DB call"  (WHERE — localizes it)
  Logs:     "service-C: query timeout, conn pool exhausted, retrying"  (WHY — root cause detail)
```

> **Think differently — monitoring vs observability.** **Monitoring** answers **known questions** —
> dashboards/alerts for failure modes you *predicted* ("known unknowns"). **Observability** lets you ask
> **new questions** about failures you *didn't* predict ("unknown unknowns") — which needs **high-
> cardinality, high-dimensional** data you can slice arbitrarily (e.g. "show p99 for users on app v2.3,
> in region X, hitting endpoint Y"). At scale most outages are novel, so you invest in observability,
> not just static dashboards. The pillars work **together**: a metric **alerts**, a trace **localizes**,
> logs give the **root cause**.

---

## 7. Alerting — alert on symptoms, not causes ⭐

Good alerting is hard; bad alerting causes **alert fatigue** (so many noisy pages that people ignore the
real one). Principles:
- **Alert on symptoms (user-facing SLO violations), not causes.** Page on "error rate > X / p99 >
  target / SLO burn rate" — things users *feel* — not on "CPU is 90%" (which may be fine). High CPU
  isn't a problem; **users getting errors** is.
- **Every page must be actionable** — if there's nothing to do, it shouldn't page (make it a dashboard/
  ticket instead). Tie alerts to a **runbook**.
- **Error-budget burn-rate alerts:** page when you're consuming the error budget too fast (e.g. "at this
  rate you'll exhaust the monthly budget in an hour") — catches real degradation, ignores trivia.
- **Severity tiers:** page (wake someone) vs ticket vs FYI. Reserve pages for "users are hurting now."

---

## 8. Deploy safety — the #1 cause of outages ⭐

Most outages come from **changes** (deploys, config), not random hardware failure. So **safe deployment
is a reliability feature**:
- **Rolling deploy:** replace instances gradually behind the LB (readiness + draining, m07) → zero
  downtime, but new+old run together briefly (needs backward compatibility).
- **Blue-green:** run two full environments (blue=current, green=new); cut traffic over at once; **roll
  back instantly** by switching back. Costs double infra during the switch.
- **Canary:** send a **small % of traffic** to the new version, **watch the metrics/SLOs**, then ramp up
  (or roll back) — catches bad releases with minimal blast radius. The senior default for risky changes.
- **Feature flags:** decouple **deploy** from **release** — ship code dark, then toggle the feature on
  for a % of users (and **kill-switch** it off instantly without a redeploy). Great for gradual rollout
  and fast mitigation.
- **Fast, practiced rollback:** the most important reliability lever — when a deploy goes bad, the fix is
  usually "**roll back now, debug later**." Make rollback one button and well-rehearsed.
- **Backward/forward compatibility:** during rolling/canary, old and new run together, and you must be
  able to roll back — so schema changes use **expand-contract** (m04) and APIs stay compatible.

> **Carry this:** *"Since deploys cause most outages, I make releasing safe: canary + feature flags +
> instant rollback, backward-compatible changes, and I watch SLOs during rollout."* This is exactly the
> discipline behind your parity-validated DCE migrations.

---

## 9. Testing failure: chaos engineering
You don't actually know your system is resilient until you **break it on purpose.** **Chaos
engineering** (Netflix's **Chaos Monkey**) **injects failures in production** (kill instances, add
latency, drop a dependency) — under control, with a hypothesis and a blast-radius limit — to **verify**
your timeouts/circuit breakers/failover actually work *before* a real incident does it for you. Related:
**game days** (rehearsed incident drills) and **fault injection** in testing. The philosophy: *the best
way to avoid being surprised by failure is to practice it.*

---

## 10. Operations: incident response, runbooks, postmortems, capacity

- **Incident response:** detect (alert) → mitigate (stop the bleeding — roll back, shed load, fail over —
  *before* root-causing) → resolve → learn. **Mitigate first, debug later.** Define roles (incident
  commander) for big ones.
- **Runbooks:** documented "if you see X, do Y" steps tied to alerts — so on-call can act fast without
  re-deriving everything at 3am.
- **Blameless postmortems** ⭐: after an incident, write up the timeline, impact, root cause, and
  **action items** — focused on **fixing the system/process, not blaming people.** People make mistakes;
  resilient systems make mistakes *safe* (you ask "why did the system *let* this happen?" not "who did
  it?"). Blame kills the honesty you need to actually learn.
- **Capacity planning & autoscaling:** provision for **peak** (m02), use **autoscaling** for predictable
  cycles, and **load-test** to find limits before users do. Watch **saturation** as the leading
  indicator.
- **Health checks** (m07): liveness (restart) vs readiness (remove from pool) underpin both LB routing
  and safe deploys.

---

## 11. From your systems 🏭
- **Reliability/correctness as a safety net:** DCE's **suppression layer** is **graceful-degradation /
  guard** logic at the exit — it blocks incorrect recommendations from reaching downstream, prioritizing
  **correctness** (a wrong auto-deny is worse than a pend). That's reliability (right answers) as a
  first-class NFR (§1), not just uptime.
- **Operational tooling = observability + mitigation:** your `claims_count` / `exclusions_count` daily
  tracking are **metrics**; `reprocess.py` / `aws_batch_reprocessing` are **mitigation/recovery** tools
  (replay, m08); `compare_results.py` parity scripts are how you **safely validated** the OpenShift→EKS /
  Azure→AWS / Snowpark migrations (§8 deploy safety, lived).
- **Running lean = operational maturity:** sustaining ~1M claims/day with the **team cut 20→4** is only
  possible with good automation, monitoring, and runbooks — the §10 story.
- **ML observability:** your AI/ML monitoring (drift via PSI/KS, alerts for **DATA_DRIFT / POSRATE_SHIFT
  / OVER_BUDGET**, retrain triggers) is observability applied to models — including a **cost** SLO
  (OVER_BUDGET) — and previews the ML-platform module (m28).
- **Health checks + safe deploys:** OpenShift/k8s **liveness/readiness** probes and rolling updates (m07)
  are §8 in production; the migration parity checks are canary-style validation.
- **Incident work:** the steady stream of JIRA defects/prod-support you handled is §10 incident
  response/triage — you can speak to "mitigate first, then root-cause."

---

## 12. Key concepts (interview-ready)
- **"Everything fails"** → design for failure + degradation, and treat deploys/monitoring/process as
  part of the system. Junior asks "make it work"; senior asks "how does it degrade, and how would I
  know?"
- **SLI (measure) ⊆ SLO (internal target) ⊆ SLA (external contract w/ penalties).** **Error budget** =
  1−SLO: spend it on velocity; if exhausted, freeze + stabilize. **More nines isn't always better.**
- **Signals:** Golden (latency, traffic, errors, saturation) / **RED** (rate, errors, duration) / USE.
  Split success vs error latency; use **percentiles**.
- **Resilience patterns:** **timeouts** (a missing one is a latent outage via Little's Law), **retries
  with backoff + jitter + budget** (naive retries → **retry storm / metastable failure**), **circuit
  breaker** (closed/open/half-open → fail fast), **bulkheads** (isolate pools → contain blast radius),
  **graceful degradation/fallbacks** (stale/cached/default vs error), **load shedding** (drop excess to
  save the core). They combine to stop **cascading failure**.
- **Observability = 3 pillars:** **metrics** (aggregate, alert — WHAT), **traces** (cross-service,
  localize — WHERE), **logs** (event detail, root cause — WHY), tied by **correlation/trace IDs**.
  **Monitoring** = known unknowns (dashboards); **observability** = unknown unknowns (high-cardinality,
  ask new questions).
- **Alert on symptoms (SLO/user-facing), not causes**; every page actionable + runbook; error-budget
  burn-rate alerts; avoid alert fatigue.
- **Deploys cause most outages** → **canary + feature flags + instant rollback + backward-compatible
  changes**; blue-green/rolling; **mitigate first, debug later**.
- **Chaos engineering** verifies resilience by injecting failure; **blameless postmortems** fix systems
  not people; provision for peak + autoscale + load-test.

---

## 13. Go deeper (the well-researched reading list)
- **Google SRE Book & SRE Workbook (free online)** — the canon: SLI/SLO/SLA, error budgets, golden
  signals, alerting, postmortems, toil. The definitive source for §2/§3/§7/§10.
- **Release It! — Michael Nygard** — the resilience-patterns bible: timeouts, circuit breakers,
  bulkheads, stability/anti-patterns (§4/§5). Highly recommended.
- **"Metastable Failures in Distributed Systems" (Bronson et al.)** + Marc Brooker's blog on retries/
  backoff/jitter — the deep §4 retry-storm material.
- **Charity Majors / Honeycomb on "observability vs monitoring"** and **OpenTelemetry docs** — for §6.
- **Netflix "Chaos Engineering" / Principles of Chaos (principlesofchaos.org)** — §9.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m10_reliability_observability/`](../../solutions/m10_reliability_observability/README.md).

🎉 **This completes PART 1 — The Building Blocks (m03–m10).** You now have the full toolkit:
networking, databases, replication/sharding, caching, load balancing, messaging, distributed-systems
theory, and reliability/ops. Say **"Module 11"** to start **PART 2 — Classic Design Problems** with the
*URL Shortener (TinyURL)* — where you finally **apply** all ten foundation modules end-to-end using the
m01 framework.
