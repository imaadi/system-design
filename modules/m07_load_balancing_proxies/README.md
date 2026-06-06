# Module 7 — Load Balancing & Proxies ⭐⭐

> m02 said "scale out, not up." m05 added replicas. But replicas are useless until something **spreads
> traffic across them** and **routes around the dead ones** — that's the **load balancer**, the single
> most ubiquitous box in every architecture diagram. This module makes you fluent in **where** it sits,
> **L4 vs L7**, the **algorithms** (and which to use when), **health checks**, the **sticky-session
> anti-pattern**, how to stop the LB from being a **single point of failure**, and the **service mesh**
> for east-west traffic.

You've run this for real: **nginx in front of uWSGI Django workers** (Questimate) and **Kubernetes
Services/Ingress** (DCE). This turns that into architect-level reasoning.

---

## 1. Why a load balancer — what it actually does ⭐

A **load balancer (LB)** sits between clients and a pool of backend servers and **distributes incoming
requests across them.** It delivers four things at once:
1. **Scalability** — spread load so you can add servers and absorb more traffic (horizontal scaling,
   m02). Throughput scales with the pool.
2. **Availability** — **health-check** backends and route **only to healthy ones**; a dead server is
   removed automatically, so one failure doesn't reach users (m02 redundancy/failover).
3. **Abstraction** — clients hit **one stable endpoint** (the LB's VIP/DNS name); backends can be
   added, removed, or replaced behind it without clients noticing → enables zero-downtime deploys.
4. **Cross-cutting work** (for L7, §3): TLS termination, compression, routing, rate-limiting, sticky
   sessions.

```
                      ┌────────────────┐    health-checked pool
   clients ──────────▶│ LOAD BALANCER  │──▶ [ S1 ✅ ]
   (one VIP/DNS name) │ (distributes + │──▶ [ S2 ✅ ]
                      │  health-checks)│──✗  [ S3 ☠ ] ← removed automatically
                      └────────────────┘──▶ [ S4 ✅ ]
```

> **The enabler beneath it: statelessness (m02).** Load balancing only works cleanly because any
> server can handle any request — that's why you push session state to Redis and files to object
> storage. A **stateful** backend forces **sticky sessions** (§6), which fights the whole model.

---

## 2. Where load balancing happens (the whole path)

"The load balancer" is really **several layers** of distribution between user and backend:
- **DNS / Global Server Load Balancing (GSLB):** routes users to the nearest/healthy **region or
  datacenter** (GeoDNS, anycast — m03). Coarse, geographic, TTL-limited. The *first* balancing step.
- **Edge / L7 LB (the main one):** the application load balancer in a region (nginx, Envoy, HAProxy,
  AWS ALB) — distributes across app servers, terminates TLS, routes by path/host.
- **Internal / east-west LB:** balances traffic **between services** (service A → service B). Often a
  **service mesh** sidecar (§8) or an L4 internal LB (k8s Service).
- **Within the backend:** the DB layer balances reads across replicas (m05), the cache cluster across
  shards (m06).

> So when asked "where's the load balancer?", the senior answer is "at several tiers" — global (DNS/
> anycast) → regional edge (L7) → internal service-to-service (mesh) → data tier.

---

## 3. L4 vs L7 load balancing — the core distinction ⭐⭐

Where in the network stack does the LB make its decision? This is the fork everything hangs on.

**L4 (transport-layer) load balancing** — operates on **TCP/UDP** (IPs + ports). It forwards
**packets/connections** without looking inside them — it doesn't know it's HTTP, can't see URLs,
headers, or cookies.
- **Pros:** **very fast** (minimal processing, just forwards), low latency/overhead, protocol-agnostic
  (works for any TCP/UDP traffic — databases, gRPC, custom protocols), can handle huge throughput.
- **Cons:** **dumb** — can't route by URL/header, can't terminate TLS (it forwards encrypted bytes),
  can't do content-based decisions. Once a connection is pinned to a backend, it stays there.
- Examples: AWS **NLB**, L4 mode in HAProxy, k8s **Service (kube-proxy)**, IPVS.

**L7 (application-layer) load balancing** — operates on **HTTP/HTTPS** (and gRPC, etc.). It
**terminates the connection, parses the request**, and routes on its **content**.
- **Pros:** **smart** — route by **path** (`/api`→service A, `/img`→service B), **host**, **header**,
  or **cookie**; **TLS termination**; compression; **content-based** routing; can retry/rewrite;
  per-request load balancing (not just per-connection); a natural place for auth/rate-limiting (m03 API
  gateway).
- **Cons:** **slower/heavier** (must parse every request, terminate TLS — more CPU), protocol-specific.
- Examples: **nginx**, **Envoy**, HAProxy (L7 mode), AWS **ALB**, k8s **Ingress**.

```
  L4:  sees  [ TCP: src/dst IP:port ]              → forward the connection (fast, blind)
  L7:  sees  [ HTTP GET /api/users  Host: ...  Cookie: ... ] → route by content (smart, slower)
```

> **The senior framing:** *"L4 is fast but blind — it balances connections by IP/port; L7 is smart but
> heavier — it terminates TLS and routes by URL/header/cookie. I use L7 at the edge for HTTP (content
> routing, TLS, gateway concerns) and L4 where I need raw throughput or non-HTTP protocols."* A common
> combo: an **L4 LB in front of L7 LBs** (L4 spreads connections across many L7 instances for scale).

---

## 4. Load-balancing algorithms — and when each is right ⭐

How does the LB pick which backend gets the next request? The algorithm matters more than people think.

- **Round robin:** rotate through servers in order. Simple, even *count* distribution. **Bad when**
  servers are **heterogeneous** (different specs) or requests have **uneven cost / long-lived
  connections** — equal *counts* ≠ equal *load*.
- **Weighted round robin:** give bigger/faster servers a higher weight (more requests). Handles
  heterogeneous hardware.
- **Least connections:** send the next request to the server with the **fewest active connections**.
  **Better for long-lived or variable-duration requests** (WebSockets, slow queries) because it tracks
  actual in-flight load, not just count. Needs the LB to track connection state.
- **Least response time:** least connections + lowest latency — routes to the fastest-responding
  server. Even more adaptive; more state.
- **IP hash / consistent hashing:** hash the client IP (or a key) → always maps to the same backend.
  Gives **affinity** (a client/key sticks to one server) — useful for **cache locality** (route the
  same key to the same backend so its cache stays warm — m06) or crude session stickiness.
- **Random + power of two choices (P2C):** see the gem below.

> **Think differently — "Power of Two Choices" (P2C).** Pure **random** routing is stateless and
> cheap but can unluckily overload a server. **Least connections** is great but needs **global state**
> (the LB must know every server's load — hard with many LBs). **P2C** is the elegant middle: pick
> **two servers at random**, send the request to the **less loaded of the two.** This simple trick gets
> **near-optimal** load distribution (exponentially better than pure random — the "power of two
> choices" result) with almost no coordination, which is why it's used in modern LBs and schedulers
> (and why it scales when you have *many* load balancers that can't share perfect global state). Saying
> "I'd use P2C to approximate least-connections without global state" is a strong senior signal.

| Algorithm | Distributes by | Best for | Cost |
|---|---|---|---|
| Round robin | count | uniform requests, homogeneous servers | trivial |
| Weighted RR | weighted count | heterogeneous hardware | trivial |
| **Least connections** | active connections | **long-lived / variable** requests | tracks state |
| Least response time | conns + latency | latency-sensitive, adaptive | more state |
| IP/consistent hash | key affinity | **cache locality**, stickiness | rehash on resize (use consistent hashing) |
| **P2C (random-of-2)** | sampled load | **many LBs, no global state** | tiny |

> ⚠️ **The cold-server trap:** **least connections** can dump a flood onto a **freshly added** server
> (it has 0 connections, so it looks "least loaded") before its cache is warm or it's ready → it gets
> overwhelmed. Fixes: **slow start / connection ramping** (gradually increase a new server's share) and
> health/readiness checks (§5).

---

## 5. Health checks — how the LB knows who's alive ⭐

The availability magic depends on **health checks**: the LB continuously probes backends and routes
only to healthy ones.
- **Active health checks:** the LB **proactively** pings each backend (e.g. `GET /health` every few
  seconds). Fast detection; configurable. The standard.
- **Passive health checks:** the LB **observes real traffic** — if a backend returns errors/timeouts,
  mark it unhealthy. No extra probes, but detection is reactive (some requests fail first).
- **Tuning:** thresholds (N consecutive failures → out; M successes → back in) and intervals trade
  **fast detection vs flapping** — too sensitive removes healthy servers on a blip; too lax keeps
  routing to a dying one. (Same tension as failover detection in m05.)

> **Liveness vs readiness (k8s vocabulary worth using):** a **liveness** check asks "is the process
> alive?" (if not → restart it); a **readiness** check asks "is it ready to serve traffic right now?"
> (if not → remove from the LB pool but **don't** kill it). Crucial during **startup** (don't send
> traffic before models/caches load — your m11 cold-start lesson) and **deploys**. *DCE/k8s used exactly
> these.*

> **Graceful shutdown / connection draining:** when removing a server (deploy, scale-down), the LB
> should **stop sending new requests** but **let in-flight ones finish** (drain) before the server
> exits — otherwise you drop live requests on every deploy. A senior detail that shows you've operated
> real systems.

---

## 6. Sticky sessions (session affinity) — the trade-off ⭐

**Sticky sessions** pin a given client to a **specific backend** for their session (via a cookie or IP
hash), so all their requests go to the same server.
- **Why you'd want it:** if a server holds **per-user state in local memory** (a session, an in-process
  cache, a WebSocket connection — m03), the user must keep hitting that server for it to work.
- **Why it's an anti-pattern (mostly):** it **reintroduces statefulness** and breaks the clean model —
  load can become **uneven** (a few sticky heavy users overload one server), **failover loses the
  state** (if that server dies, the user's session is gone), and it **complicates scaling/deploys** (you
  can't freely move users).

> **The senior position:** *"I avoid sticky sessions for HTTP by keeping the app tier **stateless** and
> putting session state in a **shared store (Redis)** — then any server handles any request and I can
> scale/deploy freely. I only accept stickiness when state genuinely must be local — e.g. **WebSocket
> connections** (m03/m14), which are inherently pinned, and there I add a pub/sub backplane instead of
> relying on stickiness for correctness."* This is exactly why Questimate puts JWT/session in Redis.

---

## 7. The load balancer is a SPOF — make it HA ⭐ ("who balances the balancer?")

If all traffic flows through one LB and it dies, **everything** is down — the LB is the ultimate single
point of failure. You must make the LB itself redundant:
- **Active-passive with a floating/virtual IP (VIP):** two LBs; a **VIP** points at the active one; if
  it fails, the VIP **fails over** to the standby (via VRRP/keepalived). Simple; the standby is idle.
- **Active-active:** multiple LBs all serving, traffic spread across them by **DNS round-robin** or
  **anycast** (m03) — the layer *above* the LBs distributes to them. Better utilization + resilience.
- **Cloud LBs (ALB/NLB):** the provider runs the LB as a managed, internally-redundant, auto-scaling
  service — they've solved LB HA for you (this is a big reason to use managed LBs).

```
        DNS / anycast (spreads across LBs)
          ├──▶ LB-A (active) ──▶ backends
          └──▶ LB-B (active) ──▶ backends     ← lose one LB, the other carries on
```

> **The recursion insight:** balancing requires a balancer, which itself needs balancing — you break
> the regress with a layer that needs **no per-request coordination**: **DNS/anycast** (stateless,
> distributed) sits above the LBs. That's "who balances the balancer."

---

## 8. Service mesh — load balancing for east-west traffic

As §2 noted, you balance **north-south** (clients→edge) *and* **east-west** (service→service). At
microservice scale, doing retries/timeouts/mTLS/LB/observability in every service's code is repetitive
and inconsistent. A **service mesh** moves that into the infrastructure:
- A **sidecar proxy** (e.g. **Envoy**) is deployed **next to every service instance**; all of that
  service's network traffic flows through its sidecar.
- The mesh (sidecars = **data plane**, a **control plane** like Istio/Linkerd configuring them)
  provides, uniformly and without app code: **client-side load balancing** (incl. P2C/least-request),
  **retries/timeouts/circuit breaking** (m10), **mTLS** between services (m03), **traffic shifting**
  (canary/blue-green), and **observability** (metrics/traces for every call).

```
  service A ─▶ [Envoy sidecar] ═mTLS═▶ [Envoy sidecar] ─▶ service B
                (LB, retries, timeouts, metrics — no app code)
```

> **The trade-off:** a mesh gives you consistent, language-agnostic networking/observability/security
> for free in app code — at the cost of **operational complexity** and a **per-hop latency/resource**
> overhead (every call goes through two proxies). Worth it at many-service scale; overkill for a few
> services. (Relevant to your k8s/OpenShift world — Istio/Linkerd run here.)

---

## 9. Reverse proxy vs forward proxy (recap + placement)
- **Reverse proxy** (nginx, Envoy): sits in **front of servers**, receives client requests and forwards
  them in. A **load balancer is a specialized reverse proxy** that also distributes across a pool. Does
  TLS termination, caching (m06), compression, routing. *(Your Questimate nginx.)*
- **Forward proxy:** sits in **front of clients**, acting on the client's behalf for outbound traffic
  (corporate egress filtering, anonymity, caching outbound). Opposite direction.
- **API gateway** (m03): a reverse proxy specialized for **API cross-cutting concerns** (auth, rate
  limiting, aggregation). The gateway and the L7 LB often overlap / are co-located.

---

## 10. From your systems 🏭
- **nginx as L7 reverse proxy + LB, lived:** Questimate runs **nginx → uWSGI** with **stateless Django
  workers** behind it. nginx terminates TLS, load-balances across workers, and serves static files —
  textbook §1/§3/§9, enabled by §6 statelessness (JWT/session in Redis, not the worker).
- **Kubernetes load balancing:** DCE on **k8s/OpenShift** uses **Services** (an **L4** LB across pods
  via kube-proxy), **Ingress** (an **L7** LB/router for HTTP), and **liveness/readiness probes** (§5) —
  you've configured exactly the L4-vs-L7 + health-check concepts here. Readiness probes are why a pod
  doesn't get traffic until its models/caches are loaded (your m11 cold-start lesson).
- **Rolling deploys + draining:** k8s rolling updates rely on **readiness gating + connection draining**
  (§5) to ship with zero dropped requests — the ops reality behind "stable endpoint, swap backends"
  (§1). You did pod operations/deploys on OpenShift→EKS, so you've lived this.
- **Service mesh territory:** the OpenShift/EKS world is where Istio/Linkerd/Envoy (§8) live — a natural
  place to discuss east-west LB + mTLS + retries if your DCE services grew chatty.
- **GSLB during migration:** routing traffic between Azure(old) and AWS(new) during the cloud migration
  is a real §2 global-traffic-management story (DNS/region routing + parity validation).

---

## 11. Key concepts (interview-ready)
- **LB = distribute traffic + health-check + stable endpoint** → scalability + availability + zero-
  downtime deploys. It works because the app tier is **stateless** (state in Redis/object store).
- **Balancing happens at several tiers:** DNS/anycast (global/region) → L7 edge → east-west (mesh) →
  data tier.
- **L4 vs L7:** **L4** = transport (IP/port), fast/blind, any protocol, no TLS/content routing; **L7** =
  application (HTTP), smart (route by path/host/header/cookie, TLS termination, gateway concerns) but
  heavier. Common combo: L4 in front of many L7s.
- **Algorithms:** round robin (uniform), weighted RR (heterogeneous), **least connections** (long-lived/
  variable), least response time, **IP/consistent hash** (affinity/cache locality), **P2C** (near-
  optimal with no global state). Beware **least-conn dumping on a cold new server** → slow start.
- **Health checks** (active vs passive) make availability work; **liveness vs readiness** (restart vs
  remove-from-pool); **connection draining** for graceful deploys. Tune to avoid flapping.
- **Sticky sessions = anti-pattern** (re-adds state, uneven load, lost-on-failover) → prefer stateless +
  shared session store; accept stickiness only when state must be local (WebSockets).
- **The LB is a SPOF** → make it HA (active-passive + floating VIP, or active-active behind DNS/anycast,
  or a managed cloud LB). "Who balances the balancer?" → DNS/anycast above it.
- **Service mesh** (Envoy sidecars + control plane) = uniform east-west LB/retries/timeouts/mTLS/
  observability in infra, not app code — at a complexity + per-hop-latency cost.

---

## 12. Go deeper (the well-researched reading list)
- **"Introduction to modern network load balancing and proxying" — Matt Klein (Envoy creator)** — the
  best single article on L4 vs L7, topologies, and the whole space. Read this for §3/§8.
- **Mitzenmacher, "The Power of Two Choices in Randomized Load Balancing"** — the §4 P2C result at its
  source (and Marc Brooker's blog posts explain it accessibly).
- **NGINX & HAProxy docs on load-balancing algorithms + health checks** — concrete §4/§5.
- **Kubernetes docs: Services, Ingress, and liveness/readiness/startup probes** — your §10 in
  authoritative form.
- **Istio / Linkerd "what is a service mesh"** — for §8 (sidecar/data-plane/control-plane).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m07_load_balancing_proxies/`](../../solutions/m07_load_balancing_proxies/README.md).

When you've done the exercises, say **"Module 8"** to build *Messaging, Queues & Stream Processing*
(queues vs logs, RabbitMQ vs Kafka, delivery guarantees, idempotency, ordering, backpressure, outbox/
CDC) — how systems talk **asynchronously** (and where the outbox/dual-write thread from m04/m06 lands).
