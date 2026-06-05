# Module 3 — Networking & Communication ⭐

> Every system is boxes connected by **arrows**, and the arrows are network calls. If you don't
> understand what an arrow *costs* and *guarantees*, your diagrams are fiction. This module makes the
> arrows real: how a request actually travels (DNS → TCP → TLS → HTTP), why each hop costs what it
> costs, and how to choose *how* two components talk (REST vs gRPC vs GraphQL; polling vs WebSockets
> vs SSE; sync vs async). The recurring theme: **network round trips are expensive and the speed of
> light is non-negotiable — good design hides round trips.**

Read in two passes: **§1–§5** (the request's journey: DNS, TCP/UDP, TLS, HTTP evolution) and
**§6–§9** (HTTP semantics, API styles, real-time/push patterns, gateways).

---

## 1. The life of a request (the spine that ties it all together) ⭐

When you type `https://api.questimate.in/portfolio` and hit enter, here's the *actual* sequence —
and the latency each step adds. Internalize this; every later section is a zoom-in on one step.

```
  ┌──────────┐  1. DNS lookup        ┌──────────┐  "api.questimate.in → 203.0.113.7"
  │ client   │ ────────────────────▶ │   DNS    │   (cached? ~0 ms. cache miss? up to ~20–120 ms)
  └────┬─────┘                       └──────────┘
       │ 2. TCP handshake (SYN, SYN-ACK, ACK)         ← 1 round trip (RTT)
       │ 3. TLS handshake (certs, keys)               ← 1 RTT (TLS 1.3) or 2 (TLS 1.2)
       │ 4. HTTP request  ──────────────────────────▶ ← travels to server
       │ 5. server work (auth, DB, render)            ← your service time
       │ 6. HTTP response ◀──────────────────────────
       ▼
   render / use
```

**The crucial insight — count the round trips.** Before a single byte of *your* data moves, a
fresh HTTPS connection costs roughly **DNS (maybe) + 1 RTT (TCP) + 1 RTT (TLS 1.3) ≈ 2–3 RTTs of
pure setup**. Within a datacenter an RTT is ~0.5 ms (setup ≈ negligible). **Cross-continent an RTT
is ~150 ms** (m02), so a cold HTTPS connection to another continent can burn **~300–450 ms before
your API even starts working.** That single fact justifies a huge amount of design:

- **Connection reuse / keep-alive & connection pools** → amortize the handshake over many requests
  (pay setup once, not per call). *This is why HTTP clients pool connections and why microservices
  hold persistent gRPC channels.*
- **CDNs / edge termination** → terminate TLS close to the user so the handshake RTT is small (m06).
- **TLS 1.3 + session resumption / 0-RTT** → fewer handshake round trips.
- **Keep chatty services in the same region/AZ** → don't pay 150 ms per hop on the critical path.

> **Carry this:** *latency is dominated by round trips, not bandwidth, for small requests.* "Make it
> faster" usually means "make fewer round trips" (batch, cache, colocate, reuse connections) far more
> often than "send fewer bytes."

---

## 2. DNS — turning names into addresses

**DNS (Domain Name System)** is the internet's phone book: it resolves a human name
(`api.questimate.in`) into an IP address (`203.0.113.7`). It's a **distributed, hierarchical,
heavily-cached** system.

**Resolution path (on a cache miss):**
```
  browser cache → OS cache → recursive resolver (your ISP / 8.8.8.8)
       → root servers (.)      → "ask the .in TLD servers"
       → TLD servers (.in)     → "ask questimate.in's authoritative server"
       → authoritative server  → "questimate.in = 203.0.113.7"
```
Each level is **cached** with a **TTL** (time-to-live). A popular domain is almost always served
from a nearby cache in ~0 ms; only the rare miss walks the hierarchy.

**Record types worth knowing:** `A` (name→IPv4), `AAAA` (→IPv6), `CNAME` (alias→another name),
`MX` (mail), `TXT` (verification/SPF), `NS` (delegation).

**DNS is also a load-balancing and availability tool** (this is the system-design relevance):
- **Round-robin DNS:** return multiple A records → clients spread across IPs (crude LB).
- **GeoDNS / latency-based routing:** return the IP of the *nearest* datacenter → cuts that
  speed-of-light RTT. (How CDNs and multi-region apps route you to the closest edge.)
- **Anycast:** *the same IP* announced from many locations; the network routes you to the topologically
  nearest one. Used by DNS providers and CDNs for both proximity and DDoS resilience.
- **Failover:** health-checked DNS can stop returning a dead region's IP (but **TTL is the catch** —
  clients cache the old answer until TTL expires, so DNS failover is *slow*; seconds-to-minutes, not
  instant. That's why DNS is a coarse availability tool, and real failover happens at the LB layer).

> **Think differently — TTL is a consistency knob.** Low TTL = fast failover/agility but more DNS
> queries (load + latency on misses). High TTL = fewer lookups but stale routing for longer. It's the
> same caching trade-off as everywhere (m06): freshness vs cost. DNS is just an eventually-consistent
> cache of name→IP.

---

## 3. The transport layer: TCP vs UDP ⭐

IP delivers packets between hosts but promises nothing (packets can drop, reorder, duplicate). On
top of IP sit two transports with opposite philosophies:

| | **TCP** | **UDP** |
|---|---|---|
| Guarantees | **Reliable, ordered, error-checked** byte stream | **Best-effort** datagrams — may drop, reorder, duplicate |
| Connection | Connection-oriented (3-way handshake first) | Connectionless (just send) |
| Overhead | Higher (handshake, ACKs, retransmits, congestion control) | Minimal (fire and forget) |
| Flow/congestion control | Yes (adapts to network capacity) | No (you build it if you need it) |
| Use when | Correctness > a little latency: web (HTTP), DBs, APIs, file transfer | Latency/throughput > perfection: video/voice (VoIP), live streaming, gaming, DNS queries, metrics |

**TCP 3-way handshake** (the 1 RTT of §1): `SYN → SYN-ACK → ACK`, then data flows. TCP then guarantees
the bytes arrive **in order and complete**, retransmitting losses and slowing down under congestion.

**Why UDP exists:** for **real-time** data, a *late* packet is worse than a *lost* one. In a video
call, you don't want TCP to stall the whole stream re-sending a packet from 2 seconds ago — you'd
rather drop it and keep going. So real-time media uses UDP (often with app-level recovery). DNS uses
UDP because a query/response is tiny and a dropped query is cheaply retried.

> **Think differently — Head-of-Line (HOL) blocking, the villain of this module.** Because TCP
> guarantees *order*, **one lost packet stalls every byte behind it** until it's retransmitted — even
> bytes that already arrived. This "head-of-line blocking" haunts every layer built on TCP (you'll see
> it bite HTTP/1.1 and HTTP/2 in §5). It's *the* reason HTTP/3 abandoned TCP for UDP-based QUIC. Keep
> this concept in your pocket — interviewers love it.

---

## 4. TLS / HTTPS (the "S", and its latency cost)

**TLS (Transport Layer Security)** encrypts the connection (the `S` in HTTPS). It gives:
**confidentiality** (eavesdroppers can't read it), **integrity** (tamper-evident), and
**authentication** (the cert proves you're really talking to questimate.in). It runs *after* the TCP
handshake and *before* HTTP.

Design-relevant facts (you don't need the crypto, you need the *costs*):
- **It adds round trips.** TLS 1.2 ≈ 2 RTTs; **TLS 1.3 ≈ 1 RTT** (and **0-RTT** resumption for
  repeat visitors). This is why §1's "hide round trips" matters and why TLS 1.3 was a big deal.
- **Termination location matters.** Terminating TLS at a **CDN edge / load balancer** near the user
  keeps the expensive handshake RTT small, then uses a warm internal connection to your origin.
- **mTLS (mutual TLS):** both sides present certs — common for **service-to-service** auth inside a
  zero-trust network / service mesh (m07).

---

## 5. HTTP and its evolution (1.1 → 2 → 3) ⭐

**HTTP** is the request/response application protocol of the web. Its versions are a 25-year war
against **latency and head-of-line blocking** — a great story to tell:

- **HTTP/1.0:** one request per TCP connection → handshake for *every* request. Brutal.
- **HTTP/1.1:** added **keep-alive** (reuse the connection for many requests) and tried
  **pipelining** (send requests without waiting). But responses must come back **in order** →
  **application-layer HOL blocking**: a slow response blocks the ones behind it. Browsers worked
  around it by opening **~6 parallel connections per domain** (and "domain sharding" hacks).
- **HTTP/2:** **multiplexing** — many concurrent requests (**streams**) interleaved over **one** TCP
  connection, plus **binary framing** and **header compression (HPACK)** (and **server push**, which
  turned out a flop and is now largely deprecated). Solves *application-layer* HOL blocking. **But** it
  still rides one TCP connection, so a single lost packet triggers **TCP-layer HOL blocking** that
  stalls *all* streams. (The villain returns one layer down.)
- **HTTP/3:** runs over **QUIC**, which is built on **UDP**. QUIC implements its own streams,
  reliability, and *per-stream* ordering, so a lost packet only stalls *its own* stream — **no
  cross-stream HOL blocking**. It also folds the transport + TLS 1.3 handshake into ~**1 RTT (or
  0-RTT)** and supports **connection migration** (your phone switching Wi-Fi→cellular keeps the
  connection). This is the current state of the art for web/edge traffic.

```
  HTTP/1.1: [conn]──req1──resp1──req2──resp2   (serial-ish; HOL at app layer; ~6 conns to fake parallelism)
  HTTP/2 :  [1 TCP conn] streams 1,2,3 interleaved  (app HOL solved, but 1 lost packet stalls ALL → TCP HOL)
  HTTP/3 :  [1 QUIC/UDP conn] independent streams    (lost packet stalls only its stream → HOL solved)
```

> **Interview-gold takeaway:** "HTTP/2 fixed head-of-line blocking at the application layer but not at
> the transport layer; HTTP/3 fixed it by replacing TCP with QUIC over UDP." Saying that sentence
> signals you understand the *why*, not just the version numbers.

---

## 6. HTTP semantics: methods, status codes, idempotency ⭐

The protocol's *vocabulary* — and two properties that matter enormously for distributed systems.

**Methods** and their two key properties:
| Method | Purpose | **Safe?** (no side effects) | **Idempotent?** (N calls == 1 call) |
|---|---|---|---|
| GET | read | ✅ | ✅ |
| HEAD | read headers only | ✅ | ✅ |
| PUT | create/replace at a known URI | ❌ | ✅ |
| DELETE | remove | ❌ | ✅ |
| POST | create / "do something" | ❌ | ❌ (not by default) |
| PATCH | partial update | ❌ | ❌ (usually) |

- **Safe** = read-only, no state change (cacheable, prefetchable).
- **Idempotent** = doing it **N times has the same effect as once.** ⭐ *This is one of the most
  important concepts in all of distributed systems*, because **networks fail and clients retry.** If a
  request times out, did it succeed? You don't know — so you retry. If the operation is idempotent,
  a retry is harmless. If not (a naive `POST /charge`), a retry could **double-charge.**

> **Think differently — the idempotency key.** To make a non-idempotent operation safe to retry,
> the client sends a unique **idempotency key** (e.g. `Idempotency-Key: <uuid>`); the server records
> "I already processed this key" and returns the *original* result on a retry instead of doing it
> again. This is exactly how Stripe makes payments retry-safe — and it's the bridge to message-queue
> "exactly-once" semantics in m08 and m21. *Any* design with money or external side effects needs this.

**Status codes** (know the families + the ones that signal design maturity):
- **2xx success:** 200 OK, 201 Created, 202 Accepted (async — "I queued it"), 204 No Content.
- **3xx redirection:** **301** (permanent — cacheable, used by URL shorteners) vs **302/307**
  (temporary). 304 Not Modified (caching).
- **4xx client error:** 400 Bad Request, **401** Unauthorized (not authenticated) vs **403**
  Forbidden (authenticated but not allowed), 404 Not Found, **409** Conflict, **422** Unprocessable,
  **429 Too Many Requests** (rate limited — m12).
- **5xx server error:** 500 Internal, **502** Bad Gateway, **503** Service Unavailable (overloaded/
  down — clients should back off), **504** Gateway Timeout.

> Knowing **401 vs 403**, **429** (rate limiting), **503 + Retry-After** (backpressure), and **202**
> (async accepted) shows you think about real failure/flow-control, not just the happy path.

**HTTP is stateless** (m02): each request is independent; the server keeps no client context between
requests (state lives in a token/cookie + a shared store like Redis). *This* is what lets you put any
request on any server behind a load balancer.

---

## 7. API styles: REST vs gRPC vs GraphQL ⭐⭐

How do two services expose/consume functionality? Three dominant styles — and the answer is **"it
depends on who's calling and the access pattern."**

### REST (Representational State Transfer)
Resources (`/users/123`) manipulated with HTTP verbs, usually **JSON over HTTP/1.1 or 2**.
- **Pros:** ubiquitous, human-readable, cacheable (uses HTTP caching natively), great tooling,
  language-agnostic, easy to debug (curl). The default for **public APIs**.
- **Cons:** **over-/under-fetching** (an endpoint returns a fixed shape — you get fields you don't
  need or must call N endpoints to assemble a screen → the "N+1 / chatty" problem); no strict schema
  by default; verbose (JSON text).

### gRPC (Google RPC)
**Binary** **Protocol Buffers** over **HTTP/2**; you call methods like local functions defined in a
`.proto` schema.
- **Pros:** **fast & compact** (binary, HTTP/2 multiplexing), **strict typed schema** (`.proto`) with
  codegen in many languages, first-class **streaming** (client/server/bidirectional), great for
  **internal east-west service-to-service** comms at scale.
- **Cons:** not human-readable, limited browser support (needs grpc-web proxy), steeper tooling,
  harder to debug by hand, HTTP-caching doesn't apply natively.

### GraphQL
A **query language**: the *client* specifies exactly which fields it wants in one request to a single
endpoint; the server resolves them.
- **Pros:** **kills over-/under-fetching** (ask for exactly what you need), one round trip to assemble
  a complex screen, strongly typed schema, great for **mobile / many-client front-ends** with varied
  data needs (a **BFF**-style aggregator).
- **Cons:** **caching is hard** (it's mostly POSTs to one URL — loses HTTP caching), **server
  complexity** (resolvers, the **N+1 resolver problem** → needs DataLoader/batching), easy to write
  **expensive queries** (need depth/complexity limits), harder rate-limiting.

| Dimension | REST | gRPC | GraphQL |
|---|---|---|---|
| Payload | JSON (text) | Protobuf (binary) | JSON (text) |
| Transport | HTTP/1.1/2 | HTTP/2 | HTTP (usually POST) |
| Schema/contract | optional (OpenAPI) | **strict (.proto)** | **strict (SDL)** |
| Fetching | fixed per endpoint | fixed per method | **client-specified** |
| Streaming | limited (SSE/WS) | **native bidi** | subscriptions |
| Caching | **native (HTTP)** | manual | hard |
| Best for | public APIs, simple CRUD | **internal microservices, low-latency, streaming** | **flexible front-end / aggregation BFF** |

> **The senior decision rule:** *Public-facing, broad ecosystem, cacheable → **REST**. Internal
> service-to-service, performance-critical, streaming → **gRPC**. A front-end that needs to assemble
> varied, nested data in one shot across many clients → **GraphQL** (often as a BFF over internal gRPC/
> REST services).* It's common to use **all three in one system**: GraphQL/REST at the edge for clients,
> gRPC between services internally.

---

## 8. Real-time & "how does the server push?" — polling, SSE, WebSockets, webhooks ⭐⭐

HTTP is **client-initiated** (client asks, server answers). But many systems need the *server* to
push data (chat messages, notifications, live prices, an LLM token stream). Here's the ladder of
techniques, weakest to strongest — a classic interview comparison (and directly relevant to chat
(m14), notifications (m15), and LLM streaming (m31)):

| Technique | How it works | Direction | Cost / when |
|---|---|---|---|
| **Short polling** | client asks "anything new?" every N seconds | client→server, repeated | simplest; wasteful (mostly empty), latency = poll interval. Fine for low-freq, low-stakes. |
| **Long polling** | client asks; server **holds** the request open until there's data (or timeout), then client re-asks | server→client (faked) | near-real-time without new tech; ties up a connection/thread per client; the classic pre-WebSocket trick. |
| **SSE (Server-Sent Events)** | one long-lived HTTP response the server streams events down | **server→client only** (one-way) | simple, auto-reconnect, works over plain HTTP; perfect for **feeds, notifications, and LLM token streaming**. Can't send client→server on the same channel. |
| **WebSockets** | one TCP connection **upgraded** to a persistent, **full-duplex** channel | **bidirectional** | true real-time both ways; best for **chat, multiplayer, collaborative editing**. Costs a persistent connection per client (stateful — see below). |
| **Webhooks** | *server-to-server* push: you register a URL, the provider POSTs you events | server→your-server | how Stripe/GitHub notify you of events; needs a public endpoint + signature verification + retries. |

```
 short poll:   C ─?─> S  (empty) ... C ─?─> S (empty) ... C ─?─> S (data!)   wasteful
 long poll:    C ─?─────────────────hold──────────> S (data!) then re-ask    near-real-time
 SSE:          C ───> S ════events════════════════> (one-way stream)         feeds, LLM tokens
 WebSocket:    C <════════ full-duplex persistent ════════> S                 chat, live both-ways
```

> **Think differently — persistent connections break statelessness.** A WebSocket/SSE connection is
> **stateful**: that user is "pinned" to one specific server for the life of the connection. This
> wrecks the easy "any server handles any request" model (m02) and forces design choices you'll face
> in m14: a **connection-manager/gateway layer** that tracks which server holds which user's socket,
> a **pub/sub backplane** (e.g. Redis) so a message produced on server A reaches a user connected to
> server B, and careful handling of **reconnection, scaling, and the OS limit of ~tens-of-thousands of
> connections per box.** "Just use WebSockets" hides a real distributed-systems cost — name it.

> **Latency-vs-resource trade:** polling wastes requests but is stateless and trivial to scale;
> persistent connections give real-time but cost memory + statefulness. Pick by *how real-time* and
> *how bidirectional* the requirement truly is — don't default to WebSockets for a notification badge
> (SSE or even long-poll is enough).

---

## 9. The edge: API Gateway, reverse proxy & BFF

In front of your services sits an **edge layer** that the public hits first. Don't conflate the pieces:

- **Reverse proxy** (e.g. **nginx**): sits in front of servers, forwards client requests to them.
  Does TLS termination, **load balancing** (m07), compression, static-file serving, basic routing.
  *(Forward proxy = sits in front of clients, for the client's benefit — the opposite direction.)*
- **API Gateway:** a smarter reverse proxy specialized for APIs. **Single entry point** that handles
  cross-cutting concerns so your services don't each reimplement them:
  - **Routing** (path/host → the right service)
  - **AuthN/AuthZ** (validate the JWT/API key once, at the edge)
  - **Rate limiting & quotas** (m12) · **request/response transformation** · **API versioning**
  - **Aggregation/composition** (fan out to several services, combine) · **caching** · **observability**
    (central logging/metrics/tracing) · **TLS termination** · **request validation**
- **BFF (Backend-for-Frontend):** a gateway/aggregation layer **tailored to one client type** (a
  web BFF, a mobile BFF) so each front-end gets exactly the shape it needs — often where GraphQL lives.

```
            ┌────────────────────── API GATEWAY ──────────────────────┐
  clients ─▶│ TLS · authN/Z · rate-limit · routing · aggregate · log   │
            └───┬──────────────┬──────────────┬────────────────────────┘
                ▼ (east-west, internal: gRPC/REST + mTLS)
           [service A]     [service B]     [service C]
```

**North-south vs east-west traffic** (vocabulary worth using):
- **North-south** = traffic in/out of the system (client ↔ edge). Public, untrusted, needs the
  gateway's auth/rate-limit/TLS. Usually **REST/GraphQL over HTTP**.
- **East-west** = traffic *between* internal services. Trusted-ish, high-volume, latency-sensitive →
  often **gRPC**, secured with **mTLS**, managed by a **service mesh** (m07).

> **Watch the trap:** the API gateway is a powerful single entry point — and therefore a potential
> **single point of failure and bottleneck.** You run it **redundantly** (multiple instances behind an
> LB) and keep it **thin** (don't put business logic in it). It's plumbing, not your app.

---

## 10. From your systems 🏭
- **REST data mesh (east-west):** Questimate's **4 microservices talk over REST with env-injected
  base URLs** — `qsocial → risk_factor`, `→ daily_return`, `→ stock_sentiment`. That's textbook
  east-west service-to-service comms. Great interview line: *"we used REST between services for
  simplicity and language-agnostic debuggability; if latency/throughput had been critical I'd have
  moved the hot internal paths to gRPC for binary payloads + HTTP/2 multiplexing."* You can argue the
  trade-off from real experience.
- **Reverse proxy + stateless app:** Questimate runs **nginx → uWSGI** with TLS at nginx and stateless
  Django workers behind it — exactly §9's reverse-proxy + §6's statelessness, with **JWT bearer auth**
  validated at the edge.
- **Async over sync (the push problem, server-side):** DCE ingests claims via **Kafka** rather than
  synchronous request/response — an **event-driven** boundary (m08) chosen because claim processing is
  heavy and bursty; you don't want a synchronous caller blocked for seconds. Contrast that with a
  sync REST call and you've shown you know *when not to use request/response.*
- **Real-time push:** Questimate's live trading (orders/holdings/positions updating) is a natural
  **WebSocket/SSE-or-polling** decision — a perfect place to discuss §8's trade-offs (you likely used
  polling/long-poll against broker APIs; you can reason about when WebSockets would've been worth the
  statefulness cost).
- **Idempotency:** placing the *same* buy order twice because of a network retry is a real
  double-execution hazard — exactly the §6 **idempotency-key** problem, and a strong story to tell.

---

## 11. Key concepts (interview-ready)
- **Latency is round-trips, not bytes** (for small requests). A cold HTTPS connection ≈ 2–3 RTTs of
  setup; cross-continent RTT ≈ 150 ms. Hide round trips: **keep-alive/pooling, CDNs/edge TLS,
  colocation, batching.**
- **DNS** = distributed, cached (TTL) name→IP; also a coarse **LB/failover** tool (GeoDNS, anycast),
  but TTL makes DNS failover slow. TTL is a freshness-vs-cost knob.
- **TCP** = reliable/ordered (web, DBs); **UDP** = best-effort/low-latency (media, DNS, gaming).
- **Head-of-line blocking:** TCP's ordering means one lost packet stalls everything behind it.
  **HTTP/1.1** had app-layer HOL; **HTTP/2** fixed app-layer (multiplexing) but not transport-layer;
  **HTTP/3 (QUIC over UDP)** fixed it with independent streams.
- **Idempotency** (N calls == 1) makes retries safe — *the* property for anything with side effects;
  add an **idempotency key** to make POST/charge retry-safe.
- **Status codes that signal maturity:** 401 vs 403, 429 (+Retry-After), 503, 202 (async accepted), 301.
- **REST vs gRPC vs GraphQL:** public/cacheable → REST; internal/fast/streaming → gRPC; flexible
  client fetching/aggregation → GraphQL. Often all three in one system (gRPC east-west, REST/GraphQL
  north-south).
- **Push patterns:** short-poll → long-poll → **SSE (one-way, great for feeds & LLM token streams)** →
  **WebSockets (bidirectional, for chat)** → webhooks (server-to-server). Persistent connections are
  **stateful** → need a connection layer + pub/sub backplane (m14).
- **API Gateway** centralizes auth/rate-limit/routing/TLS/aggregation at a **single (redundant) entry
  point**; keep it thin. **North-south** (client↔edge) vs **east-west** (service↔service).

---

## 12. Go deeper (the well-researched reading list)
- **High Performance Browser Networking (Ilya Grigorik)** — the definitive, free book on TCP/TLS/HTTP/2
  /UDP and *why round trips dominate*. The source for §1–§5.
- **HTTP/2 (RFC 9113) & HTTP/3 / QUIC (RFC 9000/9114)** — skim the motivations (HOL blocking).
- **Stripe's idempotency-keys docs** — the canonical real-world §6 pattern.
- **gRPC docs (concepts + streaming)** and **the GraphQL spec / "GraphQL vs REST" + the N+1/DataLoader
  problem** — for §7.
- **MDN HTTP reference** — methods, status codes, caching headers (authoritative + concise).
- *(Tie-in)* revisit m02's latency ladder: every protocol choice here is about minimizing the
  expensive rungs (network RTT, cross-continent).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m03_networking_communication/`](../../solutions/m03_networking_communication/README.md).

When you've done the exercises, say **"Module 4"** to build *Databases & Storage* (SQL vs NoSQL,
indexing, B-tree vs LSM, transactions, ACID, isolation levels, schema design).
