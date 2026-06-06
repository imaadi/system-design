# m07 — Exercises (Load Balancing & Proxies)

> Do these on paper / out loud before checking `solutions/m07_load_balancing_proxies/`. Goal: choose
> L4 vs L7, pick algorithms, design health checks/deploys, and make the LB tier HA.

---

## Exercise 1 — L4 or L7?
For each, choose **L4** or **L7** and justify in one line:
1. Routing `/api/*` to the API service and `/static/*` to the static service.
2. Balancing raw TCP connections to a Postgres read-replica pool.
3. Terminating HTTPS and adding auth/rate-limiting at the edge.
4. Maximum-throughput balancing of a custom binary protocol over TCP.
5. Canary: send 5% of HTTP traffic to v2 based on a header.

---

## Exercise 2 — Pick the algorithm
For each backend pool, choose a load-balancing algorithm and justify:
1. Stateless API servers, all identical hardware, short uniform requests.
2. A mix of old (2-core) and new (8-core) servers.
3. Long-lived WebSocket connections of very different durations.
4. A cache-server pool where you want the same key served by the same node (warm cache).
5. 50 independent edge load balancers that can't share global load state.

---

## Exercise 3 — The cold-server problem
You run **least connections**. You add a fresh, empty server to a busy pool and it immediately gets
overwhelmed.
1. Explain exactly why least-connections did this.
2. Give two fixes.
3. How does a readiness check help here, separately from the algorithm fix?

---

## Exercise 4 — Health checks & deploys
1. Define liveness vs readiness, and the action taken when each fails.
2. During startup, your service needs 20s to load ML models into memory before it can serve. Which
   check protects it, and what goes wrong if you only have a liveness check?
3. Describe what must happen during a rolling deploy so that **zero** in-flight requests are dropped.

---

## Exercise 5 — Sticky sessions
A teammate proposes sticky sessions so each user's in-memory session stays on one server.
1. Name three problems this introduces.
2. What's the better design, and why does it remove all three problems?
3. Name the one case where you *do* accept per-server affinity, and what you add to keep it correct.

---

## Exercise 6 — Make the LB highly available
Your single nginx LB is now a SPOF.
1. Describe an **active-passive** setup and how failover happens.
2. Describe an **active-active** setup and what sits *above* the LBs to distribute to them.
3. "Who balances the balancer?" — what's the coordination-free layer at the top, and why does the
   regress stop there?

---

## Exercise 7 — Scaling the L7 LB itself
Your L7 LB is CPU-bound on TLS termination + HTTP parsing at peak.
1. What topology lets you scale the L7 layer horizontally? (Hint: a cheaper LB in front.)
2. Name two ways to reduce the per-request cost on the L7 LB.
3. Why is L4 a good choice for the front layer here?

---

## Exercise 8 — gRPC / long-lived connection gotcha
You put an L4 load balancer in front of a gRPC service and notice traffic is very unevenly distributed.
1. Why does L4 balancing fail for gRPC?
2. What kind of balancing do you actually need, and where can it live?
3. How is this the same underlying issue as balancing WebSockets?

---

## Exercise 9 — The full chain
Trace the load-balancing tiers a request passes through, from a user in Tokyo clicking your global app
to a database read in your US region. Name the mechanism at each tier and what it balances across.

---

## Exercise 10 — Apply to your own systems 🏭
1. **Questimate (nginx → uWSGI → Django):** what layer/type of LB is nginx here, what does it terminate/
   distribute, and what makes the workers safe to balance across?
2. **DCE on Kubernetes:** explain the LB role of a **Service** vs an **Ingress** (L4 vs L7), and how
   **readiness probes** prevent traffic hitting a pod whose models aren't loaded yet.
3. **Zero-downtime deploys on OpenShift/EKS:** which two LB features make a rolling update drop zero
   requests?
4. If DCE's services became chatty east-west, what would a **service mesh** give you, and what's the
   cost?

---

When done, check `solutions/m07_load_balancing_proxies/README.md`, then say **"Module 8"**.
