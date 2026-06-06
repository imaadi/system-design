# m15 — Solutions (Notification System)

> Check the *reasoning*. Recurring lessons: it's a reliable event-driven pipeline; dedup = effectively-
> once; providers are flaky external deps (m10); preferences are a hard legal gate; priority + two
> rate-limits.

---

## Exercise 1 — Estimation
200M users, 6/day, push 60/email 30/SMS 10.
1. **Total:** 200M × 6 = 1.2B/day ÷ 10⁵ = **~12,000/sec** avg (~60k peak).
2. **Per channel/day:** push **720M**, email **360M**, SMS **120M** (SMS is the most rate-limited/
   expensive → pace it hardest).
3. **80M-user campaign** needs a **separate bulk path** because firing 80M sends at once would blow
   provider quotas and your own infra and could starve urgent traffic. You **expand the segment into
   per-user jobs in batches**, **spread them over time** (rate-paced), and run it **off the transactional
   priority lane**.

---

## Exercise 2 — The pipeline
1. **producer → Notification Service** (dedup → preferences → template → rate-limit) **→ message queue
   (priority lanes) → per-channel workers** (push/SMS/email/in-app) **→ providers (APNs/FCM/Twilio/SES)**,
   with **retries/DLQ/circuit-breakers** and **delivery tracking** writing back status. Side stores:
   preferences, templates, dedup cache (Redis), delivery log.
2. The queue gives: **decoupling** (producer doesn't block on slow/flaky provider calls), **burst
   absorption** (campaigns queue instead of overwhelming), and **independent scaling + retries +
   priority lanes** for delivery.
3. So producers stay simple and uniform ("notify user X about type Y") and the **provider/channel
   complexity** (auth, quotas, formats, failures) lives in one place — change providers without touching
   every producing service.

---

## Exercise 3 — Deduplication
1. Use an **idempotency/dedup key**: the first event is processed and its key recorded; the other 499
   (same key) are detected and **skipped** → one notification.
2. **Key** = e.g. `hash(user_id, type, event_id, channel)`, stored in **Redis** with a TTL; **check-and-
   set atomically** before sending.
3. Delivery is **at-least-once** (retries, queue redelivery, buggy producers can repeat), so you make the
   **send idempotent** via the dedup key → **effectively-once**. You've seen it in m08 (idempotent
   consumers), m14 (message dedup), and m03 (idempotency keys) — and it's DCE **suppression** in spirit.

---

## Exercise 4 — Provider reliability
1. **(a)** retry transient failures with **backoff + jitter**; **(b)** trip a **circuit breaker** on APNs
   so you stop hammering it (fail fast); **(c)** **fall back** to another channel for urgent ones; **(d)**
   **DLQ** repeated failures + alert. The **dedup key** keeps retries safe.
2. **Don't retry** — it's a **permanent failure**. Add the number to a **suppression list** and stop
   sending to it (and surface it for cleanup).
3. **Fall back** to a secondary SMS provider (or another channel) on the **priority lane** with fast
   retries — OTPs justify multi-provider fallback because they must arrive quickly.

---

## Exercise 5 — Preferences & compliance
1. **Before sending** — it's a **hard gate** because sending after an opt-out is a **legal/financial
   risk** (and drives uninstalls). Preferences are cached (Redis) for a fast check on the hot path.
2. **SMS → TCPA** (requires prior consent); **email → CAN-SPAM** (must honor unsubscribe);
   **GDPR** (consent + right to opt out) across the EU.
3. **Allowed** — security alerts are **transactional/critical**, which may **bypass *promotional*
   opt-outs**. Model preferences **per type** (not just per channel): the user muted "marketing" but
   "security" is always-allow; design the always-allow set carefully and lawfully.

---

## Exercise 6 — Priority
1. So **urgent** notifications aren't stuck behind **bulk** — separate queues/workers let OTPs flow with
   low latency while a campaign drains slowly.
2. **Transactional:** OTP, password reset, order shipped, security alert. **Bulk:** marketing campaign,
   weekly digest, product announcement, re-engagement push.
3. With one shared queue, the OTP would **queue behind millions of campaign messages** → arrive minutes
   late (or after expiry) → users can't log in. Priority lanes prevent this.

---

## Exercise 7 — Rate limiting
1. **Per-user (fatigue)** — protects the user's experience (cap frequency / digest). **Per-provider** —
   respects APNs/Twilio/SES quotas so you aren't throttled/blocked.
2. **Batching/digesting** combines many events into one notification ("12 new likes"), directly reducing
   per-user frequency (and volume to providers).
3. Workers **pace their sends** to stay under provider quotas — a **token bucket** (m12) per provider, so
   they smooth output to the allowed rate instead of bursting and getting throttled.

---

## Exercise 8 — Scheduling & digests
1. **"Send in 2 hours"** → a **delay queue** (SQS delay / a Redis sorted-set keyed by send-time that a
   worker polls). **"Daily digest at 9am"** → a **scheduler (Celery Beat / cron)** fires per interval and
   the digest job aggregates events since the last send. Both then enter the normal pipeline.
2. A **digest batches many events into one notification**, cutting the number of interruptions (12 pushes
   → 1) → less fatigue, fewer opt-outs.
3. **Celery + Celery Beat** (Questimate) for scheduled/recurring; a Redis sorted set for fine-grained
   delays.

---

## Exercise 9 — Delivery tracking
1. Via **provider callbacks/webhooks** (delivered, bounced, opened, clicked, complained) sent to your
   endpoints; you ingest them (idempotently) and update each notification's status.
2. **Bounce/complaint signals → suppression list** (stop sending to bad/complaining addresses). It
   matters because mailbox providers **penalize senders** with high bounce/complaint rates — you can get
   **blocklisted**, tanking deliverability for *everyone*.
3. Because "sent to provider" only means your worker handed it off; **actual device delivery** depends on
   the provider, network, and device state (offline, uninstalled, DND) — it's **best-effort**, confirmed
   only (partially) by async callbacks.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **DCE mirror:** ingest an **event** → **stages** (dedup → preferences → template → rate-limit) →
   **emit via channel**, with **Kafka decoupling** between stages, **idempotent processing**, **retries**,
   and a **DLQ** for failures — structurally identical to DCE's ingest → exclusions/rules/models →
   suppressions → emit-decision.
2. **DCE suppression maps to:** (a) the **dedup gate** (don't emit a duplicate notification, like
   suppressing a duplicate/incorrect recommendation) and (b) the **preference/opt-out + suppression-list
   gate** (don't emit what the user muted / to a known-bad address, like suppressing outputs that
   shouldn't leave the pipeline).
3. **Redis** → dedup keys + preference cache; **Kafka** → the decoupling/fan-out queue between service and
   workers; **Celery (Beat)** → scheduled/recurring/digest sends.
