# Module 21 — Design a Payment System / Ledger ⭐⭐ 🏭 — *closes Part 2*

> Money is different. Every other system optimized for **availability** and tolerated eventual
> consistency; a payment system optimizes for **correctness** above all — **never double-charge, never
> lose money, always reconcile.** It's the rare **CP** design (m02), built on three pillars:
> **idempotency** (the m03/m08 pattern, now with real money), the **double-entry ledger** (immutable,
> balanced, auditable — money is conserved), and **reconciliation** (assume something breaks; verify
> against the source of truth). This is **your strongest domain** — Questimate's real trading/orders and
> DCE's correctness-obsessed pipeline run through every section.

> **Format (Part 2 finale):** worked walkthrough; the **double-entry ledger**, **idempotency/exactly-
> once**, and **external-gateway + reconciliation** are the deep-dives.

---

## Step 1 — Requirements

**Functional (core):**
1. **Process a payment** (charge a customer) / **transfer** money between accounts.
2. **Record every transaction** in an auditable ledger.
3. **Refunds / reversals.**
4. Integrate with **external processors/banks** (Stripe, card networks, ACH).

**Non-functional (these *dominate* — correctness first):**
- **Correctness / exactly-once effect:** **never double-charge, never lose money, never create money.**
  *This is the #1 NFR — above availability.*
- **Strong consistency + durability:** balances must be correct (no double-spend); committed transactions
  survive crashes (ACID, m04).
- **Auditability:** a complete, immutable, verifiable history (compliance: **PCI-DSS**, financial regs).
- **Availability:** important, but **subordinate to correctness** — being **down** is better than being
  **wrong**.

> **The defining insight (CAP, the rare C choice, m02):** payments are **CP** — under a partition you
> **refuse rather than risk inconsistency**. *"A payment system that's available but wrong is worse than
> one that's briefly down."* This single reframe drives every decision (vs the AP feeds/chat you designed).

---

## Step 2 — Estimation
Payments are **lower-volume but higher-stakes** than social systems. Assume **10M transactions/day**.
- **Throughput:** 10M ÷ 10⁵ = **~115 tx/sec** avg (~1,000 peak). *Low QPS* — the challenge is **not scale,
  it's correctness** (you can afford expensive, careful, strongly-consistent processing per transaction).
- **Storage:** every transaction is an **immutable ledger entry kept forever** (audit) → steady growth, on
  durable, replicated storage. Small per entry; the value is in never losing one.

> **The estimate frames it:** unlike every prior module, **the bottleneck is correctness, not QPS.** You
> trade throughput/availability for **consistency + auditability** — the opposite optimization.

---

## Step 3 — Two pillars of money math (get these right or fail) ⭐

### Never use floating point for money ⭐
**Floats are imprecise** (`0.1 + 0.2 = 0.30000000000000004`) → rounding errors that **lose or create
money** over millions of transactions. **Always** represent money as **integers in the minor unit**
(cents/paise) or **arbitrary-precision decimal** (Python `Decimal`, SQL `NUMERIC`). Store the **currency**
explicitly; handle rounding rules deliberately. *(A classic interview gotcha — and a real bug source.)*

### Double-entry bookkeeping = the ledger ⭐⭐ (the heart)
A payment system's **source of truth is an immutable, append-only ledger of balanced entries.** The
500-year-old accounting principle:
- **Every transaction is recorded as balanced debits and credits** — money moved **out of** one account
  is moved **into** another, and **debits == credits** (the entry sums to zero). Money is **conserved** —
  you can't create or destroy it, only move it.
- You **NEVER update a mutable "balance" field in place.** Instead you **append immutable entries**; an
  account's **balance is *derived*** (the sum of its entries). This is **event sourcing for money** (m08):
  the ledger is the immutable log; balances are projections.
```
  Transfer $50 from Alice → Bob (atomic, balanced):
    Entry:  DEBIT  Alice  -50      ┐ sum = 0  (money conserved)
            CREDIT Bob    +50      ┘  append-only, immutable
    Alice's balance = SUM(Alice entries) ; Bob's = SUM(Bob entries)
```

> **Why immutable + double-entry?** (1) **Auditability** — a complete, tamper-evident history you can
> replay/verify (vs a mutable balance that hides how it got there); (2) **correctness** — the
> debits==credits invariant is a built-in check that money is conserved (any imbalance = a bug, caught);
> (3) **no lost-update on the balance** — you append, you don't read-modify-write a hot balance field
> (m04). *Every serious fintech (Stripe, banks) runs a double-entry ledger.* Saying "the ledger is an
> immutable, append-only, double-entry log; balances are derived" is the senior signal here.

---

## Step 4 — Deep Dive: idempotency & exactly-once effect ⭐⭐ (don't double-charge)

The scariest failure: a **network retry double-charges** a customer. (You hit this for real on Questimate
— a retried buy order must not execute twice.) The fix is the m03/m08 pattern, now with money:
- The client generates a unique **idempotency key** per payment intent and sends it with the request.
- The server records the key + result the **first** time; on **any retry with the same key**, it returns
  the **original result** instead of charging again.
- So **at-least-once delivery + an idempotent operation = exactly-once *effect*** (m08). Exactly-once
  *delivery* is impossible (m09), so you make the **operation** idempotent — the charge happens once even
  if the request arrives five times.

```
  POST /charge  Idempotency-Key: abc123  $50
   first time  → charge once, store (abc123 → result), return result
   retry/dup   → key seen → return SAME result, DO NOT charge again
```

> **This is the Stripe model** (their idempotency-key API is canonical). It's also why the whole pipeline
> must be idempotent end-to-end: the gateway call, the ledger write, the notification. **Idempotency is
> the bedrock of payment correctness.**

---

## Step 5 — Deep Dive: external gateways, the dual-write hazard & reconciliation ⭐⭐

A real charge involves an **external processor** (Stripe / the card network / a bank) — an **unreliable,
async, third-party** system. This creates the **dual-write problem at its scariest** (m04/m08):
- You **charge the external gateway** AND **record it in your ledger** — but they're **two systems**. If
  the **gateway succeeds but you crash before recording it** → **money moved with no record** (or you
  recorded a charge that actually failed). You **cannot** wrap an external system in your DB transaction.

**Defenses (layered):**
1. **Idempotency end-to-end** (Step 4) — send an idempotency key to the gateway too, so your retry doesn't
   double-charge there.
2. **Record intent first (a state machine):** write a `PENDING` ledger/transaction record **before**
   calling the gateway; update to `SUCCESS`/`FAILED` after. So a crash leaves a **PENDING** record you can
   resolve, not a silent gap. (An **outbox**-style durable intent, m08.)
3. **Webhooks + async confirmation:** the gateway calls **back** (webhook) with the final result; you
   process it idempotently to finalize the transaction.
4. **⭐ Reconciliation (the safety net):** **continuously compare your ledger against the processor/bank's
   records** to catch any discrepancy — a charge that succeeded externally but wasn't recorded, a mismatch,
   a stuck PENDING. **You *assume* something will go wrong and verify against the source of truth**, then
   auto-correct or flag for manual review. *This is defense-in-depth — the financial-grade "trust but
   verify."* (DCE's `compare_results.py` parity validation across environments is the same instinct.)

> **The senior line:** *"Charging an external gateway plus recording in my ledger is a dual-write I can't
> make atomic, so I use **idempotency keys end-to-end**, **record a PENDING intent before the call** and
> finalize via **webhook**, and run **reconciliation** that compares my ledger to the processor's records
> to catch and resolve any gap. In payments you design assuming a failure *will* slip through, and
> reconciliation is the net that catches it."*

---

## Step 6 — Consistency, concurrency & multi-step payments

- **Strong consistency for balances (no double-spend):** the **debit must atomically check funds** — the
  m04 lost-update problem with money. Use an **atomic conditional write** (`UPDATE … WHERE balance >=
  amount` / append-only ledger entry guarded by a constraint), **`SELECT FOR UPDATE`**, or **serializable**
  isolation on the critical path. Two concurrent withdrawals must **not** both succeed past the balance.
- **Multi-step payments → Saga, not 2PC (m09):** a payment spanning services (debit account, credit
  merchant, call gateway) is a **distributed transaction**. Avoid **2PC** (blocking coordinator holding
  locks). Use a **Saga**: a sequence of local transactions with **compensating transactions** (if a later
  step fails, **refund/reverse** the earlier ones), each step **idempotent**. The ledger's
  **reversal entries** (you never delete — you append a compensating entry) are the natural compensation.
- **Refunds/reversals are new entries**, never edits — you append a **reversing transaction** (credit back)
  so the audit trail stays complete and immutable.

---

## Step 7 — Bottlenecks, security & edge cases
- **Correctness > availability (CP):** under partition, **refuse** ambiguous operations rather than risk a
  double-charge; degrade carefully.
- **External gateway reliability:** retries (idempotent) + circuit breakers + fallback processors (m10);
  the gateway is the flaky dependency.
- **Reconciliation pipeline:** the always-on safety net (Step 5) — itself a system (batch/stream compare).
- **Precision/rounding/currency:** integers/decimal, explicit currency, deliberate rounding (Step 3).
- **Security/compliance:** **PCI-DSS** (don't store raw card data — tokenize via the processor),
  encryption, fraud detection (an ML system — Part 3 / m27), immutable **audit trail**, access controls.
- **Hot accounts:** a high-volume merchant account is a hot key on the balance — the append-only ledger
  (no in-place balance update) plus per-account sharding helps (m05).
- **Observability (m10):** transaction success/failure rate, PENDING/stuck count, reconciliation
  discrepancies (alert immediately — money at risk), gateway latency/errors, ledger imbalance checks.

**Wrap-up:** *"A **correctness-first (CP)** system whose source of truth is an **immutable, append-only,
double-entry ledger** — every transaction balanced debits/credits, balances **derived** (event-sourced),
money in **integers/decimal** never floats. **Idempotency keys end-to-end** give **exactly-once effect**
(at-least-once + idempotent op) so retries never double-charge. The **external gateway** is a dual-write
hazard, handled by **record-intent-first (PENDING state machine) + webhooks + end-to-end idempotency**,
with **reconciliation** continuously verifying my ledger against the processor as the safety net. Balance
checks are **strongly consistent** (atomic, no double-spend); multi-step payments use **Sagas with
compensating (reversal) entries**, not 2PC. The whole philosophy: **assume a failure will slip through and
verify against the source of truth.** Trade-offs: correctness/consistency over availability (the rare CP
choice), and operational overhead (reconciliation, audit) as the price of never losing money."*

---

## State-of-the-art & real-world notes 📚
- **Stripe** — the canonical **idempotency-key** API + a double-entry ledger; their engineering blog on
  idempotency/reliability is required reading.
- **Double-entry bookkeeping** (500 years old) is *the* ledger model across all of fintech/banking; modern
  "ledger" databases (**TigerBeetle**, formalized double-entry) productize it.
- **Reconciliation** is standard financial practice — every payment company runs continuous reconciliation
  against processors/banks.
- **Sagas / compensating transactions** (m09) are how multi-step payments stay consistent without 2PC.

---

## From your systems 🏭 (your strongest domain)
- **Questimate = a real financial/trading system:** you integrated **brokers (Zerodha/ICICI/Dhan)** for
  **live buy/sell orders** — and you've **named the exact hazard** (m03): *placing the same buy order twice
  on a network retry.* That's the Step-4 **idempotency** problem with real money — you faced it.
- **Double-entry / accounting math:** Questimate's **cashflow/amortization, loan valuations, DCF** are
  financial-domain accounting — you already think in debits/credits/balanced flows.
- **Reconciliation = your parity validation:** DCE's **`compare_results.py`** (compare old vs new
  environment output for parity during migrations) is **exactly reconciliation** — assume something
  diverged, compare against a source of truth, flag discrepancies. Same instinct, different domain.
- **Correctness-first + audit trail:** DCE's **suppression layer** (block wrong outputs) + **Mongo audit
  trail** of every decision is the same "correctness above throughput, with an immutable record" philosophy
  as a payment ledger. You've built correctness-critical pipelines; this is that, with money.
- **Idempotent processing + atomic ops:** your DCE idempotent reprocessing and m04 atomic-update reflex are
  the payment-correctness toolkit.

---

## Key concepts (interview-ready)
- **Payments are CP (correctness > availability):** never double-charge/lose/create money; **down beats
  wrong** (the rare CAP "choose C", m02). Low QPS, high stakes — optimize for correctness, not scale.
- **Money math:** **never floats** → integers (minor units) or **Decimal**; explicit currency + rounding.
- **Double-entry ledger (the heart):** **immutable, append-only, balanced** entries (debits == credits →
  money conserved); **balances derived** (event sourcing, m08); auditable; no in-place balance update (no
  lost-update). Refunds/reversals = **new compensating entries**, never edits.
- **Idempotency = exactly-once effect:** **idempotency key end-to-end** → at-least-once + idempotent op =
  charged once despite retries (the Stripe model, m03/m08).
- **External gateway = dual-write hazard:** **record PENDING intent first → call gateway (idempotent) →
  finalize via webhook**, and **reconcile** continuously against the processor (the safety net — assume a
  gap will happen and verify).
- **Consistency/concurrency:** **strongly-consistent atomic balance checks** (no double-spend, m04);
  **Saga + compensating (reversal) entries**, not 2PC (m09).
- **Security/compliance:** PCI-DSS (tokenize, don't store cards), immutable audit trail, fraud detection
  (ML, m27).

---

## Go deeper (reading)
- **Stripe Engineering: "Designing robust and predictable APIs with idempotency" + their ledger/reliability
  posts** — the canonical idempotency + double-entry references.
- **"Accounting for Computer Scientists" (Martin Kleppmann blog)** — double-entry bookkeeping for engineers.
- **TigerBeetle docs** — a modern purpose-built double-entry ledger database (why double-entry, debits=
  credits).
- **microservices.io: Saga pattern** — multi-step payment consistency without 2PC.
- Revisit **m02** (CAP — the CP choice), **m04** (ACID/lost-update/atomicity), **m08** (idempotency/
  exactly-once/outbox), **m09** (Saga/2PC), **m27** (fraud detection — ML on top).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m21_payment_ledger/`](../../solutions/m21_payment_ledger/README.md).

🎉 **This completes PART 2 — Classic Design Problems (m11–m21).** You've now designed eleven canonical
systems end-to-end using the m01 framework + all ten building blocks. Say **"Module 22"** to begin
**PART 3 — ML System Design** with the *ML System Design Framework* — where your **AI/ML background becomes
the differentiator** (the InsightDesk/DCE experience you already have).
