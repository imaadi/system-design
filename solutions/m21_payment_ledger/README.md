# m21 — Solutions (Payment System / Ledger)

> Check the *reasoning*. Recurring lessons: correctness > availability (CP); immutable double-entry ledger
> with derived balances; idempotency = exactly-once effect; reconciliation as the safety net.

---

## Exercise 1 — Priorities
1. **CP** because **correctness must never be sacrificed** — a double-charge/lost-money error is worse than
   downtime. **Principle: "available but wrong is worse than briefly down."**
2. Because the **stakes per transaction**, not the volume, dominate — ~115/sec is trivial throughput, so
   you can afford **expensive, careful, strongly-consistent** processing per transaction. The hard part is
   never being wrong.
3. **#1: correctness / exactly-once effect** (never double-charge/lose/create money). Follows: **strong
   consistency + durability** (correct balances, survive crashes) and **auditability** (immutable verifiable
   history for compliance).

---

## Exercise 2 — Money math
1. **Floats are binary approximations** of decimal fractions, so `0.1 + 0.2 = 0.30000000000000004` — over
   millions of transactions these rounding errors **lose or create money** and break audits.
2. **Integers in the minor unit** (cents/paise) or **arbitrary-precision decimal** (`Decimal` / SQL
   `NUMERIC`).
3. The **currency** (explicitly, per amount), and **deliberate rounding rules** (how/when to round, for FX
   and fees).

---

## Exercise 3 — The double-entry ledger
1. Entries: **DEBIT Alice −50** and **CREDIT Bob +50** (one balanced transaction). **Invariant: debits ==
   credits** (the entry sums to zero → money conserved).
2. The balance is **derived = the sum of an account's ledger entries** (not stored as a mutable column) —
   event sourcing for money.
3. **(a) Auditability** (complete tamper-evident history you can replay/verify); **(b) correctness** (the
   debits==credits invariant catches any money-conservation bug); **(c) no lost-update** (you append
   immutable entries instead of read-modify-writing a hot balance field).

---

## Exercise 4 — Idempotency
1. **Idempotency key:** client sends a unique key per payment; server stores key→result on first use and,
   on any retry with the same key, **returns the original result without charging again**.
2. **Impossible** because over an unreliable network a lost ack means you can't tell if it succeeded (m09);
   instead you make the **operation idempotent** → **exactly-once *effect*** (charged once despite
   duplicates).
3. Because **every side-effecting component** (the gateway call, the ledger write, notifications) could
   double-act on a retry — so the key must dedup at each, or a retry double-charges/double-records
   somewhere in the chain. Idempotency is a **whole-pipeline** property.

---

## Exercise 5 — The external gateway
1. You **charge Stripe successfully but crash before recording it** in your ledger → **money moved with no
   record** (or you record a charge that actually failed). The two writes aren't atomic.
2. Because the gateway is a **separate, external system** — you can't enroll it in your database transaction
   (no shared transaction coordinator); it's the dual-write problem at its worst.
3. **(1)** idempotency keys **end-to-end** (incl. to the gateway); **(2)** record a **PENDING intent before**
   the call → finalize after (crash leaves a resolvable record); **(3)** **webhooks** to finalize async,
   processed idempotently; **(4)** **reconciliation** — continuously compare your ledger to the gateway's
   records (the safety net).

---

## Exercise 6 — Reconciliation
1. A continuous process that **compares your ledger against the processor's/bank's records** to find
   discrepancies (unrecorded successful charges, stuck PENDINGs, mismatches) and correct/flag them.
2. Because in a distributed payment flow **something will eventually slip through** (crashes, lost webhooks,
   partial failures) — reconciliation is the **safety net** that detects and fixes it; you design assuming a
   gap will happen and **verify against the source of truth**.
3. **DCE `compare_results.py`** — comparing old vs new environment output for **parity** during migrations
   is the same instinct: assume divergence, compare against a reference, flag discrepancies.

---

## Exercise 7 — Concurrency
1. **Danger:** both read $100, both pass the "≥ $80?" check, both withdraw → balance goes to **−$60**
   (double-spend). **Fix:** make the debit an **atomic conditional op** (`UPDATE … WHERE balance >= 80`),
   `SELECT FOR UPDATE`, or serializable isolation so only one succeeds.
2. The **lost-update** problem (m04) — concurrent read-modify-write on the balance.
3. The **balance-check/debit path needs strong consistency** (atomic, serialized — no double-spend); the
   rest (notifications, reporting) can be looser, but the money-moving critical path is the strict part.

---

## Exercise 8 — Multi-step payments
1. **Saga** — 2PC's coordinator is a **blocking SPOF holding locks** (m09), unacceptable for a payment.
   Saga runs each step as a **local transaction** with **compensations**, each idempotent.
2. **Compensating transactions** undo earlier steps (e.g. reverse the debit); the **ledger expresses this
   as a new reversing entry** (append a balanced credit-back), never an edit.
3. A refund is a **new, reversing transaction** — append a compensating entry (credit the customer, debit
   the merchant) that references the original; the original charge **stays immutable** in the audit trail.

---

## Exercise 9 — Security & extras
1. **Tokenize** card data via the processor (store the **token**, never the raw PAN/CVV) → keeps you out of
   PCI scope for raw card handling; plus encryption + access controls + audit trail. "Never hold what you
   don't need to."
2. **Fraud detection = a real-time online ML system layered on top** (m27): score each transaction for
   risk and approve/decline/flag — low-latency, class-imbalanced, delayed labels (chargebacks).
3. **Reconciliation discrepancies** (and ledger imbalance / stuck PENDINGs) — scariest because they mean
   **the ledger and reality disagree → money may be at risk**; you alert immediately (vs a feed where you'd
   watch p99/QPS).

---

## Exercise 10 — Your-systems tie-in 🏭 + finale
1. **Placing the same buy order twice on a network retry** (you named it in m03) — the **idempotency**
   problem with real money/orders, exactly Step 4 of this module. You faced it integrating broker buy/sell.
2. **`compare_results.py` = reconciliation** (assume divergence → compare against a source of truth → flag
   discrepancies); **suppression layer + Mongo audit trail = correctness-first + immutable record** (block
   wrong outputs, keep a complete auditable history) — the payment-ledger philosophy in your domain.
3. **m02** (CAP — choose C/CP), **m04** (ACID/lost-update/atomic balance), **m05** (shard hot accounts),
   **m08** (idempotency/exactly-once/event-sourcing-ledger/outbox), **m09** (Saga vs 2PC + compensation),
   **m10** (retries/circuit breakers for the flaky gateway). (Also m27 fraud ML on top.) The payment system
   is the **correctness-maximal application of the whole course** — which is why it closes Part 2, and why
   your finance + correctness-pipeline background makes it a design you can **walk in and own**.
