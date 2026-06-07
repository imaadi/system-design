# Module 23 — Data & Feature Pipelines / Feature Stores ⭐⭐ 🏭

> Features are the model's fuel, and the **feature pipeline is the most important — and most error-prone —
> part of an ML system.** This module deep-dives Step 3 of the m22 framework: **batch vs streaming
> features** (the freshness-vs-cost trade), the **feature store** (the system that computes features once
> and serves both training and inference), and the two killers it solves — **training-serving skew** and
> **point-in-time correctness**. This is **your DCE world made explicit**: the claim-preprocessing layer
> (feature engineering), merging claims with **Redis reference data** (online feature lookup), and
> **Snowflake batch ETL** (offline features) *are* a feature-store dual-store.

> **Format (Part 3 deep-dive):** worked design of the feature platform; **batch-vs-streaming**, the
> **feature store**, and **point-in-time correctness** are the deep-dives.

---

## 1. Why features (not models) are where ML systems live or die
- **"Features beat algorithms"** (your AI/ML lesson: metadata 0.91 → text features 0.97). Good features
  often matter more than a fancier model.
- Features are computed in **two places** (m22): the **offline training pipeline** (batch, over history)
  and the **online serving path** (live, per request). Keeping these **consistent** is the central
  challenge — get it wrong and the model silently fails (**training-serving skew**).
- So the system-design problem isn't "what features?" (that's modeling) — it's **"how do I compute, store,
  and serve features consistently, freshly, and reusably at scale?"** That's the **feature store**.

---

## 2. Batch vs streaming (real-time) features ⭐⭐ (the freshness-vs-cost trade)

Features differ by **how fast they change** and **how fresh they must be at prediction time**:

- **Batch features:** computed **periodically** (hourly/daily) over historical data in a **batch job**
  (Spark/SQL/Snowflake). E.g. *"user's average purchase over the last 30 days", "merchant's lifetime
  chargeback rate".*
  - **Pro:** cheap, simple, high-throughput; fine for **slow-changing** features.
  - **Con:** **stale** — up to a full batch interval behind (yesterday's aggregate).
- **Streaming / real-time features:** computed **on-the-fly** from **recent events** via a stream processor
  (Flink/Kafka Streams/Spark Streaming). E.g. *"number of transactions in the last 5 minutes", "clicks in
  this session".*
  - **Pro:** **fresh** (seconds old) — essential when recent behavior matters (fraud velocity, live
    session context).
  - **Con:** **expensive + complex** (maintain real-time aggregates, windowing, state, exactly-once, m08).

| | **Batch features** | **Streaming features** |
|---|---|---|
| Computed | periodically (hourly/daily) | continuously, per-event |
| Freshness | stale (interval-behind) | fresh (seconds) |
| Cost/complexity | low | high (stateful stream processing) |
| Use for | slow-changing aggregates, history | fast-changing / velocity / session features |
| Tools | Spark, SQL, Snowflake | Flink, Kafka Streams, Spark Streaming |

> **The senior framing:** *"I choose batch vs streaming **per feature** by how fast it changes and how
> much freshness moves the prediction. Slow aggregates ('30-day avg') go **batch** (cheap); velocity/
> session features ('txns in last 5 min') go **streaming** (worth the cost). Most systems are **hybrid**:
> batch for the bulk, streaming for the few features where recency is critical."* (Fraud, m27, leans on
> streaming velocity features.)

---

## 3. The feature store ⭐⭐ (the centerpiece)

A **feature store** is a centralized system to **define, compute, store, serve, and reuse features** —
the linchpin that keeps the offline (training) and online (serving) paths consistent.

### The dual-store architecture (the key design) ⭐
The same feature needs **two storage shapes** (the m04 OLTP-vs-OLAP split applied to features):
- **Offline store** (for **training**): historical feature values, optimized for **high-throughput
  batch reads** to build training datasets → a **warehouse/columnar store** (Snowflake/BigQuery/S3
  Parquet). *(Your DCE Snowflake.)*
- **Online store** (for **serving**): the **latest** feature values per entity, optimized for **low-latency
  point lookups** at inference (<10 ms) → a **KV store** (Redis/DynamoDB/Cassandra). *(Your DCE Redis
  reference-data cache.)*

```
   feature pipelines (batch + streaming)
        │ write the SAME computed features to BOTH
        ├──────────────▶ OFFLINE store (warehouse) ──▶ training (build datasets, point-in-time joins)
        └──────────────▶ ONLINE store (Redis/KV)   ──▶ serving (low-latency lookup at inference)
   feature registry/catalog: definitions, ownership, discovery, reuse
```

### What the feature store gives you (4 things)
1. **Consistency (kills training-serving skew):** features are defined **once** and the **same values**
   flow to both stores → training and serving see identical features (§4).
2. **Low-latency serving:** the online store answers feature lookups in ms at inference time.
3. **Reuse + discovery (a feature platform):** a **registry/catalog** lets teams **find and share**
   features — "compute `user_avg_purchase` once, every model reuses it" — instead of N teams writing N
   (subtly different, skew-causing) implementations. *Centralization avoids the dual-write problem (m04).*
4. **Point-in-time correctness for training (§4).**

> **Say this:** *"A feature store is a **dual-store**: an **offline store (warehouse)** for high-throughput
> training reads and an **online store (KV/Redis)** for low-latency serving, fed by **shared feature
> pipelines** so both see identical features — plus a **registry** for reuse. It exists to solve the two
> killers: **training-serving skew** and **point-in-time correctness**."*

---

## 4. The two killers the feature store solves ⭐⭐

### (a) Training-serving skew (recap from m22, the fix here)
Skew = features computed **differently** offline vs online → silent degradation. The feature store fixes
it by **computing features once from one definition** and serving the **same values** to both training and
inference — so the feature a model trains on is computed identically to the one it serves on. No two code
paths, no drift between them.

### (b) Point-in-time correctness ⭐ (the subtle, deadly one)
When you build a **training example** for an event at time **T** (with its label), you must compute its
features using **only data known *as of* T** — **never** a feature value that was updated *after* T. If you
use a "current" feature value to label a *past* event, you **leak the future** → the model trains on info
it won't have at prediction time → **great offline, fails online** (a form of data leakage, m22).

```
  Event at T (label known later). To build the training row:
    ✗ WRONG: join the user's CURRENT avg_purchase (computed today) → leaks the future
    ✓ RIGHT: join the user's avg_purchase AS OF time T (point-in-time / "time-travel" join)
```
The feature store supports **point-in-time joins** (it stores **timestamped** feature history and joins
each training event to the feature values **valid at that event's time**) — so training data mirrors
exactly what serving would have seen. *Most data scientists get this subtly wrong by hand; the feature
store makes it correct by construction.*

> **The senior line:** *"Two silent killers: **training-serving skew** (different feature code offline vs
> online) and **point-in-time leakage** (using future feature values to label past events). A feature
> store fixes both — **one definition served to both paths**, and **point-in-time joins** that build each
> training row from feature values **as-of the event time**. Both bugs are invisible (offline looks great)
> until production tanks — which is why this is the foundation of ML system design."*

---

## 5. Architecture: the pipelines feeding the stores
- **Feature pipelines** transform raw data into features:
  - **Batch pipeline:** scheduled jobs (Spark/SQL/Snowflake) over the warehouse/lake → write to offline
    store (and push the latest to online store).
  - **Streaming pipeline:** a stream processor (Flink/Kafka Streams) over event streams (Kafka, m08) →
    write fresh features to the online store (and log to offline for training).
- **Lambda vs Kappa architecture** (worth naming): **Lambda** = a **batch layer** (accurate, complete) +
  a **speed/streaming layer** (fresh) whose results are merged — robust but **duplicate logic** in two
  places (a skew risk!). **Kappa** = **stream-only** — treat everything as a stream and **reprocess by
  replaying** the log (one code path, simpler, no batch/stream duplication). Kappa is increasingly favored
  to avoid maintaining two implementations.
- **Feature engineering patterns:** windowed **aggregations** (sum/avg/count over time windows),
  **embeddings** (learned vector features, your m05 AI work), **encoding** (one-hot/ordinal/target),
  **normalization/scaling**, and **joins** with reference/dimension data (your DCE eligibility/auth merges).

---

## 6. Bottlenecks, scale & edge cases
- **Freshness vs cost:** streaming features are expensive — only stream what truly needs recency; batch
  the rest. (The §2 trade.)
- **Online-store latency:** feature lookups are on the inference critical path → KV store (Redis/Dynamo)
  with caching; sometimes **fetch many features in one batched lookup** to hit the latency budget.
- **Backfilling:** when you add a new feature, you must **compute its history** for past training data
  (a big batch job with point-in-time correctness) — non-trivial.
- **Feature drift / monitoring:** features can drift (m22/m28) — monitor distributions; a broken upstream
  pipeline silently corrupts features (a top failure mode).
- **Online/offline consistency:** even with a feature store, ensure the batch-computed and streaming-
  computed versions of a feature agree (the Lambda duplication risk).
- **Governance:** ownership, versioning, PII/compliance on features (your healthcare PHI world).

**Wrap-up:** *"Features are computed by **batch pipelines** (cheap, stale — slow aggregates) and
**streaming pipelines** (fresh, expensive — velocity/session), chosen **per feature** by freshness need. A
**feature store** is the linchpin: a **dual-store** (offline warehouse for training, online KV for
low-latency serving) fed by **shared feature definitions**, with a **registry** for reuse. It solves the
two silent killers — **training-serving skew** (one definition → both paths see identical features) and
**point-in-time correctness** (timestamped feature history + point-in-time joins → no future leakage into
training). It's the m04 (OLTP/OLAP), m06 (caching), m08 (streaming) toolkit assembled for features. Trade-
offs: freshness vs cost (streaming), online-store latency, backfilling, and avoiding Lambda's duplicate
batch/stream logic (Kappa)."*

---

## 7. State-of-the-art & real-world notes 📚
- **Uber Michelangelo** (the system that popularized the feature store) and **Airbnb Zipline** — the
  origin stories. **Feast** (open-source feature store), **Tecton** (commercial, by Michelangelo's
  creators), **Databricks/SageMaker/Vertex Feature Stores** — the productized versions.
- **Point-in-time joins / "time-travel"** is the defining feature-store capability (Feast/Tecton docs).
- **Lambda vs Kappa** (Nathan Marz / Jay Kreps) — the batch+stream vs stream-only architectures.
- The recurring theme: a feature store is **m04 (dual storage) + m06 (cache) + m08 (streaming) +
  governance**, specialized for ML features.

---

## 8. From your systems 🏭 (you've built this without the label)
- **DCE *is* a feature-store dual-store:** **Snowflake/Snowpark batch ETL** = the **offline store /
  feature computation**; the **Redis real-time reference-data cache** that the pipeline **merges into each
  claim at scoring time** = the **online store / low-latency feature lookup at inference**. You move
  eligibility/auth/reference features into Redis precisely so scoring reads them fast — that's online
  feature serving.
- **Claim-preprocessing = feature engineering + consistency:** turning raw claim JSON into **100+
  structured fields** every model consumes is feature engineering *and* the **single consistent
  definition** that prevents training-serving skew (§4a).
- **Your feature patterns:** membership-overlap, person-matching, **date-overlap windows (30/90/180-day)**,
  AIM-grouper matching, provider NPI/Tax-ID matching — these are exactly §5's windowed aggregations +
  joins with reference data.
- **Point-in-time awareness:** a claim must be scored on data **as of the claim's service/process date**,
  not "today's" eligibility — the §4b point-in-time-correctness concern lives in healthcare claims.
- **Embeddings as features:** your AI/ML embedding/TF-IDF work is §5's learned/encoded features.

---

## 9. Key concepts (interview-ready)
- **Features are where ML systems live or die** ("features beat algorithms"); the system problem is
  **computing/storing/serving features consistently, freshly, reusably** — the **feature store**.
- **Batch vs streaming features** = **freshness vs cost**, chosen **per feature**: batch (cheap, stale,
  slow aggregates) vs streaming (fresh, expensive, velocity/session). Most systems are **hybrid**.
- **Feature store = a dual-store**: **offline (warehouse)** for high-throughput training reads + **online
  (KV/Redis)** for low-latency serving, fed by **shared feature pipelines**, with a **registry** for reuse.
- **It solves the two silent killers:** **training-serving skew** (one definition → identical features in
  train + serve) and **point-in-time correctness** (timestamped history + **point-in-time joins** → no
  future leakage into training). Both bugs are invisible until production fails.
- **Architecture:** batch + streaming pipelines (m08); **Lambda** (batch+speed layers, duplicate logic =
  skew risk) vs **Kappa** (stream-only, replay — one code path). Patterns: windowed aggregations,
  embeddings, encoding, joins with reference data.
- **It's m04 (OLTP/OLAP) + m06 (cache) + m08 (streaming)** assembled for features. Bottlenecks: freshness/
  cost, online-store latency, backfilling, feature drift/monitoring.

---

## 10. Go deeper (reading)
- **Chip Huyen, "Designing ML Systems" (feature engineering + data engineering chapters)** — the framework.
- **Uber "Michelangelo" + "Michelangelo Palette/feature store" blog posts**, **Airbnb "Zipline"** — the
  origin feature stores.
- **Feast docs (point-in-time joins, online/offline stores)** and **Tecton's feature-store explainers** —
  the productized concepts.
- **"Questioning the Lambda Architecture" (Jay Kreps)** — Lambda vs Kappa.
- Revisit **m04** (OLTP/OLAP, dual storage), **m06** (caching = online store), **m08** (streaming/Kafka),
  **m22** (where skew/point-in-time were introduced), **m28** (feature monitoring/drift).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m23_feature_pipelines_stores/`](../../solutions/m23_feature_pipelines_stores/README.md).

When you've done the exercises, say **"Module 24"** to design *Model Serving & Inference at Scale* — online
vs batch serving, dynamic batching, the model registry, and safe rollout (shadow/canary/A-B) — how the
model + these features actually reach production.
