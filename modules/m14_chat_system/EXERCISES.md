# m14 — Exercises (Chat System)

> Do these on paper / out loud before checking `solutions/m14_chat_system/`. Connection management and
> message routing are the core; later ones are extensions.

---

## Exercise 1 — Estimation
Assume **800M DAU**, **30 messages/user/day**, **~12% online concurrently at peak**, message ≈ **200 B**.
1. Messages/sec (avg + peak at 4×)?
2. Concurrent connections at peak? If a server holds 80k connections, how many connection servers?
3. Message storage/day and /year?
4. Which of these numbers most shapes the architecture, and why?

---

## Exercise 2 — Why WebSockets
1. Why can't plain HTTP request/response power chat?
2. Why WebSockets over SSE or long-polling specifically?
3. What property of WebSockets creates the central design challenge?

---

## Exercise 3 — The routing problem
A is connected to connection-server-1; B is connected to connection-server-7.
1. Trace exactly how a message from A reaches B (name every component).
2. What two pieces of infrastructure make cross-server delivery possible?
3. Redis pub/sub vs Kafka for the backplane — one-line trade-off.

---

## Exercise 4 — Connection management
1. Why can't connection servers be treated as stateless and load-balanced per request?
2. What is the session registry and what does it map?
3. How do you detect a dead connection, and what problem (from m09) does this echo?
4. A connection server dies. Walk through what the affected clients do.

---

## Exercise 5 — Ordering & delivery
1. How do you guarantee messages appear in order within a conversation? Why not use timestamps?
2. What delivery guarantee do you give, and how do you avoid showing duplicates?
3. Why must you persist the message *before* ACKing the sender?
4. Explain how sent/delivered/read receipts work.

---

## Exercise 6 — Offline delivery
1. The recipient is offline when a message is sent. What happens to the message?
2. How is the recipient alerted, and what happens when they reconnect?
3. Why is store-and-forward only possible because of a choice you made in Ex 5.3?

---

## Exercise 7 — Group chat
1. How do you deliver a message to a 20-person group (which fan-out)?
2. A Discord channel has 200,000 members. Why does the small-group approach break, and what do you do
   instead?
3. How is this the same problem as the celebrity in m13?

---

## Exercise 8 — Presence
1. Why is naive "broadcast my online status to all contacts" a problem?
2. Give three mitigations.
3. What consistency model does presence use, and why is that acceptable?

---

## Exercise 9 — Extensions
1. **Media:** how do you handle a user sending a 20 MB video? Where does it live, how is it delivered?
2. **E2E encryption:** what changes, and name two features it complicates.
3. **Slack vs WhatsApp:** name two design differences and why.

---

## Exercise 10 — Bottlenecks + your-systems tie-in 🏭
1. Name the main bottlenecks/SPOFs and one mitigation each.
2. Map the **pub/sub backplane** and **effectively-once delivery** to tools/patterns you've used
   (Redis/Kafka; DCE idempotent reprocessing).
3. Questimate's real-time trading needed server-push too — frame the WebSocket-vs-polling decision you'd
   make and the trade-off.

---

When done, check `solutions/m14_chat_system/README.md`, then say **"Module 15"**.
