# m02 — Exercises (Core Fundamentals)

> Do these on paper / out loud before checking `solutions/m02_fundamentals/`. The goal is fluency
> with the vocabulary and the numbers — you'll use both in every later module.

---

## Exercise 1 — Classify the requirement
For each, say whether it's **functional** or **non-functional**, and if non-functional, **which
"-ility"** (scalability / latency / availability / consistency / durability):
1. "Users can upload a profile photo."
2. "99.99% uptime."
3. "A posted comment is visible to the author immediately."
4. "Handle 1M concurrent users."
5. "A completed payment is never lost, even if a server crashes."
6. "Search results return in under 200 ms at p99."
7. "Two users transferring money never cause the balance to go negative."

---

## Exercise 2 — The latency ladder
Order these from fastest to slowest, and give the rough number for each: HDD seek · L1 cache · RAM
reference · cross-continent round trip · SSD random read · same-datacenter round trip. Then answer:
**how many times slower is a disk seek than a RAM reference (order of magnitude)?**

---

## Exercise 3 — Back-of-the-envelope (show steps, round hard)
A photo-sharing app:
- **300M daily active users**, each views **20 photos/day** and uploads **2 photos/day**.
- Average photo (stored, post-compression) **= 300 KB**; metadata row **= 1 KB**.
- Keep everything for **5 years**, **3× replication**.

Compute, and state **one architectural implication** for each:
1. Photo **reads/sec** (avg + peak at 3×) and **uploads/sec** (avg).
2. **Read:write ratio.**
3. **New photo storage/day** and **photo storage over 5 years** (with replication).
4. **Metadata storage over 5 years** (with replication) — compare its size to photo storage.
5. **Read bandwidth** (photos served/sec × size).
6. Where should photos live vs metadata? Why?

---

## Exercise 4 — Consistency model picker
For each, pick the **weakest consistency model that's still acceptable** (strong / read-your-writes /
eventual) and justify in one line:
1. A "likes" counter on a post.
2. A bank account balance during a withdrawal.
3. Your own newly-posted tweet appearing in your timeline.
4. The number of items left in stock during a flash sale.
5. A user's profile bio they just edited (shown to themselves).
6. A globally-distributed DNS record.

---

## Exercise 5 — CAP / PACELC scenarios
1. A network partition splits your 3-replica datastore (2 nodes on one side, 1 on the other). A
   write arrives at the minority side. Describe what a **CP** system does vs an **AP** system does.
2. Classify each as leaning **CP** or **AP**, and say why: a banking core ledger; a social-media
   "who's online" presence service; a shopping cart; a distributed lock service.
3. Give the **PACELC** label (PA/EL, PC/EC, etc.) you'd want for: a payment ledger; a product
   recommendation feed. Explain the "Else" (no-partition) half of each.

---

## Exercise 6 — Availability math
1. A request must pass through an API gateway (99.9%), an auth service (99.9%), and a database
   (99.95%), all **in series**. Rough end-to-end availability? Is it better or worse than any single
   component?
2. You replace the single database with **two redundant replicas** (each 99.9%) where the request
   succeeds if **either** is up. Rough availability of that pair?
3. What does this tell you about (a) long dependency chains and (b) redundancy?

---

## Exercise 7 — Latency vs throughput diagnosis
For each complaint, say whether it's a **latency** problem or a **throughput** problem, and one fix:
1. "Each search feels snappy, but the site falls over during the 9am rush."
2. "Traffic is low, but every page takes 4 seconds to load."
3. "Our batch job finishes fine, but real-time API p99 spiked to 2s after we added batching."
4. "We can serve 10K rps total, but we need 50K rps."

---

## Exercise 8 — Little's Law (capacity from first principles)
1. A payment API receives **3,000 requests/sec**; each request spends **40 ms** in the service
   (incl. a DB call). How many requests are **in flight** at any instant? How big a worker/connection
   pool would you provision (with headroom)?
2. The downstream DB slows down so each request now takes **120 ms**. If your pool is **fixed at
   150 workers**, what's the new **maximum throughput** the service can sustain? What happens to the
   requests beyond that rate, and how does this become an outage?
3. State the one-line lesson Little's Law teaches about the latency↔throughput↔concurrency triangle.

---

## Exercise 9 — Amdahl & the Universal Scalability Law (the limits of "add machines")
1. A job is **10% inherently serial** (a global lock). By Amdahl's Law, what's the **maximum
   speedup** you can ever get, even with infinite machines?
2. Your team doubles the cluster from 50 → 100 nodes and **throughput drops**. Name the law that
   predicts this and the mechanism causing it.
3. Give **three concrete design moves** that reduce the coordination/serial fraction so scaling stays
   closer to linear.

---

## Exercise 10 — Linearizability vs serializability
For each, say whether it primarily needs **linearizability**, **serializability**, **both (strict
serializability)**, or **neither**, and one line of why:
1. "Read my account balance and always see my most recent deposit."
2. "Transfer $100 from A to B — either both the debit and credit happen, or neither, and no
   interleaving with other transfers corrupts balances."
3. "A `like` count that can lag a bit."
4. "A globally-distributed bank that needs transaction isolation **and** real-time-fresh reads."

---

## Exercise 11 — Apply to your own system 🏭
For **Questimate** (or DCE), answer:
1. Name the tier that is **stateless** and how state was moved out of it.
2. Name one place you traded **throughput/background work for hot-path latency**.
3. Name one feature that's **eventually consistent** and one that's **strongly consistent**, and
   why each choice is correct.
4. What is the system's **dominant "-ility"** (availability? durability? latency? consistency?), and
   what architectural decision did it drive?

---

When done, check `solutions/m02_fundamentals/README.md`, then say **"Module 3"**.
