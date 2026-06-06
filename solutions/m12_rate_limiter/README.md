# m12 — Solutions (Rate Limiter)

> Check the *reasoning*. Recurring lessons: pick the algorithm by burst/accuracy/memory; distributed =
> shared atomic counter (lost-update again); fail-open for protection, fail-closed for security.

---

## Exercise 1 — Pick the algorithm
1. **Token bucket** — allows bursts up to capacity while capping the long-run average at the refill rate.
2. **Leaky bucket** — enforces a strictly constant output rate (50/sec), queuing/dropping bursts to
   protect the fixed-capacity downstream.
3. **Sliding window counter** — O(1) memory, smooths the boundary burst, accurate enough.
4. **Fixed window** — simplest, O(1), tolerable inaccuracy at low traffic.
5. **Sliding window log** — exact count, at the cost of storing every timestamp (memory).

---

## Exercise 2 — The boundary burst
1. With 100/min fixed windows: send **100 requests at `11:00:59`** (fills the 11:00 window) and **100 at
   `11:01:00`** (fresh 11:01 window) → **~200 requests in ~1 second**, both legal.
2. **Sliding window log** (counts the actual trailing 60s, so the 11:00:59 batch still counts at
   11:01:00 → rejects) and **sliding window counter** (weights the previous window into the estimate, so
   the trailing burst is accounted for). Both consider the *real* recent window, not a reset bucket.
3. 40s into the minute → previous window overlaps the trailing 60s by **(60−40)/60 = 1/3 ≈ 0.33**.
   `estimate = current(30) + previous(80) × 0.33 ≈ 30 + 26.4 ≈ 56`. **56 < 100 → allowed.**

---

## Exercise 3 — Token bucket mechanics
1. Bucket full (10 tokens) → **all 10 allowed** (each consumes a token; bucket now empty).
2. Bucket is empty → **all 5 rejected** (429).
3. 3 seconds × 2 tokens/sec = 6 tokens refilled, capped at capacity → **6 tokens available** (not more,
   capacity = 10 but only 6 accrued).
4. **`capacity`** = the **maximum burst** (how many can fire at once when saved up); **`refill_rate`** =
   the **sustained long-run average** rate.

---

## Exercise 4 — The atomicity bug
1. Both requests run `GET count` → both read **99** → both check `99 < 100` → **both pass** → both `INCR`
   → count ends at **101** → limit exceeded (and a third could too). The check used a stale read.
2. The **lost-update race** (m04) — concurrent read-modify-write where one update is based on a stale read.
3. **Atomic fixes:** (a) **`INCR`** — it atomically increments *and returns* the new value, so you
   increment first then check the returned value (`if new_value > limit → reject`); the increment is a
   single atomic op, no stale read. (b) **A Lua script** that reads, checks, and increments/decrements in
   one atomic execution (Redis runs Lua atomically). Both eliminate the interleaving that caused the
   over-count.

---

## Exercise 5 — Distributed enforcement
1. **Local-only breaks** because each node only sees *its* share of the user's requests; a user spread by
   the LB across 10 nodes could send `limit` to *each* → up to **10× the global limit**.
2. **Centralized Redis:** all nodes increment a **shared counter** in Redis (atomically), so the limit is
   global and accurate. **Cost:** a **network hop to Redis per request** (latency) and Redis as a **hot
   path / SPOF** → must replicate + shard it.
3. **Hybrid:** each node counts **locally** and **periodically syncs** (or gossips) with a central store
   to approximate the global count. **Trade-off:** low per-request latency (local check) but **approximate**
   accuracy with a **sync-lag window** where the limit can be slightly exceeded.
4. **Distributed rate limiting trades accuracy vs latency vs coordination** — central = accurate but a
   hop/SPOF; local = fast but inaccurate; hybrid = approximate-global at low latency.

---

## Exercise 6 — Fail-open vs fail-closed
1. **Fail-open (allow):** keep serving requests unprotected → **risk:** overload/abuse goes unchecked
   while Redis is down (but the service stays up).
2. **Fail-closed (deny):** reject requests → **risk:** you cause an **outage** (legit users blocked) even
   though nothing was actually wrong with the app.
3. **Public read API → fail-open** (availability matters; the limiter is a protection, not correctness —
   better unprotected than down). **Login/OTP → fail-closed** (security-critical: allowing unlimited
   credential/OTP attempts is worse than being briefly unavailable). Decide by "what's worse: being down,
   or being unprotected?"

---

## Exercise 7 — Placement & response
1. **At the edge — the API gateway/reverse proxy** (m03/m07) — so all services are protected uniformly
   and abusive traffic is rejected **before** consuming app resources (cheapest, earliest).
2. **`429 Too Many Requests`** + **`Retry-After`**, **`X-RateLimit-Remaining`**, **`X-RateLimit-Reset`**
   (and `X-RateLimit-Limit`).
3. **Back off** — wait `Retry-After`, then retry with **exponential backoff + jitter** (m10) rather than
   immediately hammering again.

---

## Exercise 8 — Keys & tiers
1. **Per-IP:** catches anonymous abuse, no auth needed — but **coarse** (NAT/proxies share an IP →
   throttles innocents / misses IP-rotating attackers). **Per-API-key:** the real **fairness unit**
   (per customer), works behind NAT, ties to plans — but requires the request to be authenticated.
2. Behind NAT/carriers many users share one IP, so an IP limit punishes them all (or hides an attacker
   among them). **Instead** limit on **authenticated user/API key** when available; for anonymous, use
   generous IP limits + device fingerprint + WAF/anomaly detection.
3. Store **limits per tier in a hot-reloadable config**; key on the API key, look up the user's tier →
   its limit, apply the algorithm with that limit — change limits by editing config, **no redeploy**.
4. Maintain **two counters** (a per-second and a per-hour limit) and check **both**; reject if **either**
   is exceeded.

---

## Exercise 9 — Token bucket in Redis
1. Store **`tokens`** (current count) and **`last_refill_ts`** (when you last refilled) per key.
2. **Single Lua script** because refill (compute elapsed × rate, cap at capacity) and consume (decrement
   if ≥1) must be **atomic** — otherwise concurrent requests race on the token count (the lost-update bug).
   Redis executes a Lua script atomically (single-threaded).
3. Pseudocode:
   ```
   now = TIME()
   elapsed = now - last_refill_ts
   tokens = min(capacity, tokens + elapsed * refill_rate)
   if tokens >= 1:
       tokens -= 1; last_refill_ts = now
       return ALLOW, remaining=tokens
   else:
       return DENY, retry_after=(1 - tokens)/refill_rate
   ```
   The caller **allows** the request (and may set `X-RateLimit-Remaining`) or returns **429 + Retry-After**.

---

## Exercise 10 — Apply to m11 + your systems 🏭
1. **m11 create endpoint:** key on **user/API-key** (fall back to IP for anonymous), **token bucket**
   (allow short bursts of legit creates, cap the average), enforced at the **API gateway/edge** before it
   reaches the app + KGS, and **fail-open** (a limiter outage shouldn't stop URL creation — it's
   protection, not correctness). Add malware-scan + per-IP coarse limit as defense-in-depth.
2. **Your stack:** **Redis counters** live in your existing Redis (atomic `INCR`/Lua — same store you use
   for sessions/reference data); the **edge enforcement point** is **nginx** (Questimate's reverse proxy,
   where JWT is already validated). **DCE's exclusion-rules-at-entry** are the same idea: **reject/keep
   out at the boundary to protect the expensive core** — admission control, of which rate limiting is one
   form.
3. The **atomic-INCR** requirement is the **lost-update fix** (m04): a counter is a read-modify-write, and
   concurrent requests would race on a non-atomic check → over-count. Your reflex to use atomic ops /
   `SELECT FOR UPDATE` / version checks transfers directly — "make the read-modify-write atomic."
