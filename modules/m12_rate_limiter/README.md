# Module 12 — Design a Rate Limiter ⭐⭐

> A **rate limiter** caps how many requests a client may make in a time window — the system's
> immune system against **abuse, DoS, runaway costs, and overload**. It's a favorite interview problem
> because it's small in scope but **algorithmically rich** (five classic algorithms with real
> trade-offs) and exposes a deep distributed-systems issue (a **shared counter** = coordination, m09).
> We bolt this onto m11's create endpoint and any public API.

> **Format reminder (Part 2):** this README is a worked design via the m01 framework; the **algorithms**
> and **distributed counting** are the deep-dives. `CROSS_QUESTIONS.md` drills the follow-ups.

---

## Step 1 — Requirements

**Functional:**
1. **Limit** requests to **N per time window** per **key** (key = user ID / API key / IP / endpoint).
2. **Reject** over-limit requests with **HTTP 429 Too Many Requests** (+ `Retry-After`).
3. **Configurable rules** (different limits per endpoint, per user tier, etc.).

**Non-functional (these shape it):**
- **Low latency overhead:** it runs on **every request** → must add **sub-millisecond** cost. *It must
  not become the bottleneck it's protecting against.*
- **Accurate enough:** don't badly over- or under-count (the accuracy bar depends on the use case).
- **Distributed:** the limit must hold across **many app-server instances** (a user hitting different
  servers shouldn't get N× the limit) → shared state.
- **Highly available + fail-safe:** if the limiter's datastore is down, decide **fail-open vs
  fail-closed** (a real design choice — Step 7).
- **Scalable:** millions of keys, high QPS.

> **Clarifying questions to ask:** limit per *what* (user/IP/API-key/endpoint)? Global or per-region?
> Do we allow **bursts** or enforce a **steady rate**? What happens on limiter failure? These answers
> pick the algorithm and topology.

---

## Step 2 — Estimation (lightweight, but note the constraints)
The limiter is mostly about **latency + memory**, not throughput math:
- It's on the **critical path of every request** → budget **< 1 ms** added latency → the counter store
  must be **in-memory** (Redis / local memory), never a disk DB.
- **Memory:** per active key you store a counter (a few bytes) or, for a log algorithm, **every
  timestamp** (much more). At, say, 10M active keys: counters ≈ tens of MB (cheap); a sliding-window
  **log** for high-volume keys could be **GBs** (the memory-vs-accuracy trade, Step 4).
- Throughput: the limiter must sustain the **full request QPS** (e.g. 100k/s from m11) → atomic,
  O(1) operations only.

---

## Step 3 — API / where it sits
The rate limiter is usually **middleware at the edge** (API gateway / reverse proxy, m03/m07) so every
service is protected uniformly and bad traffic is rejected **before** it reaches your app logic.

```
  client ─▶ API Gateway ─[rate limiter checks key]─┬─ under limit ─▶ app/service
                                                    └─ over limit  ─▶ 429 Too Many Requests
```

**Response on limit:** **`429 Too Many Requests`** + headers:
- `Retry-After: 30` (seconds to wait) · `X-RateLimit-Limit: 100` · `X-RateLimit-Remaining: 0` ·
  `X-RateLimit-Reset: <epoch>`. Good clients back off (exponential backoff + jitter, m10).

---

## Step 4 — Deep Dive: the five algorithms ⭐⭐ (the heart)

Each trades **accuracy vs memory vs burst behavior**. Know all five and when to use each.

### (1) Fixed Window Counter — simplest
Divide time into fixed windows (e.g. 1-minute buckets). Keep **one counter per key per window**;
increment on each request; reject when it exceeds the limit; the counter resets at the window boundary.
- **Pro:** trivially simple, **O(1)** memory/time (one integer per key).
- **Con:** ⚠️ the **boundary burst** — a client can send up to **2× the limit** around a boundary: with
  100/min, 100 requests at `11:00:59` and 100 at `11:01:00` = **200 in ~1 second**, all "legal" because
  they fall in different windows.
```
  limit 100/min:  |--- 11:00 window ---|--- 11:01 window ---|
                              ▲100 reqs  ▲100 reqs   → 200 in ~1s across the boundary
```

### (2) Sliding Window Log — perfectly accurate
Store a **timestamp for every request** (per key, e.g. in a Redis sorted set). On each request, drop
timestamps older than the window, **count** what's left; if `< limit`, add the new timestamp and allow.
- **Pro:** **exact** — no boundary problem; the window truly slides.
- **Con:** **memory-heavy** — stores *every* request's timestamp (a hot key = thousands of entries), and
  more compute per request (trim + count). Expensive at scale.

### (3) Sliding Window Counter — the practical sweet spot ⭐
An **approximation** that fixes the boundary burst cheaply. Keep counters for the **current** and
**previous** fixed windows; estimate the count over the sliding window as:
`count ≈ current_window_count + previous_window_count × (overlap fraction)`.
- E.g. 30s into the current minute: `estimate = curr + prev × 0.5`. If `estimate < limit`, allow.
- **Pro:** **O(1)** memory (two counters), **smooths the boundary burst**, accurate enough for almost
  everything. *This is what Cloudflare uses.*
- **Con:** approximate (assumes the previous window's traffic was uniform) — slight inaccuracy, rarely
  matters.

### (4) Token Bucket — allows bursts (most popular) ⭐
A bucket holds up to **`capacity`** tokens and **refills at `refill_rate` tokens/sec**. Each request
**consumes one token**; if the bucket is empty → reject (429). You store just **(token_count,
last_refill_time)** per key and lazily compute refill on access.
- **Pro:** **allows bursts** up to `capacity` (great for bursty-but-bounded traffic — uses idle
  capacity), smooth long-run average = `refill_rate`, O(1) memory. **The industry default** (AWS, Stripe,
  many gateways).
- **Con:** a full bucket lets `capacity` requests through at once (a burst by design).
```
  Token Bucket:  [🪙🪙🪙🪙🪙] capacity=5, refill 1/sec
   request → take a token (allow) ; empty → reject ; tokens regenerate over time → bursts then steady
```

### (5) Leaky Bucket — smooths to a constant rate
Requests enter a **queue** (the bucket); they're processed (**leak**) at a **constant rate**. If the
queue is full → reject. Output is a **steady stream** regardless of bursty input.
- **Pro:** **enforces a constant output rate** — ideal for protecting a **fixed-capacity downstream**
  (it never lets a burst through).
- **Con:** bursts are **queued (added latency) or dropped**; it **can't** use idle capacity for a spike
  (the opposite of token bucket); needs a queue.

| Algorithm | Memory | Accuracy | Bursts? | Notes |
|---|---|---|---|---|
| Fixed window | O(1) | low (boundary burst) | — | simplest; 2× at edges |
| Sliding window **log** | **O(N)** per key | **exact** | no | memory-heavy |
| Sliding window **counter** | O(1) | high (approx) | smoothed | **practical sweet spot (Cloudflare)** |
| **Token bucket** | O(1) | good | **allows bursts** | **most popular (AWS/Stripe)** |
| Leaky bucket | O(queue) | good | **smooths/blocks** | constant output rate |

> **The senior decision rule:** *Token bucket if you want to **allow bursts** (most APIs — be friendly to
> spiky-but-bounded clients). Leaky bucket if you must **protect a fixed-rate downstream** (smooth the
> flow). Sliding-window counter when you want accuracy + tiny memory without burst tolerance. Fixed
> window only for the simplest cases (and know its boundary flaw). Sliding-window log only when you need
> exactness and can pay the memory.*

---

## Step 5 — Deep Dive: distributed rate limiting ⭐⭐ (the genuinely hard part)

One server is easy (a local in-memory counter). The hard part: **you have many app-server instances**
behind a load balancer (m07), and the limit must hold **across all of them** — otherwise a user spread
across 10 servers gets 10× the limit. This is a **shared-counter / coordination** problem (m09).

**Approach A — Centralized counter in Redis (the standard).** All instances read/write counters in a
**shared Redis**. Accurate global limits.
- **The atomicity trap (this is the m04 lost-update bug!):** a naive `GET count → if < limit → INCR` is a
  **race** — two concurrent requests both read `99`, both allow, both increment to `101` → limit
  breached. **Fix:** use an **atomic operation** — `INCR` (returns the new value, so check-and-increment
  is one step) with `EXPIRE` set on first increment, or a **Lua script** that does the whole
  check-decrement atomically (essential for token bucket: compute refill + consume in one atomic
  script). *Redis is single-threaded, so a Lua script runs atomically.*
- **Cost:** a **network hop to Redis on every request** (adds latency) and **Redis becomes a hot
  path / potential bottleneck + SPOF** → replicate it, shard counters across a Redis cluster, and
  decide fail-open/closed (Step 7).

**Approach B — Local counters (per node).** Each node enforces `limit / number_of_nodes` locally — no
coordination, fastest.
- **Pro:** zero network hop, no SPOF. **Con:** **inaccurate** — uneven load across nodes means some
  users get throttled early and others late; the LB algorithm (m07) affects fairness.

**Approach C — Hybrid (local + async sync).** Each node counts locally and **periodically syncs** with a
central store (or gossips) to approximate the global count.
- **Pro:** low per-request latency (local check) with approximate global accuracy. **Con:** complexity +
  a sync lag window where limits can be slightly exceeded.

> **The trade-off triangle (say this):** *"Distributed rate limiting trades **accuracy vs latency vs
> coordination cost** (m09). A **central Redis counter** is accurate but adds a hop and is a hot
> path/SPOF; **per-node local** counters are fastest but inaccurate; a **hybrid** (local + periodic
> sync) approximates global limits cheaply. I'd default to **Redis with atomic Lua scripts** for
> correctness, sharded + replicated, and consider hybrid if Redis latency/throughput becomes the
> bottleneck."*

---

## Step 6 — Rules & keys
- **Key dimensions:** per **user/API-key** (the usual — fair per-customer limits), per **IP** (anonymous
  abuse, but NAT/proxies share IPs so it's coarse), per **endpoint** (protect expensive routes), or
  combinations. Often **tiered by plan** (free = 100/min, pro = 10k/min).
- **Rule storage:** keep limit rules in a config store (with hot-reload) so you can change limits without
  redeploys; the limiter looks up the rule per request.
- **Multiple limits at once:** e.g. 10/sec **and** 1000/hour — check all; reject if any is exceeded.

---

## Step 7 — Bottlenecks, failure & edge cases (stress it)

- **The limiter as SPOF/bottleneck:** it's on every request, so it **must be HA and fast** — replicate
  Redis, shard counters, keep ops O(1) and atomic. A slow limiter slows *everything* (Little's Law, m02).
- **Fail-open vs fail-closed** ⭐ — *what if Redis/the limiter is down?*
  - **Fail-open (allow):** keep serving (unprotected) → preserves **availability**; you risk overload but
    don't cause an outage yourself. **Usual choice for rate limiting** (it's a protection, not
    correctness).
  - **Fail-closed (deny):** block everything → protects the downstream but **you go down**. Choose this
    for **security-critical** limits (e.g. login/OTP attempts) where allowing unlimited is worse than
    being down.
- **Hot key:** a single abusive user hammering one key = a **hot key** on one Redis shard (m05/m06) →
  shard counters, and consider local pre-filtering for the worst offenders.
- **Race conditions:** solved by **atomic INCR / Lua** (Step 5) — never GET-then-set.
- **Clock issues:** windows/refill depend on time; rely on the **central store's clock** (or monotonic
  refill math) rather than per-node wall clocks (m09 clocks).
- **Abuse evasion:** attackers rotate IPs/keys → combine keys (user+IP), add anomaly detection, and tie
  to broader DoS protection (CDN/WAF).
- **Good DX:** return clear `429 + Retry-After + X-RateLimit-*` so well-behaved clients back off
  gracefully (m10) instead of hammering.

**Wrap-up:** *"A fast, edge-deployed limiter using a **token bucket** (to allow bursts) with counters in
**atomic Redis (Lua)** for accurate distributed limits, sharded + replicated, **failing open** so a
limiter outage doesn't take down the API, returning **429 + Retry-After**. Key trade-offs: token bucket
(bursts) vs leaky bucket (smoothing); central Redis (accurate, a hop) vs local (fast, approximate); and
fail-open (availability) vs fail-closed (protection). It's the proactive cousin of load shedding (m10)."*

---

## State-of-the-art & real-world notes 📚
- **Cloudflare** popularized the **sliding-window counter** approximation (great accuracy at O(1)
  memory). **Stripe/AWS/Kong/Envoy** commonly use **token bucket**. **Nginx** ships a **leaky-bucket**
  limiter (`limit_req`).
- **Redis is the de-facto distributed counter store** — `INCR`+`EXPIRE` for windows, **Lua scripts** for
  atomic token-bucket logic; libraries like `redis-cell` (a Redis module implementing GCRA, a token-
  bucket variant) exist.
- **GCRA (Generic Cell Rate Algorithm):** a precise token-bucket-equivalent used in high-end limiters —
  worth name-dropping as the "fancy token bucket."
- Rate limiting sits in a **defense-in-depth** stack: CDN/WAF (volumetric DDoS) → gateway rate limiting
  (per-key) → app-level quotas → downstream load shedding (m10).

---

## From your systems 🏭
- **Redis as the counter store:** you already use **Redis** for hot, atomic state (sessions, reference
  data) — a rate limiter is the same pattern: atomic `INCR`/Lua counters in Redis on the hot path. Natural
  fit for your stack.
- **Gateway/edge enforcement:** Questimate's **nginx + JWT-at-the-edge** is exactly where a rate limiter
  belongs (m03/m07) — reject abusive create traffic before it reaches Django.
- **Admission control analogy:** DCE's **exclusion rules at pipeline entry** ("keep certain claims out of
  the pipeline") are conceptually **admission control / gating** — the same "reject early at the boundary
  to protect the expensive core" instinct as rate limiting.
- **Lost-update reflex:** the atomic-INCR-vs-GET-then-set race **is** the m04 lost-update problem — your
  reflex to use atomic ops / `SELECT FOR UPDATE` transfers directly.
- **Fail-open thinking:** matches your reliability instincts (m10) — a protection layer shouldn't itself
  cause the outage it's meant to prevent.

---

## Key concepts (interview-ready)
- **Rate limiting = the proactive cousin of load shedding (m10):** protect against abuse/DoS/cost/overload
  by capping requests/window/key; reject with **429 + Retry-After + X-RateLimit-***.
- **Five algorithms:** **fixed window** (simple, **boundary burst → 2×**), **sliding window log** (exact,
  **memory-heavy**), **sliding window counter** (O(1), smoothed, **the sweet spot — Cloudflare**),
  **token bucket** (**allows bursts**, most popular — AWS/Stripe), **leaky bucket** (**smooths to constant
  rate**, protects fixed-capacity downstream). Pick by **burst tolerance + accuracy + memory**.
- **Distributed = shared counter = coordination (m09):** **central Redis** (accurate, a hop, hot
  path/SPOF) vs **per-node local** (fast, inaccurate) vs **hybrid** (local + sync). Use **atomic
  INCR/Lua** — naive GET-then-set is the **lost-update race** (m04).
- **Place it at the edge/gateway** (m03/m07); keep it **O(1) and HA** (it's on every request — Little's
  Law).
- **Fail-open vs fail-closed:** rate limiting usually **fails open** (availability); security limits
  **fail closed** (protection).
- **Keys:** per user/API-key (fair), per IP (coarse, NAT issues), per endpoint, tiered by plan; support
  multiple simultaneous limits.

---

## Go deeper (reading)
- **Cloudflare blog: "How we built rate limiting… sliding window"** — the canonical sliding-window-counter
  writeup (Step 4).
- **Stripe blog: "Scaling your API with rate limiters"** — token bucket + multiple limit types in
  production.
- **System Design Interview (Alex Xu), "Design a rate limiter"** — the standard interview treatment.
- **Redis docs: `INCR` pattern + Lua scripting + `redis-cell`/GCRA** — the distributed-counter mechanics
  (Step 5).
- Revisit **m04** (lost update/atomicity), **m05/m06** (hot keys), **m09** (coordination), **m10** (load
  shedding) — this problem is those applied.

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m12_rate_limiter/`](../../solutions/m12_rate_limiter/README.md).

When you've done the exercises, say **"Module 13"** to design a *News Feed / Twitter Timeline*
(fan-out on write vs read, the celebrity problem, feed ranking) — the first "big" product design, where
the hot-key/celebrity thread from m05/m06 becomes the central challenge.
