# m19 — Cross-Questions ("if-and-buts") on Video Streaming

> Answer out loud in 2–3 sentences before reading the model answer. Bandwidth/CDN, the transcode pipeline,
> and adaptive bitrate are where this is won.

---

### Q1. What's the single biggest challenge in a video system — storage or bandwidth?
**A.** **Bandwidth (egress).** Storage is comparatively cheap and bounded; **delivery is exabyte-scale** —
billions of watch-hours at multi-Mbps each. So the design **centers on the CDN** (serve from edge caches
near users) and on **cutting bytes** (better codecs, per-title encoding, right-sized bitrates). "Video is a
bandwidth problem, not a storage problem" is the framing — everything optimizes the egress bill.

---

### Q2. Why do you transcode an uploaded video into multiple versions?
**A.** Because viewers have **wildly different devices and bandwidths** — a phone on 3G can't play the same
4K stream as a TV on fiber. So you transcode the source into many **renditions**: multiple **resolutions**
(240p…4K) × **codecs** (H.264/H.265/AV1), so each viewer gets an appropriate one — and so **adaptive
bitrate** has multiple qualities to switch between. One-size-fits-all would either buffer on slow links or
waste bandwidth on fast ones.

---

### Q3. How do you transcode fast (a 1-hour video is a long encode)?
**A.** **Segment and parallelize.** Chop the video into **small chunks** (a few seconds each) and
**transcode the chunks concurrently** across a worker fleet (a **queue + workers**, m08), then reassemble.
This turns a long **serial** encode into hundreds of independent chunk-jobs done in **parallel** — minutes
instead of hours. It's the classic "parallelize by partitioning the work" move, and the whole pipeline is
**async** so the uploader isn't blocked.

---

### Q4. Walk me through what happens after a user uploads a video.
**A.** The upload goes (via **multipart upload**, m18) to an **object store** as the raw source, and the
user immediately sees "**processing…**" (it's **async**, m08 — they don't wait). The **transcode pipeline**
then validates it, **splits it into segments**, **transcodes each segment in parallel** into all target
bitrates/codecs, **packages** them into HLS/DASH with a **manifest**, stores segments + manifest in the
object store, and flips the **metadata to "ready."** Popular/creator content may be **pre-warmed** into the
CDN.

---

### Q5. What is adaptive bitrate streaming and why does it matter?
**A.** **ABR** encodes the video at **multiple bitrates**, chops each into small **segments** (~2–10 s),
and lists them in a **manifest** (HLS `.m3u8` / DASH `.mpd`). The **player chooses which bitrate to fetch
for the next segment** based on its **current bandwidth + buffer**, switching up or down per segment. It
matters because it lets playback **degrade quality instead of buffering** when bandwidth drops — a blurry
moment beats a spinning wheel — which is the core of smooth playback on real, fluctuating networks.

---

### Q6. In ABR, who decides the quality — the server or the client?
**A.** The **client (player)**. ABR inverts the usual model: the server just serves **static segments at
every bitrate**, and the **player decides** which bitrate to pull per segment, reacting to real-time
bandwidth/buffer. This is smart because the client has the best, most current view of its own conditions,
and it keeps the server simple (static file delivery, perfect for a CDN). You pre-encode every bitrate
precisely so the client always has a choice to make.

---

### Q7. Why is the CDN so central to a video system?
**A.** Because video segments are **static, immutable files** and **egress dominates** — you must serve
them from **edge caches near users**, not from a distant origin (which would be slow and ruinously
expensive). The CDN handles the exabyte-scale delivery, absorbs the **heavily skewed popularity** (cache
hot content hard), and cuts origin bandwidth to near-zero for popular content. For video, **the CDN *is*
the architecture** — the rest (transcode, storage) feeds it.

---

### Q8. What is Netflix Open Connect and why is it clever?
**A.** Netflix puts its **own cache appliances *inside* ISP networks**, so popular content is served from a
box **in the viewer's ISP** — it often never crosses the public internet backbone. It's the ultimate
"**move the data to the user**" (m03 speed-of-light): minimal latency, minimal backbone bandwidth, better
quality, lower cost for both Netflix and the ISP. It's CDN taken to its logical extreme — caching as
physically close to viewers as possible.

---

### Q9. How do you handle the fact that a few videos get most of the views?
**A.** **Popularity is extremely skewed** (a tiny fraction of videos = most views — a hot-key/celebrity
pattern, m05/m06/m13). You **cache the hot content hard at the edge** (and **pre-warm** new releases via
push CDN), so popular videos are always served from a nearby cache; the **long tail** (rarely watched) stays
in **cheap cold origin storage** and is fetched on demand. Edge caching turns the skew from a problem into
a cost-saver — you cache exactly the small set that matters.

---

### Q10. How do you reduce buffering and speed up startup?
**A.** Several levers: serve the **manifest + first segments from the nearest edge** (low RTT); start at a
**lower initial bitrate** and ramp up once the buffer fills (fast start, then quality); use **small
segments** so the player adapts quickly; **pre-warm** popular content; and keep a healthy client **buffer**.
ABR itself prevents most buffering by dropping quality under bandwidth stress. You measure **rebuffer ratio**
and **startup time** as the key UX metrics (m10).

---

### Q11. What's the role of codecs, and what's the trade-off?
**A.** Codecs determine **file size → bandwidth cost**: newer codecs (**H.265/HEVC**, **AV1**) compress
much better than **H.264**, cutting egress (the dominant bill) substantially. The trade-offs: **more encode
CPU** (slower/costlier transcoding) and **device/browser support + licensing** concerns (H.264 plays
everywhere; AV1 is newer). So you often encode **multiple codecs** and serve the best one a device supports —
spending compute to save far-larger bandwidth costs.

---

### Q12. What is per-title (or per-scene) encoding?
**A.** Instead of a fixed bitrate ladder for every video, **tune the bitrate to each video's (or scene's)
complexity**: a simple cartoon or talking-head needs far fewer bits for the same quality than a fast action
scene. **Netflix's per-title encoding** analyzes each title and picks an optimal bitrate ladder, squeezing
the **egress bill** without hurting perceived quality. It's a pure bandwidth-cost optimization — spend
analysis/encode compute once to save delivery bytes on every single view.

---

### Q13. Where do the video segments actually live?
**A.** In an **object store** (S3/MinIO, m18) — they're **large, immutable, write-once blobs** fetched by
key, exactly the object-store pattern. The **CDN caches** them at the edge; the object store is the
**origin**. **Metadata** (title, available renditions, manifest location, view count) lives in a separate
**database** (the m04/m18 metadata-vs-blobs split). Cold, rarely-watched renditions tier down to cheaper
archival storage.

---

### Q14. How is live streaming different from on-demand video?
**A.** **Latency.** On-demand can **pre-transcode** the whole video offline and serve from cache; **live**
must **transcode in real time** as the stream arrives (ingest → real-time encode → segment → CDN) with a
tight **end-to-end latency** budget (seconds, sometimes sub-second for interactive). You use **smaller
segments** (lower latency but more overhead), and the latency-vs-quality-vs-cost trade is sharper. You also
can't cache the future, so the CDN fills as segments are produced. It's the harder, lower-latency cousin of
on-demand.

---

### Q15. How do you design the upload to be reliable for large files?
**A.** **Multipart upload** (m18): split the file into parts, upload them **in parallel** (throughput) and
**independently retryable** (resume a failed part, don't restart the whole upload), then assemble. Combined
with **client-side checksums** for integrity and a **resumable** protocol, a multi-GB upload survives flaky
networks. The upload writes to the object store; transcoding then proceeds **async** from there.

---

### Q16. Designing YouTube vs Netflix — what's different?
**A.** **YouTube (UGC):** anyone uploads, so the **upload + transcode pipeline is huge** and continuous,
plus **content moderation + copyright (Content ID)** and search/recommendations over an enormous catalog.
**Netflix (curated):** a **fixed, professionally-produced catalog**, so it's **watch-heavy** (few uploads),
optimized hard for delivery (**Open Connect, per-title encoding, DRM**). Both are CDN-centric with adaptive
streaming; the difference is **UGC ingest scale + moderation (YouTube)** vs **delivery optimization +
licensing/DRM (Netflix)**.

---

### Q17. How does recommendation/search fit into a video platform?
**A.** They're **separate systems layered on top** of the storage/delivery core: **search** over the
catalog (inverted index + ranking, m16 — like your Sennzo multi-filter search), and **recommendations**
(what to watch next — candidate generation + ranking, m24) driving most viewing. They consume **metadata +
watch/engagement data** and shape demand (which content gets hot → what you cache). The video pipeline
delivers; recs/search decide **what** people watch — both matter, but they're distinct subsystems.

---

### Q18. What metrics do you monitor for a video service?
**A.** UX-centric: **rebuffer ratio** (% of playback spent buffering — the key quality metric), **startup
time** (time to first frame), **bitrate distribution** (are viewers getting good quality?), and **play
failures**. Infra: **CDN cache hit ratio**, **egress cost per region** (the bill), **transcode queue lag**,
storage growth. You alert on rebuffer/startup regressions and CDN hit-ratio drops — those directly hit user
experience and cost (m10).

---

### Q19. How do you protect licensed content (DRM)?
**A.** **Encrypt the segments** and use **DRM** (Widevine/PlayReady/FairPlay): the player obtains a
**license/decryption key** from a license server (after auth/entitlement checks) to decrypt and play, and
the key handling is hardware-backed on the device to prevent extraction. This stops casual copying of
premium content (Netflix). UGC platforms instead focus on **copyright detection** (YouTube's Content ID
fingerprints uploads against known works) and **moderation**. Different content models → different
protection.

---

### Q20. Summarize why this design reduces to "feed a CDN."
**A.** Because the read path is **immutable static segments** and **egress is the dominant cost/challenge**,
the entire architecture exists to **get the right segment from an edge cache near the user**: transcoding
**produces** the segments (many bitrates/codecs) and stores them in an object store **origin**, ABR lets the
**client pull the right one**, and the **CDN serves them at the edge** (with hot content cached hard, Open
Connect inside ISPs). Storage and compute feed the CDN; the CDN is what viewers actually hit. "Cut the
bytes, cache at the edge, let the client adapt" is the whole game.
