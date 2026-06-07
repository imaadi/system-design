# m20 — Exercises (Proximity / Ride-hailing)

> Do these on paper / out loud before checking `solutions/m20_proximity_ridehailing/`. Geospatial indexing
> and the write-heavy/matching split are the core.

---

## Exercise 1 — Estimation
Assume **8M active drivers**, location update every **3 s**, plus heavy proximity-query traffic.
1. Location updates/sec? Why is this *the* number for Uber?
2. Can you durably write all of these to a disk DB? What do you do instead?
3. Roughly how much memory to hold all current locations (~50 B each)?

---

## Exercise 2 — Why naive 2D fails
1. Why can't a B-tree on (lat, long) efficiently answer "within radius R"?
2. What's the fundamental mismatch between a B-tree and a 2D proximity query?

---

## Exercise 3 — Geohash
1. Explain in two sentences how geohash turns 2D proximity into a 1D problem.
2. Why does "nearby = shared prefix" let you use a normal index?
3. How do you choose the geohash precision/length for a given query radius?

---

## Exercise 4 — Quadtree vs H3
1. When is a quadtree better than a flat geohash grid?
2. Why does Uber use hexagons (H3) — what property of hexagons helps?
3. What is S2 and who uses it?

---

## Exercise 5 — The boundary problem
1. Describe a case where querying only your own cell misses the nearest result.
2. What's the fix (geohash vs H3)?
3. After gathering candidate cells, what two steps finish the query?

---

## Exercise 6 — Yelp vs Uber
1. Why is Yelp read-heavy and Uber write-heavy, given both use a geo-index?
2. How do you serve "restaurants within 2 km" for Yelp end-to-end?
3. What's different about Uber's storage of locations?

---

## Exercise 7 — Matching
1. Walk through matching a rider to a driver, naming the geo step and the dispatch step.
2. Why isn't straight-line distance enough — what do you actually optimize?
3. How do you prevent the same driver being matched to two riders?

---

## Exercise 8 — Hot cells & sharding
1. How do you shard the geo-index across servers?
2. What's the hot-cell problem and three mitigations?
3. Why do density-adaptive structures (quadtree/H3 multi-res) help here?

---

## Exercise 9 — Real-time & routing
1. How do you provide live trip tracking, and which earlier module does it reuse?
2. What consistency does location data need, and what one operation needs stronger consistency?
3. How does Google Maps routing work at a high level (graph + algorithm + trick)?

---

## Exercise 10 — Tie-in 🏭 + Part 3 preview
1. Map the **in-memory geo-index** and **live tracking** to tools/patterns you've used.
2. How is Yelp's geo-query the same shape as your **Sennzo multi-filter search**?
3. Name three places **ML** sits on top of this geo backbone (your Part-3 strength).

---

When done, check `solutions/m20_proximity_ridehailing/README.md`, then say **"Module 21"**.
