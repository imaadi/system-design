# m03 — Exercises (Networking & Communication)

> Do these on paper / out loud before checking `solutions/m03_networking_communication/`. The goal:
> reason about *what an arrow costs and guarantees*, and *choose how components talk.*

---

## Exercise 1 — Count the round trips (latency budget)
A user in **Mumbai** calls your API server in **Virginia (US-East)**. Cross-continent RTT ≈ **200 ms**.
The connection is **cold** (no keep-alive), HTTPS with **TLS 1.3**.
1. List the setup steps before any application data flows, and the RTTs each costs.
2. Roughly how much time is spent in **setup** before the server even starts working?
3. Name **three** changes that would cut this, and say which round trip each removes.
4. If the API itself does 3 sequential internal calls (each a 1 ms same-DC RTT), does that matter
   compared to the cross-continent setup? What's the lesson?

---

## Exercise 2 — TCP or UDP?
For each, choose TCP or UDP and justify in one line:
1. A banking REST API.
2. A live video call.
3. A DNS query.
4. Streaming a stored movie (Netflix-style VOD).
5. A multiplayer first-person shooter's position updates.
6. Uploading a file to object storage.

---

## Exercise 3 — The HTTP head-of-line story
Explain, in 4 sentences, the progression HTTP/1.1 → HTTP/2 → HTTP/3 **specifically through the lens of
head-of-line blocking.** Your answer must use the words "application layer," "transport layer,"
"multiplexing," and "QUIC/UDP." Then: **on a very lossy mobile network, could HTTP/2 actually be worse
than HTTP/1.1?** Why?

---

## Exercise 4 — Idempotency design
You're building `POST /payments` (charge a card). A client's request times out and it retries.
1. Why is a naive implementation dangerous?
2. Design the **idempotency-key** mechanism: what does the client send, what does the server store,
   and what happens on (a) first request, (b) a retry with the same key, (c) two *different* charges?
3. Which HTTP methods are idempotent *by default*, and why is GET also "safe"?
4. How does this connect to message-queue delivery guarantees (forward-reference to m08)?

---

## Exercise 5 — Pick the API style
For each, choose **REST / gRPC / GraphQL** and justify:
1. A public developer API for a payments company (third parties integrate).
2. Internal communication between 30 latency-sensitive microservices.
3. A mobile app + web app that each need differently-shaped, nested data from many backend services.
4. A service that needs **bidirectional streaming** between two internal components.
5. A simple internal CRUD admin tool.
Then: describe a realistic system that uses **all three** and where each lives.

---

## Exercise 6 — Choose the real-time transport
For each, pick **short-poll / long-poll / SSE / WebSocket / webhook** and justify:
1. A "you have 3 new notifications" badge that can lag a few seconds.
2. A two-person live chat with typing indicators.
3. Streaming an LLM's answer token-by-token to a browser.
4. Stripe telling *your* backend that a payment succeeded.
5. A live sports score ticker (display only).
6. A collaborative document editor (Google-Docs-style).

---

## Exercise 7 — Scaling persistent connections
You must support **2 million concurrent users** on a chat app over WebSockets.
1. Why can't you treat the WebSocket servers as stateless the way you would HTTP app servers?
2. A box holds ~50k connections. Roughly how many connection servers do you need (ignore HA)?
3. User A (connected to server S1) sends a message to user B (connected to server S7). How does the
   message get from S1 to B? Name the component you need.
4. What has to happen when a connection server dies or you deploy a new version?

---

## Exercise 8 — Status codes & flow control
Your API is getting overwhelmed by a misbehaving client retrying aggressively.
1. What status code do you return to say "slow down," and what header do you add?
2. What status code says "I'm overloaded right now, try later"?
3. What should a *well-behaved* client do on receiving these (name the algorithm)?
4. Why is returning **200 OK** with an error message in the body (instead of a proper 4xx/5xx) a bad
   idea for clients, caches, and monitoring?

---

## Exercise 9 — Apply to your own systems 🏭
1. Questimate's services talk over **REST with env-injected base URLs**. Argue *for* that choice, then
   describe the conditions under which you'd migrate a hot internal path to **gRPC**, and the costs.
2. DCE ingests claims via **Kafka** rather than synchronous REST. Frame this as a sync-vs-async
   decision: what does async buy here, and what does it cost (name two)?
3. Where does **JWT bearer auth** get validated in Questimate's request path, and why is doing it at
   that layer (vs in every service) the right call?

---

When done, check `solutions/m03_networking_communication/README.md`, then say **"Module 4"**.
