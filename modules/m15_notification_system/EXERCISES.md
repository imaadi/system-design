# m15 — Exercises (Notification System)

> Do these on paper / out loud before checking `solutions/m15_notification_system/`. Dedup, provider
> reliability, preferences, and priority are the core.

---

## Exercise 1 — Estimation
Assume **200M users**, avg **6 notifications/user/day**, channel split push 60% / email 30% / SMS 10%.
1. Notifications/sec (avg + peak at 5×)?
2. Per-channel volume/day for each channel?
3. A campaign targets 80M users at once — why does this need a separate path, and what do you do?

---

## Exercise 2 — The pipeline
1. Draw the end-to-end pipeline from producer to provider, naming every component.
2. What does the queue between the Notification Service and the workers buy you (three things)?
3. Why are producers kept ignorant of APNs/Twilio specifics?

---

## Exercise 3 — Deduplication
1. A producer bug fires the same "order shipped" event 500 times. How do you ensure one notification?
2. Design the dedup key and where it's stored.
3. Why is this "at-least-once + dedup = effectively-once," and where have you seen this before?

---

## Exercise 4 — Provider reliability
1. APNs starts timing out 50% of requests. What do you do (name four mechanisms)?
2. A phone number is invalid (permanent failure). Retry or not? What do you do with it?
3. An OTP must arrive and APNs/push is down. What's your move?

---

## Exercise 5 — Preferences & compliance
1. When do you check user preferences, and why is it a hard gate (not optional)?
2. Name the laws relevant to SMS and email opt-out/consent.
3. A user opted out of marketing but you need to send a security alert. Allowed? How do you model this?

---

## Exercise 6 — Priority
1. Why do you need separate priority lanes?
2. Give two examples of transactional vs two of bulk notifications.
3. What happens to an OTP if everything shares one queue during a campaign?

---

## Exercise 7 — Rate limiting
1. Name the two distinct rate limits and what each protects.
2. How does batching/digesting help with one of them?
3. How do workers avoid getting throttled by a provider?

---

## Exercise 8 — Scheduling & digests
1. How do you implement "send this in 2 hours" and "send a daily digest at 9am"?
2. How does a digest reduce notification fatigue?
3. Which of your real tools would you use for scheduling?

---

## Exercise 9 — Delivery tracking
1. How do you know whether a notification was delivered/opened/bounced?
2. What do you do with bounce/complaint signals, and why does it matter for deliverability?
3. Why is "sent to provider" not the same as "delivered to device"?

---

## Exercise 10 — Your-systems tie-in 🏭
1. Explain how this notification pipeline mirrors the **DCE claims pipeline** stage-by-stage.
2. Map DCE **suppression rules** to two features of this design.
3. Which of your tools (Redis, Kafka, Celery) would you use for: dedup keys, decoupling/fan-out,
   scheduled sends?

---

When done, check `solutions/m15_notification_system/README.md`, then say **"Module 16"**.
