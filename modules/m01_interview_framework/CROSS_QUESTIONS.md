# m01 — Cross-Questions ("if-and-buts") on the Framework

> These are the **meta** follow-ups — about *running* the interview, not a specific system.
> Answer each out loud in 2–3 sentences before reading the model answer. The model answers are how
> a senior would respond.

---

### Q1. The interviewer just says "Design Instagram." Where do you start? Don't you waste time asking questions?
**A.** I start with **requirements**, and no — clarifying is the opposite of wasting time; designing
the wrong system wastes time. I'd spend ~5 min: "What's the one core feature — photo upload + feed?
Are we including stories, DMs, search? What scale — millions or billions of users? Read-heavy
(viewing) I assume?" Then I state assumptions and get a nod. Scoping is itself a graded skill.

---

### Q2. What if the interviewer is silent / gives you nothing and just watches?
**A.** Then *I* drive. I state my assumptions explicitly ("I'll assume 100M DAU, read-heavy, global,
managed cloud OK — stop me if that's off") and proceed. A silent interviewer is often testing
whether I can lead under ambiguity. I narrate continuously and check in at milestones.

---

### Q3. How do you decide functional vs non-functional requirements quickly?
**A.** **Functional = what it does** (verbs/features: "post a tweet", "see a feed"). **Non-functional
= how well it must do it** (the -ilities: scale, latency, availability, consistency, durability).
Quick test: if it's a feature a PM would list, it's functional; if it's a quality an SRE cares
about, it's non-functional. NFRs drive architecture more than FRs do.

---

### Q4. You estimated the QPS. So what? Why does the interviewer care about your arithmetic?
**A.** They don't care about the digits — they care that I **convert scale into an architectural
decision**. The number's job is to answer "one machine or a thousand?", "cache enough or shard?".
So I always pair the estimate with its implication: "~50 GB/day → I'll need partitioning within a
year, so I'll design for it now." The implication is the point.

---

### Q5. Average QPS vs peak QPS — which do you design for, and why?
**A.** You **provision for peak**, but you reason from average. Peak ≈ 2–10× average for typical
consumer traffic (state your multiplier). Designing for average gets you paged during the daily
spike; designing for 100× peak wastes money. I size capacity for realistic peak plus headroom, and
use autoscaling + queues to absorb bursts beyond that.

---

### Q6. When do you choose the database — and isn't picking NoSQL "to scale" always right?
**A.** I choose it **after** I know the **access patterns and consistency needs**, in the data-model
step — not reflexively. "NoSQL scales" is a junior reflex; modern SQL (Postgres, Spanner, Aurora)
scales massively and gives transactions for free. I pick based on: point-key access vs complex
joins, transactional needs, read/write ratio, and consistency. The honest line: "I'd default to a
relational store unless an access pattern or scale property specifically pushes me off it."

---

### Q7. How much should you draw before talking about scale?
**A.** Draw the **minimal happy-path** that satisfies the functional requirements first (single
region, the core services, one DB, a cache). *Then* evolve it when a bottleneck or NFR forces a
change. Drawing 20 boxes up front is a red flag — it looks like pattern-matching, not reasoning.

---

### Q8. The interviewer pushes back: "Why not just use one big SQL database?" How do you respond?
**A.** With curiosity, not defense. "Great question — for the stated scale, could a single
well-tuned Postgres with read replicas handle it? Let me check the numbers… at ~1K writes/sec, yes,
honestly it could, and it'd be simpler. I'd only shard when writes exceed what one primary can take
or the dataset outgrows one node. So my answer depends on the scale we agreed on." Showing you'd
*not* over-engineer is itself senior.

---

### Q9. You're 25 minutes in and barely past the high-level design. What do you do?
**A.** Time-triage out loud: "I want to make sure we cover a deep-dive and failure modes, so I'll
lock the high-level here and go deep on [the most load-bearing component]." Better to deliberately
cut than to run out of time mid-diagram. Naming the trade-off ("I'm prioritizing depth over the
analytics feature") is the senior move.

---

### Q10. How deep is "deep enough" in a deep-dive?
**A.** Three levels: **component → data structure/algorithm → failure mode**. E.g. cache → "I'd use
an LRU in Redis, keyed by X" → "on a node loss, the ring rehashes ~1/N keys; I warm critical keys /
use replicas to avoid a thundering herd on the DB." If I can name the data structure, the algorithm,
and what breaks, that's deep enough for one component.

---

### Q11. You don't know a technology the interviewer mentions (e.g. "would you use Spanner?"). What now?
**A.** Be honest and reason from properties. "I haven't run Spanner in production, but what I'd want
here is **external consistency + horizontal scale with transactions** — if Spanner provides that,
it's a strong fit; I'd validate the latency and cost trade-offs." Reasoning about the *property* you
need, even without hands-on, is far better than bluffing — bluffs get probed and collapse.

---

### Q12. Functional requirements keep growing ("now add stories, now add DMs"). How do you keep control?
**A.** Treat each addition as a scoped extension: "Adding DMs changes the data model — it's a
write-heavy, point-to-point pattern vs the fan-out feed, so I'd add a separate messages store and
maybe a queue. Want me to design that fully or keep it as an extension?" I integrate it through the
framework rather than bolting a box on, and I keep checking scope so I don't silently over-commit.

---

### Q13. Should you mention specific products (Kafka, Redis, DynamoDB) or stay generic ("a message queue")?
**A.** Both, in order: name the **generic component and the property** first ("I need a durable,
ordered, replayable log → a distributed log"), *then* anchor it ("e.g. Kafka"). Leading with the
property proves understanding; the product name shows familiarity. Leading with only the product is
buzzword soup.

---

### Q14. How do you handle "what if traffic grows 100×?" mid-design?
**A.** Walk the request path and find what saturates first. "At 100×: the app tier scales
horizontally (stateless), the cache tier needs more shards/consistent hashing, the DB write path is
the real limit → shard by key or move to a wide-column store; reads are absorbed by cache + CDN.
I'd also add backpressure/queues so a spike degrades gracefully instead of falling over." The skill
is *locating the bottleneck*, not listing scaling tricks.

---

### Q15. The interviewer asks for a feature you think is a bad idea. Do you push back?
**A.** Yes, respectfully, with a trade-off framing: "We can do strong consistency on the feed, but
it'll cost write latency and availability during partitions (CAP) — for a social feed, eventual
consistency is usually the better trade. Do you want me to design the strongly-consistent version
anyway?" Disagreeing *with reasoning* and then deferring to their call is senior; blindly complying
or blindly refusing both score worse.

---

### Q16. How do you close strong in the last 5 minutes?
**A.** Three moves: (1) **summarize** the design in 3 sentences + the 2–3 key trade-offs I made,
(2) name the **bottlenecks/SPOFs** and how I'd mitigate them, (3) say **what I'd do next with more
time** ("add multi-region for availability, instrument p99 and cache hit rate, load-test the shard
rebalance"). Ending on trade-offs + next steps signals I know a design is never "done."

---

### Q17. Is there a difference between a "product design" and an "infrastructure design" interview?
**A.** Yes. **Product/feature design** (design Twitter) emphasizes FRs, data model, and the
read/write path. **Infra design** (design a rate limiter / a message queue / a key-value store)
emphasizes algorithms, consistency, and failure semantics — lighter on product FRs, heavier on the
deep-dive. Same framework; you just shift weight from steps 3–4 toward step 6.

---

### Q18. How is an ML/LLM system-design interview different from a classic one?
**A.** Same 7-step backbone, plus ML-specific layers: in *requirements* you frame it **as an ML
problem** (what's predicted, what's the label, online vs batch); *data/features* becomes a first-class
step (pipelines, **training-serving skew**, feature store); *serving* adds latency budgets, batching,
and model rollout (shadow/canary/A-B); and *evaluation* splits into **offline metrics + online
metrics** with a feedback loop. We build that explicitly in m22 (ML) and m29–m34 (LLM).

---

### Q19. What's the most common reason strong engineers still fail these?
**A.** Two: (1) **skipping requirements** and designing the wrong thing well, and (2) **going silent
/ not narrating trade-offs** — doing good work in their head that the interviewer can't see. The fix
for both is the framework: scope first, then think out loud at every decision with a number and a
trade-off.

---

### Q20. How do you avoid both under-engineering and over-engineering?
**A.** Anchor every component to a **requirement or a number**. If I can't trace a box back to an FR,
an NFR, or an estimate, I shouldn't add it (that's over-engineering). If an NFR (latency,
availability, scale) has no component serving it, I'm under-engineering. The estimate is the referee:
"the numbers say I need X" or "the numbers say one box is fine."
