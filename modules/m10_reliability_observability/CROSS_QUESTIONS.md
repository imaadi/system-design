# m10 — Cross-Questions ("if-and-buts") on Reliability, Observability & Operations

> Answer out loud in 2–3 sentences before reading the model answer. These reveal whether you've *run*
> systems in production, not just built them.

---

### Q1. SLA vs SLO vs SLI — define them and how they relate.
**A.** **SLI** is the **measurement** (e.g. % of requests < 200 ms, % non-error). **SLO** is your
**internal target** for that SLI (e.g. 99.9% success over 30 days) — what you design and alert against.
**SLA** is the **external contract** with customers, with **penalties** if breached. They nest by
strictness: you measure (SLI), set a stricter internal target (SLO), and promise a looser external one
(SLA) — so you detect and fix problems before the contract breaks.

---

### Q2. What's an error budget and why is it useful?
**A.** It's **1 − SLO**: a 99.9% SLO means you're *allowed* ~0.1% failure (~43 min/month) — that's the
budget. It reframes reliability as a **resource you spend on velocity**: if budget remains, ship fast
and take risks; if it's exhausted, **freeze features and stabilize**. It turns "dev wants speed, ops
wants stability" into a shared, objective trade-off — and it stops the team from over- or under-investing
in reliability.

---

### Q3. Is 100% uptime (or "more nines") always the goal?
**A.** No. Each extra nine costs ~10× more (m02), and chasing 100% **kills velocity** — you ship nothing
for fear of breaking it, and you can't control dependencies/networks anyway. The right goal is an **SLO
that matches the business need** (a ledger needs more nines than an internal dashboard), managed via an
**error budget**. Reliability is a deliberate trade-off against feature speed, not a number to maximize
blindly.

---

### Q4. What are the golden signals you'd monitor for a service?
**A.** Google's four: **Latency** (how slow — in percentiles, splitting success vs error), **Traffic**
(demand/QPS), **Errors** (failure rate), and **Saturation** (how full — CPU/memory/queue/pool, i.e.
nearness to a limit). For request services the **RED** subset (Rate, Errors, Duration) is a great quick
frame; for resources, **USE** (Utilization, Saturation, Errors). I always use **percentiles, not
averages**, and pair latency with error rate.

---

### Q5. Why measure success-latency and error-latency separately?
**A.** Because a flood of **fast failures** can make average (or even p99) latency look *great* while the
service is actually broken — errors often return quickly (a fast 500), dragging the latency number down.
If you only watch latency you'd miss the outage. So I separate the latency of successful vs failed
requests and **always pair latency with the error rate** — the two together tell the real story.

---

### Q6. Why is a missing timeout a latent outage?
**A.** Because a slow dependency without a timeout **ties up your threads/connections** while you wait.
By **Little's Law (m02)**, if requests take much longer (W↑) and your pool is fixed (L capped), your
**throughput collapses** (λ = L/W) — new requests queue, latency climbs, and one slow downstream service
**takes you down too**. So every network call needs a timeout; it's the most basic resilience control
and the most commonly forgotten.

---

### Q7. How can retries make an outage *worse*?
**A.** Naive/aggressive retries **amplify load on an already-struggling service**: it's slow/failing →
clients retry immediately → even more load → it fails harder → more retries → a **retry storm**, which
can become a **metastable failure** (the system stays collapsed even after the original trigger clears,
because retries keep the load pinned high). The fix: **exponential backoff + jitter** (back off and
desynchronize), a **retry budget/cap**, **circuit breakers**, and only retry **idempotent**, **retryable**
errors.

---

### Q8. Why exponential backoff *and* jitter — isn't backoff enough?
**A.** Backoff stops a single client from hammering, but if **many clients** fail at once and all back
off by the same schedule, they **retry in lockstep** — synchronized waves that re-overload the service
(a thundering herd, m06). **Jitter** randomizes each client's wait so retries **spread out** in time,
smoothing the load. You need both: backoff to reduce frequency, jitter to desynchronize.

---

### Q9. Explain the circuit breaker pattern and its states.
**A.** It wraps a dependency and **trips when it's failing** so you **fail fast** instead of piling up.
**Closed:** normal, requests flow, failures counted. **Open:** failure threshold exceeded → calls
**immediately fail/fallback without hitting the dependency** (giving it time to recover and protecting
your own threads). After a cooldown → **Half-open:** let a few trial requests through; success → close
(recovered), failure → reopen. It's the antidote to retry storms — convert "keep hammering a dead
service" into "shed it fast, recover gracefully."

---

### Q10. What's a bulkhead and what does it protect against?
**A.** Named after a ship's watertight compartments — you **isolate resources** so one failing part can't
sink the whole. Concretely: **separate thread/connection pools per dependency**, so if dependency X hangs
it exhausts only *X's* pool, leaving threads free to serve everything else. It **contains the blast
radius** — without it, one hung dependency consumes all your threads (Little's Law) and the whole service
dies. Also used to isolate tenants or critical vs non-critical paths.

---

### Q11. What is graceful degradation? Give an example.
**A.** Serving a **reduced but useful** experience when a dependency fails, instead of erroring out. E.g.
if the recommendation service is down, show a **default/popular** list; if the live price feed is down,
show the **last cached** price with a "stale" note; if a non-essential widget's backend fails, **hide
it** and render the rest; if the write store is down, **queue** the write (m08). You decide per feature
what "degraded but useful" means — better than a blank error page.

---

### Q12. What is load shedding and how does it differ from rate limiting?
**A.** **Load shedding** is **reactive**: when overwhelmed, you **deliberately drop/reject** some
requests (preferably low-priority) to keep the core healthy for the rest — better to serve 90% well than
fail 100%. **Rate limiting** (m12) is **proactive**: cap each client's request rate to prevent overload
in the first place. Both protect the system; shedding kicks in under stress (return 429/503 +
Retry-After), rate limiting enforces a steady cap. Together they prevent meltdown.

---

### Q13. Walk me through how one slow dependency cascades into a full outage, and how you'd prevent it.
**A.** Service A calls B; B slows; A's calls to B pile up (no timeout) → A's thread pool exhausts (Little's
Law) → A stops responding → A's callers pile up → the chain collapses from one slow leaf, amplified by
**tail latency** in any fan-out. Prevention is the combined toolkit: **timeouts** (don't wait on B),
**circuit breaker** (stop calling a sick B), **bulkheads** (B exhausts only its pool), **graceful
fallback** (serve without B), **load shedding** (drop excess). Goal: one bad dependency degrades **one
feature**, not the whole system.

---

### Q14. What are the three pillars of observability and what does each answer?
**A.** **Metrics** = aggregate time-series (QPS, p99, error rate, saturation) → answer **"what / how
much"** and drive dashboards + alerts. **Traces** = follow one request across services as spans tied by a
trace ID → answer **"where"** the latency/error is (essential in microservices/async, m08). **Logs** =
detailed event records (structured, with correlation IDs) → answer **"why"** (root cause). They work
together: a metric **alerts** you, a trace **localizes**, logs give the **detail**.

---

### Q15. What's the difference between monitoring and observability?
**A.** **Monitoring** answers **questions you knew to ask** — dashboards/alerts for failure modes you
predicted ("known unknowns"). **Observability** lets you answer **questions you didn't predict** ("unknown
unknowns") by collecting **high-cardinality, high-dimensional** data you can slice arbitrarily after the
fact (e.g. "p99 for users on app v2.3, region X, endpoint Y"). At scale most incidents are novel, so you
invest in observability, not just static dashboards — though you need both.

---

### Q16. How do you debug a slow request in a microservices/async system?
**A.** **Distributed tracing**: propagate a **trace/correlation ID** through every hop (sync calls and
message metadata, m08) so all spans for one logical request are stitched into a trace; the trace shows
**which service/span ate the time** (e.g. "1.8s of 2s was service-C's DB call"). Then I pull that
service's **logs by the same correlation ID** for root cause, and check its **metrics** (saturation,
error rate) for context. Without tracing there's no single call stack across services, so you'd be
guessing.

---

### Q17. What makes a good alert, and what causes alert fatigue?
**A.** A good alert is **actionable** and fires on **symptoms users feel** (SLO violations: error rate,
p99, error-budget burn rate), tied to a **runbook**. **Alert fatigue** comes from paging on **causes**
(CPU 90%, disk 70%) that may be harmless, and from non-actionable noise — people start ignoring pages and
miss the real one. So I **alert on symptoms not causes**, page only when "users are hurting now," and
route everything else to dashboards/tickets with severity tiers.

---

### Q18. Why alert on symptoms instead of causes? Isn't high CPU a problem?
**A.** Not necessarily — a service can run at 90% CPU and serve every request within SLO; that's
*efficient*, not broken. What matters is whether **users are getting errors or slow responses**. If you
page on CPU you get false alarms (fatigue) and you can also **miss** real problems that don't show up as
high CPU. So I page on user-facing symptoms (SLO burn), and use causes like CPU as **diagnostic context**
(dashboards) once a symptom alert fires.

---

### Q19. What's the most common cause of outages, and what follows from that?
**A.** **Changes** — deploys and config changes — not random hardware failure. It follows that **safe
deployment is a core reliability feature**: **canary** releases (small % first, watch SLOs), **feature
flags** (decouple deploy from release, instant kill-switch), **fast rehearsed rollback** ("roll back
now, debug later"), **backward-compatible** changes (expand-contract, m04), and watching SLOs during
rollout. Redundancy guards against hardware; deploy discipline guards against the bigger risk.

---

### Q20. Canary vs blue-green vs rolling deploy?
**A.** **Rolling:** replace instances gradually behind the LB (readiness + draining) — zero downtime, but
old+new run together (needs compatibility) and rollback is slower. **Blue-green:** two full environments;
switch all traffic at once, **roll back instantly** by switching back — fast rollback but double infra
during the switch. **Canary:** send a **small % of traffic** to the new version, **watch metrics/SLOs**,
then ramp — smallest blast radius, best for risky changes, but needs good metrics + automation. I default
to **canary + feature flags** for anything risky.

---

### Q21. What's a feature flag and how does it help reliability?
**A.** A feature flag **decouples deploy from release**: you ship the code "dark" (off), then toggle the
feature on — for a % of users, specific cohorts, or everyone — **without a redeploy**, and **kill-switch
it off instantly** if it misbehaves. Reliability wins: gradual rollout limits blast radius, mitigation is
a toggle (no emergency deploy), and you can A/B or ramp safely. The cost is flag sprawl/tech debt, so you
clean up stale flags.

---

### Q22. What is chaos engineering and why would you intentionally break production?
**A.** **Chaos engineering** injects controlled failures into (often) production — kill instances, add
latency, drop a dependency — with a hypothesis and a **limited blast radius**, to **verify your
resilience actually works** (timeouts, circuit breakers, failover) *before* a real incident tests it for
you. The philosophy: you don't truly know the system is resilient until you've broken it on purpose;
better to discover a missing timeout during a controlled experiment than during a 3am outage. Netflix's
Chaos Monkey is the canonical example.

---

### Q23. What's a blameless postmortem and why "blameless"?
**A.** After an incident you write up the **timeline, impact, root cause, and action items** — focused on
**fixing the system and process, not blaming individuals**. It's blameless because people make mistakes,
and the real question is "**why did the system let a normal human action cause an outage?**" (missing
guardrail, no canary, bad default). Blame makes people hide information and stop reporting near-misses,
which destroys the learning you need to actually get more reliable. You fix the system so the next person
can't make the same mistake.

---

### Q24. During an active incident, what's your first priority?
**A.** **Mitigate first, root-cause later** — stop the user impact before understanding *why*. The fastest
mitigations are usually **roll back the recent deploy**, **flip a feature flag off**, **fail over** to a
healthy replica/region, or **shed load**. Once the bleeding stops and users are okay, *then* investigate
root cause calmly and write a blameless postmortem. For big incidents, assign an **incident commander** to
coordinate. Heroically debugging while users suffer is the junior mistake; restoring service first is the
senior one.
