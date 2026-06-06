# m07 — Cross-Questions ("if-and-buts") on Load Balancing & Proxies

> Answer out loud in 2–3 sentences before reading the model answer. These probe whether you understand
> *why* an LB choice matters operationally, not just that LBs exist.

---

### Q1. What does a load balancer actually give you, beyond "spreading traffic"?
**A.** Four things: **scalability** (spread load across a pool so you can add servers), **availability**
(health-check backends and route only to healthy ones → a dead server never reaches users),
**abstraction** (clients hit one stable endpoint while you add/remove/replace backends behind it →
zero-downtime deploys), and at L7, **cross-cutting work** (TLS termination, routing, compression, rate-
limiting). And it only works cleanly because the app tier is **stateless** — any server can serve any
request.

---

### Q2. L4 vs L7 load balancing — explain the difference and when you'd use each.
**A.** **L4** operates at the transport layer (TCP/UDP, IP+port) — it forwards connections **without
looking inside**, so it's **very fast** and protocol-agnostic but **blind** (no URL/header routing, no
TLS termination). **L7** operates at the application layer (HTTP) — it **terminates and parses** the
request, so it can route by **path/host/header/cookie**, terminate TLS, and do gateway concerns, but
it's **heavier** (more CPU per request). I use L7 at the edge for HTTP (content routing, TLS) and L4
for raw throughput or non-HTTP protocols — often L4 in front of many L7 instances.

---

### Q3. Why is L4 faster than L7?
**A.** L4 just forwards packets/connections by IP+port — minimal processing, no parsing, and it doesn't
terminate TLS (it passes encrypted bytes straight through). L7 must **terminate the TLS connection,
parse the HTTP request**, and make a content-based decision on **every request** — far more CPU and
latency per request. The trade is speed (L4) vs intelligence (L7: route by URL/header, retries,
rewrites, per-request balancing).

---

### Q4. Round robin vs least connections — when does round robin fail?
**A.** Round robin distributes by **count** (rotate evenly), which is fine for **uniform, short
requests on homogeneous servers**. It fails when requests have **uneven cost or duration** (e.g.
long-lived WebSockets, slow queries) or servers are **heterogeneous** — equal counts then mean very
**unequal load**, so some servers pile up while others idle. **Least connections** fixes this by routing
to the server with the **fewest active connections** (tracking real in-flight load), which is better for
long-lived/variable work.

---

### Q5. What's the "power of two choices" and why is it clever?
**A.** Instead of routing purely at random (cheap but can unluckily overload a server) or using least-
connections (great but needs **global state** about every server's load — hard with many LBs), **P2C**
picks **two backends at random and sends the request to the less-loaded one.** This simple trick yields
**near-optimal** load distribution — exponentially better than pure random — with **almost no
coordination**, so it scales when you have many independent load balancers that can't share perfect
global state. It's how modern proxies/schedulers approximate least-connections cheaply.

---

### Q6. When would you use IP hash / consistent hashing in a load balancer?
**A.** When you want **affinity** — the same client or key always routed to the same backend. Two uses:
**(1) cache locality** — route requests for the same key to the same server so its local cache stays
warm (much better hit ratio than spreading them, m06); **(2) crude session stickiness** without
cookies. I'd use **consistent hashing** (not plain `hash % N`) so that adding/removing a backend
remaps only ~K/N keys rather than reshuffling everything (m05).

---

### Q7. You add a fresh server to the pool and it immediately gets slammed and falls over. Why?
**A.** Classic **least-connections (or least-load) cold-server trap**: the new server has **zero active
connections**, so it looks "least loaded" and the LB routes a **flood** of new requests to it — before
its cache is warm or it's fully ready — overwhelming it. Fixes: **slow start / connection ramping**
(gradually increase the new server's share over a warm-up window) and a proper **readiness check** so
it only joins the pool when truly ready.

---

### Q8. Active vs passive health checks?
**A.** **Active:** the LB **proactively probes** each backend (e.g. `GET /health` every few seconds) —
fast, predictable detection; the standard. **Passive:** the LB **infers health from real traffic** — if
a backend starts returning errors/timeouts, mark it unhealthy — no extra probes but detection is
**reactive** (some real requests fail first). Many setups use both: active for steady detection, passive
to catch issues active probes miss. Tuning thresholds trades fast detection vs flapping.

---

### Q9. Liveness vs readiness checks — what's the difference and why does it matter?
**A.** **Liveness** asks "is the process alive/not deadlocked?" — if it fails, the orchestrator
**restarts** the container. **Readiness** asks "is it ready to serve traffic *right now*?" — if it
fails, the instance is **removed from the LB pool but not killed**. The distinction is critical at
**startup** (don't send traffic before models/caches finish loading — the m11 cold-start lesson) and
during **deploys/temporary overload** (drain traffic without restarting a healthy-but-busy process).
Confusing them causes restart loops or traffic to not-ready servers.

---

### Q10. What is connection draining and why does it matter for deploys?
**A.** **Draining** = when removing a server (deploy, scale-down), the LB **stops sending it new
requests** but **lets in-flight requests finish** before the server shuts down. Without it, every deploy
**drops live requests** (users see errors mid-request). With graceful shutdown + draining + readiness
gating, you get **zero-downtime rolling deploys** — swap backends behind the stable endpoint while
users notice nothing. It's a hallmark of having operated real production systems (you did, on
OpenShift/EKS).

---

### Q11. Are sticky sessions good or bad?
**A.** Mostly an **anti-pattern**, because they **reintroduce statefulness**: load becomes uneven (a few
heavy sticky users overload one server), **failover loses the session** (if that server dies the user's
state is gone), and you **can't freely scale/move** users. The better design is a **stateless app tier
+ shared session store (Redis)** so any server handles any request. I accept stickiness only when state
**must** be local — e.g. **WebSocket** connections, which are inherently pinned — and even then I add a
pub/sub backplane so correctness doesn't depend on stickiness.

---

### Q12. If everything flows through the load balancer, isn't it a single point of failure?
**A.** Yes — so you make the **LB itself redundant**. Options: **active-passive with a floating/virtual
IP** (keepalived/VRRP fails the VIP over to a standby), **active-active** with multiple LBs behind
**DNS round-robin or anycast** (the layer above spreads to them), or a **managed cloud LB** (ALB/NLB)
that's internally redundant and auto-scaled. The general principle: "who balances the balancer?" → a
**stateless, coordination-free layer (DNS/anycast)** sits above the LBs.

---

### Q13. How does traffic get routed to the *nearest* region in the first place?
**A.** **Global Server Load Balancing (GSLB)** — usually **DNS-based geo/latency routing** or
**anycast** (m03): DNS returns the IP of the closest/healthy region, or anycast announces one IP from
many locations and the network routes to the nearest. This is the **first** balancing tier (region/DC
selection) before the regional L7 LB distributes across app servers. Caveat: DNS-based GSLB is **TTL-
limited**, so failover between regions is coarse (seconds-to-minutes), which is why fine-grained
failover happens lower down.

---

### Q14. Walk me through the full chain of load balancing from a user's click to a DB read.
**A.** User → **DNS/anycast (GSLB)** picks the nearest region → regional **L7 LB** (nginx/ALB) terminates
TLS and routes the HTTP request to a healthy **app server** (round-robin/least-conn/P2C) → the app makes
**east-west** calls balanced by an **internal LB or service-mesh sidecar** → the data tier balances
**reads across DB replicas** (m05) and **cache shards** (m06). So "the load balancer" is really
balancing at four tiers, each with its own algorithm and health checks.

---

### Q15. What is a service mesh and what problem does it solve?
**A.** A service mesh moves **service-to-service (east-west) networking concerns** out of application
code and into the infrastructure via a **sidecar proxy (Envoy)** next to every service instance, managed
by a control plane (Istio/Linkerd). It provides, uniformly and language-agnostically: **client-side load
balancing, retries/timeouts/circuit breaking (m10), mTLS between services (m03), traffic shifting
(canary), and per-call observability** — so you don't reimplement all that in every service. The cost
is **operational complexity** and a **per-hop latency/resource overhead** (every call traverses two
proxies), so it's worth it at many-service scale, overkill for a handful.

---

### Q16. Reverse proxy vs forward proxy vs load balancer — disentangle them.
**A.** A **reverse proxy** sits in **front of servers**, receiving client requests and forwarding them
in (TLS termination, caching, routing) — e.g. nginx. A **load balancer is a reverse proxy that also
distributes across a pool** with health checks. A **forward proxy** sits in **front of clients**, acting
on their behalf for **outbound** traffic (egress filtering, anonymity) — the opposite direction. An
**API gateway** is a reverse proxy specialized for API cross-cutting concerns (auth, rate-limit,
aggregation); it often overlaps with the L7 LB.

---

### Q17. Should you terminate TLS at the load balancer or pass it through to backends?
**A.** Usually **terminate at the L7 LB/edge**: it offloads expensive crypto from app servers, lets the
LB route by HTTP content, and centralizes cert management — then it uses a warm internal connection (or
re-encrypts with mTLS) to backends. **TLS passthrough** (L4, encrypted bytes forwarded to the backend)
is used when the backend must terminate TLS itself (end-to-end encryption requirements, or the LB must
stay L4/blind). The common secure pattern: **terminate at the edge, re-encrypt internally with mTLS**
(m03/§8).

---

### Q18. How do load balancers enable zero-downtime deployments?
**A.** Because clients hit a **stable endpoint** while backends are swapped behind it: roll out new
instances, wait for their **readiness checks** to pass, add them to the pool, then **drain** old
instances (stop new traffic, finish in-flight) and remove them. Combined with health checks this is a
**rolling update** — users keep hitting healthy servers throughout. (Blue-green and canary are variants:
shift traffic between two pools, or send a small % to the new version first — often done at the L7 LB or
service mesh.)

---

### Q19. Your L7 LB is becoming a CPU bottleneck under TLS + parsing load. What do you do?
**A.** Scale the LB tier itself: run **multiple L7 LB instances** and put an **L4 LB (or DNS/anycast)
in front** to spread connections across them (L4 is cheap, so it scales the expensive L7 layer).
Also: **offload TLS** (terminate at a CDN/edge so the origin LB does less crypto, m06), enable **HTTP
keep-alive/connection reuse** (m03) to amortize handshakes, and right-size instances. The pattern is
"**L4 fan-out to many L7s**" plus pushing TLS/static handling to the edge.

---

### Q20. How does this connect to scaling WebSockets (from m03)?
**A.** WebSockets are **persistent, stateful** connections, so they're inherently **pinned** to one
server (a form of forced stickiness) — you can't freely rebalance an open connection. So you **L4-
balance the initial connection** across connection servers, accept the per-connection affinity, and use
a **pub/sub backplane** (Redis/Kafka) so a message produced on server A reaches a user connected to
server B. Health checks + reconnection handle a server dying (clients reconnect and get rebalanced).
This is the m14 chat design — load balancing of stateful connections is its own discipline.

---

### Q21. How would you load balance gRPC, given it's long-lived HTTP/2?
**A.** Naively L4-balancing gRPC is bad: gRPC multiplexes many requests over **one long-lived HTTP/2
connection**, so an L4 LB pins that connection to one backend and **all** requests stick there — no real
balancing. You need **L7 (request-level) balancing** that load-balances individual gRPC *streams/
requests* (Envoy/a gRPC-aware L7 LB or **client-side load balancing** via the mesh), or periodic
connection rebalancing. This is a classic gotcha: per-connection L4 balancing breaks down for any
multiplexed, long-lived protocol (HTTP/2, gRPC, WebSockets).

---

### Q22. How is a Kubernetes Service different from an Ingress, in LB terms?
**A.** A **Service** (ClusterIP/NodePort/LoadBalancer) is an **L4** construct — kube-proxy/IPVS load-
balances TCP/UDP connections across the healthy pods backing the service (by IP:port, no HTTP
awareness). An **Ingress** (backed by an ingress controller like nginx/Envoy) is an **L7** construct —
it routes **HTTP** by host/path to different services, terminates TLS, etc. So Service = internal L4
pod balancing; Ingress = external L7 HTTP routing into the cluster. You used both on DCE's k8s/
OpenShift.
