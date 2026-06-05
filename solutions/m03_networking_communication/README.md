# m03 — Solutions (Networking & Communication)

> Check the *reasoning*. The recurring lesson: round trips dominate small-request latency, and you
> choose *how* components talk based on who's calling and the access/real-time pattern.

---

## Exercise 1 — Count the round trips
RTT ≈ 200 ms, cold HTTPS, TLS 1.3.
1. **Setup steps:** (a) DNS lookup — ~0 if cached, else up to a few RTTs on a full miss; (b) **TCP
   3-way handshake = 1 RTT** (200 ms); (c) **TLS 1.3 handshake = 1 RTT** (200 ms). *Then* the HTTP
   request/response is another RTT.
2. **~400 ms of pure setup** (TCP + TLS) before the server starts — plus DNS if uncached, plus
   ~200 ms for the actual request/response. So ~600 ms wall-clock for a cold call that the server
   might serve in 10 ms.
3. Three fixes: **(a) keep-alive / connection pooling** → removes the TCP+TLS handshake on subsequent
   calls (saves ~400 ms each); **(b) CDN/edge TLS termination near Mumbai** → the handshake RTT
   becomes ~5 ms instead of 200 ms; **(c) TLS 1.3 0-RTT session resumption** → removes the TLS
   handshake RTT for repeat clients. (Also: put a replica in Mumbai → kills the cross-continent RTT
   entirely.)
4. **Three same-DC calls = ~3 ms**, utterly negligible next to ~400–600 ms of cross-continent setup.
   **Lesson:** optimize the *expensive* round trips first (the cross-continent / handshake ones); a
   few internal same-DC hops barely register. Don't micro-optimize cheap rungs while ignoring the
   expensive one.

---

## Exercise 2 — TCP or UDP?
1. Banking API → **TCP** (correctness/ordering essential).
2. Live video call → **UDP** (a late packet is worthless; drop and continue beats stalling).
3. DNS query → **UDP** (tiny request/reply; cheap to retry; low overhead). *(Large/secure DNS can use
   TCP/DoH.)*
4. Stored movie / VOD → **TCP** (it's buffered ahead; you want every byte, and HTTP-over-TCP/CDN works
   great — note: *live* low-latency streaming may use UDP/WebRTC, but classic VOD is TCP).
5. FPS position updates → **UDP** (newest position matters; don't re-send stale positions).
6. File upload → **TCP** (every byte must arrive intact and in order).

---

## Exercise 3 — Head-of-line story
> "**HTTP/1.1** suffers **application-layer** HOL blocking — responses return in order on a connection,
> so one slow response blocks the rest (browsers opened ~6 connections to fake parallelism). **HTTP/2**
> introduced **multiplexing** — many streams interleaved on one connection — fixing app-layer HOL, but
> it still rides a single TCP connection, so a lost packet causes **transport-layer** HOL across all
> streams. **HTTP/3** replaces TCP with **QUIC over UDP**, giving independent per-stream ordering, so a
> lost packet stalls only its own stream — eliminating HOL at both layers."
**Could HTTP/2 be worse than HTTP/1.1 on a lossy mobile network?** Yes — because HTTP/2's single TCP
connection means one dropped packet stalls *every* multiplexed stream (transport HOL), whereas
HTTP/1.1's separate connections fail more independently. On high-loss links that can make HTTP/2 lose;
it's exactly the weakness HTTP/3/QUIC was designed to fix.

---

## Exercise 4 — Idempotency design
1. **Danger:** a timeout doesn't tell the client whether the charge succeeded, so it retries — a naive
   server would charge **twice** (double-charge the customer).
2. **Idempotency key:** (client) generate a unique key (UUID) and send `Idempotency-Key: <uuid>`;
   (server) store the key → (status, result) the first time. **(a) First request:** no key on record
   → process the charge, persist (key → result), return it. **(b) Retry with same key:** key found →
   **return the stored original result** without charging again. **(c) Two different charges:** different
   keys → both processed normally. (Edge cases: a key seen but still "in progress" → return 409/“retry
   later”; keys expire after a window.)
3. **GET, HEAD, PUT, DELETE** are idempotent by default; **GET/HEAD are also "safe"** (no state change
   at all, so cacheable/prefetchable). PUT/DELETE change state but repeating them lands the same final
   state.
4. **m08 tie-in:** message queues typically guarantee **at-least-once** delivery (a message can be
   redelivered), so consumers must be **idempotent** — same idempotency-key / dedup idea turns
   at-least-once *delivery* into at-most-once *effect* ("effectively exactly-once").

---

## Exercise 5 — Pick the API style
1. Public payments API → **REST** (ubiquitous, language-agnostic, easy for third parties to integrate
   and debug, HTTP-cacheable, great tooling).
2. 30 internal latency-sensitive services → **gRPC** (binary Protobuf, HTTP/2 multiplexing, strict
   schemas + codegen, streaming; ideal east-west).
3. Mobile + web needing varied nested data from many services → **GraphQL** (client picks fields,
   one round trip to assemble a screen; a BFF aggregating internal services).
4. Bidirectional streaming between two internal components → **gRPC** (native bidi streaming).
5. Simple internal CRUD admin tool → **REST** (simplest; no need for gRPC tooling or GraphQL
   complexity).
**All three together:** clients hit a **GraphQL/REST API gateway/BFF** at the edge (north-south); that
layer fans out to internal services over **gRPC** (east-west); a public partner API is exposed as
**REST**. Each style where its strength matches the caller.

---

## Exercise 6 — Real-time transport
1. Notification badge (can lag) → **long-poll** or even **short-poll** (low stakes; don't pay for a
   persistent connection). SSE is also fine.
2. Two-person live chat + typing indicators → **WebSocket** (bidirectional, low-latency both ways).
3. LLM answer token-by-token → **SSE** (one-way server→client stream; simple; auto-reconnect; the
   prompt itself is a normal POST that starts the stream).
4. Stripe → your backend on payment success → **webhook** (server-to-server POST to your registered
   URL; verify signature + handle retries/idempotency).
5. Live score ticker (display only) → **SSE** (one-way push; no client→server needed).
6. Collaborative doc editor → **WebSocket** (frequent low-latency bidirectional edits + presence).

---

## Exercise 7 — Scaling persistent connections (2M concurrent)
1. **Not stateless:** each WebSocket is a long-lived connection pinned to one server holding in-memory
   per-connection state; you can't freely route a given user's frames to any box, and losing a box
   drops its users' live connections.
2. ~50k/box → **~40 connection servers** for 2M (plus headroom + HA → more).
3. **S1 → B (on S7):** S1 publishes the message to a shared **pub/sub backplane** (e.g. Redis
   pub/sub or Kafka) keyed by recipient; S7 is subscribed for the users it holds, receives it, and
   pushes it down B's socket. You also need a **session registry** (which server holds which user) or
   a topic-per-user model. The needed component: a **pub/sub backplane + connection/session manager.**
4. **On death/deploy:** affected clients' connections drop → clients **auto-reconnect** (with backoff)
   and get routed (via the LB) to a healthy server, which re-registers them in the session registry
   and re-subscribes; you design for graceful reconnection and avoid a reconnect stampede (jitter).

---

## Exercise 8 — Status codes & flow control
1. **429 Too Many Requests**, with a **`Retry-After`** header.
2. **503 Service Unavailable** (also often with `Retry-After`).
3. **Exponential backoff with jitter** — wait increasing, randomized intervals between retries so
   clients don't synchronize and stampede (the "thundering herd" / retry storm).
4. **200-with-error-in-body is harmful** because: clients/SDKs key off status codes (a 200 looks like
   success → they won't retry or handle the error), **caches** may cache the "successful" error
   response, **load balancers/monitoring** count it as healthy (your error-rate dashboards lie), and it
   breaks standard HTTP semantics. Use the correct 4xx/5xx so the whole ecosystem behaves correctly.

---

## Exercise 9 — Apply to your own systems 🏭 (model)
1. **REST between Questimate services:** *for* — simple, language-agnostic, human-debuggable (curl),
   no codegen/build coupling, and the calls (pull factor returns / yield curve / residuals) are
   coarse-grained and not on a tight latency budget. **Migrate to gRPC when:** a path becomes
   chatty/latency-critical or high-QPS, where binary Protobuf + HTTP/2 multiplexing + streaming would
   pay off. **Costs:** loss of human-readability/easy debugging, schema/codegen discipline and build
   coupling, no native HTTP caching, and migration effort. (This is a real, defensible trade-off you
   can speak to.)
2. **DCE via Kafka (async):** async **buys** decoupling (producers don't block on slow scoring),
   **burst absorption** (a surge queues instead of overwhelming), **resilience** (consumers can be down
   briefly and catch up), and independent scaling of ingestion vs processing. It **costs** (a) added
   end-to-end **latency** + eventual (not immediate) processing, and (b) **complexity**: ordering,
   idempotency/dedup on redelivery, and harder end-to-end debugging/tracing (m08).
3. **JWT validated at the edge (nginx/gateway + the auth layer), with the token key in Redis.** Doing
   it once at the entry point (vs in every service) centralizes the cross-cutting auth concern, keeps
   downstream services simpler, and enables **instant revocation** by deleting the Redis key — the
   right altitude for auth (§9 north-south concern), and it keeps the app tier stateless.
