# m15 — Cross-Questions ("if-and-buts") on the Notification System

> Answer out loud in 2–3 sentences before reading the model answer. Dedup, provider reliability,
> preferences/compliance, and priority are where this is won.

---

### Q1. What's the overall architecture of a notification system?
**A.** A **reliable event-driven pipeline** (m08): producers (any service) call a thin **Notification
Service** with `(user, type, data, idempotency_key)`; it **dedups, checks preferences, renders the
template, and rate-limits**, then **enqueues** per-channel jobs onto a **message queue** with **priority
lanes**; **per-channel workers** consume and call third-party **providers** (APNs/FCM/Twilio/SES) with
**retries/backoff/circuit-breakers/DLQ**, tracking delivery. The queue decouples producers from slow,
flaky delivery and absorbs bursts.

---

### Q2. How do you make sure a user doesn't receive the same notification multiple times?
**A.** **Deduplication via an idempotency key** — a key per logical notification (e.g.
`hash(user_id, type, event_id, channel)`) stored in **Redis** with a TTL. Before sending, **check-and-set**
the key atomically; if it already exists, **skip**. This converts the inevitable **at-least-once** delivery
(retries, queue redelivery, buggy producers) into an **effectively-once** send. It's the same idempotency
pattern as m08/m14, and conceptually the same as DCE **suppression** (stop the duplicate output from
going out).

---

### Q3. Third-party providers (APNs, Twilio) fail and rate-limit you. How do you handle that?
**A.** Treat them as **unreliable external dependencies** (m10): **retry transient failures** (timeouts,
503s) with **exponential backoff + jitter**; **don't retry permanent failures** (invalid number, hard
bounce) — add the address to a **suppression list**; **DLQ** after N attempts; put a **circuit breaker**
per provider so a failing provider isn't hammered (fail fast); and for **urgent** notifications,
**fall back** to another channel (push down → SMS). The dedup key makes all this retrying safe.

---

### Q4. Why do you need user preferences, and when do you check them?
**A.** Users opt out per **channel** and per **type** (mute marketing, keep security alerts), and you must
honor it — both for UX (fatigue) and **legally** (CAN-SPAM, **TCPA** for SMS, **GDPR** consent). You check
preferences **before sending** — it's a **hard gate**, not a nicety; sending after an opt-out is a
real legal/financial risk. Preferences live in a fast store (cached in Redis). Truly transactional/
critical messages (OTP, security) may bypass *promotional* opt-outs, designed carefully.

---

### Q5. An OTP and a 10M-user marketing campaign are sent around the same time. How do you keep the OTP fast?
**A.** **Priority lanes** — separate queues by priority. A **high-priority/transactional** lane (OTP,
security, password reset) has dedicated workers and low latency; a **bulk/promotional** lane handles
campaigns and can lag. The OTP goes on the transactional lane and **skips the bulk backlog** entirely, so
it arrives in seconds even while millions of marketing messages are queued. Never let urgent traffic queue
behind bulk.

---

### Q6. How do you send a campaign to 50M users without melting the system or the providers?
**A.** A **dedicated bulk path** separate from transactional: **expand the segment into per-user jobs in
batches**, **spread the sends over time** (rate-paced — don't fire 50M provider calls in one second, which
would blow provider quotas and your own infra), and keep it **off the priority lane**. It's the m13
fan-out-storm pattern: process the huge fan-out asynchronously, batched and throttled, while real-time
transactional notifications flow unaffected.

---

### Q7. What are the two different rate limits in a notification system?
**A.** **(1) Per-user (notification fatigue):** cap how many notifications a user receives (e.g. ≤ N/hour)
and/or **batch into digests** — over-notifying causes uninstalls and opt-outs. **(2) Per-provider:**
respect APNs/Twilio/SES **rate limits/quotas** so you aren't throttled or blocked — workers pace sends
(token bucket, m12) to stay under provider limits. They protect different things: the user's experience vs
your relationship with the provider.

---

### Q8. "Sent" vs "delivered" — what can you actually guarantee?
**A.** You can confirm **"sent to the provider"** (your worker successfully handed it to APNs/Twilio), but
**actual delivery to the device** depends on the provider, network, and device (offline, app uninstalled,
DND). Delivery is **best-effort**. You track what you can via **provider callbacks/webhooks** (delivered,
bounced, opened) and update status + prune bad addresses (suppression list) — but you don't promise
end-device delivery, only that you reliably attempted it.

---

### Q9. How do you avoid "notification fatigue"?
**A.** Several levers: **per-user rate limits** (cap frequency), **digests/batching** (combine "12 new
likes" into one notification instead of 12), **smart timing** (respect quiet hours/timezone), **good
defaults + easy preferences** (let users tune channels/types), and **relevance** (don't send low-value
notifications). Fatigue is a real business metric — over-notifying drives opt-outs and uninstalls, which is
why batching/digests and per-user limits are first-class, not afterthoughts.

---

### Q10. Walk me through what happens when an "order shipped" event fires.
**A.** The order service calls the **Notification Service** with `(user_id, type=order_shipped, data,
idempotency_key)`. The service **dedups** (already processed this key? skip), **loads preferences**
(user wants push+email, not SMS for this type), **renders templates** per channel/locale, **checks rate
limits**, and **enqueues** a push job and an email job on the appropriate (transactional) lane. **Workers**
call **FCM/APNs** and **SES**, retrying on transient failure with backoff, DLQ on repeated failure, and
log delivery status. The user gets exactly one push + one email.

---

### Q11. Where does the message queue fit and why is it essential?
**A.** Between the **Notification Service** and the **channel workers**. It's essential because delivery is
**slow and flaky** (external provider calls), so you **decouple** the fast producer path from it (the
producer doesn't block on Twilio), **absorb bursts** (a campaign queues instead of overwhelming),
**retry** failed sends, scale workers independently, and implement **priority lanes**. It's the m08
queue/log decoupling applied — Kafka (durable, replayable) or SQS, with per-channel/priority topics.

---

### Q12. How do you handle scheduled and recurring notifications?
**A.** A **scheduler** (a delay queue, or **Celery Beat** / a cron-like service) enqueues notifications at
their due time onto the normal pipeline. For recurring (daily digest), the scheduler fires per interval and
the digest job aggregates the relevant events since last send. Delay queues (SQS delay, Redis sorted-set by
timestamp) handle "send in 2 hours"; a time-bucketed scheduler scans due items. The send itself reuses the
same dedup/preferences/worker pipeline.

---

### Q13. How is this system the same as a claims/ML pipeline (your DCE work)?
**A.** Structurally **identical**: ingest an **event** → run it through **stages** (here: dedup →
preferences → template → rate-limit; there: exclusions → rules → models → suppressions) → **emit an output
via a channel**, with **Kafka decoupling** between stages, **idempotent processing**, **retries**, and a
**DLQ-style** path for failures. DCE's **suppression rules** (block incorrect/unwanted outputs) are the
direct analog of **dedup + preference gating** (block duplicate/muted notifications). Same pipeline shape,
different domain.

---

### Q14. A producer has a bug and fires the same notification event 1,000 times. What happens?
**A.** The **idempotency/dedup key** saves you: the first event is processed and its key recorded in
Redis; the other 999 (same key) are **detected and skipped** — the user gets **one** notification, not
1,000. This is exactly why dedup is the defining correctness feature: it makes the system robust to buggy
producers, retries, and at-least-once redelivery. (You'd also alarm on the abnormal event rate via
observability.)

---

### Q15. How do you support multiple channels with different content/format?
**A.** **Templating**: producers send **structured data + a type**, and the system renders per-channel,
per-locale templates (a push has a short title/body, an email is rich HTML, an SMS is plain text + length
limits). Templates live in a **central template store** (versioned). This decouples producers (who
shouldn't format strings) from presentation, and lets you change copy/localization without touching the
producing services.

---

### Q16. What's a suppression list and why do you need one?
**A.** A list of addresses/devices you must **stop sending to**: hard email bounces, invalid phone
numbers, unsubscribes, and complaint reports (spam marks). You consult it **before** sending and skip
suppressed targets. It's essential for **deliverability** (mailbox providers penalize senders with high
bounce/complaint rates — you can get blocklisted) and **compliance** (honoring unsubscribes). Provider
delivery callbacks feed it automatically. It's the same "don't emit a known-bad output" idea as DCE
suppression.

---

### Q17. How do you guarantee an OTP is delivered reliably and quickly?
**A.** Put it on the **high-priority transactional lane** (skips bulk), use a **reliable channel with
fallback** (SMS via a primary provider, **fall back** to a secondary provider or push if the first fails),
**retry fast** with short backoff (it's time-sensitive), and **dedup** so a retry doesn't send two codes.
Track delivery via provider callbacks. OTPs are the canonical "urgent + must-arrive" case — they justify
multi-provider fallback and a dedicated fast path that bulk marketing never touches.

---

### Q18. What are the main bottlenecks and SPOFs?
**A.** **Third-party providers** (rate limits/outages → circuit breakers, fallback, multi-provider); the
**message queue** (HA — it's the pipeline backbone); the **Notification Service** (stateless → scale
horizontally; the dedup/preference stores must be fast + HA Redis); and **bulk fan-out** (a campaign
storm → dedicated batched/throttled path). Observability (m10): per-channel/provider delivery+failure+
bounce rates, queue lag, dedup-hit rate, DLQ depth, opt-out rate — alert on provider failure spikes and
queue backlog.

---

### Q19. How do you do delivery tracking and analytics?
**A.** Providers send **asynchronous callbacks/webhooks** (delivered, bounced, opened, clicked, complained)
to your endpoints; you ingest these (idempotently — they can be retried), update each notification's
**status**, feed the **suppression list** (bounces/complaints), and aggregate into **analytics** (delivery
rate, open rate per type/channel) — often via a stream (Kafka) into a columnar store (m04 OLAP). This
closes the loop: you learn what was actually delivered/engaged and improve targeting + deliverability.

---

### Q20. How does this connect back to the chat system (m14)?
**A.** Chat's **"wake the offline user"** path **is** a notification-system use case: when a chat
recipient is offline, the chat service becomes a **producer** that emits a notification, and this pipeline
delivers it via **push (APNs/FCM)**. m14 was a specific instance; m15 generalizes it — multi-channel,
dedup, preferences, priority, retries, compliance. Many systems (chat, orders, security) are *producers*
feeding one shared notification platform, which is exactly why you build it as a reusable, decoupled
pipeline.
