# m19 — Exercises (Video Streaming)

> Do these on paper / out loud before checking `solutions/m19_video_streaming/`. Bandwidth/CDN, the
> transcode pipeline, and ABR are the core.

---

## Exercise 1 — Estimation (the bandwidth point)
Assume **2B watch-hours/day**, avg **3.5 Mbps**.
1. Rough total **egress per day** (order of magnitude)?
2. Why does this number make the CDN non-optional?
3. Storage is also large — but why is it the *secondary* concern vs egress?

---

## Exercise 2 — The two pipelines
1. Draw the upload/write path and the watch/read path, naming each component.
2. Why is upload/transcoding asynchronous?
3. Where do segments live, and what fronts them for delivery?

---

## Exercise 3 — Transcoding
1. Why transcode into multiple versions at all?
2. How do you transcode a 1-hour video quickly? (The key trick.)
3. Name three target dimensions you produce (resolution, codec, ...).

---

## Exercise 4 — Adaptive bitrate
1. What's in a manifest file, and what format is it (name one)?
2. Who decides which quality to fetch, and based on what?
3. What does ABR do when bandwidth suddenly drops, and why is that better than the alternative?

---

## Exercise 5 — The CDN
1. Why are video segments ideal for a CDN (what property)?
2. How does popularity skew help rather than hurt you here?
3. What is Netflix Open Connect and what problem does it solve?

---

## Exercise 6 — Cutting bytes
1. Codecs: name the bandwidth-vs-compute trade-off (H.264 vs H.265/AV1).
2. What is per-title encoding and what does it optimize?
3. Why is "spend compute to save bytes" the right trade at this scale?

---

## Exercise 7 — Playback quality
1. Name two techniques to reduce startup latency.
2. What metric measures buffering, and why is it the key quality signal?
3. How does ABR itself prevent most buffering?

---

## Exercise 8 — Live streaming
1. What's the fundamental difference from on-demand?
2. What can't you do for live that you can for on-demand?
3. What's the latency-vs-overhead trade with segment size?

---

## Exercise 9 — YouTube vs Netflix + extras
1. Name two design differences between YouTube (UGC) and Netflix (curated).
2. How do search and recommendations relate to the core video pipeline?
3. How do you protect premium content vs detect copyright on UGC?

---

## Exercise 10 — Your-systems tie-in 🏭
1. **Sennzo (500k+ titles):** which half of a streaming service did you build, and which half does this
   module add?
2. Map **segment storage** and the **transcode pipeline** to tools/patterns you've used (object store;
   queue+workers).
3. How is the CDN/Open-Connect idea the same lesson as a concept from m03?

---

When done, check `solutions/m19_video_streaming/README.md`, then say **"Module 20"**.
