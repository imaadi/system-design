# m20 — Solutions (Proximity / Ride-hailing)

> Check the *reasoning*. Recurring lessons: map 2D→indexable (geohash/quadtree/H3); check neighbor cells;
> Yelp read-heavy static vs Uber write-heavy volatile; geo finds candidates, matching optimizes.

---

## Exercise 1 — Estimation
8M drivers, update every 3 s.
1. **8M ÷ 3 ≈ ~2.7 million location updates/sec.** It's *the* number because that write flood (volatile,
   continuous) defines Uber's architecture — you can't persist it normally.
2. **No** — ~2.7M durable disk writes/sec is infeasible/ruinous. Instead keep **current locations in an
   in-memory geo-index** (Redis GEO / geo-sharded service), **updated in place** (matching needs only the
   current position, not history); buffer the ping stream via Kafka if needed; trip history is separate.
3. 8M × ~50 B ≈ **~400 MB** — comfortably fits in memory (which is why an in-memory index works).

---

## Exercise 2 — Why naive 2D fails
1. A B-tree is sorted on **one** key, so a `lat BETWEEN … AND long BETWEEN …` query can only use the index
   on **one** dimension (scanning a wide strip) then filters the rest — and "radius R" is a **circle**, not
   a box, so even the bounding box over-fetches. Slow at scale.
2. A **B-tree is 1-dimensional/linear ordering**, but **proximity is 2D** — two points close in space
   aren't necessarily close in a single-column sort order. You need to map 2D space to something the
   1D index can exploit (geohash) or use a structure built for 2D (quadtree/H3).

---

## Exercise 3 — Geohash
1. Geohash **interleaves the bits of lat and long** into one string by recursively bisecting the ranges,
   so a 2D point becomes a **1D string** where **string-prefix ≈ spatial neighborhood**.
2. Because **nearby points share a long common prefix**, "find points near me" = "**find geohashes with my
   prefix**" — a **prefix range query**, which a normal **B-tree** does efficiently (m04). 2D proximity →
   1D prefix lookup.
3. **Cell size ≈ query radius:** longer geohash = smaller cell. Pick a length whose cell roughly matches
   the "within X" you serve — too coarse over-fetches (cell ≫ radius), too fine means your radius spans
   many cells (more lookups).

---

## Exercise 4 — Quadtree vs H3
1. When **density is very uneven** — a quadtree **subdivides only dense cells**, so Manhattan splits deeply
   while the ocean stays coarse (no wasted/overloaded cells); a flat geohash grid uses uniform cells
   everywhere.
2. **Hexagons have uniform neighbor distances** (all 6 neighbors equidistant; no ambiguous diagonals/
   corners like squares), making **nearest-neighbor rings and distance reasoning clean**; plus H3 is
   **multi-resolution** for aggregation/zoom.
3. **S2** is Google's **sphere-based hierarchical** spatial index (handles globe curvature) — used by
   **Google Maps**.

---

## Exercise 5 — The boundary problem
1. A driver just **across a cell edge** from you is geographically close but in a **different cell** — query
   only your cell and you miss them (they could be the nearest).
2. **Query your cell + its neighbors** — geohash has **8 neighbors**; H3 has a surrounding **ring** of cells.
3. **Compute exact distances** for all candidates and **filter/sort** by the actual radius (the cells just
   shortlist; exact distance finalizes).

---

## Exercise 6 — Yelp vs Uber
1. **Yelp's POIs are static** (restaurants don't move) → index once, query many = **read-heavy**. **Uber's
   drivers move constantly** (updates every few seconds) → **write-heavy + real-time**. Same geo-index,
   opposite workload.
2. **Yelp "within 2 km":** index POIs by geohash (cell ≈ 2 km); on query compute the user's geohash, look
   up **cell + 8 neighbors**, fetch candidate POIs, compute exact distance, **filter** (≤ 2 km, open now,
   category) and **sort by distance**; cache popular areas.
3. **Uber stores current locations in-memory, updated in place** (volatile, no history persisted), vs
   Yelp's static POIs that live in an indexed (often cached) durable store.

---

## Exercise 7 — Matching
1. **Geo step:** query the in-memory index for **available drivers in the rider's cell + neighbors**.
   **Dispatch step:** apply matching logic (closest/lowest **ETA**, fairness, acceptance prob, surge) to
   pick + assign the best driver, notify both, remove from pool.
2. **Optimize ETA over the road network with traffic**, not straight-line distance — a closer-by-air driver
   can be farther in **time** (rivers, one-ways, traffic). Geo distance just shortlists; ETA decides.
3. **Make the assignment atomic** — atomically mark the chosen driver **unavailable** (compare-and-set /
   short lock) so a concurrent match can't pick them too (the lost-update/mutual-exclusion fix, m04/m09).

---

## Exercise 8 — Hot cells & sharding
1. **Geo-shard by region / geohash cell** across servers (consistent hashing or region map, m05) — each
   shard owns an area, queries/updates route to the relevant shard.
2. **Hot cell:** a dense area at peak (downtown rush hour) overloads the one shard owning that cell — a geo
   hot-key (m05/m06). **Mitigations:** adaptive/finer cells in dense areas (quadtree subdivision / H3 higher
   resolution → split load), **replicate** hot cells, and load-balance.
3. Because they **split dense areas into more cells** (so load spreads across more shards) while keeping
   sparse areas coarse — directly attacking the skew that creates hot cells, unlike a uniform grid.

---

## Exercise 9 — Real-time & routing
1. **WebSocket** (m14) — the driver streams location, the rider receives live updates to render the moving
   car; reuses the chat-system persistent-connection model.
2. Location data needs only **eventual consistency** (AP — slightly stale is fine); the **driver
   assignment** needs **stronger consistency** (atomic, so no double-assignment).
3. The **road network is a graph** (intersections = nodes, roads = weighted edges by travel time);
   routing = **shortest path (Dijkstra/A\*)** with heavy **precomputation** (contraction hierarchies/
   shortcuts) so cross-country routes return in ms; edge weights include real-time traffic (+ ML).

---

## Exercise 10 — Tie-in 🏭 + Part 3 preview
1. **In-memory geo-index = Redis** (native `GEOADD`/`GEOSEARCH`; "keep volatile current state in memory,
   don't persist every update" = your reference-data/session caching instinct, m06); **live tracking =
   WebSocket** (m14) + **location streams = Kafka** (m08) — your real-time/async toolset.
2. **Yelp's geo-query = geo-filtered search**: spatial index as the first filter, then category/rating/
   open-now filters + distance sort — structurally the **same multi-filter search you built for Sennzo**
   (m16), with a geospatial first stage.
3. **ML on top:** **ETA prediction** (pickup/dropoff time from road+traffic+history), **demand forecasting**
   (pre-position drivers), and **surge/dynamic pricing** (balance supply/demand) — plus dispatch
   optimization. These are ML systems over the geo backbone — **Part 3** territory where your AI/ML
   background leads.
