# m20 — Cross-Questions ("if-and-buts") on Proximity / Ride-hailing

> Answer out loud in 2–3 sentences before reading the model answer. Geospatial indexing, the write-heavy
> location flood, and matching are where this is won.

---

### Q1. Why can't you just store lat/long as two columns and query a bounding box?
**A.** A **B-tree indexes one dimension**, so it can't efficiently do a **2D range** — `WHERE lat BETWEEN
… AND long BETWEEN …` forces it to scan a huge strip on one column then filter, which is slow at scale.
And "within radius R" is a **circle**, not a box. You need a **geospatial index** purpose-built for 2D
proximity — geohash, quadtree, or H3/S2 — that maps 2D space to something a normal index can handle fast.

---

### Q2. Explain geohash and why it's clever.
**A.** **Geohash** encodes a (lat, long) into a short **string** by recursively bisecting the lat/long
ranges, interleaving the bits, and base32-encoding. The magic: **nearby locations share a common prefix**
(`9q8yy*` is a small area inside `9q8y*`). So "find points near me" becomes "**find geohashes with this
prefix**" — a **1D prefix query** that a standard **B-tree index** handles fast. It **collapses a 2D
proximity problem into 1D prefix matching** — that's the brilliant trick.

---

### Q3. What controls geohash precision, and how do you pick it?
**A.** **Length:** a **longer geohash = a smaller cell = more precise** (each extra char subdivides the
area). You pick the prefix length so the **cell size ≈ your query radius** — too coarse and a cell covers
way more than you need (you fetch and filter lots of far points); too fine and your radius spans many
cells (more lookups). Match cell size to the typical "find within X km" you serve.

---

### Q4. What's a quadtree and when is it better than geohash?
**A.** A **quadtree** recursively divides 2D space into **four quadrants**, subdividing a cell **only when
it holds too many points** — so **dense areas (Manhattan) subdivide deeply while sparse areas (ocean) stay
coarse**. It's better than a flat geohash grid when **density is very uneven**, because it **adapts to the
data** (no wasted cells on empty space, no overloaded cells in dense areas). The cost: it's more complex
to update/rebalance than a simple geohash string.

---

### Q5. What does Uber use, and why hexagons (H3)?
**A.** Uber uses **H3**, a **hexagonal hierarchical grid**. Hexagons are nice because **every neighbor is
equidistant** (unlike a square grid where diagonal neighbors are farther and corners are ambiguous), which
makes **nearest-neighbor "rings" and distance reasoning cleaner**, and the **multiple resolutions** let you
aggregate/zoom. So H3 gives uniform neighbor behavior + multi-resolution hierarchy — ideal for dispatch,
demand aggregation, and surge by cell. (Google's **S2** is the sphere-based analog used in Maps.)

---

### Q6. What is the boundary problem and how do you handle it?
**A.** A point near a **cell edge** may have its **nearest neighbor in the adjacent cell**, so querying
only the point's own cell **misses close-but-across-the-border results**. The fix: query the **target cell
plus its neighbor cells** (geohash has 8 neighbors; H3 has a "ring" of surrounding cells), gather all
candidates, then compute **exact distances** and sort/filter by the actual radius. Forgetting neighbor
cells is a classic bug that silently drops nearby results.

---

### Q7. How is Yelp's problem different from Uber's?
**A.** **Yelp** queries **mostly static** POIs (restaurants don't move) → it's **read-heavy**: index POIs
once by geohash, cache popular areas, and at query time fetch the cell + neighbors, filter (open now,
category), and sort by distance — basically a **geo + filter search** (m16). **Uber** queries **constantly
moving** drivers → it's **write-heavy + real-time**: millions of location updates/sec into an in-memory
geo-index, plus matching and live tracking. Same geo-index concept, opposite workload (read-heavy static
vs write-heavy volatile).

---

### Q8. Uber has millions of drivers pinging location every few seconds. How do you handle that write volume?
**A.** ~5M drivers ÷ ~4 s ≈ **~1.25M location updates/sec** — far too much to durably write to a disk DB.
You keep **current driver locations in an in-memory geo-index** (Redis GEO / a geo-sharded location
service), **updating in place** (overwrite the old location — matching only needs the *current* position,
not history). You **don't persist every ping**; the location state is **ephemeral** and small (~hundreds of
MB). If needed, buffer the ping stream through **Kafka** to absorb bursts. Trip history is a separate,
durable, append-only store.

---

### Q9. How do you scale the geo-index across servers?
**A.** **Geo-shard** it — partition by **region / geohash cell** across servers, so each shard owns an area
and a query/update routes to the relevant shard (consistent hashing or a region map, m05). This spreads
the global load and keeps each shard's data local to a geography. The catch is **hot cells** (dense areas)
overloading a shard — handled by adaptive cell size (subdivide dense cells) and replicating hot cells (Q10).

---

### Q10. What's the "hot cell" problem and how do you address it?
**A.** A **dense area at peak** (downtown at rush hour) packs huge numbers of drivers + requests into **one
geohash/H3 cell**, so the **shard owning that cell saturates** — a geo **hot-key** problem (m05/m06). Fixes:
use **adaptive/finer cells** in dense areas (quadtree subdivision or H3 higher resolution) so the load
splits across more cells/shards; **replicate** hot cells for read scaling; and load-balance. Uniform grids
make this worse, which is part of why density-adaptive (quadtree) and multi-res (H3) structures exist.

---

### Q11. Walk me through matching a rider to a driver.
**A.** Rider requests a ride at a location → **geo-query** the in-memory index for **available drivers in
the rider's cell + neighbor cells** → get candidates → apply **dispatch logic**: closest / lowest **ETA**
(not just straight-line distance — road network + traffic), plus fairness, driver acceptance probability,
and **surge** → **assign** the best driver, notify both, and remove that driver from the available pool →
then **stream live tracking** (WebSocket, m14). The geo-index *finds* candidates; the *matching* is a
separate optimization step on top.

---

### Q12. Is straight-line (geo) distance enough for matching? Why or why not?
**A.** No — **geo distance is just a first filter** to get nearby candidates. The real cost is **ETA over
the road network with traffic** (a driver 200 m away across a river or behind a one-way maze may be 10 min
out, while one 500 m away on a clear road is closer in time). So you use the **geo-index to shortlist**
nearby drivers, then compute **route-based ETA** (a routing engine, increasingly **ML-predicted**) to pick
the best. Geo proximity ≠ time proximity.

---

### Q13. How do you provide real-time trip tracking?
**A.** With a **persistent connection** — the driver's app streams location, and the rider's app receives
live updates over a **WebSocket** (m14) to render the moving car on the map. This reuses the chat-system
connection model (session registry + the driver→rider routing). Location is **eventually consistent** (a
slightly stale position is fine), and updates are frequent but small. Once the trip ends, the connection
closes.

---

### Q14. What consistency model does driver location need?
**A.** **Eventual consistency (AP)** — a driver's position being a second or two stale is harmless for
matching and tracking, and chasing strong consistency on ~1.25M updates/sec would be absurdly expensive.
You favor **availability + low latency**: always answer "nearby drivers" quickly even if positions are
slightly behind. The one place you want care is **not double-assigning** a driver (the assignment must be
atomic/consistent — a small, low-volume operation), but the location data itself is happily eventual.

---

### Q15. How do you prevent assigning the same driver to two riders at once?
**A.** Make the **assignment atomic** — when you match a driver, **atomically mark them unavailable**
(remove from the available pool) so a concurrent match can't pick them too. This is the **lost-update /
mutual-exclusion** problem (m04/m09): use an atomic operation (a compare-and-set / conditional update in
the index, or a short lock on the driver record) so only one match wins. The location flood is eventual,
but the **assignment** is a small consistent operation — different consistency for different data.

---

### Q16. How does Google Maps routing fit in (shortest path)?
**A.** The **road network is a graph** (intersections = nodes, roads = weighted edges with travel time),
and routing is a **shortest-path** problem — **Dijkstra/A\*** with heavy **precomputation** (contraction
hierarchies, precomputed shortcuts) so a cross-country route returns in milliseconds rather than searching
the whole graph live. Edge weights incorporate **real-time traffic** (and ML predictions). It's another
"**precompute the expensive work** so queries are fast" instance — the recurring theme, applied to graphs.

---

### Q17. Where does ML enter a ride-hailing system?
**A.** On top of the geo backbone: **ETA prediction** (how long until pickup/dropoff, from road + traffic
+ historical data), **demand forecasting** (predict where rides will be needed → pre-position drivers),
**surge/dynamic pricing** (balance supply/demand), and **dispatch optimization** (global matching). These
are **ML systems** over the location/trip data — exactly the **ML system design** territory of Part 3,
where your AI/ML background is the differentiator. The geo-index finds candidates; ML makes the marketplace
smart.

---

### Q18. What are the main bottlenecks and how do you address them?
**A.** **Location write volume** (~1.25M/s → in-memory in-place updates, don't persist every ping, Kafka
buffer); **hot cells** (adaptive subdivision + replication); the **boundary problem** (query neighbors);
**matching latency** (fast geo-shortlist then ETA); and **real-time tracking** (WebSocket fan-out).
Observability (m10): query/update p99, **per-cell load** (spot hot cells), match latency/success rate,
stale-location rate. The geo-index design (cell size, sharding) is the lever for most of these.

---

### Q19. How would you implement "find restaurants within 2 km" for Yelp specifically?
**A.** Index each POI by its **geohash** (precision chosen so a cell ≈ ~2 km, or use a couple of
precisions). On query: compute the user's geohash, look up the **matching cell + its 8 neighbors** (the
boundary fix), fetch candidate POIs in those cells, compute **exact distance**, **filter** (≤ 2 km, open
now, category, rating), and **sort by distance** (or relevance). Heavily **cache** popular areas (m06).
It's a **geo-filtered search** — the same shape as the multi-filter search I built for Sennzo, with a
spatial index as the first filter.

---

### Q20. How does this module reuse the building blocks, and what's next?
**A.** It reuses **B-tree indexing** (m04 — why 2D is hard, and how geohash makes it 1D-indexable),
**sharding + hot keys** (m05/m06 — geo-sharding + hot cells), **caching** (m06 — static POIs / hot areas),
**async streams** (m08 — location pings via Kafka), **WebSockets** (m14 — live tracking), and **search +
filters** (m16). It's the building blocks applied to a spatial/real-time domain. **What's next: payments
(m21)** closes Part 2, then **Part 3 (ML system design)** — including the **ETA/demand/surge ML** that
sits on *this* geo backbone, where my AI/ML background leads.
