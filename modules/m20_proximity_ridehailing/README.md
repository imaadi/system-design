# Module 20 — Design Proximity / Ride-hailing (Uber, Yelp, Google Maps) ⭐⭐

> "Find what's near me" — and for Uber, "match me to a nearby driver whose location changes every few
> seconds." The star concept is **geospatial indexing**: a brilliant trick to make **2D proximity queries
> fast** by turning them into a **1D indexable problem** (geohash) or an **adaptive grid** (quadtree/H3).
> Two flavors expose a familiar split: **Yelp** (static POIs, **read-heavy**) vs **Uber** (moving
> drivers, **write-heavy + real-time matching**). It's the **penultimate classic problem** (payments/
> ledger in m21 closes Part 2 before ML system design begins).

> **Format (Part 2):** worked walkthrough; **geospatial indexing** and **Uber's write-heavy location +
> matching** are the deep-dives.

---

## Step 1 — Requirements

**Functional (core):**
- **Proximity (Yelp):** given a location + radius, return **nearby entities** (restaurants, drivers),
  sorted by distance.
- **Ride-hailing (Uber), additionally:** **drivers continuously report location**; a rider requests a
  ride → **match to a nearby driver**; **track the trip** in real time; compute **ETA**.

**Non-functional (these drive it):**
- **Low-latency proximity queries** (find-nearby must be fast — it's the core interaction).
- **(Uber) very high write throughput:** millions of drivers updating location every few seconds → a
  *flood* of location writes. *This is Uber's defining challenge.*
- **Real-time** matching + tracking (seconds).
- **Scalable + available + geographically distributed** (cities worldwide; hot dense areas).

> **The defining insight:** the hard part is **efficiently answering "what's near (lat, long)?"** — which
> needs a **geospatial index**, because you **can't** efficiently do it with a normal 2D range query.
> Uber adds a **write-heavy, real-time** twist on the same index.

---

## Step 2 — Estimation
**Uber-style:** assume **5M active drivers**, each updates location every **~4 s**; millions of ride
requests/day.
- **⭐ Location writes:** 5M ÷ 4 s = **~1.25 million location updates/sec** — *the* number. You **cannot**
  write this to a durable disk DB sustainably → keep current locations in an **in-memory geo-index**
  (Redis/geo-sharded service), updated in place (the old location is overwritten; you don't need history
  for matching).
- **Proximity queries:** ride requests + map views → high read QPS, latency-critical.
- **Storage:** current-location state is small (5M × ~50 B ≈ 250 MB — fits in memory); trip history is
  separate (durable, append-only).

> **The estimate frames it:** Yelp is **read-heavy over static data** (index once, query lots); Uber is
> **write-heavy over volatile data** (~1.25M updates/sec → **in-memory** geo-index, not a disk DB). Same
> index, opposite workload.

---

## Step 3 — Deep Dive: geospatial indexing ⭐⭐ (the star)

### Why naive doesn't work
You might store `lat` and `long` as two columns and query `WHERE lat BETWEEN ... AND long BETWEEN ...`.
But a B-tree indexes **one dimension** — it can't efficiently do a **2D range** (it'd scan a huge strip),
and "within radius R" is a circle, not a box. At scale this is too slow. You need a structure built for
**2D proximity**.

### Geohash — turn 2D into 1D ⭐ (the brilliant trick)
**Geohash** encodes a (lat, long) into a **short string** (e.g. `9q8yyk`) by recursively bisecting the
lat/long ranges and interleaving the bits, then base32-encoding. The magic property: **nearby locations
share a common prefix** — `9q8yy*` is a small area, `9q8y*` a bigger one containing it.
```
  (lat, long)  ──geohash──▶  "9q8yyk"
  nearby points → shared prefix → "find near me" = "find geohashes with prefix 9q8yy"
  → a 1D PREFIX query, which a normal B-tree index handles! (m04)
```
So you **collapse a 2D proximity problem into 1D prefix matching** — now it's a standard, fast indexed
lookup. **Longer geohash = smaller cell = more precise** (pick the prefix length ≈ your query radius).
*(Redis `GEOADD`/`GEOSEARCH` use geohash under the hood.)*

### Quadtree — adaptive to density ⭐
A **quadtree** recursively divides 2D space into **4 quadrants**, subdividing a cell only when it holds
too many points. So **dense areas (Manhattan) subdivide deeply; sparse areas (ocean) stay coarse** —
adapting to skew (a uniform grid wastes cells on empty space and overloads dense ones). A proximity query
walks down to the relevant cell(s). Great when density is very uneven; rebuilding/updating is more work
than a flat geohash.

### H3 / S2 — hierarchical grids (what the giants use)
- **Uber's H3:** a **hexagonal** hierarchical grid (hexagons tile nicely — uniform neighbor distances,
  no corner ambiguity). Multiple **resolutions**; cells have stable IDs; great for aggregation + nearest-
  neighbor "rings."
- **Google's S2:** maps the sphere to a hierarchical grid of cells (handles the globe's curvature). Used
  in Maps.

| Approach | Idea | Strength | Used by |
|---|---|---|---|
| **Geohash** | 2D → 1D prefix string | simple, indexable with a B-tree, prefix = proximity | Redis GEO, many |
| **Quadtree** | recursive 4-way split | **adapts to density** | classic spatial DBs |
| **H3** | hex hierarchical grid | uniform neighbors, multi-res, aggregation | **Uber** |
| **S2** | sphere → hierarchical cells | handles globe curvature | **Google Maps** |

> **The senior framing:** *"You can't efficiently 2D-range a B-tree, so you map space to an indexable
> key: **geohash** turns proximity into a **1D prefix** (nearby → shared prefix → standard index);
> **quadtree** adapts to density; **H3/S2** are hierarchical grids the big players use. Pick cell size ≈
> query radius, and **handle the boundary problem** by also checking neighbor cells."*

### ⚠️ The boundary problem
A point near a **cell edge** may have its nearest neighbor in the **adjacent** cell. So a proximity query
must check the **target cell + its neighbors** (geohash has 8 neighbors; H3 has a "ring"), then compute
exact distances and sort. Forgetting this misses close-but-across-the-border results.

---

## Step 4 — Yelp (read-heavy, static) vs Uber (write-heavy, dynamic)

### Yelp — find nearby POIs (read-heavy)
POIs are **mostly static** → **index once** by geohash (a column/table indexed by geohash), heavily
**cache** popular areas (m06). Query: compute the user's geohash → look up that cell + neighbors → fetch
candidate POIs → **filter** (open now, category) + **sort by exact distance** → return. It's basically a
**search problem** (m16) with a geo-index + filters — similar to your Sennzo multi-filter search.

### Uber — match riders to moving drivers (write-heavy + real-time) ⭐⭐
1. **Location updates (the flood):** ~1.25M/sec → keep **current driver locations in an in-memory geo-
   index** (Redis GEO / a geo-sharded "location service"), **updating in place** (overwrite old location;
   matching doesn't need history). Don't durably write every ping — it's ephemeral state.
2. **Geo-sharding:** partition the index **by region/geohash cell** across servers, so a city's load is
   spread and a query/update hits the relevant shard. (Consistent hashing/region mapping, m05.)
3. **Matching (dispatch):** rider requests → **geo-query** the index for nearby available drivers (cell +
   neighbors) → apply **dispatch logic** (closest, lowest ETA, fairness, driver acceptance, surge) →
   assign → notify both. The match is a separate concern from the index.
4. **Real-time tracking:** once matched, stream the driver's live location to the rider via **WebSocket**
   (m14) for the moving-car-on-map view.
5. **ETA / routing:** estimate arrival via a routing engine + **ML ETA prediction** (real Uber uses ML;
   ties to Part 3) over the road network (Google Maps is a graph + shortest-path problem — Dijkstra/A*/
   contraction hierarchies).

---

## Step 5 — Bottlenecks, scale & edge cases

- **Hot regions (the geo hot-key):** downtown at rush hour = a **hot cell** (m05/m06) — tons of drivers
  + requests in one geohash/H3 cell → that shard saturates. Mitigate: **adaptive cell size** (subdivide
  dense cells — quadtree/H3 multi-res), replicate hot cells, and load-balance.
- **Write volume (Uber):** in-memory geo-index updated in place; absorb the location-ping flood via a
  **stream (Kafka)** if needed; don't persist every ping.
- **Boundary problem:** always query neighbor cells (above).
- **The matching is its own problem:** dispatch optimization (global assignment, surge pricing, ETA) is
  increasingly **ML** (demand forecasting, ETA, pricing) — ties to Part 3.
- **Routing (Maps):** the road network is a **graph**; shortest path uses Dijkstra/A*/contraction
  hierarchies with precomputation (another "precompute for fast reads").
- **Consistency:** location/availability is **eventually consistent** (AP) — a slightly stale driver
  position is fine; matching tolerates it.
- **Observability (m10):** query/update p99, per-cell load (hot cells), match latency/success, stale-
  location rate.

**Wrap-up:** *"The core is a **geospatial index** that makes 'what's near (lat, long)?' fast: **geohash**
(2D→1D prefix, B-tree-indexable — nearby share a prefix), **quadtree** (density-adaptive), or **H3/S2**
(hierarchical grids the giants use), always checking **neighbor cells** for the boundary problem. **Yelp**
is read-heavy over **static** POIs → index once + cache + filter + sort. **Uber** is write-heavy +
real-time: ~1.25M location updates/sec kept in an **in-memory, geo-sharded index** (updated in place),
proximity-query for nearby drivers, run **dispatch/matching** (closest/ETA/surge, increasingly ML), and
**WebSocket** live tracking. Key challenges: **hot cells** (adaptive subdivision + replication) and the
**boundary problem**."*

---

## State-of-the-art & real-world notes 📚
- **Uber H3** (open-source hexagonal grid) — the canonical ride-hailing geo-index; Uber's marketplace/
  dispatch + ML (ETA, demand, surge) sit on top. **Google S2** powers Maps' spatial indexing.
- **Geohash** is the classic, simple approach (and what **Redis GEO** and many DBs use). **PostGIS**
  (Postgres spatial extension, R-tree/GiST indexes) is the go-to spatial DB.
- **Routing:** Google Maps = graph shortest-path with heavy precomputation (contraction hierarchies);
  ETA increasingly **ML-predicted**.

---

## From your systems 🏭
- **Redis GEO + in-memory hot state:** the in-memory geo-index is your **Redis** comfort zone (Redis has
  native `GEOADD`/`GEOSEARCH`); the "keep volatile current state in memory, don't persist every update"
  is the same instinct as your reference-data/session caching (m06).
- **Real-time streaming + WebSocket:** location-update streams (Kafka, m08) + live trip tracking
  (WebSocket, m14) reuse your real-time + async toolset; Questimate's real-time data handling is adjacent.
- **Read-heavy vs write-heavy split:** Yelp (static, read-heavy) vs Uber (volatile, write-heavy) is the
  same workload-classification you do daily (m01/m04) — and Yelp's geo+filter query is your **Sennzo
  multi-filter search** with a spatial index.
- **ML on top (Part 3 preview):** Uber's ETA/demand/surge are ML systems over this geo backbone — exactly
  the ML-system-design territory your AI/ML background makes you strong in (next modules).

---

## Key concepts (interview-ready)
- **The problem is geospatial indexing:** you **can't efficiently 2D-range a B-tree**, so map space to an
  indexable key. **Geohash:** 2D → **1D prefix string** (nearby = shared prefix → standard B-tree lookup);
  cell size ≈ query radius. **Quadtree:** recursive 4-way split, **adapts to density**. **H3 (Uber, hex) /
  S2 (Google, sphere):** hierarchical grids. **Always check neighbor cells (boundary problem).**
- **Yelp = read-heavy over static POIs:** index once by geohash + cache + filter + exact-distance sort
  (a geo search problem, m16).
- **Uber = write-heavy + real-time:** ~1.25M location updates/sec → **in-memory, geo-sharded index updated
  in place** (don't persist every ping); proximity-query nearby drivers; **dispatch/matching** (closest/
  ETA/surge, increasingly **ML**); **WebSocket** live tracking (m14).
- **Hot cells** (downtown rush hour) = geo hot-key (m05/m06) → adaptive subdivision + replication.
  **Eventual consistency** for locations (AP). **Routing (Maps)** = graph shortest-path + precomputation.

---

## Go deeper (reading)
- **Uber Engineering: H3** (the hexagonal grid) and their marketplace/dispatch + ETA-ML posts.
- **Google S2 geometry library** and **geohash** explainers; **Redis GEO** + **PostGIS** docs (the
  practical tools).
- **System Design Interview (Alex Xu): "Proximity service" / "Design Uber"** — the geo-index + matching
  treatment.
- Revisit **m04** (B-tree indexing — why 2D is hard), **m05** (geo-sharding/hot cells), **m06** (caching/
  hot keys), **m14** (WebSocket tracking), **m16** (geo as a search+filter problem), **Part 3** (the ML on
  top — ETA/demand/surge).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m20_proximity_ridehailing/`](../../solutions/m20_proximity_ridehailing/README.md).

Say **"Module 21"** to design a *Payment System / Ledger* (idempotency, exactly-once effects,
double-entry bookkeeping, reconciliation) — the **final classic problem (and the close of Part 2)**,
squarely your **Questimate financial / DCE correctness** territory. After that, **Part 3: ML System
Design** begins (m22), where your AI/ML strength leads.
