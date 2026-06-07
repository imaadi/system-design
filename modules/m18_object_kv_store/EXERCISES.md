# m18 — Exercises (KV Store + Object Store)

> Do these on paper / out loud before checking `solutions/m18_object_kv_store/`. Part A rewards naming the
> Dynamo primitive + why; Part B rewards the erasure-coding cost trade.

---

## Exercise 1 — Assemble Dynamo
List the **seven** Dynamo techniques and the problem each solves (one line each). Note which module each
comes from.

---

## Exercise 2 — put/get walkthrough
1. Trace a `put(key, value)` with N=3, W=2, including what happens if one replica is down.
2. Trace a `get(key)` with R=2 when the two replicas return different versions.
3. With N=3, choose W and R for: (a) fast/available writes, (b) strong-ish reads, (c) balanced. Show W+R.

---

## Exercise 3 — Conflicts
1. Why does "always writable" produce conflicting versions?
2. What detects a conflict vs a simple newer-version, and how?
3. Three resolution strategies and the trade-off of each (use the Amazon cart as an example).

---

## Exercise 4 — Vector clocks vs timestamps
1. What can a vector clock tell you that a wall-clock timestamp cannot?
2. Why is Last-Write-Wins dangerous?
3. When is LWW acceptable anyway?

---

## Exercise 5 — Gossip, hinted handoff, sloppy quorum
1. What does gossip provide, and why is it decentralized (no SPOF)?
2. A write arrives but a target replica is down. What keeps the write available?
3. After the replica recovers, how does it get the missed write?

---

## Exercise 6 — Object store architecture
1. Why separate metadata from object data? What goes where?
2. What does multipart upload solve?
3. Why does immutability simplify the design?

---

## Exercise 7 — Erasure coding (the cost trade)
1. 3× replication: what's the storage overhead?
2. Reed-Solomon (10+4): how many failures can you survive, and what's the overhead?
3. State when you'd replicate vs erasure-code, and the cost you pay for erasure coding.
4. An object store has 100 PB of cold data. Rough storage cost with 3× vs (10+4)?

---

## Exercise 8 — Durability & integrity
1. How do you achieve 11 nines of durability (two ingredients)?
2. How do you detect and repair silent bit-rot?
3. Why isn't "we have 3 copies" enough by itself?

---

## Exercise 9 — Choosing the store
For each, pick KV store / object store / relational and justify:
1. User shopping carts at huge scale, must always accept writes.
2. 50 GB ML model artifacts retrieved by name.
3. Financial transactions with multi-row consistency.
4. User-uploaded videos.
5. Session tokens looked up by id, sub-ms.

---

## Exercise 10 — Your-systems tie-in 🏭
1. DCE stored model artifacts in **MinIO** and claims in **Mongo**. Map this to Part B's metadata/data
   split and explain why model files go to object storage, not a DB.
2. Which Dynamo primitives have you operated (via Mongo replica sets, Redis Cluster, Mongo read/write
   concerns)? Name three.
3. How is this module the "payoff" of m05 and m09 — what can you now *derive* instead of memorize?

---

When done, check `solutions/m18_object_kv_store/README.md`, then say **"Module 19"**.
