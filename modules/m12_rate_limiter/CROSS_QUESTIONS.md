# m12 — Cross-Questions ("if-and-buts") on the Rate Limiter

> Answer out loud in 2–3 sentences before reading the model answer. The algorithm trade-offs and the
> distributed-counter atomicity are where this is won.

---

### Q1. Why do you need a rate limiter at all?
**A.** To protect the system from **abuse/DoS** (one client overwhelming you), enforce **fairness** (one
user can't starve others), control **cost** (esp. expensive endpoints / paid downstreams like LLM APIs),
and **prevent overload/cascading failure** (m10) by shedding excess proactively. It's the system's immune
system — the **proactive cousin of load shedding**: rate limiting caps load before it arrives; load
shedding drops it under stress.

---

### Q2. Walk me through the main algorithms and their trade-offs.
**A.** **Fixed window** (count per fixed bucket — simple, O(1), but a **2× boundary burst**); **sliding
window log** (store every timestamp — **exact** but **memory-heavy**); **sliding window counter**
(current+previous window estimate — O(1), smooths the burst, the **practical sweet spot**); **token
bucket** (tokens refill at a rate, consume per request — **allows bursts**, most popular); **leaky
bucket** (queue leaks at constant rate — **smooths output**, protects fixed-capacity downstreams). I pick
by **burst tolerance, accuracy, and memory**.

---

### Q3. What's the boundary-burst problem with fixed windows?
**A.** A fixed window resets at the boundary, so a client can send the full limit at the **end** of one
window and the full limit at the **start** of the next — e.g. with 100/min, 100 requests at `11:00:59`
and 100 at `11:01:00` → **200 in ~1 second**, all "legal" because they're in different windows. So fixed
windows can allow up to **2× the intended rate** at the edges. The sliding-window counter (or log) fixes
this by considering the actual trailing window.

---

### Q4. How does the sliding window counter work and why is it the "sweet spot"?
**A.** Keep counters for the **current** and **previous** fixed windows and estimate the trailing-window
count as `current + previous × (overlap fraction)` — e.g. 30s into the minute, `estimate = curr + prev ×
0.5`. It's **O(1) memory** (two counters) like fixed window, but it **smooths the boundary burst** like
the log, with only slight approximation (it assumes the previous window's traffic was uniform). That
accuracy-for-almost-no-memory is why Cloudflare uses it for most cases.

---

### Q5. Token bucket vs leaky bucket — when each?
**A.** **Token bucket** refills tokens at a steady rate up to a capacity and lets requests consume them —
so it **allows bursts** up to the bucket size (using idle capacity for spikes), with a smooth long-run
average. **Leaky bucket** queues requests and processes them at a **constant rate**, so output is always
smooth and **bursts are queued or dropped**. Use **token bucket** when you want to be burst-friendly (most
APIs); use **leaky bucket** when you must **protect a fixed-capacity downstream** with a strictly steady
rate (no bursts allowed through).

---

### Q6. Which algorithm uses the most memory, and why does that matter?
**A.** The **sliding window log** — it stores a **timestamp for every request** per key, so a high-volume
key can hold thousands of entries (and you trim + count on each request). At scale that's GBs of memory
and more CPU per request. It matters because the limiter is on the **hot path of every request**, so the
counter store must stay cheap and O(1) — which is why the **sliding window counter** or **token bucket**
(both O(1)) are preferred unless you truly need the log's exactness.

---

### Q7. How do you enforce a single limit across many app-server instances?
**A.** You need **shared state**. The standard is a **centralized counter in Redis** that all instances
read/write, so the limit is global regardless of which server a request lands on. The cost is a **network
hop to Redis per request** (latency) and Redis becoming a **hot path / potential SPOF** (replicate +
shard it). Alternatives: **per-node local** counters (`limit/N` each — fast but inaccurate) or a
**hybrid** (local counting + periodic sync) for approximate global limits at low latency.

---

### Q8. What's the race condition in distributed rate limiting and how do you fix it?
**A.** It's the **lost-update race** (m04): a naive `GET count → if < limit → INCR` lets two concurrent
requests both read `99`, both pass the check, and both increment to `101` → the limit is breached. The
fix is **atomicity**: use **`INCR`** (which atomically increments *and returns* the new value, so
check-and-increment is one step) with `EXPIRE` on first increment, or a **Lua script** that does the
whole check-decrement atomically (essential for token bucket: refill + consume in one script). Redis runs
Lua scripts atomically (single-threaded), so no race.

---

### Q9. The rate limiter is on every request — isn't it a bottleneck and SPOF?
**A.** Yes, which is why it must be **fast (O(1), in-memory) and highly available**. A slow limiter slows
*every* request (Little's Law, m02), and a dead one could block everything. Mitigations: **replicate and
shard** the Redis counter store, keep operations atomic O(1), deploy the limiter as **horizontally-scaled
edge middleware** (m07), and choose **fail-open** so a limiter outage degrades to "unprotected but up"
rather than "down" (Q10). You engineer the limiter with the same rigor as any critical tier.

---

### Q10. If the rate limiter's datastore (Redis) goes down, do you allow or block requests?
**A.** That's the **fail-open vs fail-closed** decision. **Fail-open (allow)** keeps the service
**available** but temporarily unprotected — the usual choice for general rate limiting, since the limiter
is a *protection*, not a correctness requirement, and you'd rather risk some overload than cause an
outage. **Fail-closed (deny)** protects the downstream but **takes you down** — appropriate for
**security-critical** limits (login/OTP/password-reset attempts) where allowing unlimited attempts is
worse than being unavailable. Decide per use case.

---

### Q11. What HTTP response and headers do you return when rate-limited?
**A.** **`429 Too Many Requests`**, with **`Retry-After`** (seconds until the client may retry) and the
**`X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset`** headers so well-behaved clients
know their budget and when it resets. Good clients then back off with **exponential backoff + jitter**
(m10). Clear headers prevent clients from blindly hammering and turn rate limiting into a cooperative
protocol rather than a wall.

---

### Q12. What should you key the rate limit on?
**A.** Depends on the goal. **Per user/API-key** is the usual — fair per-customer limits and works behind
NAT. **Per IP** catches anonymous abuse but is **coarse** (NAT/proxies share an IP, so you can throttle
many innocent users or miss attackers rotating IPs). **Per endpoint** protects expensive routes. Often
you **combine** (user+IP) and apply **tiered limits by plan** (free vs pro). You can also enforce
**multiple limits simultaneously** (e.g. 10/sec *and* 1000/hour) and reject if any is exceeded.

---

### Q13. Where in the architecture should the rate limiter live?
**A.** Usually as **middleware at the edge** — the **API gateway / reverse proxy** (m03/m07) — so every
service is protected uniformly and abusive traffic is rejected **before** it consumes app resources. It
can also live as a library in each service for service-specific limits, or as a dedicated rate-limiting
service the gateway calls. Edge placement is the default because it stops bad traffic earliest (cheapest)
and centralizes the policy.

---

### Q14. How do you allow bursts but still cap the long-run rate?
**A.** **Token bucket** is purpose-built for this: the bucket **capacity** sets the max burst (how many
requests can fire at once when tokens are saved up), while the **refill rate** sets the sustained
long-run average. So a client that's been quiet accumulates tokens and can spike up to `capacity`, then
is throttled to `refill_rate` — burst-friendly *and* bounded. (Leaky bucket, by contrast, would smooth
the spike away entirely.)

---

### Q15. A single abusive user is hammering one key — what happens to your Redis counters?
**A.** That key becomes a **hot key** (m05/m06) — all that user's counter ops hit **one Redis shard**,
which can saturate it. Mitigations: **shard counters** so load spreads (though one key still lands on one
shard), add a **local pre-filter** on each node for egregious offenders (reject locally before the Redis
hop), and tie into **DDoS/WAF** upstream for volumetric abuse. The limiter both *detects* the abuse and
must *survive* it — so the hot-key defenses from caching apply here too.

---

### Q16. How do clock/time issues affect the rate limiter?
**A.** Windows and token refill depend on **time**, and per-node wall clocks **drift/skew** (m09), so if
each node used its own clock you'd get inconsistent windows across nodes. The fix: rely on the **central
store's clock/time** (Redis `TIME`) or use **monotonic refill math** based on elapsed time stored with
the counter (`last_refill_time` + rate). This keeps the limit consistent regardless of individual server
clock differences — another reason the central store is convenient.

---

### Q17. How is rate limiting different from load shedding (m10)?
**A.** **Rate limiting** is **proactive and per-key**: it enforces a *predefined cap* per client/window,
regardless of current system load (you get 100/min whether the system is idle or busy). **Load shedding**
is **reactive and system-wide**: under *actual overload*, the system drops/rejects excess requests
(often low-priority first) to survive. They're complementary layers: rate limiting prevents any one
client from over-consuming; load shedding protects the whole system when aggregate demand exceeds
capacity anyway.

---

### Q18. How would you implement a token bucket in Redis safely?
**A.** Store **`{tokens, last_refill_ts}`** per key, and use a **Lua script** (atomic in Redis) that, on
each request: (1) computes elapsed time since `last_refill_ts`, (2) adds `elapsed × refill_rate` tokens
capped at `capacity`, (3) if `tokens ≥ 1`, decrements and returns "allow"; else returns "deny" with the
time until the next token. Doing it all in one Lua script makes refill+consume **atomic**, avoiding the
lost-update race, and keeps it O(1). (`redis-cell`/GCRA packages this for you.)

---

### Q19. Your API has free and paid tiers with different limits. How do you handle that?
**A.** Store **rules per tier** in a config store (hot-reloadable so you can change limits without
redeploys), key the limit on the **API key/user**, look up the user's tier → its limit, and apply the
chosen algorithm with that limit. You can layer **multiple limits** (per-second burst + per-day quota) and
different limits per endpoint. The limiter stays generic; the **rule lookup** is what varies — which is
why you externalize rules rather than hard-coding them.

---

### Q20. How do you rate-limit fairly behind shared IPs (corporate NAT, mobile carriers)?
**A.** IP-only limiting is **coarse and unfair** here — thousands of users behind one NAT'd IP share a
budget (you throttle innocents) and attackers can hide among them. Better: limit on a **more specific
identity** when available — **authenticated user/API key** (the real fairness unit), or a combination
(user when logged in, IP+device fingerprint for anonymous). For anonymous traffic you accept IP as a
coarse signal but set generous limits and lean on **WAF/anomaly detection** for abuse, rather than
punishing everyone behind a shared IP.
