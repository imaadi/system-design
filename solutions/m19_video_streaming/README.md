# m19 — Solutions (Video Streaming)

> Check the *reasoning*. Recurring lessons: video = bandwidth problem → CDN is the architecture; transcode
> = parallel by segmenting; ABR = client adapts quality per segment; cut bytes (codecs/per-title).

---

## Exercise 1 — Estimation
2B watch-hours/day, 3.5 Mbps.
1. **Egress:** 2B × 3600 s × 3.5 Mbit ÷ 8 ≈ **~3.15 ZB?** — let's keep it order-of-magnitude: 2B hrs ×
   3.5 Mbps ≈ **petabytes-to-exabytes per day** (≈ 2×10⁹ × 3600 × 3.5×10⁶ bits ÷ 8 ≈ ~3×10¹⁸ bytes ≈
   **~3 exabytes/day**). The point: it's **astronomical**.
2. Serving exabytes/day **from origin is impossible/ruinous** (origin bandwidth + cross-continent latency)
   — it must come from **edge caches near users**. The CDN isn't an optimization here; it's the only way to
   deliver at all affordably.
3. Storage (PB-scale) is **bounded and cheap per GB**, and you write each video once; **egress is unbounded
   and expensive** (every view re-sends bytes). You pay for delivery **per view**, so bandwidth dominates.

---

## Exercise 2 — The two pipelines
1. **Write:** uploader → multipart upload → object store (raw) → **transcode pipeline** (segment →
   transcode chunks in parallel → package HLS/DASH + manifest) → object store (segments + manifests) →
   metadata "ready." **Read:** viewer → get **manifest** → player picks bitrate → fetch **segments from CDN
   edge** (miss → origin object store) → play, adapting bitrate.
2. **Async** because transcoding into many renditions takes minutes and the uploader shouldn't wait — they
   see "processing," and the pipeline (queue + workers, m08) runs in the background, flipping to "ready"
   when done.
3. **Segments live in the object store (m18)** (immutable blobs); the **CDN** fronts them for edge delivery.

---

## Exercise 3 — Transcoding
1. Viewers have **different devices/bandwidths** — a phone on 3G vs a 4K TV — so you produce versions each
   can play, and ABR needs multiple qualities to switch between.
2. **Segment + parallelize:** chop into small chunks and **transcode chunks concurrently** across a worker
   fleet (queue, m08), then reassemble — serial hours → parallel minutes.
3. **Resolution** (240p…4K), **codec** (H.264/H.265/AV1), **bitrate** (the ladder) — plus packaging format
   (HLS/DASH).

---

## Exercise 4 — Adaptive bitrate
1. A **manifest** lists the **available bitrates and the segment URLs** at each; formats: **HLS `.m3u8`**
   or **DASH `.mpd`**.
2. The **client/player** decides, based on its **current measured bandwidth + buffer level**, per segment.
3. ABR **drops to a lower bitrate** for the next segment (lower quality, no interruption) — **better than
   buffering** because a momentarily blurry frame beats a spinning wheel / stalled playback.

---

## Exercise 5 — The CDN
1. Segments are **static, immutable files** → perfectly cacheable (no invalidation, versioned URLs), ideal
   for edge caching.
2. **Skew means a small hot set serves most views** → you cache exactly that small set hard at the edge
   and serve most traffic from cache; the rarely-watched long tail stays in cheap cold storage. Skew makes
   caching highly effective.
3. **Open Connect** = Netflix's **cache appliances inside ISP networks**, serving popular content from a box
   *in the viewer's ISP* — minimizing backbone bandwidth + latency (the ultimate "move data to the user,"
   m03).

---

## Exercise 6 — Cutting bytes
1. **H.265/AV1 compress better** (smaller files → less bandwidth) but cost **more encode CPU** + have
   **device-support/licensing** concerns; **H.264** is universal but larger.
2. **Per-title encoding** tunes the bitrate ladder to **each video's complexity** (a cartoon needs fewer
   bits than an action scene), optimizing the **egress bill** without hurting perceived quality.
3. Because **egress (every view) dwarfs encode (once)** — you encode a video a handful of times but deliver
   it millions of times, so spending more compute up front to shrink each delivered byte pays off massively.

---

## Exercise 7 — Playback quality
1. Serve **manifest + first segments from the nearest edge** (low RTT) and **start at a lower bitrate then
   ramp up** (fast first frame); also small segments + pre-warming.
2. **Rebuffer ratio** (fraction of playback time spent buffering) — it directly measures the worst UX
   (stalls), which is what viewers hate most.
3. ABR **lowers quality instead of stalling** when bandwidth drops, keeping the buffer fed — so most
   potential buffering becomes a (tolerable) quality dip instead.

---

## Exercise 8 — Live streaming
1. **Latency** — live must deliver in seconds (or sub-second), so it **transcodes in real time** as the
   stream arrives, vs on-demand which pre-transcodes offline.
2. You **can't pre-transcode or pre-cache the future** — segments are produced as the event happens, so the
   CDN fills on the fly and you can't do leisurely offline optimization.
3. **Smaller segments = lower latency** (the player gets fresh content sooner) **but more overhead/
   requests**; larger segments are more efficient but add latency — a tighter trade than on-demand.

---

## Exercise 9 — YouTube vs Netflix + extras
1. **YouTube (UGC):** huge continuous **upload/transcode** pipeline + **moderation/copyright (Content ID)**.
   **Netflix (curated):** fixed catalog, **watch-heavy**, optimized delivery (**Open Connect, per-title
   encoding, DRM**). (UGC ingest+moderation vs delivery optimization+licensing.)
2. They're **separate systems on top**: **search** over the catalog (m16) and **recommendations** (m24)
   decide **what** people watch (shaping which content gets hot/cached), consuming metadata + watch data;
   the video pipeline **delivers** it.
3. **Premium content → DRM/encryption** (Widevine/PlayReady/FairPlay + license server); **UGC → copyright
   detection** (Content ID fingerprinting) + moderation. Different content models, different protection.

---

## Exercise 10 — Your-systems tie-in 🏭
1. **Sennzo (500k+ titles):** you built the **catalog/metadata + multi-filter search** half (browse/
   discover, m16); this module adds the **upload → transcode → store → deliver** half (the media pipeline +
   CDN).
2. **Segment storage = object store** (S3/MinIO, your DCE experience, m18 — immutable blobs by key, CDN-
   fronted); **transcode pipeline = queue + workers** (Kafka/Celery, your DCE/Questimate async-job pattern,
   m08 — enqueue chunk jobs, workers process in parallel).
3. The **CDN/Open-Connect "cache inside the ISP"** idea is the **m03 speed-of-light** lesson taken to the
   extreme — you can't beat the speed of light, so you **move the data physically close to the user**
   (the same reason you'd place a replica/cache near users).
