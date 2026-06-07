# m21 — Exercises (Payment System / Ledger)

> Do these on paper / out loud before checking `solutions/m21_payment_ledger/`. Correctness, the
> double-entry ledger, idempotency, and reconciliation are the core.

---

## Exercise 1 — Priorities
1. Why is a payment system CP, not AP? State the one-line principle.
2. The throughput is only ~115 tx/sec. Why isn't scale the main challenge?
3. Name the #1 non-functional requirement and two others that follow from it.

---

## Exercise 2 — Money math
1. Why is `0.1 + 0.2 == 0.3` false, and why is that catastrophic for money?
2. What two representations should you use instead?
3. What else must you store/handle besides the amount?

---

## Exercise 3 — The double-entry ledger
1. Record "transfer $50 from Alice to Bob" as ledger entries. What invariant must hold?
2. Where is an account's balance, if not in a "balance" column?
3. Give three reasons to use an immutable double-entry ledger over a mutable balance.

---

## Exercise 4 — Idempotency
1. A retry double-charges a customer. Design the fix.
2. Why is "exactly-once delivery" impossible, and what do you achieve instead?
3. Why must the idempotency key flow end-to-end (not just at the API)?

---

## Exercise 5 — The external gateway
1. You charge Stripe AND record in your ledger. Describe the failure that loses consistency.
2. Why can't you just wrap both in one transaction?
3. List the four layered defenses, including the safety net.

---

## Exercise 6 — Reconciliation
1. What is reconciliation, and what does it compare?
2. Why is it essential rather than optional?
3. Which tool/pattern from your DCE work is the same idea?

---

## Exercise 7 — Concurrency
1. Two withdrawals of $80 hit a $100 balance simultaneously. What's the danger, and the fix?
2. Which named problem (from m04) is this?
3. What consistency level does the balance-check path need, vs the rest of the system?

---

## Exercise 8 — Multi-step payments
1. A payment debits an account, credits a merchant, and calls a gateway. 2PC or Saga? Why?
2. A later step fails — what undoes the earlier ones, and how does the ledger express it?
3. How do you process a refund without violating immutability?

---

## Exercise 9 — Security & extras
1. How do you handle card data to minimize PCI-DSS scope?
2. Where does fraud detection fit, and what kind of system is it?
3. Which metric/alert is the scariest for a payment system, and why?

---

## Exercise 10 — Your-systems tie-in 🏭 + course finale
1. You integrated brokers for live buy/sell on Questimate. Which exact payment-correctness problem did you
   face (hint: you named it in m03)?
2. Map DCE **`compare_results.py`** and the **suppression layer + audit trail** to two concepts in this
   module.
3. Name five building-block modules (m02–m10) whose ideas appear in this payment design.

---

When done, check `solutions/m21_payment_ledger/README.md`, then say **"Module 22"** (Part 3 — ML System
Design begins).
