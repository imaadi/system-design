# m23 — Solutions (Feature Pipelines & Stores)

> Check the *reasoning*. Recurring lessons: batch vs streaming = freshness vs cost; dual-store (offline
> warehouse + online KV); the feature store kills training-serving skew + point-in-time leakage.

---

## Exercise 1 — Batch or streaming?
1. **Batch** — a 90-day average is slow-changing; daily recompute is fine and cheap.
2. **Streaming** — "last 5 minutes" velocity must be **fresh** (seconds); essential for fraud.
3. **Batch** — lifetime chargeback rate changes slowly; periodic recompute.
4. **Streaming** — current-session clicks need real-time freshness.
5. **Batch** (or near-real-time) — all-time review count changes slowly; a daily/hourly batch is fine
   (stream only if "reviews in last minute" mattered).

---

## Exercise 2 — The dual-store
1. Because **training and serving have opposite access patterns** — training scans huge history (batch),
   serving does low-latency point lookups (per request); one store can't be great at both (m04 OLTP/OLAP).
2. **Offline = warehouse/columnar** (Snowflake/BigQuery/Parquet) — optimized for **high-throughput batch
   reads** over history to build training datasets.
3. **Online = KV store** (Redis/DynamoDB) — optimized for **low-latency point lookups** (<10 ms) of the
   latest values at inference.
4. **DCE:** **Snowflake/Snowpark = offline** (batch ETL/feature computation); **Redis reference-data cache
   = online** (merged into each claim at scoring time for fast lookups).

---

## Exercise 3 — Training-serving skew
1. Features computed **differently** offline (training) vs online (serving) → the model sees a different
   distribution in production than it trained on → worse predictions, **no error thrown** (silent).
2. Training computes "avg purchase 30d" in a Spark/SQL job; serving re-implements it in app code with a
   different window/null-handling/rounding → same feature, different values.
3. The feature store **computes the feature once from one definition** and serves the **same values** to
   both training and serving → no second implementation to diverge.

---

## Exercise 4 — Point-in-time correctness
1. The feature values **as of last March** (the transaction's time) — what serving *would have seen* then.
2. You **leak the future** — a "current" value reflects data from *after* the event, so the model trains on
   info it won't have at prediction time → looks great offline, fails online.
3. **Point-in-time ("as-of") joins**, which require storing **timestamped feature history** (every value
   with the time it became valid) so each training event joins to the values valid at its time.

---

## Exercise 5 — Spot the leakage
1. `account_total_chargebacks` is a **lifetime** count that **includes chargebacks that happened *after*
   the transaction** being scored — so for a past transaction it contains **future** information.
2. Compute it **point-in-time**: the chargeback count **as of the transaction's time** (only chargebacks
   that had occurred before it), via a point-in-time join over timestamped history.
3. **Great offline** because the leaked future signal is highly predictive on historical data (the model
   "sees the answer"); **fails online** because at real prediction time those future chargebacks **don't
   exist yet**, so the powerful feature is unavailable/wrong.

---

## Exercise 6 — Architecture
1. **Batch pipeline** (Spark/SQL/Snowflake over warehouse/lake) → writes features to **offline store** +
   pushes latest to **online store**. **Streaming pipeline** (Flink/Kafka Streams over the Kafka event
   stream) → writes fresh features to the **online store** + logs to offline for training. Serving reads
   the online store; training reads offline with point-in-time joins.
2. **Lambda** = a **batch layer** (accurate/complete) + a **speed/streaming layer** (fresh), merged — but
   you **maintain the same feature logic twice**, a **skew risk + maintenance burden**. **Kappa** =
   **stream-only**, reprocess history by **replaying the log** — one code path.
3. **Kappa avoids duplicating feature logic** across two layers (the exact skew problem), simplifying
   maintenance and guaranteeing batch/stream consistency.

---

## Exercise 7 — Serving latency
1. Serve features from the **low-latency online KV store** (Redis/Dynamo), **batch all 20 lookups into one
   round trip** (pre-join the entity's features into one record so it's a single O(1) read), and cache hot
   entities — keeping well under the 30 ms budget.
2. A **hot entity** (celebrity user read constantly) is a hot key (m05/m06) on the online store → its node
   saturates. Handle by **replicating/caching** that entity's feature record across nodes / a local cache
   tier.

---

## Exercise 8 — Operational hazards
1. Every downstream model now reads that feature in the **wrong unit** (100× off) with **no error** →
   predictions silently degrade. Catch with **feature monitoring** (distribution/range checks, alert on a
   feature's stats deviating) + pipeline data validation (m28).
2. **Backfilling** — you must compute the new feature's **history** for past training data with **point-in-
   time correctness** (a big, careful batch job); until then it can't be used in models trained on old data.
3. **Embedding-version skew** — if serving uses a new embedding version but the model trained on the old
   one (or vice versa), inputs mismatch → skew. Avoid by **versioning embeddings** and ensuring train +
   serve use the **same version** (pin it with the model).

---

## Exercise 9 — Do you even need one?
1. **Over-engineering** for a **single model with a few features** — you can compute them in a shared
   library/request path, carefully kept consistent; a full feature store adds unneeded complexity.
2. It **earns its complexity** with: **many models/teams** (reuse + governance), **both batch and streaming
   features**, real **point-in-time-correctness** needs, or **scale** demanding a fast online store. (Any
   three.)
3. Even without the product, **training-serving consistency** (one definition for both paths) and
   **point-in-time correctness** (no future leakage) still apply — they're concepts, not just tools.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **DCE = a feature-store dual-store:** **Snowflake/Snowpark batch ETL = offline store/feature
   computation**; the **Redis real-time reference-data cache merged into each claim at scoring = online
   store/low-latency feature lookup**; the **claim-preprocessing layer = feature engineering + the single
   consistent definition** every model consumes (skew prevention). You built the pattern without the label.
2. **Membership-overlap / consecutive-period merge → windowed aggregations + temporal joins**;
   **date-overlap (30/90/180-day) features → windowed aggregations**; **NPI/Tax-ID + person/diagnosis
   matching → joins with reference/dimension data** — exactly §5's aggregation + reference-join patterns.
3. **Point-in-time in claims:** a claim must be scored against **eligibility/auth/reference data as of the
   claim's service or process date**, not "today's" — using current membership/auth to judge a past claim
   would leak future state (e.g. a member who became eligible *after* the service date). That's the
   point-in-time-correctness concern lived in healthcare.
