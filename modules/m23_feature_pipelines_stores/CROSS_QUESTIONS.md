# m23 — Cross-Questions ("if-and-buts") on Feature Pipelines & Stores

> Answer out loud in 2–3 sentences before reading the model answer. Batch-vs-streaming, the dual-store, and
> point-in-time correctness are where this is won.

---

### Q1. Why are features such a big deal in an ML system — isn't the model the point?
**A.** No — **features are the model's fuel, and often matter more than the model** ("features beat
algorithms" — better features beat better algorithms on the same data). And the **systems** problem isn't
"what features" (modeling) but "**how do I compute, store, and serve features consistently, freshly, and
reusably at scale across the offline-training and online-serving paths?**" Getting that wrong causes
**training-serving skew** and **leakage** — silent failures that the fanciest model can't survive. The
feature pipeline is the most error-prone part of ML production.

---

### Q2. Batch vs streaming features — what's the difference and how do you choose?
**A.** **Batch** features are computed **periodically** (hourly/daily) over historical data — cheap and
simple but **stale** (interval-behind), good for slow-changing aggregates ("30-day avg purchase").
**Streaming** features are computed **continuously from recent events** — **fresh** (seconds old) but
**expensive/complex** (stateful stream processing), needed for fast-changing signals ("txns in last 5
min"). I choose **per feature** by **how fast it changes and how much freshness moves the prediction** —
most systems are **hybrid** (batch the bulk, stream the few features where recency is critical, like fraud
velocity).

---

### Q3. What is a feature store and why do you need one?
**A.** A **feature store** is a centralized system to **define, compute, store, serve, and reuse
features**. You need it because features must reach **two places consistently** — the **offline path**
(training) and the **online path** (serving) — and doing that by hand causes skew and leakage. The feature
store provides a **dual-store** (offline warehouse + online KV), **shared feature definitions** (so both
paths see identical features), **low-latency serving**, **point-in-time correctness** for training, and a
**registry** for reuse. It's the linchpin of a production ML system.

---

### Q4. Explain the dual-store architecture of a feature store.
**A.** The same feature needs **two storage shapes** (the m04 OLTP/OLAP split applied to features). The
**offline store** (warehouse/columnar — Snowflake/BigQuery/Parquet) holds **historical** feature values
for **high-throughput batch reads** to build training datasets. The **online store** (KV — Redis/DynamoDB)
holds the **latest** value per entity for **low-latency point lookups** (<10 ms) at inference. **Shared
feature pipelines write the same computed features to both**, so training and serving stay consistent. My
DCE setup is exactly this: **Snowflake offline, Redis reference-data online.**

---

### Q5. How does a feature store prevent training-serving skew?
**A.** By **computing each feature once from a single definition** and serving the **same values** to both
the training (offline store) and serving (online store) paths. So the feature a model trains on is computed
**identically** to the one it serves on — there's no second, subtly-different implementation to drift
apart. Without a feature store, a data scientist writes the training feature in Spark/SQL and an engineer
re-implements it in serving code, and they inevitably diverge (different windows, null handling, rounding)
→ skew. One definition, two destinations, no skew.

---

### Q6. What is point-in-time correctness and why does it matter?
**A.** When building a **training example** for an event at time **T**, you must compute its features using
**only data known as of T** — never a feature value updated **after** T. If you join a *current* feature
value onto a *past* event, you **leak the future**: the model trains on information it won't have at
prediction time, so it looks great offline but **fails online** (data leakage). It matters because it's
**silent and flattering** — the leak inflates offline metrics, hiding the problem until production tanks.
You fix it with **point-in-time joins** over timestamped feature history.

---

### Q7. What's a point-in-time join, concretely?
**A.** The feature store keeps **timestamped feature history** (every feature value with the time it became
valid). A **point-in-time join** matches each training event (at time T) to the feature values that were
**valid at T** — "as-of" join / time-travel — rather than the latest values. So a training row for a
transaction last March uses the user's stats **as they were last March**, not today's. This makes the
training data mirror **exactly what serving would have seen** at that moment, eliminating future leakage.
Most people do this wrong by hand; the feature store makes it correct by construction.

---

### Q8. Why store features in two different databases instead of one?
**A.** Because **training and serving have opposite access patterns** (the m04 OLTP-vs-OLAP split):
training does **high-throughput scans over huge historical data** (→ a columnar warehouse), while serving
does **low-latency point lookups for one entity** at inference (→ a KV store like Redis). One database
can't be great at both — a warehouse is too slow for per-request serving, and a KV store can't efficiently
build big training datasets or hold full history. So you store the **same features in both shapes**, fed by
one pipeline.

---

### Q9. How does the feature store enable feature reuse, and why does that matter?
**A.** Via a **registry/catalog** of feature definitions with ownership and metadata, so teams can
**discover and reuse** existing features instead of re-implementing them. It matters because **N teams
writing N implementations of `user_avg_purchase`** is both wasteful **and** a skew factory (each
implementation differs slightly). Centralizing to **one definition** everyone reuses avoids the
**dual-write/duplication problem** (m04), saves work, and guarantees consistency. A mature feature store is
really a **feature platform** — features become shared, governed assets.

---

### Q10. A feature lookup is on the inference critical path. How do you keep it fast?
**A.** The **online store** is a **low-latency KV store** (Redis/DynamoDB) sized for <10 ms point reads,
often **cached** (m06). To hit a tight latency budget when a model needs many features, you **batch the
lookups** (fetch all features for the entity in one round trip rather than N) and pre-warm/replicate hot
entities. The online store holds only the **latest** values (small), so lookups are O(1). If features come
from multiple sources, you may pre-join them into one online record. Feature-serving latency is part of the
serving SLO (m24).

---

### Q11. What's the difference between Lambda and Kappa architectures?
**A.** **Lambda** = a **batch layer** (accurate, complete, slow) **+ a speed/streaming layer** (fresh,
approximate), whose outputs are merged — robust but you **maintain the same logic twice** (batch and
stream), which is a **skew risk** and a maintenance burden. **Kappa** = **stream-only**: treat everything
as a stream and **reprocess history by replaying the log** (Kafka) — **one code path**, no batch/stream
duplication. Kappa is increasingly favored precisely because duplicating feature logic across two layers is
exactly the skew problem you're trying to avoid.

---

### Q12. You add a new feature. What's the complication for training?
**A.** **Backfilling** — to train on historical data, you need the new feature's **values for the past**,
which means **recomputing its history** with **point-in-time correctness** (each past row gets the value
as-of its time). That's a big batch job and easy to get subtly wrong (leaking the future during backfill).
Until backfilled, you can't use the feature in models trained on old data. Backfilling is a real operational
cost of feature engineering that the feature store helps standardize but doesn't make free.

---

### Q13. How can a feature pipeline silently corrupt your model?
**A.** A **broken upstream pipeline** (a schema change, a failed job, a null surge, a unit change like
dollars→cents) produces **wrong feature values**, and the model **keeps serving** on garbage with **no
error** — predictions just get worse. It's a top ML failure mode because the model can't tell good features
from bad. You defend with **feature monitoring** (distribution checks, null-rate alerts, freshness checks —
m28), data validation in the pipeline, and alerting when a feature's stats deviate from expected. "Garbage
in, silent garbage out."

---

### Q14. Where do batch and streaming features physically land, and how do they reach serving?
**A.** **Batch pipelines** (Spark/SQL/Snowflake over the warehouse/lake) write features to the **offline
store** and push the latest to the **online store**. **Streaming pipelines** (Flink/Kafka Streams over the
event stream, m08) write **fresh** features straight to the **online store** (and log them to the offline
store for future training). At inference, the model service does a **low-latency lookup from the online
store**; at training, the pipeline reads **history with point-in-time joins from the offline store**. The
feature store coordinates both so the values agree.

---

### Q15. How do embeddings fit into a feature store?
**A.** **Embeddings are learned vector features** (your AI/ML m05 work) — e.g. a user embedding, an item
embedding, a text embedding. They're computed (often by a separate model/job), then **stored and served
like any feature**: the latest embedding per entity in the **online store** for low-latency lookup, history
in the **offline store** for training. The wrinkle is that embeddings **change when their model is
retrained** (versioning matters — train and serve must use the same embedding version, or you reintroduce
skew). Vector/ANN serving (m18-style) may back the lookup if you need nearest-neighbor over them (m24).

---

### Q16. Do you always need a full feature store?
**A.** No — for a **simple system with one model and a few features**, a feature store is over-engineering;
you might just compute features in the request path or a single shared library, carefully kept consistent.
The feature store earns its complexity when you have **many models/teams** (reuse + governance matter),
**both batch and streaming features**, **real point-in-time-correctness needs**, or **scale** that demands
a fast online store. Like everything (m01), match the tool to the requirement — but **the *concepts*
(consistency, point-in-time) apply even without the product**.

---

### Q17. How do you guarantee the batch and streaming versions of a feature agree?
**A.** This is the **Lambda duplication risk** — if "txns in last hour" is computed one way in the batch
backfill and another way in the streaming job, they diverge. Defenses: **share the aggregation logic**
(one definition compiled to both, or Kappa's single stream path), **reconcile** (periodically compare batch
vs streaming values and alert on drift — like reconciliation in m21), and **define features
declaratively** in the feature store so the framework generates both implementations consistently. Avoiding
two hand-written implementations is the whole point.

---

### Q18. How does this module reuse the earlier building blocks?
**A.** Heavily: the **offline/online dual-store** is the **m04 OLTP-vs-OLAP** split; the **online store** is
a **cache/KV** (m06, Redis/Dynamo); **streaming feature pipelines** are **Kafka + stream processing**
(m08); **batch pipelines** read the **data warehouse/lake** (m04); **sharding** the online store at scale is
m05; and **monitoring** features is m10/m28. A feature store is **the Parts 0–2 toolkit assembled for ML
features** — which is exactly why those came first. You're not learning new infra; you're specializing it.

---

### Q19. What governance/compliance concerns apply to features?
**A.** **Ownership + versioning** (who owns a feature, which version a model uses — a version mismatch
between train and serve reintroduces skew), **lineage** (what raw data + transforms produced it, for
debugging/audit), and **PII/PHI handling** (a feature derived from sensitive data inherits its compliance
constraints — directly relevant to your healthcare claims). The feature store's **registry** is where this
governance lives. In a regulated domain you also need **access controls** and an **audit trail** on
features, just like the payment ledger (m21).

---

### Q20. Pitch a feature store in one paragraph to a skeptical engineer.
**A.** *"Right now, our data scientists compute features one way for training and our engineers
re-implement them for serving — so they silently diverge (**training-serving skew**), and when we build
training data we accidentally use future feature values (**point-in-time leakage**). Both make models that
look great offline and fail in production, and we can't reuse features across teams. A **feature store**
fixes all of it: **define a feature once**, it's served to both training and serving identically, training
uses **point-in-time-correct** history, the **online store** serves it in milliseconds, and a **registry**
lets every team reuse it. It's the m04/m06/m08 infra we already run, specialized so features stop being our
silent failure mode."*
