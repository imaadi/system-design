# m36 — Rapid-Fire Recall (the whole curriculum, one-liners)

> Quick-recall drills spanning **all 36 modules**. Cover the answer, say it out loud in **one or two
> sentences**, reveal, move on. Do a **15-minute block daily** in your prep window. If any feels shaky,
> re-read that module. (These are deliberately terse — the *depth* is in m01–m35.)

---

## Part 0 — Framework & fundamentals (m01–m02)
1. **The 7-step framework?** → Reqs/NFRs → estimate → API → data model → high-level → deep-dive →
   bottlenecks/trade-offs.
2. **First thing in any answer?** → Clarify requirements + **which NFR dominates** (scale/latency/
   consistency/availability), then scope down.
3. **CAP?** → Under a partition, choose **Consistency or Availability**. Most systems are AP with tunable
   consistency; pick **per feature**.
4. **Strong vs eventual consistency — when each?** → Strong for money/inventory/auth-revoke; eventual for
   feeds/likes/analytics.
5. **Latency: RAM vs disk vs cross-continent?** → ~100ns / ~100µs(SSD)–10ms(disk) / ~150ms. Network
   dominates.
6. **Little's Law?** → L = λ × W (concurrency = arrival-rate × time-in-system). Sizing + queue intuition.
7. **Why does linear scaling break down (USL)?** → Coordination + contention are the tax; adding nodes has
   diminishing (then negative) returns.

## Part 1 — Building blocks (m03–m10)
8. **REST vs gRPC vs GraphQL?** → public/simple / internal-low-latency / client-shaped aggregation.
9. **WebSocket vs SSE vs long-poll?** → bidirectional / one-way server-push (LLM streaming!) / fallback.
10. **SQL vs NoSQL?** → relations+txns vs scale+flexible-schema+known-access-pattern; polyglot by access
    pattern.
11. **Replication: sync vs async?** → sync = no data loss, higher latency; async = fast, risk loss on
    failover.
12. **Sharding key — what matters?** → even distribution + queries hit one shard; avoid hotspots; resharding
    is painful → consistent hashing.
13. **SQL scaling order?** → index → cache → read replicas → vertical → shard (last).
14. **Cache-aside vs write-through vs write-back?** → lazy/read-heavy / sync-write / async-write (fastest,
    loss risk).
15. **Cache invalidation?** → TTL + event-based; the hard problem. Watch thundering herd + stampede.
16. **L4 vs L7 load balancer?** → transport (fast, opaque) vs application (content/path routing, smarter).
17. **Kafka vs RabbitMQ?** → log/replay/high-throughput/stream vs flexible routing/per-message queue.
18. **At-least-once + idempotency = ?** → effectively-once (dedup keys); exactly-once is mostly a myth.
19. **Quorum (R + W > N)?** → overlapping read/write sets → read sees latest write; tune R/W for consistency
    vs latency.
20. **Circuit breaker / retry+backoff / bulkhead?** → stop hammering a dead dep / retry transient with
    jitter / isolate failure.
21. **The four golden signals?** → latency, traffic, errors, saturation.

## Part 2 — Classic problems (m11–m21)
22. **URL shortener — the crux?** → read-heavy + latency → cache-fronted KV + pre-generated keys; analytics
    async.
23. **Rate limiter — the algorithm?** → token bucket (allows bursts); distributed → Redis + atomic
    Lua/INCR.
24. **News feed — push vs pull?** → fan-out-on-write for most, pull for celebrities → hybrid.
25. **Chat — what dominates?** → persistent WebSocket connections + session registry + pub/sub backplane +
    ordering.
26. **Notifications — the must-have?** → idempotency/dedup + retries + per-channel providers + user prefs.
27. **Typeahead — the structure?** → prefix trie / FST + top-k cached per node; edge-served; eventually
    updated.
28. **Web crawler — the hard parts?** → politeness/dedup (URL + content)/frontier prioritization/traps.
29. **Object store (S3) — the model?** → flat namespace + metadata index + erasure coding + replication.
30. **Video streaming — the key idea?** → transcode to ABR ladder + HLS/DASH chunks + CDN.
31. **Proximity (Uber/Yelp) — the index?** → geospatial (geohash / quadtree / S2/H3) for "nearby."
32. **Payments — the #1 rule?** → correctness > availability; **double-entry ledger + idempotency +
    reconciliation**; down-beats-wrong.

## Part 3 — ML system design (m22–m28)
33. **The #1 ML-SD difference?** → it **fails silently** (the world drifts, the model doesn't) → you must
    monitor + retrain.
34. **Training-serving skew?** → features computed differently in training vs serving → a feature store
    (one definition).
35. **Point-in-time correctness?** → only use features known as-of the event time; else label leakage.
36. **Online vs batch serving?** → real-time low-latency vs throughput; precompute when you can.
37. **The recsys funnel?** → retrieval (candidate gen, two-tower/ANN) → ranking → re-ranking (diversity/
    business).
38. **CTR — calibration: why ads need it?** → the auction multiplies bid × **absolute** pCTR; AUC is
    calibration-invariant.
39. **Fraud — the 4 inversions?** → extreme imbalance / delayed-noisy labels / adversarial drift /
    asymmetric cost.
40. **Rules+ML hybrid + 3-way decision?** → rules (fast/explainable/guardrail) + ML (fuzzy) → allow/block/
    review (DCE!).
41. **MLOps = ?** → DevOps + data & model dimensions; version code+data+model; eval-gated retraining (CT).

## Part 4 — LLM system design (m29–m34)
42. **LLM serving — the inversion?** → memory-bound + autoregressive (not compute, not one pass).
43. **KV cache — why it matters?** → makes decode cheap but grows with seq-len, dominates memory →
    PagedAttention.
44. **Continuous batching?** → token-level add/remove keeps the GPU saturated (10–20×) despite variable
    output lengths.
45. **TTFT vs TPOT?** → time-to-first-token (≈prefill, perceived; stream it) vs time-per-output-token
    (decode).
46. **RAG = knowledge, fine-tune = ?** → behavior. Order: prompt → RAG → fine-tune.
47. **RAG #1 lever + #1 failure?** → chunking (structure-aware); "most hallucinations are retrieval misses
    — fix retrieval first."
48. **RAG eval metrics?** → faithfulness + context precision/recall + answer relevancy (RAGAS), CI-gated.
49. **Chatbot — the core problem?** → LLM is stateless → "memory" = context-window management (recent +
    summary + retrieved).
50. **Agent = ?** → LLM(reasoning) + tools(actions) + loop + state; reliability is the hard part (error
    compounding).
51. **Tool descriptions decide ?** → routing (vague → wrong tool; my 3/6 vs 6/6).
52. **Semantic caching?** → cache by meaning (embed query → similarity search); risk = false hits → tune
    threshold.
53. **Model routing?** → cheap-model-first/escalate (cost varies ~100×) — the dominant LLM cost lever.
54. **Prompt injection — solved?** → No (esp. indirect); defend in depth + **limit blast radius**
    (least-privilege + human approval).
55. **LLM eval — why hard?** → open-ended, non-deterministic, no ground truth → LLM-as-judge (biased) +
    human + proxies.

## Part 5 — Your systems & interview (m35–m36)
56. **Your DCE one-liner?** → ~1M claims/day rules+ML hybrid (exclusions→models→suppressions), correctness
    #1, $675K→$50K.
57. **Your Questimate one-liner?** → 4 Django microservices, Redis-JWT (stateless+revoke), snapshot
    precompute → O(1) reads.
58. **Turn "have you done X?" into?** → "Yes — on [DCE/Questimate/InsightDesk] I…" + a specific + a number.
59. **The senior tells?** → quote a number, name the trade-off, "it depends on X," surface a failure mode,
    name the pattern.
60. **The one thing to never skip?** → clarify requirements + which NFR dominates, *before* designing.

---

> Score yourself: **50+/60 cold = interview-ready.** Any miss → re-read that module. Re-run this list
> before each interview.
