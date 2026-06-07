# Module 19 — Design Video Streaming (YouTube / Netflix) ⭐⭐

> Upload, process, store, and stream video to **billions** of viewers smoothly. The headline lesson:
> **video is a *bandwidth* problem, not a storage problem** — the dominant cost and challenge is
> delivering **petabytes of egress**, which makes the **CDN** the whole game. Two pipelines: the
> **upload→transcode** write path (parallelized via queues, m08, into many renditions) and the
> **adaptive-bitrate streaming** read path (the client adapts quality to its bandwidth). Your **Sennzo
> movie platform (500k+ titles)** is the content-catalog side of this.

> **Format (Part 2):** worked walkthrough; **transcoding**, **adaptive bitrate (ABR)**, and **CDN** are
> the deep-dives.

---

## Step 1 — Requirements

**Functional (core):**
1. **Upload** a video; **process** it (transcode into multiple resolutions/formats); make it watchable.
2. **Stream/watch** with **smooth playback** (minimal buffering, fast startup) on any device/bandwidth.
3. *(Stretch)* search, recommendations, comments/likes, live streaming.

**Non-functional (these drive it):**
- **Massive scale + global:** hundreds of millions of viewers worldwide.
- **Low buffering + fast startup:** the core UX — playback must be smooth and start in <~1–2 s.
- **Huge storage + (especially) bandwidth:** PB of content, **far more PB of egress** — *bandwidth is
  the dominant cost.*
- **High availability + durability:** content must always play; never lose uploads.
- **Cost-efficient:** at this bandwidth/storage scale, cost is a first-class design constraint.

> **The defining insight:** **storage is cheap; egress/bandwidth is expensive and dominant** → the design
> centers on the **CDN** (deliver from the edge) and on **cutting bytes** (better codecs, right-sized
> bitrates). The compute challenge is **transcoding**; the UX challenge is **adaptive bitrate**.

---

## Step 2 — Estimation (bandwidth is the scary number) ⭐
Assume **1M videos uploaded/day**, **avg 10 min**, and **~1B watch-hours/day**.
- **Storage:** each video is transcoded into **multiple renditions** (240p…4K) × **codecs**, each ~hundreds
  of MB–GBs. 1M/day × (say) ~1–5 GB total across renditions = **~PBs/day** of new storage → **object
  store** (m18) + lifecycle tiering (hot vs cold).
- **⭐ Egress bandwidth (the real number):** ~1B watch-hours/day at avg **~3 Mbps** ≈ 1B × 3600 s × 3
  Mbit/8 ≈ **~1.3 *exabytes*/day** of delivery (order-of-magnitude). **This is why CDN is the whole
  game** — serving this from origin is impossible/ruinous; it must come from edge caches near users.
- **Transcode compute:** 1M videos/day × (minutes of CPU/GPU per rendition × several renditions) → a
  large, **parallelized** transcoding fleet.

> **The estimate frames it:** PB-scale storage (→ object store + tiering), **exabyte-scale egress** (→
> CDN, the dominant cost), and heavy transcode compute (→ parallel pipeline). **Watch-bandwidth ≫
> everything else.**

---

## Step 3 — High-level design (two pipelines) ⭐

```
  UPLOAD / WRITE PATH (async):
   uploader ─▶ (multipart upload) ─▶ Object store (raw) ─▶ [Transcode pipeline]
                                                              │ split into segments
                                                              │ transcode each → many bitrates/codecs (parallel, m08)
                                                              │ package (HLS/DASH) + generate manifest
                                                              ▼
                                                       Object store (segments + manifests) ─▶ Metadata DB
                                                                                              "processing → ready"

  WATCH / READ PATH:
   viewer ─▶ get manifest ─▶ player picks a bitrate ─▶ fetch segments from CDN edge ──HIT──▶ play
                                                            │ MISS → origin (object store) → cache → play
                                          (player keeps adapting bitrate per segment to its bandwidth)
```

- **Upload is async** (m08): you upload (multipart, m18), get "**processing…**", and the **transcode
  pipeline runs in the background** — the user doesn't wait. When done, metadata flips to "**ready**."
- **Watch** is mostly **static segment delivery via CDN** — the player fetches a **manifest**, then pulls
  small **segments** from the nearest edge, **adapting the bitrate** as it goes.

---

## Step 4 — Deep Dive: the transcode pipeline ⭐

**Why transcode?** A raw upload is one format/resolution, but viewers have **wildly different devices and
bandwidths** (a phone on 3G vs a 4K TV on fiber). So you **transcode** the source into **many renditions**
— multiple **resolutions** (240p, 480p, 720p, 1080p, 4K) × **codecs** (H.264, H.265/HEVC, AV1) — so each
viewer gets an appropriate one.

**The pipeline is a parallelized DAG of jobs:**
1. **Validate** the upload (format, malware, duration).
2. **Split** the video into **segments** (chunks of a few seconds).
3. **Transcode each segment in parallel** across a worker fleet (a **queue + workers**, m08) into each
   target bitrate/codec — *segmenting is what makes transcoding fast*: a 1-hour video becomes hundreds of
   independent chunk-jobs done concurrently, not one long serial encode.
4. **Package** the segments into streaming formats (**HLS / DASH**) and generate the **manifest**.
5. **Store** segments + manifests in the **object store** (m18); update **metadata** → "ready"; warm the
   CDN for likely-popular content.

> **Say this:** *"Transcoding is parallelized by **segmenting** — chop the video into chunks and transcode
> them concurrently across a worker fleet via a queue, then reassemble — turning a long serial encode into
> a fast parallel one. It's async (m08), so the uploader sees 'processing' and the pipeline runs in the
> background, producing many resolution/codec renditions for adaptive streaming."*

> **Cost lever — codecs & per-title encoding:** a better codec (**H.265/AV1** vs H.264) cuts file size
> (and thus **bandwidth cost**) substantially, at the cost of more **encode CPU** + device-support
> concerns. **Netflix's per-title (and per-scene) encoding** tunes the bitrate to each video's complexity
> (a cartoon needs fewer bits than an action scene) — squeezing the egress bill. Bytes = money at this
> scale.

---

## Step 5 — Deep Dive: Adaptive Bitrate Streaming (ABR) ⭐⭐ (the key streaming concept)

How do you stream smoothly when a viewer's bandwidth **fluctuates** (Wi-Fi → cellular, congestion)?
**Adaptive Bitrate Streaming** (HLS/DASH):
- The video is encoded at **multiple bitrates** and each is chopped into small **segments** (~2–10 s).
- A **manifest file** (HLS **`.m3u8`** / DASH **`.mpd`**) lists the available bitrates and the segment
  URLs at each.
- The **player (client) chooses which bitrate to fetch for the *next* segment** based on its **current
  measured bandwidth + buffer level** — and can **switch up or down per segment**.

```
   manifest lists:   240p segments | 480p | 720p | 1080p | 4K
   player on good wifi → fetch 1080p segments
   bandwidth drops    → next segment fetched at 480p (no buffering, just lower quality)
   bandwidth recovers → back up to 1080p
```

> **Think differently — the *client* adapts, not the server.** ABR inverts the usual model: the **player
> decides** which quality to pull, segment by segment, reacting to real-time conditions. The result:
> **degrade quality instead of buffering** (a momentarily blurry frame beats a spinning wheel). The server
> just serves static segments; the intelligence is at the edge/client. This is why playback feels smooth
> on a flaky connection — and why you pre-encode every bitrate so the client always has a choice.

---

## Step 6 — Deep Dive: the CDN (the whole game) ⭐⭐

Video segments are **static, immutable files** → perfect for a **CDN** (m06). Since **egress dominates**:
- **Serve segments from edge PoPs near users** — a watch request should hit a nearby cache, **never the
  origin**. Cache hit ratio is everything (a miss to a distant origin = latency + cost).
- **Popularity is extremely skewed:** a tiny fraction of videos get the vast majority of views → **cache
  the hot content hard at the edge**; the **long tail** (rarely watched) stays in cheaper cold origin
  storage. (The hot-key/celebrity thread again, m05/m06/m13 — solved by edge caching.)
- **Netflix Open Connect (the famous move):** Netflix puts its **own cache appliances *inside ISP*
  networks** — so popular content is served from a box in *your ISP*, not even crossing the public
  internet. The ultimate "move the data to the user" (m03 speed-of-light) — it slashes backbone bandwidth
  and improves quality.
- **Versioned/immutable URLs** → no CDN invalidation needed (m06/m18); segments never change.

> **The senior line:** *"Video is a bandwidth problem, so the CDN *is* the architecture — serve immutable
> segments from edge caches near users, cache the heavily-skewed popular content hard, and tier the cold
> tail to cheap origin. At Netflix's extreme, push caches inside ISPs (Open Connect). Storage is cheap;
> egress is the bill you're optimizing."*

---

## Step 7 — Bottlenecks, scale & edge cases

- **Bandwidth cost (the #1 issue):** CDN + edge caching + better codecs + per-title encoding + right-sized
  bitrates. Everything here ultimately serves cutting egress.
- **Transcode compute:** parallel (segment-level) + queue-based + autoscale; prioritize popular/creator
  content; use GPUs/hardware encoders.
- **Popularity skew (hot videos):** edge-cache hard, pre-warm a new release (push CDN, m06); cold tail →
  cheap storage.
- **Startup latency:** fast manifest + first-segment delivery from edge; small initial segment / lower
  initial bitrate then ramp up.
- **Storage:** object store (m18) for segments/renditions; lifecycle tiering (hot → cold → archive);
  delete unwatched renditions of dead content.
- **Live streaming (a different beast):** **low latency** (seconds), can't pre-transcode the whole thing →
  ingest → **real-time transcode** → segment → CDN with low buffer; latency vs quality trade is tighter.
  (Mention as an extension.)
- **Metadata/recommendations/search:** the catalog, view counts, and recs are separate systems (search
  m16, recs m24, feed-like ranking m13) layered on top.
- **DRM/security:** licensed content needs DRM/encryption (Netflix); UGC needs content moderation +
  copyright (Content ID).
- **Observability (m10):** rebuffer ratio, startup time, CDN hit ratio, bitrate distribution, transcode
  queue lag, egress cost per region.

**Wrap-up:** *"Two pipelines. **Upload→transcode** (async, m08): multipart-upload the source to an object
store, then a **parallel, segment-level transcode** fleet produces many **bitrate/codec renditions**,
packages them as **HLS/DASH** with a **manifest**, and stores segments in the object store (m18). **Watch**
(read path): the player fetches the **manifest** and pulls small **segments from a CDN edge**, using
**adaptive bitrate** — the *client* picks quality per segment by its bandwidth, degrading quality instead
of buffering. The **CDN is the architecture** because **egress dominates**: serve immutable segments from
the edge, cache the heavily-skewed popular content hard (Netflix even puts caches *inside ISPs*), tier the
cold tail, and cut bytes with better codecs/per-title encoding. Storage cheap, bandwidth the bill."*

---

## State-of-the-art & real-world notes 📚
- **YouTube** (UGC: huge upload + transcode pipeline) vs **Netflix** (curated: watch-heavy, **Open
  Connect** ISP caches, **per-title encoding**). Both are CDN-centric.
- **HLS** (Apple) and **MPEG-DASH** are the adaptive-streaming standards (segments + manifest).
- **Codecs:** **H.264** (universal), **H.265/HEVC** + **AV1** (smaller, more CPU/licensing) — the
  bandwidth-vs-compute lever.
- **Netflix Open Connect** and **Akamai/Cloudflare/CloudFront** CDNs; **Netflix's per-title/per-scene
  encoding** is the canonical "cut bytes" case study.

---

## From your systems 🏭
- **Sennzo (500k+ titles):** you built a **movie platform** with **multi-filter search** — the **catalog/
  metadata + search** layer of a streaming service (m16). You've built the "browse/discover" half; this
  module adds the upload/transcode/deliver half.
- **Object store for segments (m18):** segments live in **object storage** (S3/MinIO — your DCE
  experience) — immutable blobs by key, exactly the m18 pattern (and CDN-fronted, m06).
- **Async transcode pipeline = your Kafka/Celery:** the segment-transcode fleet is a **queue + worker**
  pipeline (m08) — your DCE/Questimate pattern (producer enqueues jobs, workers process in parallel).
- **"Move data to the user":** the CDN/Open-Connect idea is the m03 speed-of-light lesson taken to the
  extreme — the same reason you'd put a replica/cache near users.

---

## Key concepts (interview-ready)
- **Video = a bandwidth problem, not storage:** **egress dominates** → the **CDN is the architecture**;
  storage is cheap. Optimize bytes (codecs, **per-title encoding**, right-sized bitrates).
- **Upload→transcode pipeline (async, m08):** multipart upload → **segment** → **transcode chunks in
  parallel** into many **resolutions × codecs** → package **HLS/DASH** + **manifest** → object store
  (m18). Segmenting is what makes transcoding fast; uploader sees "processing."
- **Adaptive Bitrate Streaming (ABR):** multiple bitrates, small **segments**, a **manifest** (`.m3u8`/
  `.mpd`); the **client picks quality per segment by its bandwidth** → **degrade quality, don't buffer.**
  Intelligence at the client, static segments at the server.
- **CDN:** serve immutable segments from edge PoPs near users; **cache heavily-skewed popular content
  hard**, tier the cold tail; **Netflix Open Connect = caches inside ISPs**; versioned URLs (no
  invalidation).
- **Bottlenecks:** bandwidth cost (#1), transcode compute (parallel + autoscale), popularity skew (edge
  cache/pre-warm), startup latency, live streaming (low-latency variant), DRM/moderation.

---

## Go deeper (reading)
- **Netflix Tech Blog:** "Open Connect," "Per-Title Encode Optimization," and their encoding/CDN posts —
  the canonical bandwidth-optimization case studies.
- **HLS (Apple) and MPEG-DASH** specs/overviews — adaptive streaming (segments + manifest).
- **YouTube/Google talks on the video processing pipeline** — UGC transcode at scale.
- **System Design Interview (Alex Xu): "Design YouTube"** — the upload/transcode/CDN treatment.
- Revisit **m06** (CDN/hot keys), **m08** (async transcode pipeline), **m18** (object store for segments),
  **m03** (speed of light / edge), **m16/m24** (search/recs layer).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) before
checking [`solutions/m19_video_streaming/`](../../solutions/m19_video_streaming/README.md).

When you've done the exercises, say **"Module 20"** to design *Proximity / Ride-hailing (Uber, Yelp,
Google Maps)* — geospatial indexing (geohash/quadtree/H3) and real-time matching, the last classic
problem before we move into ML system design.
