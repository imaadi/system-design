# m12 — Exercises (Rate Limiter)

> Do these on paper / out loud before checking `solutions/m12_rate_limiter/`. Focus on the algorithms and
> the distributed/atomicity details — that's where it's graded.

---

## Exercise 1 — Pick the algorithm
For each requirement, choose an algorithm and justify in one line:
1. "Be friendly to spiky clients but cap the long-run average."
2. "Protect a legacy downstream that can only handle exactly 50 req/sec, no bursts."
3. "We need accuracy with minimal memory and no boundary bursts."
4. "Dead-simple, low traffic, we can tolerate some inaccuracy."
5. "We need a perfectly exact count and can pay the memory."

---

## Exercise 2 — The boundary burst
Limit = 100 requests/minute, fixed window.
1. Construct a sequence where a client legally sends ~200 requests in ~1 second. Give timestamps.
2. Which two algorithms fix this, and how does each avoid it?
3. For the sliding-window counter: 40 seconds into the current minute, previous window had 80 requests
   and the current has 30. What's the estimated count, and is a new request allowed?

---

## Exercise 3 — Token bucket mechanics
A token bucket has capacity = 10, refill = 2 tokens/sec, currently full.
1. A burst of 10 requests arrives instantly. How many are allowed?
2. Immediately after, 5 more arrive in the same instant. Allowed or rejected?
3. After 3 seconds of silence, how many tokens are available (cap applies)?
4. What do `capacity` and `refill_rate` each control, in plain terms?

---

## Exercise 4 — The atomicity bug
Two requests for the same key arrive concurrently; the limit is 100 and the current count is 99.
1. Show how a naive `GET → check → INCR` lets both through (count ends at 101).
2. What is this bug called (you've seen it before)?
3. Give two atomic fixes and say why each is safe.

---

## Exercise 5 — Distributed enforcement
You run 10 app-server instances behind a load balancer; the limit is 1000/min per user.
1. Why does a purely local counter on each node break the limit?
2. Describe the centralized-Redis approach and its cost.
3. Describe a hybrid approach and its trade-off.
4. State the accuracy/latency/coordination trade-off in one sentence.

---

## Exercise 6 — Fail-open vs fail-closed
Your Redis counter store goes down.
1. What does fail-open do, and what's the risk?
2. What does fail-closed do, and what's the risk?
3. Which would you pick for: a public read API? a login/OTP endpoint? Justify each.

---

## Exercise 7 — Placement & response
1. Where in the architecture do you put the limiter, and why there?
2. What status code and which three headers do you return when a client is limited?
3. What should a well-behaved client do on receiving them?

---

## Exercise 8 — Keys & tiers
1. Pros/cons of limiting per-IP vs per-API-key.
2. Why is per-IP unfair behind corporate NAT / mobile carriers, and what do you do instead?
3. How would you support free (100/min) vs pro (10k/min) tiers without redeploying to change limits?
4. How do you enforce "10/sec AND 1000/hour" at the same time?

---

## Exercise 9 — Implement token bucket in Redis
Sketch (pseudocode) an atomic token-bucket check in Redis.
1. What two values do you store per key?
2. Why must the refill + consume happen in a single Lua script?
3. What does the script return, and what does the caller do with it?

---

## Exercise 10 — Apply it to m11 + your systems 🏭
1. Add rate limiting to m11's `POST /urls` create endpoint: which key, which algorithm, where enforced,
   and what failure mode (open/closed)?
2. Map this design to your stack: where would the **Redis counters** live, where is the **edge
   enforcement** point (hint: nginx), and how is DCE's **exclusion-rules-at-entry** the same "reject
   early to protect the core" idea?
3. Why is the atomic-INCR requirement the same instinct as your m04 lost-update fix?

---

When done, check `solutions/m12_rate_limiter/README.md`, then say **"Module 13"**.
