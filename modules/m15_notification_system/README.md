# Module 15 — Design a Notification System ⭐ 🏭

> Send notifications to users across **push, SMS, email, in-app** — reliably, at scale, without spamming
> them or violating the law. It looks like a CRUD app but is really a **reliable event-driven pipeline**:
> decouple producers from delivery (m08), fan out across channels, **dedup** so nobody gets 5 copies,
> respect **preferences/compliance**, **rate-limit** (the user *and* the providers, m12), and survive
> flaky **third-party providers** (m10). This is **your DCE pipeline shape** — ingest event → stages →
> emit output, with **suppression/dedup** and retries.

> **Format (Part 2):** worked m01 walkthrough; the **pipeline + reliability** is the deep-dive.

---

## Step 1 — Requirements

**Functional (core):**
1. **Send a notification** to a user via one or more **channels**: **push** (mobile/web), **SMS**,
   **email**, **in-app**.
2. **Notification types** (transactional: OTP, order shipped; vs promotional: marketing digest).
3. **User preferences / opt-out** (per channel, per type) — *checked before sending*.
4. **Templating** (content per type/locale) and **scheduling** (send now / later / recurring).
5. *(Stretch)* delivery tracking/analytics, A/B, batching/digests.

**Non-functional (these drive it):**
- **Reliability:** don't **lose** important notifications (at-least-once) — but **don't duplicate** them
  (no "you got 5 copies of the same alert").
- **High throughput + fan-out:** "notify 10M users of an outage" must not melt the system.
- **Low latency for urgent** (OTP must arrive in seconds) while **bulk** can be slower → **priority**.
- **Respect external limits:** third-party providers (APNs/FCM/Twilio/SES) have **rate limits** and
  **fail** → handle gracefully.
- **Compliance:** legal opt-out/consent (TCPA for SMS, CAN-SPAM for email, GDPR consent) — *not optional.*

> **The defining insight:** this is a **reliability + fan-out pipeline** problem, and the hard parts are
> **deduplication**, **preferences/compliance**, and **surviving unreliable providers** — not the
> "send" itself.

---

## Step 2 — Estimation
Assume **100M users**, avg **~5 notifications/user/day** + occasional bulk campaigns.
- **Volume:** 100M × 5 = 500M/day ÷ 10⁵ = **~5,800/sec** avg (~30k peak). Plus **bulk bursts** (a campaign
  = up to 100M at once → a separate bulk path).
- **Channel split (rough):** push ~60%, email ~30%, SMS ~10% → each channel's workers + provider limits
  sized accordingly (SMS is the most rate-limited/expensive).
- **Fan-out:** one *event* can become many notifications (e.g. "X posted" → all followers, the m13 fan-out
  again) → the system multiplies events into per-user, per-channel sends.
- **Dedup state:** an idempotency/dedup key per (user, notification, channel) in Redis (small, TTL'd).

> **The estimate frames it:** moderate steady volume but **huge bursty fan-out** (campaigns) and lots of
> **external provider calls** → **queues + async workers per channel + retries + two paths (transactional
> vs bulk)**.

---

## Step 3 — API
```
POST /api/notifications           { user_id, type, channels?, data, idempotency_key }
POST /api/notifications/bulk      { segment, type, template_id, schedule? }      (campaigns)
GET/PUT /api/users/{id}/preferences   { push:bool, sms:bool, email:bool, per-type... }
```
- **Producers** (any service: order service, chat's offline path, security) just call "notify user X about
  type Y with data Z" + an **idempotency_key** — the platform handles channel selection, preferences,
  dedup, templating, rate-limiting, retries. *Producers don't know about APNs/Twilio.*

---

## Step 4 — High-level design ⭐ (the pipeline)

```
  producers ──▶ Notification Service ──▶ [Message Queue / Kafka] ──▶ per-channel workers ──▶ providers
  (any svc)        │ validate                  (decouple, buffer,        │ Push worker ─▶ APNs / FCM
                   │ dedup (idempotency key)    priority lanes)          │ SMS worker  ─▶ Twilio
                   │ load user PREFERENCES ─────────────────────────────│ Email worker─▶ SES/SendGrid
                   │ render TEMPLATE                                     │ In-app      ─▶ store + WS push (m14)
                   │ rate-limit (user + provider)                       └─ retries/backoff, DLQ, track
                   ▼
            Preferences store · Template store · Dedup cache (Redis) · Delivery log
```

**Flow for one notification:**
1. A producer calls the **Notification Service** with `(user, type, data, idempotency_key)`.
2. The service **deduplicates** (has this idempotency_key been processed? → drop if yes), **loads the
   user's preferences** (opted out of SMS? skip SMS), **picks channels**, **renders the template**, and
   **checks rate limits** (per-user fatigue + per-provider).
3. It **enqueues** one job per (channel) onto a **message queue** (Kafka/SQS) — decoupling producers from
   the slow, flaky delivery, buffering bursts, enabling **priority lanes**.
4. **Per-channel workers** consume jobs and call the **third-party provider** (APNs/FCM/Twilio/SES); on
   failure they **retry with backoff** (m10) and **DLQ** after N attempts.
5. **Delivery status** (sent/failed/bounced) is **logged/tracked** (and fed back for analytics +
   suppression of bad addresses).

> This is a **classic event-driven pipeline** (m08): producers emit, a queue decouples, workers deliver.
> The Notification Service is a thin orchestrator; the queue + channel workers do the heavy, retryable
> work. **It's the same shape as your DCE claims pipeline.**

---

## Step 5 — Deep Dive: reliability & deduplication ⭐⭐

The two hardest correctness problems:

### Deduplication (don't send 5 copies) ⭐
A retried event, a buggy producer, or at-least-once queue redelivery (m08) can trigger the same
notification multiple times. **Fix: an idempotency/dedup key** per logical notification (e.g.
`hash(user_id, notification_type, event_id, channel)`), recorded in **Redis** (with a TTL). Before
sending, **check-and-set** the key atomically; if it's already present, **skip** (don't resend). This is
**at-least-once delivery + idempotent send = effectively-once** — the exact m08/m14 pattern, and your DCE
**suppression** instinct (stop the duplicate/incorrect output from going out).

### Retries, backoff, DLQ, circuit breakers (providers are unreliable) ⭐
Third-party providers **time out, rate-limit, and go down** — they're external dependencies (m10):
- **Retry transient failures** with **exponential backoff + jitter** (a 503/timeout might succeed later).
- **Don't retry permanent failures** (invalid number, hard email bounce) → mark the address bad
  (**suppression list**) and stop.
- **Dead-letter queue (DLQ)** after N attempts for inspection.
- **Circuit breaker** per provider — if APNs is failing, stop hammering it (fail fast), and possibly
  **fall back** to another channel (push down → SMS) for urgent notifications.
- **At-least-once + dedup** means a retry is safe (the dedup key prevents a double-send if the first
  actually went through but the ack was lost).

---

## Step 6 — Deep Dive: preferences, priority, rate limiting

### Preferences & compliance (legal, first-class) ⭐
- **Check preferences BEFORE sending** — per channel and per type (a user can mute marketing but keep
  security alerts). Store in a fast **preferences store** (cached in Redis).
- **Compliance is mandatory:** honor **opt-out/unsubscribe** (CAN-SPAM email, **TCPA** for SMS requires
  consent, **GDPR** consent) — sending after opt-out is a **legal/financial risk**. The preference check
  is a hard gate, not a nicety.
- **Always-allow exceptions:** truly transactional/critical (OTP, security, legally-required) may bypass
  *promotional* opt-outs — but design this carefully.

### Priority lanes ⭐
An **OTP** must not get stuck behind a **10M-user marketing batch**. Use **separate queues by priority**:
a **high-priority/transactional** lane (low latency, dedicated workers) and a **bulk/promotional** lane
(throughput-optimized, can lag). Urgent notifications skip the bulk backlog.

### Rate limiting (two different limits, m12)
1. **Per-user (notification fatigue):** cap how many notifications a user gets (e.g. ≤ N/hour) and/or
   **batch into digests** — over-notifying drives uninstalls/opt-outs.
2. **Per-provider:** respect APNs/Twilio/SES **rate limits** so you aren't throttled/blocked — the workers
   pace their sends (token bucket, m12) to stay under provider quotas.

---

## Step 7 — Bottlenecks, scale & edge cases

- **Bulk fan-out (campaigns):** "notify 50M users" is a **fan-out storm** → a **dedicated bulk path**:
  expand the segment into per-user jobs **in batches**, **spread over time** (don't fire 50M provider
  calls in one second), and keep it **off the transactional lane** (priority). (Same two-speed idea as
  m13 fan-out.)
- **Provider outage:** circuit-break + queue + retry; for urgent, **fall back** to another channel.
- **Duplicate sends:** the dedup key (Step 5) — *the* defining correctness feature.
- **Delivery tracking:** providers give callbacks/webhooks (delivered, bounced, opened) → ingest these to
  update status, prune bad addresses (suppression list), and power analytics. Note: "sent to provider" ≠
  "delivered to device" — track what you can; delivery is best-effort.
- **Scheduling/digests:** a scheduler (Celery Beat / a delay queue) enqueues future/recurring sends and
  batches frequent events into a **digest** (e.g. "12 new likes" instead of 12 pushes) — fights fatigue.
- **Templating/localization:** centralize templates (per type/locale) so producers send **data**, not
  formatted strings.
- **Observability (m10):** send rate, delivery/failure/bounce rates per channel+provider, queue lag,
  dedup hits, DLQ depth, opt-out rate; alert on provider failure spikes + queue backlog.

**Wrap-up:** *"A reliable event-driven pipeline: producers emit `(user, type, data, idempotency_key)` to a
thin **Notification Service** that **dedups** (idempotency key in Redis), **checks preferences/compliance**,
**renders templates**, and **rate-limits** (user fatigue + provider quotas), then enqueues per-channel jobs
onto **priority lanes** (transactional vs bulk). **Per-channel workers** call providers (APNs/FCM/Twilio/
SES) with **retries+backoff, circuit breakers, fallback, and a DLQ**, tracking delivery. Bulk campaigns run
on a separate, time-spread path. Key correctness features: **at-least-once + dedup = effectively-once** and
**preferences as a hard gate**. It's the same ingest→process→suppress→emit shape as a claims pipeline."*

---

## State-of-the-art & real-world notes 📚
- **Notification platforms** (Courier, Knock, OneSignal, AWS SNS/Pinpoint, Firebase) productize exactly
  this: channel routing, preferences, templating, dedup, retries, delivery tracking.
- **The pattern is generic event-driven architecture** (m08): a thin orchestrator + queue + per-channel
  consumers — reused for any "react to events with side effects" system.
- **Providers:** **APNs** (Apple push), **FCM** (Android/web push), **Twilio** (SMS), **SES/SendGrid**
  (email). Each has quotas, async delivery callbacks, and failure modes you must handle.

---

## From your systems 🏭 (this maps almost 1:1)
- **It IS the DCE pipeline shape:** ingest an **event** → run it through **stages** (dedup, preferences,
  template, rate-limit) → **emit output via a channel** — exactly DCE's ingest → exclusions/rules/models →
  emit-decision flow, with **Kafka decoupling** (m08) between stages.
- **Suppression ≈ dedup + preference gating:** DCE's **suppression rules** stop **incorrect/unwanted
  outputs** from leaving the pipeline — *identical* in spirit to the **dedup key** (don't send the same
  notification twice) and **preference/opt-out gate** (don't send what the user muted). Your "stop the bad
  output at the exit" instinct is the core of this design.
- **Idempotent/effectively-once:** at-least-once + dedup is your **idempotent reprocessing** reflex (DCE).
- **Redis + Celery:** dedup keys + preference cache in **Redis**; **Celery Beat** for scheduled/recurring/
  digest sends (Questimate) — your exact tools.
- **External-dependency resilience:** Questimate integrated flaky **broker/3rd-party APIs** → the same
  retries/backoff/circuit-breaker/fallback discipline (m10) you'd apply to APNs/Twilio.

---

## Key concepts (interview-ready)
- **A notification system is a reliable event-driven pipeline (m08):** thin **Notification Service**
  (dedup, preferences, template, rate-limit) → **queue (priority lanes)** → **per-channel workers** →
  **providers (APNs/FCM/Twilio/SES)**. Producers just say "notify X about Y."
- **Dedup is the defining correctness feature:** **idempotency/dedup key** (per user+type+event+channel)
  in Redis → **at-least-once + dedup = effectively-once** (no 5 copies). The DCE-suppression analogy.
- **Providers are unreliable external deps (m10):** **retries+backoff+jitter**, **don't retry permanent
  failures** (suppression list), **DLQ**, **circuit breaker** per provider, **channel fallback** for
  urgent.
- **Preferences/compliance = a hard gate** checked **before** sending (per channel/type; TCPA/CAN-SPAM/
  GDPR). Transactional may bypass *promotional* opt-outs.
- **Priority lanes:** transactional (urgent, low-latency) vs bulk/promotional (throughput) — OTP never
  stuck behind a campaign.
- **Two rate limits (m12):** per-user (fatigue → cap/digest) and per-provider (respect quotas).
- **Bulk campaigns** = fan-out storm → dedicated path, **batched + time-spread**, off the transactional
  lane.
- **Delivery is best-effort** ("sent to provider" ≠ "delivered"); ingest provider callbacks for tracking +
  bad-address suppression.

---

## Go deeper (reading)
- **System Design Interview (Alex Xu): "Design a notification system"** — the standard treatment (pipeline
  + dedup + retries).
- **AWS SNS / Pinpoint** and **Firebase Cloud Messaging** docs — real multi-channel notification infra.
- **Courier / Knock engineering blogs** — modern notification-platform design (preferences, batching,
  dedup, deliverability).
- Revisit **m08** (queues/idempotency/effectively-once), **m10** (retries/circuit breakers/DLQ), **m12**
  (rate limiting), **m13** (fan-out), **m14** (the push-to-offline-user path this generalizes).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m15_notification_system/`](../../solutions/m15_notification_system/README.md).

When you've done the exercises, say **"Module 16"** to design *Typeahead / Autocomplete + Search* (tries,
prefix sharding, ranking, the inverted-index pipeline) — a different flavor: latency-obsessed read systems
and the search-indexing pipeline.
