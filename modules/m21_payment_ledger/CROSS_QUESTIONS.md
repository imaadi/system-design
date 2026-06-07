# m21 — Cross-Questions ("if-and-buts") on the Payment System / Ledger

> Answer out loud in 2–3 sentences before reading the model answer. Correctness, the double-entry ledger,
> idempotency, and reconciliation are where this is won.

---

### Q1. How is designing a payment system different from designing a social feed?
**A.** Priorities **invert**. A feed is **AP** — optimize availability + scale, tolerate eventual
consistency (a stale like is fine). A payment system is **CP** — **correctness above all**: never
double-charge, lose, or create money, even at the cost of availability. **"Available but wrong" is worse
than "briefly down."** It's also low-QPS but high-stakes, so you can afford expensive, careful,
strongly-consistent per-transaction processing — the opposite optimization from everything before it.

---

### Q2. Why must a payment system never use floating-point for money?
**A.** Floats are **imprecise** — `0.1 + 0.2 = 0.30000000000000004` — so rounding errors accumulate over
millions of transactions and **lose or create money** (and fail audits). Always represent money as
**integers in the minor unit** (cents/paise) or **arbitrary-precision decimal** (`Decimal`, SQL
`NUMERIC`), store the **currency** explicitly, and apply **deliberate rounding rules**. It's a classic
gotcha and a real production bug source — "use integers/decimal, never floats" is the expected answer.

---

### Q3. What is double-entry bookkeeping and why use it?
**A.** Every transaction is recorded as **balanced debits and credits** — money out of one account equals
money into another, so each entry **sums to zero** and **money is conserved** (you can't create/destroy
it, only move it). You **never update a mutable balance in place**; you **append immutable entries**, and
an account's **balance is *derived*** (the sum of its entries) — event sourcing for money. It gives
**auditability** (complete tamper-evident history), **correctness** (debits==credits is a built-in
invariant check), and **no lost-update** on a hot balance field.

---

### Q4. Why store an immutable ledger instead of just a balance column you update?
**A.** A mutable balance **hides history** (you can't see how it got there → hard to audit/debug), and
updating it is a **read-modify-write** on a hot field → the **lost-update** race under concurrency (m04).
An **immutable append-only ledger** keeps a **complete auditable history**, makes the balance a
**verifiable derivation** (sum of entries), and turns concurrent activity into **appends** (no in-place
contention on a balance). Compliance and correctness both demand the immutable record — every serious
fintech runs a double-entry ledger.

---

### Q5. How do you prevent a customer being double-charged on a network retry?
**A.** **Idempotency keys.** The client generates a unique key per payment intent; the server records the
key + result the **first** time and, on **any retry with the same key**, returns the **original result**
instead of charging again. So **at-least-once delivery + an idempotent charge = exactly-once *effect***
(m08) — the charge happens once even if the request arrives five times. This is the **Stripe model**, and
it must be applied **end-to-end** (including the call to the external gateway).

---

### Q6. Why can't you guarantee "exactly-once" and what do you do instead?
**A.** Exactly-once **delivery** is impossible over an unreliable network (m09 — a lost ack means you can't
tell if it succeeded). So you don't try; you make the **operation idempotent** so duplicates are harmless,
achieving exactly-once **effect**. With money, that's an **idempotency key** + a dedup record, so a
retried/duplicated charge returns the original result rather than charging again. "You can't get
exactly-once delivery, so you make the effect idempotent" is the senior framing.

---

### Q7. You charge an external gateway (Stripe) AND record it in your ledger. What can go wrong?
**A.** It's a **dual-write** across two systems you can't wrap in one transaction (m04/m08): if the
**gateway succeeds but you crash before recording it**, money moved with **no ledger record** (or you
record a charge that actually failed). Naively "call gateway then write ledger" is **not atomic**. This is
the scariest version of the dual-write problem because it involves real money and an external,
asynchronous, unreliable system.

---

### Q8. So how do you keep your ledger and the external gateway consistent?
**A.** Layered defenses: **(1)** **idempotency keys end-to-end** (send one to the gateway too, so retries
don't double-charge there); **(2)** **record a PENDING intent first** — write the transaction as PENDING
*before* calling the gateway, then finalize to SUCCESS/FAILED, so a crash leaves a resolvable PENDING, not
a silent gap; **(3)** **webhooks** — process the gateway's async callback idempotently to finalize; and
**(4)** **reconciliation** — continuously compare your ledger against the gateway's records to catch and
fix any discrepancy. You assume a gap *will* happen and verify against the source of truth.

---

### Q9. What is reconciliation and why is it essential?
**A.** **Reconciliation** continuously **compares your ledger against the processor's/bank's records** to
detect discrepancies — a charge that succeeded externally but wasn't recorded, a stuck PENDING, a mismatch
— and then auto-corrects or flags for manual review. It's essential because, in a distributed payment flow,
**something *will* eventually fall through** (crashes, lost webhooks, partial failures), and reconciliation
is the **safety net** that catches it. It's the financial-grade "**trust but verify**" — defense in depth.
(Your DCE `compare_results.py` parity check is the same idea.)

---

### Q10. How do you prevent double-spend (two concurrent withdrawals draining one balance)?
**A.** Make the **debit atomically check funds** — this is the m04 **lost-update** problem with money. Use
an **atomic conditional operation** (`UPDATE … SET balance = balance - amt WHERE balance >= amt`, or an
append-only ledger entry guarded by a balance constraint), a **`SELECT FOR UPDATE`** row lock, or
**serializable** isolation on the critical path — so two concurrent withdrawals can't *both* pass the
balance check and overdraw. The balance check must be **strongly consistent and serialized**, unlike the
AP data elsewhere.

---

### Q11. Multi-step payment across services — 2PC or Saga?
**A.** **Saga**, not 2PC. **2PC**'s coordinator is a **blocking SPOF holding locks** (m09) — unacceptable
for a payment flow. A **Saga** runs the steps as **local transactions** (debit account, credit merchant,
call gateway) with **compensating transactions** to undo earlier steps if a later one fails (e.g. **refund/
reverse**), each step **idempotent**. The **ledger's reversal entries** are the natural compensation — you
**append a reversing entry**, never edit, keeping the audit trail intact. So: Saga + idempotent steps +
compensating reversal entries.

---

### Q12. How do you handle a refund?
**A.** A refund is a **new, reversing transaction** — you **append a compensating ledger entry** (credit
the customer back, debit the merchant) rather than deleting or editing the original charge. The original
stays in the immutable record; the refund is its own balanced entry referencing it. This preserves the
**complete audit trail** (you can see the charge *and* its refund), keeps double-entry balanced, and is the
natural Saga **compensation**. Never mutate history — always append the correction.

---

### Q13. What happens during a network partition — do you stay available?
**A.** No — payments are **CP** (m02), so under a partition you **refuse or hold ambiguous operations
rather than risk inconsistency** (a double-charge or lost money). You'd rather a payment **fail/retry
later** than process it wrongly. Concretely: the balance check / debit requires the strongly-consistent
path, so if it can't reach a quorum, it errors. Reconciliation later cleans up any PENDING. This is the
deliberate opposite of the AP feeds/chat you designed — "down beats wrong" for money.

---

### Q14. How does the idempotency key flow through the whole system?
**A.** **End-to-end.** The client sends it on the request; the **payment service** dedups on it (returns the
stored result for a repeat); it's passed to the **external gateway** (so the gateway also won't double-
charge on retry); and the **ledger write** is keyed to it (so a reprocessed message doesn't create a
duplicate entry, m08). Every component that has a side effect must be idempotent on that key — otherwise a
retry double-acts somewhere in the chain. Idempotency is a **property of the whole pipeline**, not one
endpoint.

---

### Q15. What's the role of a state machine in a payment?
**A.** A payment moves through explicit states — e.g. **CREATED → PENDING → SUCCESS / FAILED → (REFUNDED)**
— persisted **before** each side-effecting step. Recording **PENDING before calling the gateway** means a
crash leaves a **resolvable** record (reconciliation/retry can finish it), not a silent gap; only a
confirmed gateway result (sync or via webhook) advances it to SUCCESS/FAILED. The state machine makes the
flow **recoverable and idempotent** (you can safely re-drive a PENDING) and gives a clear audit of where
each payment is.

---

### Q16. How do you store card data securely (PCI-DSS)?
**A.** You **don't store raw card data** — you **tokenize** via the processor (Stripe/Braintree returns a
token representing the card; you store the **token**, not the PAN/CVV). This keeps you **out of PCI scope**
for handling raw card numbers (a huge compliance + liability reduction). Plus encryption in transit/at
rest, strict access controls, and an immutable audit trail. The principle: **never hold what you don't need
to** — let the certified processor handle the sensitive data and give you a safe token.

---

### Q17. How does fraud detection fit into a payment system?
**A.** As an **ML system layered on top** (m27): before/around processing, a model scores each transaction
for fraud risk (from features like amount, location, velocity, device, history) and the system **approves,
declines, or flags for review** in real time. It's a **low-latency online ML scoring** problem with **class
imbalance** and **delayed labels** (chargebacks arrive later) — exactly the ML-system-design territory (and
your DCE rules+ML hybrid experience). The payment ledger handles correctness; fraud ML handles *should this
payment happen at all*.

---

### Q18. What metrics and alerts matter most for a payment system?
**A.** **Correctness-centric:** **reconciliation discrepancies** (alert immediately — money may be at risk),
**stuck PENDING count** (payments that didn't finalize), **ledger imbalance** (debits ≠ credits → a bug),
**success/decline/error rates**, and **gateway latency/errors**. Unlike a feed (where you watch p99/QPS),
here the scariest signal is a **discrepancy** — any sign the ledger and reality disagree. You also alert on
duplicate-charge rate (idempotency failures) and refund anomalies (m10).

---

### Q19. How do you handle a high-volume merchant account (a hot balance)?
**A.** The **append-only double-entry ledger** already helps — you **append entries** instead of
contending on an in-place balance field, so concurrent activity doesn't serialize on one row. For extreme
volume you **shard by account** (m05) and may compute the balance from periodic **snapshots + recent
entries** (so you don't sum a huge history each read — a materialized balance you can rebuild from the
ledger). The immutable ledger stays the source of truth; the balance is a fast, rebuildable projection.

---

### Q20. Tie it together — how do all the building blocks show up in this design?
**A.** **CAP/CP** (m02 — choose consistency); **ACID + lost-update/atomicity** (m04 — balance checks);
**event sourcing / idempotency / outbox / exactly-once** (m08 — the immutable ledger + idempotency keys);
**Saga / 2PC** (m09 — multi-step payments + compensation); **sharding hot accounts** (m05); **retries/
circuit breakers** (m10 — flaky gateway); and **fraud ML** (m27). The payment system is the **correctness-
maximal application** of the whole course — which is why it closes Part 2 and why your DCE/Questimate
correctness-and-finance background makes it your strongest design to *walk in and own*.
