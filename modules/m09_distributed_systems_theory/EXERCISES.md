# m09 — Exercises (Distributed Systems Theory)

> Do these on paper / out loud before checking `solutions/m09_distributed_systems_theory/`. Theory, but
> every answer should land on a **practical implication**.

---

## Exercise 1 — Spot the fallacy
For each design statement, name the distributed-computing fallacy it violates and the fix:
1. "We'll just call the inventory service synchronously; it'll respond instantly."
2. "No need to retry — the message will get there."
3. "We hard-coded the service IPs in config; they never change."
4. "Internal traffic doesn't need encryption."
5. "We'll send the whole 50 MB document on every request."

---

## Exercise 2 — Slow vs dead
Node A stops getting heartbeats from leader B.
1. List four things that could actually be true about B.
2. Why can't A know which one it is?
3. A's timeout is the deciding tool. Describe the failure if it's too short, and if it's too long.
4. How does this ambiguity force you to make your write operations idempotent?

---

## Exercise 3 — Clocks
1. Two servers each timestamp an event with their wall clock; you order the events by timestamp. Why
   might the order be wrong?
2. Why is wall-clock-based Last-Write-Wins dangerous for conflict resolution?
3. Which clock do you use to measure "how long did this operation take," and why?
4. What does a vector clock let you detect that a Lamport clock cannot?

---

## Exercise 4 — Consensus basics
1. State the three properties a consensus protocol must provide.
2. What does FLP impossibility say, and how do practical systems (e.g. Raft) get around it?
3. In Raft, why must a leader be elected by a **majority**, and how does that prevent split-brain?
4. A consensus cluster has 5 nodes. How many failures can it tolerate and still make progress? What
   about 4 nodes — why are odd sizes preferred?

---

## Exercise 5 — Where does consensus belong?
1. Your system needs: (a) to elect a primary, (b) to store the shard→node map, (c) to handle 500k
   writes/sec of user events. Which of these should go through consensus, and which absolutely should
   not? Why?
2. State the general principle this illustrates (connect it to "coordination is the tax on scale").

---

## Exercise 6 — Coordination services
1. Name three things you'd use etcd/ZooKeeper for.
2. Are coordination services CP or AP, and why is that the *right* choice for their job?
3. Why do you keep application data *out* of etcd?
4. Which coordination service does Kubernetes use, and for what?

---

## Exercise 7 — 2PC vs Saga
A "book trip" operation must reserve a flight (service A), reserve a hotel (service B), and charge a card
(service C).
1. Describe how 2PC would do this and the specific failure that makes it dangerous.
2. Describe the Saga version, including what happens if "charge card" fails.
3. Choreography vs orchestration — pick one for this and justify.
4. What two properties must each Saga step have?

---

## Exercise 8 — Distributed lock gone wrong
Worker W1 acquires a distributed lock (TTL 30s) to process job J, then suffers a 40s GC pause. Meanwhile
the lock expires and worker W2 acquires it and starts J.
1. What's the danger now?
2. Name the failure principle this is an instance of.
3. Design the fix with fencing tokens — what does the lock service issue, and what does the *resource* do?
4. What's an even better option than relying on the lock for correctness?

---

## Exercise 9 — Pick the coordination level
For each, choose **consensus**, **leader/quorum replication**, or **CRDT/eventual** and justify:
1. Agreeing which node is the primary.
2. A collaborative text editor's shared document across offline clients.
3. A globally-incrementing "likes" counter that can be approximate.
4. Committing the next entry in a replicated write-ahead log.

---

## Exercise 10 — Apply to your own systems 🏭
1. **MongoDB replica-set election / Redis Sentinel failover:** explain it using slow-vs-dead, majority
   election, and why a minority partition can't elect (no split-brain).
2. **Kubernetes / Kafka:** which consensus/coordination service backs each, and what state does it hold?
3. **DCE claim pipeline:** frame it as a Saga — what are the steps, and how do suppressions act like
   guard/compensation logic? Why don't you use 2PC across Mongo + downstream?
4. **Idempotent claim reprocessing:** explain why it's the practical answer to "you can't tell slow from
   dead."

---

When done, check `solutions/m09_distributed_systems_theory/README.md`, then say **"Module 10"**.
