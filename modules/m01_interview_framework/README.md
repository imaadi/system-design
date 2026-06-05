# Module 1 — The Interview Framework & Thinking Like an Architect ⭐

> Before any "design Twitter", you need a **method** — a repeatable way to drive a 45-minute
> open-ended conversation from a vague one-liner to a defensible architecture. This module is that
> method. Master it and every later module just fills in the *content*; the *process* is always this.

**Why this matters most:** the interviewer is not grading whether you know the "right answer"
(there isn't one). They're grading a senior signal: *can this person take ambiguity, structure it,
make trade-offs out loud, quantify, and adjust under pressure?* The framework is how you broadcast
that signal on purpose instead of hoping it shows.

---

## 1. What a system-design interview actually tests

It's a **simulation of you on the job**, compressed into ~45 min. Five things are being scored:

| Signal | What it looks like | How to show it |
|---|---|---|
| **Problem structuring** | You don't panic at ambiguity; you carve it into pieces. | Drive the framework below. Lead, don't wait to be led. |
| **Breadth** | You know the building blocks (LB, cache, queue, DB types…). | Name the right tool *and why*, not just buzzwords. |
| **Depth** | You can go 3 levels deep on at least one component. | Pick a component, zoom in, discuss data structures/algorithms/failure. |
| **Trade-off reasoning** | You say "X buys us A but costs B; I'd pick X because…". | Never present a choice as free. Always name the cost. |
| **Communication** | Clear, structured, collaborative, quantified. | Think out loud, draw, check in, use numbers. |

> **The senior mindset in one line:** *"There is no best design — only the best design for **these**
> requirements and constraints."* Everything you say should be traceable back to a requirement or
> a number. If you can't justify it, don't add it.

**Levels of seniority (how the same question is graded differently):**
- **Junior:** lists components, can build the happy path.
- **Mid:** picks reasonable tech, knows the trade-offs when prompted.
- **Senior:** *drives* the conversation, **brings up** trade-offs/failures/scale unprompted,
  quantifies, and ties every decision to a requirement. ← **this is your target.**

---

## 2. The framework (memorize the shape, not the words)

Many acronyms exist (RESHADED, RADIO, PEDALS…). They're the same idea. Here's the one I want you
to run — **7 steps, ~45 min**. The time budget is a guide, not a rule; the *order* matters most.

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  1. REQUIREMENTS    (functional + non-functional + constraints)   ~5 min   │
   │  2. ESTIMATION      (scale: QPS, storage, bandwidth — back-of-envelope) ~5  │
   │  3. API / INTERFACE (the contract: endpoints / methods)            ~3 min   │
   │  4. DATA MODEL      (entities, which database, access patterns)    ~5 min   │
   │  5. HIGH-LEVEL DESIGN (boxes & arrows: the happy path end-to-end)  ~10 min  │
   │  6. DEEP DIVE       (zoom into 1–2 components the interviewer cares about) ~12 │
   │  7. BOTTLENECKS & WRAP (scale it, fail it, secure it, trade-offs)  ~5 min   │
   └──────────────────────────────────────────────────────────────────────────┘
        ▲                                                                 │
        └──────────── iterate: new requirement → revisit earlier steps ◄──┘
```

The arrow back is important: design is **not** linear. A deep-dive often reveals you need to revisit
the data model. Say so out loud — *"this changes my storage choice, let me update the diagram."*
That visible iteration is a green flag.

---

### Step 1 — Requirements (the most important 5 minutes) ⭐⭐

Most failed interviews are **lost here**, not in the tech. If you design the wrong system
beautifully, you fail. **Slow down and scope.**

**1a. Functional requirements (FRs)** — *what the system does* (features, as verbs):
> "Users can shorten a URL. Users can visit a short URL and get redirected. (Maybe: custom alias,
> expiry, analytics.)"

Write them as a short list. Then **explicitly cut scope**: *"For a 45-min design I'll focus on
shorten + redirect, and treat analytics as a stretch goal — does that work?"* Cutting scope on
purpose is a senior move; trying to design everything is a junior one.

**1b. Non-functional requirements (NFRs)** — *the qualities* (the "-ilities"). These **drive the
architecture** far more than features do:
- **Scale:** how many users / requests / data? (drives estimation → next step)
- **Latency:** how fast? (p50/p99 — "redirect must be <100 ms p99")
- **Availability:** how much uptime? (99.9%? 99.99%? → redundancy decisions)
- **Consistency:** does everyone need to see the same value instantly, or is "eventually" fine?
  (This single question reshapes the whole design — see m02 CAP.)
- **Durability:** can we ever lose data? (a payment: never. a "like": maybe okay.)
- **Read vs write ratio:** read-heavy (URL shortener ~100:1) vs write-heavy (logging) → totally
  different designs.

**1c. Constraints & assumptions** — surface the boundaries:
> "Assume global users. Assume we can use managed cloud services. Assume ~100M new URLs/day. Is
> there a cost ceiling? Single region or multi-region?"

**How to actually run this step:** *ask clarifying questions, then state your assumptions and get a
nod.* You're co-designing. Example openers:
- "Who are the users and what's the single most important thing they do?"
- "Roughly what scale — thousands, millions, billions?"
- "Is this read-heavy or write-heavy?"
- "What matters more here: that it's always available, or always perfectly consistent?"

> ⚠️ **The trap:** jumping to "I'll use Kafka and Cassandra" before you know the requirements.
> That's solution-first. **Requirements-first** is the senior pattern.

---

### Step 2 — Estimation (back-of-the-envelope) ⭐

Now put **numbers** on the NFRs. You're not after precision — you're after the **order of magnitude**
that decides architecture (does this fit on one machine, or do I need 1,000?). Full number tables
are in **m02**; the *method* lives here.

**The three things to estimate:**
1. **Throughput (QPS):** requests/sec, average **and peak**.
2. **Storage:** bytes/item × items × years (+ replication factor).
3. **Bandwidth:** QPS × payload size.

**The technique (round aggressively):**
- Convert per-day to per-second using **1 day ≈ 10⁵ seconds** (86,400 ≈ 100,000).
  - *"100M writes/day ÷ 10⁵ s ≈ **1,000 writes/sec** average."*
- **Peak ≈ 2–10× average.** State your multiplier. ("I'll assume 5× → ~5,000 writes/sec peak.")
- **Read:write ratio** scales reads off writes. ("100:1 read-heavy → ~100K reads/sec average.")
- **Storage:** *items/day × bytes/item × seconds_per_day_factor × retention × replication.*
  - *"100M URLs/day × 500 bytes ≈ 50 GB/day → ~18 TB/year → ×3 replicas ≈ 55 TB/yr."*

**Powers of two / data sizes you must know cold** (full table in m02):
- 2¹⁰ ≈ 1 thousand (KB), 2²⁰ ≈ 1 million (MB), 2³⁰ ≈ 1 billion (GB), 2⁴⁰ ≈ 1 trillion (TB).
- A char ≈ 1 byte; a typical short text record ≈ hundreds of bytes; an image ≈ 100s of KB–MBs.

**Why it matters:** the number *chooses the architecture for you*. 1,000 QPS fits on a couple of
servers + a cache. 1,000,000 QPS needs sharding, fan-out, CDNs. **State the implication**, e.g.
*"~50 GB/day means we'll need partitioning within a year, so I'll design for it now."*

> 💡 Don't burn 10 minutes on arithmetic. Get the magnitude, state the architectural consequence,
> move on. Interviewers love *"~X, which means I need Y"* — the *implication* is the point.

---

### Step 3 — API / Interface (the contract)

Define the **interface** before the internals. This pins down what the system promises and forces
you to confirm the FRs are covered. Keep it small — the 2–4 core operations.

```
POST /urls            { long_url, custom_alias?, expiry? }  -> { short_url }
GET  /{short_code}    -> 302 redirect to long_url
GET  /urls/{code}/stats -> { clicks, ... }   (stretch)
```

Decisions to *mention* (don't over-engineer): REST vs gRPC vs GraphQL (m03), idempotency of
writes, pagination for list endpoints (`limit` + cursor), auth (who can call this). For an
*internal* high-throughput call you might say "I'd use gRPC here for lower overhead"; for a public
API, "REST/JSON for ubiquity."

> Tip: choosing the API also reveals **idempotency** needs early — *"POST /urls should be
> idempotent on (user, long_url) so retries don't create duplicates."* That's a senior detail.

---

### Step 4 — Data Model (entities, storage choice, access patterns)

List the **entities** and their key fields, then choose **where each lives** and **how it's
accessed** — because *access pattern drives the database choice*, not the other way around.

```
URL:  { short_code (PK), long_url, user_id, created_at, expiry }
User: { user_id (PK), email, ... }
Click (analytics, optional): { short_code, ts, ip, ua, ... }  ← append-only, huge, time-series
```

Then reason out loud:
- **What are the queries?** "Lookup by short_code (point read, the hot path), list by user_id."
- **SQL or NoSQL?** "Point reads by key, massive scale, simple relations → a **key-value/wide-column
  store** is a fine fit. If I needed multi-row transactions I'd lean SQL." (Full decision tree in m04.)
- **Index** the columns you query (m04). **Partition/shard** by the access key if it won't fit on
  one node (m05).
- Separate **hot, small, transactional** data from **cold, huge, append-only** data (e.g. clicks →
  a time-series/columnar store or a data lake), so analytics doesn't slow the redirect path.

> The senior move: *"I'd store URLs in a KV store keyed by short_code for O(1) redirects, and stream
> click events to a separate analytics pipeline so the write path doesn't touch the read path."*

---

### Step 5 — High-Level Design (boxes & arrows: the happy path)

Now draw the **end-to-end flow** for the core FRs — client → edge → services → data — and **trace a
request through it out loud**. Start simple (single server), then add the obvious pieces.

```
            ┌─────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────────────┐
  client ──▶│   DNS   │──▶│ Load Balancer│──▶│  App servers   │──▶│   Cache      │
            └─────────┘   │   (L7)       │   │ (stateless,    │   │ (Redis)      │
                          └──────────────┘   │  many replicas)│   └──────┬───────┘
                                             └───────┬────────┘          │ miss
                                                     │                   ▼
                                                     │            ┌──────────────┐
                                                     └───────────▶│   Database   │
                                                                  │ (KV, sharded)│
                                                                  └──────────────┘
```

**Narrate the path:** *"A redirect hits DNS → L7 load balancer → a stateless app server → it checks
Redis for the short_code; on a hit it 302-redirects in <1 ms; on a miss it reads the DB, populates
the cache, and redirects. Writes go straight to the DB and invalidate/populate cache."*

Principles to apply here (each is a later module):
- **Statelessness** so any server can handle any request → easy horizontal scaling (m02, m07).
- **Cache the hot read path** (m06). **Load-balance** across replicas (m07).
- **Separate read and write paths**; push slow/async work onto a **queue** (m08).
- Add a **CDN** for static/global content (m06).

> Keep it *clean*. A common failure is drawing 20 boxes immediately. Draw the **minimum that
> satisfies the FRs**, then *grow* it in the deep-dive when a bottleneck forces you to.

---

### Step 6 — Deep Dive (depth on 1–2 components) ⭐⭐

This is where senior is won or lost. The interviewer will steer: *"how does the cache stay
consistent?", "what happens when a shard is hot?", "how do you generate unique short codes?"* Pick
the meaty component and go **three levels down**: data structure → algorithm → failure mode.

How to choose what to deep-dive (if they let you choose): pick the part that's **most novel or most
load-bearing** for *this* problem. For a URL shortener that's **unique ID/short-code generation** and
**the cache**. For a chat app it's **connection management + message fan-out**. For a feed it's
**fan-out on write vs read**.

A good deep-dive sounds like: *"Short-code generation. Option A: hash(long_url) + take 7 chars —
risks collisions, need a check-and-retry. Option B: a global counter, base-62 encoded — no
collisions but the counter is a bottleneck and a single point of failure. Option C: pre-generate
ranges per server from a key-generation service (like a ticket server / Snowflake-style IDs) — no
coordination on the hot path. I'd pick C because it scales horizontally; the cost is operational
complexity of the KGS, which I'd make HA with replication."* — Notice: **options, trade-offs, a
pick, a justification, the residual cost.** That's the template for *every* deep-dive.

---

### Step 7 — Bottlenecks, Failure & Wrap-up

Proactively stress your own design. **Bringing these up unprompted is the strongest senior signal.**

Run this checklist out loud:
- **Single points of failure (SPOF):** "The KGS is a SPOF → I'd replicate it / pre-fetch ranges so
  a brief outage doesn't block writes."
- **Scaling the bottleneck:** "At 10× traffic the DB write path saturates → shard by short_code;
  reads are already absorbed by cache + read replicas."
- **Failure modes:** "If the cache dies, reads fall through to the DB — I size DB read capacity for
  that, or use a replicated cache." (See **circuit breakers, retries, graceful degradation** — m10.)
- **Hot spots:** "A viral link is a hot key → I'd add a CDN / local cache tier for it." (m06)
- **Consistency edge:** "On a write, there's a window where cache is stale → short TTL or
  write-through." (m06)
- **Security/abuse:** "Rate-limit creation to stop abuse (m12); validate/blocklist malicious URLs."
- **Observability:** "I'd emit metrics (QPS, p99, cache hit rate, error rate) and alerts." (m10)

Then **summarize**: restate the design in 3 sentences, the 2–3 key trade-offs you made, and *what
you'd do next with more time*. Ending with *"with more time I'd add X and measure Y"* shows you
know it's never done.

---

## 3. The skills *around* the framework

### Communicate like a senior
- **Think out loud — always.** Silence reads as "stuck." Narrate even your uncertainty: *"I'm
  weighing SQL vs a wide-column store here; let me think about the access pattern."*
- **Structure your speech:** "There are three options… I'll go with the second, because…"
- **Check in:** "Does that scope sound right?" / "Want me to go deeper on the cache or the
  sharding?" It's collaborative, not a monologue.
- **Be quantitative:** attach a number to every claim you can. "~1K QPS, so a single Redis handles
  it." (See m02 numbers.)
- **Be honest about trade-offs and gaps:** *"I don't have production experience with Spanner, but
  the property I want is external consistency — I'd evaluate it for that."* Honesty > bluffing;
  interviewers probe bluffs.

### Draw like a senior
- Boxes = services/datastores; arrows = data flow (label them: "read", "async event").
- **Left→right** = request flow; **top** = clients/edge, **bottom** = storage.
- Group by tier (edge / app / data). Keep it legible; you'll keep editing it.
- On a whiteboard/excalidraw, leave space — you *will* add a cache, a queue, a replica later.

### Manage the clock
- If you're 20 min in and still gathering requirements, you've over-scoped — **cut**.
- Don't rabbit-hole arithmetic or one tiny feature. **Breadth first, then one deep dive.**
- Leave ~5 min for bottlenecks/wrap — interviewers remember the ending.

---

## 3.5 What a great answer literally *sounds like* (the first 4 minutes)

Abstract steps are easy to nod at and hard to do. Here's the **actual cadence** of a strong opening
on *"Design a URL shortener"* — notice it's a *conversation*, every claim has a number or a
trade-off, and the candidate is **driving**:

> **You:** "Before I design anything — let me scope it. The two core features are *create a short
> URL* and *redirect a short URL to the original*. I'll treat custom aliases, expiry, and analytics
> as stretch goals. Sound right?"
> **Interviewer:** "Yes, focus on create + redirect."
> **You:** "Good. Non-functionally, the big one is this is **massively read-heavy** — people click
> links far more than they create them, I'd guess ~100:1 — and **redirect latency is on the user's
> critical path**, so I'll target sub-100 ms p99. Availability matters a lot: a dead redirect breaks
> every link ever shared, so I want at least three nines. For consistency, eventual is fine — a new
> link being visible a second late won't hurt — but the mapping must be **durable**, links live
> forever. Let me put rough numbers on it…"
> **You:** "Say **100M new URLs/day**. That's 100M over ~10⁵ seconds ≈ **1,000 writes/sec** average,
> maybe 5,000 at peak. At 100:1 reads that's **~100K reads/sec** — *that* number tells me the whole
> game is the **read path**: writes are trivial for a single primary, but reads need a cache and
> probably a CDN. Storage: 100M × ~500 bytes ≈ **50 GB/day**, ~18 TB/year, so I'll need partitioning
> within a year — I'll design for it now. Let me sketch the interface, then the data model…"
> **You:** "Two endpoints: `POST /urls` returns a short code; `GET /{code}` 302-redirects. I'd make
> the POST **idempotent on (user, long_url)** so retries don't create duplicates — small detail, but
> it bites in production. Now the interesting question is *how I generate the short code*, which I
> think is the real meat here — want me to go deep on that, or finish the high-level first?"

That's it. In four minutes you've shown: scoping, the read-heavy insight, quantification *with
implications*, an idempotency detail, and you've **teed up your own deep-dive while checking in.**
Every sentence is traceable to a requirement or a number. **That's the sound of senior.** Record
yourself doing this for one problem and play it back — the gaps (silence, un-justified choices, no
numbers) are obvious on replay.

---

## 4. Red flags vs green flags (how you're actually scored)

| 🚩 Red flags (avoid) | ✅ Green flags (do) |
|---|---|
| Jumping to tech before requirements | Scope FRs/NFRs first, then design |
| Buzzword soup ("I'll use Kafka, K8s, Cassandra…") with no *why* | Name a tool **and the property** it gives you |
| Presenting choices as free | "X buys A, costs B; I pick X because…" |
| Going silent | Continuous narration |
| Over-engineering for imaginary scale | Design for the *stated* scale; mention how it'd evolve |
| Ignoring failure until asked | Proactively raise SPOFs, hot keys, partial failure |
| No numbers | Back-of-envelope at every decision point |
| Defensive when challenged | "Good point — that changes things; let me adjust" |
| One giant diagram, never revisited | Start minimal, evolve it as bottlenecks appear |

> The single biggest one: **handling pushback gracefully.** When the interviewer says "but what if
> X?", they're usually *helping* — they found the interesting edge. The right reaction is curiosity
> and adjustment, never defensiveness.

---

## 5. From your systems 🏭 (rehearse these — they're your edge)

You've *driven* this framework for real. Frame your experience in its language:

- **Requirements/NFRs:** DCE's hard NFR is **correctness + auditability** on ~1M claims/day — a
  wrong auto-deny has real consequences, so you built **suppression rules at the exit** as a safety
  net. That's an NFR (correctness/durability of decisions) shaping architecture. Questimate's NFR
  was **fast reads on expensive analytics**, which is why you **precomputed risk into snapshot
  tables** — latency NFR → architecture.
- **Estimation:** "~1M claims/day ≈ ~12/sec average, but bursty around payer cycles, so I size for
  peak." You can *speak* in QPS about your own system.
- **Data model → storage choice:** DCE keeps **claims in MongoDB** (flexible nested claim docs) and
  **reference data in Redis** (hot, low-latency lookups during scoring) — a textbook "right store
  for the access pattern" decision you can defend.
- **High-level + async:** DCE's **Kafka** stream decouples ingestion from scoring; Questimate's
  **Celery Beat** pipeline does heavy compute off the request path. That's "push slow work async."
- **Deep dive:** the **Redis-backed JWT** in Questimate (stateless verification *plus* instant
  server-side revocation) is a perfect "options + trade-off + pick" story.
- **Bottlenecks/migrations:** OpenShift→EKS, Azure→AWS, ENSO→K8s, Lambda/Glue→Snowpark — you've
  done real **evolution under cost/scale pressure**, with measured savings. That's step 7, lived.

> In an interview, when asked "have you done this?", you can say *"yes — on a 1M-claims/day pipeline
> I…"* That sentence alone separates you from most candidates.

---

## 6. Key concepts (interview-ready)
- **Run the 7 steps:** requirements → estimation → API → data model → high-level → deep-dive →
  bottlenecks. *Drive* it; don't wait to be driven.
- **Requirements first, always.** FRs (features) + NFRs (the -ilities) + constraints. The NFRs
  (scale, latency, availability, consistency, durability, read/write ratio) shape the architecture.
- **Quantify everything.** 1 day ≈ 10⁵ s; peak ≈ 2–10× avg; storage = items × bytes × retention ×
  replicas. State the *implication* of the number.
- **There is no best design**, only best-for-these-requirements. Every choice = benefit + cost.
- **Depth = data structure → algorithm → failure mode** on at least one component.
- **Proactively stress your own design** (SPOF, hot keys, partial failure, consistency window).
- **Handle pushback with curiosity**, iterate the diagram visibly.
- The senior signals: structure, trade-offs out loud, numbers, and graceful adjustment.

---

## 7. How to get better at this (deliberate practice)
1. **Time yourself.** Set a 45-min timer; force the step budget. You'll feel where you over-spend.
2. **Talk to the rubber duck.** Do a full design out loud, alone, recording yourself. Play it back —
   silence and hand-waving are obvious on replay.
3. **One deep-dive per problem.** After each practice design, pick one component and go 3 levels down.
4. **Build a story bank** (we formalize this in m35): 5–6 stories from Questimate/DCE mapped to
   framework steps, so any "have you…?" has a ready, quantified answer.
5. **Re-derive, don't memorize.** For every later module, practice *deriving* the design from
   requirements using this framework — that's the skill that transfers to an unseen question.

---

## Practice
➡️ Drill the follow-ups in [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md) (answer out loud first),
then do [`EXERCISES.md`](EXERCISES.md) before checking
[`solutions/m01_interview_framework/`](../../solutions/m01_interview_framework/README.md).

When you've done the exercises, say **"Module 2"** to build *Core Fundamentals* — the entire
vocabulary (scalability, latency, availability, CAP, consistency models, and the numbers).
