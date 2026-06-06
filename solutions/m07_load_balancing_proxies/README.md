# m07 — Solutions (Load Balancing & Proxies)

> Check the *reasoning*. Recurring lessons: L4 = fast/blind, L7 = smart/heavy; match algorithm to
> request shape; the LB is a SPOF you must make HA; statelessness beats stickiness.

---

## Exercise 1 — L4 or L7?
1. **L7** — path-based routing needs to read the HTTP URL.
2. **L4** — raw TCP to a DB pool; no HTTP awareness needed, want speed.
3. **L7** — TLS termination + auth/rate-limit are application-layer (gateway) concerns.
4. **L4** — custom binary protocol, max throughput, no need to parse content.
5. **L7** — header-based traffic splitting requires reading the HTTP header.

---

## Exercise 2 — Pick the algorithm
1. **Round robin** — uniform requests + identical servers; equal counts ≈ equal load.
2. **Weighted round robin** — give the 8-core boxes higher weight than the 2-core ones.
3. **Least connections** — long-lived, variable-duration connections; balance by actual in-flight load,
   not count.
4. **Consistent hashing (IP/key hash)** — route the same key to the same node for cache locality (warm
   cache, better hit ratio); consistent hashing so resizing remaps only ~K/N.
5. **Power of two choices (P2C)** — near-optimal distribution with no global state, ideal for many
   independent LBs.

---

## Exercise 3 — The cold-server problem
1. **Why:** the new server has **0 active connections**, so under least-connections it looks "least
   loaded" and the LB routes a **disproportionate flood** of new requests to it — faster than it can
   warm its cache / handle, so it tips over.
2. **Fixes:** **slow start / connection ramping** (gradually raise the new server's traffic share over a
   warm-up window) and/or temporarily **cap** its concurrent connections until warm.
3. **Readiness check:** ensures the server only **joins the pool when it's actually ready** (caches
   warmed, dependencies up). It's complementary — readiness controls *when* it gets any traffic;
   slow-start controls *how fast* the traffic ramps once it's in.

---

## Exercise 4 — Health checks & deploys
1. **Liveness** = "is the process alive/not stuck?" → failure **restarts** the container. **Readiness**
   = "ready to serve now?" → failure **removes it from the LB pool** (but doesn't kill it).
2. **Readiness** protects startup (don't route traffic until the 20s model load finishes). With **only a
   liveness check**, the LB sends traffic immediately (errors during load) — or worse, if liveness
   fails during the slow load, it **restart-loops** forever and never becomes ready.
3. **Zero-drop rolling deploy:** bring up new instances → wait for their **readiness** to pass → add to
   pool → **drain** old instances (stop new traffic, let in-flight finish, honor graceful shutdown) →
   remove old. Health checks + draining + readiness gating = no dropped requests.

---

## Exercise 5 — Sticky sessions
1. **Problems:** (a) **uneven load** — heavy sticky users overload their server; (b) **lost state on
   failover** — if that server dies, the user's session is gone; (c) **scaling/deploy friction** — you
   can't freely move users, and a removed server strands its sessions.
2. **Better design:** **stateless app tier + shared session store (Redis)** — any server handles any
   request, so load balances evenly, a server death loses nothing (state is in Redis), and you scale/
   deploy freely. Removes all three problems.
3. **Accept affinity for:** **WebSocket** (and other persistent) connections, which are inherently
   pinned to one server. To keep it correct you add a **pub/sub backplane** (Redis/Kafka) so messages
   reach a user regardless of which server holds their socket — correctness doesn't depend on stickiness.

---

## Exercise 6 — Make the LB HA
1. **Active-passive:** two LBs, a **floating/virtual IP** points at the active one; **keepalived/VRRP**
   monitors it and **moves the VIP to the standby** on failure. Standby is idle until needed.
2. **Active-active:** multiple LBs all serving; **DNS round-robin or anycast** sits *above* them and
   spreads client traffic across the LBs. Better utilization + resilience.
3. **Who balances the balancer:** **DNS/anycast** — it's **stateless and coordination-free** (no
   per-request shared state), so it doesn't itself need a balancer; the regress stops at a layer that
   distributes without coordination (and is globally distributed/redundant by nature). Managed cloud LBs
   solve this for you internally.

---

## Exercise 7 — Scaling the L7 LB itself
1. **L4 in front of many L7s:** a cheap **L4 LB (or DNS/anycast)** spreads connections across multiple
   **L7 LB** instances, so the expensive L7 layer scales horizontally.
2. **Reduce per-request cost:** **offload TLS to a CDN/edge** so the origin L7 does less crypto; enable
   **HTTP keep-alive/connection reuse** so handshakes amortize; cache at the edge so fewer requests
   reach the L7 LB; right-size instances.
3. **L4 is a good front layer** because it's **cheap and fast** (just forwards by IP:port, no parsing/
   TLS) — so it can fan a huge connection volume out to the costly L7 tier without itself becoming the
   bottleneck.

---

## Exercise 8 — gRPC / long-lived connection gotcha
1. **Why L4 fails:** gRPC multiplexes many requests over **one long-lived HTTP/2 connection**; an L4 LB
   balances **connections**, so it pins that whole connection to a single backend and **all** the
   client's requests land there — no real per-request balancing → uneven load.
2. **Need request-level (L7) balancing:** an L7/gRPC-aware proxy (Envoy) or **client-side load
   balancing** (often via the service mesh) that distributes individual **streams/requests**; or
   periodic connection rebalancing.
3. **Same issue as WebSockets:** both are **persistent, multiplexed/long-lived** connections, so
   per-connection (L4) balancing can't spread the work — you must balance at the request level or pin
   the connection and balance *its establishment* + use a backplane.

---

## Exercise 9 — The full chain (Tokyo user → US DB read)
- **DNS / anycast (GSLB):** routes the user to the nearest/healthy **region** (e.g. an Asia edge, or US
  if that's where the app lives). Balances across **regions/DCs**.
- **Regional L7 LB** (nginx/ALB/Ingress): terminates TLS, routes the HTTP request across **app servers**
  (round-robin/least-conn/P2C), health-checked.
- **East-west LB / service mesh sidecar:** balances the app's **service-to-service** calls across
  **service instances**.
- **Data tier:** balances **reads across DB replicas** (m05) and requests across **cache shards** (m06).
Four tiers, each with its own mechanism and health checks.

---

## Exercise 10 — Apply to your own systems 🏭 (model)
1. **nginx (Questimate):** an **L7 reverse proxy + load balancer** — it **terminates TLS**, serves
   static files, and **distributes HTTP requests across the uWSGI Django workers**. The workers are safe
   to balance across because they're **stateless** (JWT/session in Redis, not worker memory), so any
   worker can serve any request.
2. **k8s Service vs Ingress (DCE):** a **Service** is **L4** — kube-proxy/IPVS load-balances TCP
   connections across the healthy pods (by IP:port, no HTTP awareness). An **Ingress** is **L7** —
   routes external **HTTP** by host/path to services, terminates TLS. **Readiness probes** keep a pod
   **out of the Service's endpoint pool until it passes** — so traffic never hits a pod whose ML models
   aren't loaded yet (no cold-start errors).
3. **Zero-downtime rolling updates:** **readiness gating** (new pods get traffic only when ready) +
   **connection draining / graceful shutdown** (old pods stop taking new requests but finish in-flight
   ones before terminating). Together → no dropped requests.
4. **Service mesh for chatty east-west:** uniform, language-agnostic **client-side load balancing,
   retries/timeouts/circuit breaking, mTLS between services, traffic shifting (canary), and per-call
   metrics/traces** — without changing app code. **Cost:** operational complexity (control plane +
   sidecars) and a **per-hop latency/resource overhead** (every call traverses two Envoy proxies); worth
   it at many-service scale, overkill for a few.
