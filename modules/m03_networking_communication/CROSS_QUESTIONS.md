# m03 — Cross-Questions ("if-and-buts") on Networking & Communication

> Answer each out loud in 2–3 sentences before reading the model answer. These are the follow-ups
> that separate "I know the words" from "I understand the trade-offs."

---

### Q1. What actually happens, step by step, when you type a URL and press enter?
**A.** DNS resolves the name → IP (browser/OS/resolver caches, else walk root→TLD→authoritative).
Then a **TCP 3-way handshake** (1 RTT), then a **TLS handshake** (~1 RTT in 1.3) to encrypt, then the
**HTTP request** goes up, the server does its work (auth, DB, render), and the **HTTP response** comes
back. The key insight: ~2–3 RTTs of *setup* happen before any of my data moves — cross-continent
that's ~300–450 ms, which is why connection reuse, CDNs, and colocation matter.

---

### Q2. For a small API call, is latency dominated by bandwidth or round trips?
**A.** **Round trips, overwhelmingly.** A 1 KB request isn't bandwidth-limited; it's limited by how
many times data must cross the network and how far. The speed of light fixes cross-continent RTT at
~150 ms regardless of your pipe size. So "make it faster" usually means **fewer round trips** (batch,
cache, reuse connections, colocate) far more than "send fewer bytes." Bandwidth matters for *large*
payloads (video, files), not chatty small calls.

---

### Q3. Why does connection reuse (keep-alive / pooling) matter so much?
**A.** Because each *new* HTTPS connection pays the TCP + TLS handshake (2–3 RTTs) before useful work.
If every request opened a fresh connection, you'd pay that tax every time — devastating cross-region.
**Keep-alive** reuses one TCP/TLS connection for many requests; **connection pools** keep a set of warm
connections ready. This amortizes setup to ~zero per request and is why HTTP clients pool and why
microservices hold persistent gRPC channels rather than dialing per call.

---

### Q4. TCP vs UDP — when would you ever choose the "unreliable" one?
**A.** When **timeliness beats completeness**: real-time media (VoIP, video calls, live streaming),
online gaming, and small request/reply like DNS. In a voice call, retransmitting a packet from 2
seconds ago (what TCP would do) is useless and harmful — better to drop it and keep the stream live.
UDP gives you that "fire and forget, build your own recovery if needed" model. For anything where
correctness matters (APIs, DBs, file transfer), TCP's reliability/ordering wins.

---

### Q5. Explain head-of-line blocking and how each HTTP version deals with it.
**A.** Because TCP delivers bytes **in order**, one lost packet stalls everything queued behind it
until it's retransmitted — **head-of-line (HOL) blocking**. **HTTP/1.1** also has *application-layer*
HOL (responses come back in order on a connection), so browsers opened ~6 connections to fake
parallelism. **HTTP/2** multiplexes many streams over one connection → fixes app-layer HOL, **but**
it's still one TCP connection, so a lost packet causes *transport-layer* HOL across all streams.
**HTTP/3** runs over **QUIC (UDP)** with independent per-stream ordering, so a lost packet stalls only
its own stream — HOL solved at both layers.

---

### Q6. If HTTP/2 multiplexes over one connection, why isn't it always strictly better than HTTP/1.1?
**A.** Because of **transport-layer HOL blocking**: on a lossy network, one dropped TCP packet stalls
*all* of HTTP/2's interleaved streams (they share the one TCP connection), whereas HTTP/1.1's separate
connections degrade more independently. So on high-loss links HTTP/2 can underperform. That exact
weakness is why HTTP/3 moved to QUIC/UDP with per-stream reliability. (On good networks HTTP/2 is a
clear win.)

---

### Q7. What makes an operation idempotent, and why do you care in a distributed system?
**A.** Idempotent = performing it **N times has the same effect as once** (GET, PUT, DELETE are;
POST/PATCH usually aren't). I care because **networks fail and clients retry**: when a request times
out, you don't know if it succeeded, so you retry — and if the operation isn't idempotent, the retry
can double-charge / double-create. Idempotency makes retries *safe*, which is the foundation of
reliable communication, queues, and payments.

---

### Q8. How do you make a non-idempotent operation (like "charge a card") safe to retry?
**A.** With an **idempotency key**: the client generates a unique key (UUID) and sends it with the
request; the server records the key + result the first time, and on any retry with the same key it
returns the *original* result instead of charging again. This is how Stripe does payments. It turns
an at-least-once delivery (inevitable with retries) into an at-most-once *effect* — the same idea that
underlies "exactly-once processing" in message queues (m08) and ledgers (m21).

---

### Q9. 401 vs 403 — what's the difference, and why does it matter?
**A.** **401 Unauthorized** = "I don't know who you are" (missing/invalid credentials — authentication
failed). **403 Forbidden** = "I know who you are, but you're not allowed to do this" (authorization
failed). The distinction matters for correct client behavior (401 → prompt login / refresh token; 403
→ don't retry, you simply lack permission) and for not leaking information. Mixing them up is a common
tell of imprecision.

---

### Q10. A client gets a 429 or a 503 — what should it do, and what does that signal about your design?
**A.** **429 Too Many Requests** = you're rate-limited; back off and respect the `Retry-After` header.
**503 Service Unavailable** = the server is overloaded/down; back off (also often with `Retry-After`).
Both signal **backpressure** — the system is protecting itself, and a good client responds with
**exponential backoff + jitter** rather than hammering. Designing these in (rate limits, load
shedding, Retry-After) shows you think about flow control and overload, not just the happy path (m10,
m12).

---

### Q11. When would you choose gRPC over REST?
**A.** For **internal, east-west, performance-critical** service-to-service communication: gRPC gives
binary Protobuf (compact/fast), HTTP/2 multiplexing, a **strict typed schema** with codegen across
languages, and **native streaming** (incl. bidirectional). I'd keep **REST** for public-facing APIs
where ubiquity, human-readability, HTTP caching, and easy debugging matter more. A common pattern:
REST/GraphQL at the edge for clients, gRPC between services internally.

---

### Q12. GraphQL solves over-fetching — so why not use it for everything?
**A.** Because it trades problems. GraphQL kills over/under-fetching and is great for varied
front-ends, but: **HTTP caching breaks** (it's POSTs to one endpoint), it adds **server complexity**
(resolvers + the **N+1 resolver problem**, needing DataLoader-style batching), clients can write
**expensive/deep queries** (you must add depth/complexity limits and harder rate-limiting), and it's
overkill for simple CRUD. So I'd use it where flexible client-driven fetching/aggregation is the real
need (a BFF for mobile+web), not as a blanket replacement for REST.

---

### Q13. What's the "N+1 problem" and where does it show up here?
**A.** It's when fetching a list triggers one query, then **one more query per item** (1 + N) instead
of a single batched query — killing performance. In **REST** it appears as a chatty client making N
follow-up calls to assemble a screen. In **GraphQL** it appears server-side: a resolver for a list
field fires a downstream call **per element**. The fix is **batching** (a single `WHERE id IN (...)`
or a DataLoader that coalesces per-tick requests). Recognizing it across REST/GraphQL/DB layers shows
depth.

---

### Q14. The server needs to push data to clients (e.g. live notifications). Walk me through the options.
**A.** Cheapest to richest: **short polling** (ask every N sec — simple, wasteful, latency = interval);
**long polling** (server holds the request until data — near real-time, no new tech, but ties up a
connection); **SSE** (a one-way server→client HTTP stream — ideal for feeds/notifications and **LLM
token streaming**, with auto-reconnect); **WebSockets** (a persistent **bidirectional** channel —
needed for chat/collab); and **webhooks** (server-to-server push to a registered URL — Stripe/GitHub
style). I pick by *how real-time* and *how bidirectional* the requirement is — I wouldn't use
WebSockets for a notification badge when SSE or long-poll suffices.

---

### Q15. SSE vs WebSockets — when each?
**A.** **SSE** is one-way (server→client) over plain HTTP, simple, with built-in reconnection — perfect
when the client only needs to *receive* a stream: notifications, live dashboards, feeds, and
**streaming LLM tokens**. **WebSockets** are full-duplex (both directions) over an upgraded persistent
connection — needed when the client also sends frequently and low-latency: **chat, multiplayer games,
collaborative editing**. If you don't need client→server on the same channel, SSE is simpler and
cheaper; reach for WebSockets only when bidirectionality is real.

---

### Q16. Why do WebSockets complicate horizontal scaling?
**A.** A WebSocket is a **persistent, stateful connection** — that user is pinned to one specific
server for the connection's life, which breaks the "any server handles any request" model. So you
need: a **connection-manager/gateway** layer tracking which server holds which user's socket, a
**pub/sub backplane** (e.g. Redis pub/sub or Kafka) so a message produced on server A reaches a user
connected to server B, sticky routing, graceful **reconnection** handling, and awareness of the
**~tens-of-thousands-of-connections-per-box** OS limit. "Just use WebSockets" hides this real cost —
it's the core challenge of designing chat (m14).

---

### Q17. What's the difference between a forward proxy, a reverse proxy, an API gateway, and a load balancer?
**A.** A **forward proxy** sits in front of *clients* and acts on their behalf (egress filtering,
anonymity). A **reverse proxy** (nginx) sits in front of *servers*, receiving client requests and
forwarding them in (TLS termination, routing, static files). A **load balancer** is a reverse proxy
specialized to **distribute traffic across replicas** with health checks (m07). An **API gateway** is a
reverse proxy specialized for APIs, adding **auth, rate-limiting, routing, aggregation, versioning,
observability** at a single entry point. They overlap; the gateway often *is* the LB + reverse proxy +
cross-cutting concerns bundled.

---

### Q18. Isn't the API gateway a single point of failure and a bottleneck?
**A.** It can be, so you mitigate: run it **redundantly** (multiple instances behind a load balancer
with health checks), keep it **stateless and thin** (no business logic — just plumbing: auth,
routing, rate-limit), scale it **horizontally**, and make sure it degrades gracefully. The benefit
(centralizing cross-cutting concerns so every service doesn't reimplement auth/rate-limiting) outweighs
the risk *if* you treat it as critical infrastructure and make it HA — same as any other tier.

---

### Q19. Why is HTTP described as "stateless," and how do logins work then?
**A.** Stateless means **each request is independent** — the server keeps no client context in its own
memory between requests; everything needed is in the request. Logins work by the client carrying a
**token/cookie** (e.g. a JWT or session ID) on every request; the server validates it (and looks up any
session state in a **shared store like Redis**, not local memory). That's *why* HTTP is stateless by
design: it lets any server behind a load balancer handle any request, enabling horizontal scaling
(m02). Questimate's Redis-backed JWT is exactly this.

---

### Q20. How does DNS help with availability and global latency — and what's its limitation?
**A.** DNS can return the **nearest** datacenter's IP (**GeoDNS / latency-based routing**) or use
**anycast** (same IP announced from many sites, network routes to the closest) to cut the speed-of-
light RTT and absorb DDoS. It can also do **health-checked failover** (stop returning a dead region).
**The limitation is TTL**: clients cache the resolved IP until TTL expires, so DNS-level failover is
**slow** (seconds-to-minutes) and not precise — which is why fine-grained failover happens at the
**load balancer** layer, and DNS is used for coarse, geographic routing.

---

### Q21. You're designing an LLM chat UI that streams tokens as they're generated. What transport?
**A.** **SSE** is the natural fit: it's a one-way server→client stream over plain HTTP, simple, with
auto-reconnect — exactly matching "server emits tokens, client displays them." I don't need full
duplex on the same channel (the user's prompt is a normal POST that *starts* the stream), so
WebSockets would be over-engineering. This is precisely the pattern in m31 (and how the InsightDesk
`/ask/stream` endpoint worked) — and the senior detail is that streaming improves **perceived latency
(TTFT)** even when total time is unchanged.

---

### Q22. Two internal microservices are chatty and latency-sensitive. They currently use REST/JSON. What would you change and what's the trade-off?
**A.** I'd consider moving the hot path to **gRPC**: binary Protobuf (smaller/faster to (de)serialize
than JSON), HTTP/2 multiplexing (no per-call connection setup, many concurrent streams), a **strict
schema** (fewer integration bugs), and streaming if needed. The **trade-offs**: it's not
human-readable (harder ad-hoc debugging), needs codegen/tooling and schema discipline, loses native
HTTP caching, and adds a migration cost. So I'd do it only where the latency/throughput win is real —
which is exactly the judgment call I'd make on Questimate's inter-service REST calls if they became a
bottleneck.
