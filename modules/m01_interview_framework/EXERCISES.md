# m01 — Exercises (the framework)

> Goal: build muscle memory for **driving the 7 steps**, not for any specific system yet. Do these
> out loud / on paper. Try them **before** opening `solutions/m01_interview_framework/`.

---

## Exercise 1 — Scope a vague prompt (requirements muscle)
Prompt: **"Design a system to shorten URLs."**
Write down, in 5 minutes:
- 3–5 **functional** requirements (and explicitly cut 2 to "stretch").
- 5 **non-functional** requirements with a rough target each (scale, latency, availability,
  consistency, read/write ratio).
- 3 clarifying questions you'd ask, and the **assumption** you'd state if the interviewer shrugs.

---

## Exercise 2 — Back-of-the-envelope drills (estimation muscle)
No calculator. Round hard. Show the steps.
1. A service gets **500M reads/day** and **5M writes/day**. Average reads/sec? writes/sec?
   read:write ratio? Estimate **peak** reads/sec (state your multiplier).
2. Each write stores a **~1 KB** record. How much **new storage per day**? Per **year**? With
   **3× replication**?
3. A photo service stores **10M photos/day** at **~1.5 MB** each. Daily storage? 5-year storage?
4. Convert: **2 billion requests/day** → requests/sec (use 1 day ≈ 10⁵ s).
5. For each result above, write **one architectural implication** ("…therefore I'd …").

---

## Exercise 3 — Name the property, then the product
For each need, write (a) the **generic component + the property** it provides, then (b) a concrete
product you'd name:
1. "I need to serve the same hot value to 100K reads/sec with sub-ms latency."
2. "I need to decouple a fast producer from a slow consumer and survive consumer downtime."
3. "I need an ordered, durable, **replayable** event log multiple consumers can read independently."
4. "I need to spread stateless app traffic across N replicas and route around dead ones."
5. "I need globally low-latency delivery of static images/video."

---

## Exercise 4 — Trade-off framing (say the cost)
Rewrite each junior statement into a senior one of the form *"X buys A, costs B; I'd pick X because…"*:
1. "I'll just use NoSQL because it scales."
2. "I'll cache everything."
3. "I'll make it strongly consistent."
4. "I'll add a message queue."
5. "I'll shard the database."

---

## Exercise 5 — The deep-dive template
Pick **unique short-code generation** for a URL shortener. Produce a 60–90 second deep-dive that
includes: at least **3 options**, the **trade-off** of each, your **pick**, the **justification**,
and the **residual cost/failure mode** of your pick. (This is the template you'll reuse in every
problem module.)

---

## Exercise 6 — Stress your own design (bottleneck muscle)
Take the simple URL-shortener high-level design (DNS → LB → stateless app → cache → KV DB). Without
being prompted, list:
- 2 **single points of failure** and a mitigation for each.
- 1 **hot-key** scenario and how you'd handle it.
- 1 **consistency window** (where could a read see stale data?) and how you'd bound it.
- What happens **if the cache tier dies entirely** — does the system survive? At what cost?

---

## Exercise 7 — Map your own work to the framework 🏭
For **DCE** (or Questimate), fill in the 7 steps as if it were an interview answer:
1. Requirements (FRs + the dominant NFR).
2. Scale estimate (claims/sec average & peak reasoning).
3. The core "API"/interface (what enters and leaves the pipeline).
4. Data model + **why MongoDB for claims and Redis for reference data**.
5. High-level flow (ingestion → exclusions → models+rules → suppressions → output).
6. One deep-dive (e.g. the suppression layer, or Redis-backed JWT on Questimate).
7. Bottlenecks/evolution (a real migration you did, with the savings number).

> This exercise *is* your m35 story bank in seed form. Get it tight; you'll use it in real interviews.

---

## Exercise 8 — Spot the red flag
For each candidate behavior, name the red flag and the senior alternative:
1. Starts drawing Kafka + Cassandra + K8s 30 seconds in.
2. Goes quiet for 2 minutes while thinking.
3. Says "this design handles billions of users" for a stated 10K-user internal tool.
4. When challenged on a choice, defends it harder instead of reconsidering.
5. Finishes the happy path and says "I think that's it" with 15 minutes left.

---

When done, check `solutions/m01_interview_framework/README.md`, then say **"Module 2"**.
