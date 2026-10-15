Amazon S3 is used to store the video chunks. Netflix also provides open connect servers to internet service providers, which acts like a cache of movies. Most requests to Netflix can be served by this cache, and the remaining are sent over the network. This reduces the bandwidth and time required for Netflix to operate at scale. Synergy at it's finest.

# Netflix System Design: Video Onboarding and Processing at Scale

Based on the video description you shared. The description covers chunking, shot-to-scene grouping, S3 storage, and Open Connect caching. The rest of the pipeline below is standard design reasoning, so check the details against the video.

---

## 1. Problem Statement

Design the system that takes a newly produced movie or show and makes it available to millions of viewers worldwide. The system must:

1. Ingest raw studio masters (very large, high-quality video files)
2. Split and process them into formats that play on many devices and network speeds
3. Store the processed content durably
4. Deliver it to viewers with low startup time and smooth playback
5. Reduce bandwidth cost at scale

---

## 2. Clarifying Requirements

**Functional**
- Accept uploads of source video from studios and partners
- Break video into chunks that can be processed in parallel
- Encode each chunk into multiple resolutions, bitrates, and codecs
- Detect shots and group them into scenes for metadata and features
- Publish the processed title so the streaming service can serve it
- Serve playback requests from the nearest cache

**Non-functional**
- **Throughput:** onboarding can take a long time for one title, but many titles arrive in parallel
- **Durability:** source masters must never be lost
- **Low startup latency** for viewers
- **Cost efficiency:** bandwidth is the biggest operating expense
- **Fault tolerance:** a failed chunk should be retried without restarting the whole title

**Scale assumptions to state**
- Thousands of titles onboarded per year, each made of many hours of source footage
- Billions of playback requests over time, served mostly from caches close to users
- Each title is encoded into many variants (different resolutions, bitrates, and codecs), so one source file produces hundreds of outputs

Doing this math aloud shows the interviewer you understand why parallelism and caching matter.

---

## 3. High-Level Architecture

| Component | Responsibility |
|---|---|
| Ingest service | Accepts uploads, validates files, records title metadata |
| Object storage (Amazon S3) | Stores source masters, chunks, and encoded outputs |
| Chunking service | Splits the source into fixed-length or shot-aligned segments |
| Encoding workers | Transcode each chunk into multiple formats in parallel |
| Shot and scene detection | Finds shot boundaries and groups shots into scenes |
| Metadata service | Stores title details, encoding variants, scene information |
| Quality control | Checks encoded output for errors, artifacts, or sync issues |
| Publishing service | Marks a title as available and pushes it to the catalog |
| Content catalog and API | What the app queries to show titles and playback options |
| Open Connect appliances | Caches inside ISP networks that serve most playback requests |
| Playback and steering service | Chooses the best cache and bitrate for each viewer |
| Message queue and workflow engine | Coordinates the pipeline stages and retries failed steps |

---

## 4. Video Onboarding Pipeline

1. **Ingest:** the studio uploads the source master to S3. The ingest service records the title, checksum, and status.
2. **Chunking:** the source is split into segments. Shot-aligned chunks avoid cutting in the middle of a scene's visual changes.
3. **Shot detection:** the system analyzes frames to find where one camera shot ends and another begins. Shots are short, around four seconds in the video description.
4. **Scene grouping:** consecutive shots that share a setting or action are collated into scenes. This powers features like skip-intro, scene previews, and search.
5. **Parallel encoding:** each chunk goes to a worker pool. Every worker encodes its chunk into all target variants. Results are written back to S3.
6. **Assembly:** outputs are stitched or indexed so each variant is a playable stream.
7. **Quality checks:** automated checks look for missing frames, audio drift, and encoding failures. Failed chunks are re-encoded.
8. **Publishing:** the metadata service marks the title ready and it appears in the catalog for the regions where it is licensed.

Interview point: chunking is what makes the pipeline parallel. Without it, one long movie would take one machine many hours.

---

## 5. Why Chunking and Parallel Processing Matter

- A two-hour film split into 4-second shots gives about 1,800 units of work.
- Those units can be processed by thousands of workers at once, so the total time drops from hours to minutes.
- Failures are isolated. A bad chunk is retried alone, not the whole movie.
- Work can be scheduled by cost. Cheap spot capacity can process non-urgent variants.

---

## 6. Storage Design

- **Source masters:** stored in S3 with versioning and cross-region replication. These are the most sensitive assets and must not be lost.
- **Encoded outputs:** many variants per title, stored in S3 under a clear key scheme such as title_id, then variant, then chunk.
- **Metadata:** title, episode, scene, and variant records in a relational or document store. The queries are structured and need consistency.
- **Lifecycle policies:** rarely used variants or intermediate chunks can move to cheaper storage tiers.

---

## 7. Delivery: Open Connect and Caching

The description highlights a key idea: most viewers should never hit the central cloud.

- **Open Connect appliances** are servers placed inside ISP networks, and sometimes at internet exchange points.
- Popular titles are pushed to these caches during off-peak hours, based on predicted demand. This is called **proactive fill**.
- When a viewer presses play, the **steering service** picks the best appliance for that viewer, considering network health, load, and what is cached.
- If the title is not cached nearby, the request falls back to a larger cache or the cloud origin.

**Why this works**
- Most traffic is served from inside the viewer's ISP. That cuts network distance, latency, and transit costs.
- The internet backbone carries less traffic, and the ISP also benefits from less peering cost.
- Startup time drops because the nearest cache is only a few hops away.

Interview point: this is a classic cache-hierarchy design. Emphasize that the cache is positioned close to users and filled in advance, not only on demand.

---

## 8. Adaptive Streaming and Playback

- The player downloads video in small segments and switches bitrates depending on current bandwidth. This is adaptive bitrate streaming.
- Multiple variants per title make this possible. Each variant is encoded at a different resolution and bitrate.
- Scene and shot metadata can drive features like the skip-intro button, next-episode preview, and thumbnail selection.

---

## 9. Catalog and Playback Requests

1. The app queries the catalog API for titles available in the viewer's region.
2. The viewer presses play.
3. The playback service authorizes the request, checks licensing, and issues a signed, short-lived URL.
4. The steering service returns the best cache endpoint.
5. The player fetches segments directly from the appliance.

Keep the control plane (catalog, licensing, authorization) separate from the data plane (video segments). Only the control plane needs strict consistency. Video delivery tolerates eventual consistency and benefits from caching.

---

## 10. Reliability and Fault Tolerance

- **Idempotent pipeline stages:** re-running a stage on the same chunk produces the same output, so retries are safe.
- **Workflow engine:** tracks each title's progress and retries failed steps.
- **Checksums:** verify integrity after every upload and transfer.
- **Redundancy:** multiple cache appliances per region. If one fails, the steering service routes around it.
- **Graceful degradation:** if a cache is unavailable, playback falls back to a higher-tier cache or origin, with possibly lower quality.

---

## 11. Likely Follow-Up Questions

**Q: Why split the video into chunks at all?**
A: It enables parallel encoding, isolates failures, and lets the pipeline scale out with more workers.

**Q: Why use shot detection instead of fixed-length chunks?**
A: Shot-aligned chunks avoid splitting a scene mid-action, which reduces encoding artifacts at chunk boundaries and enables scene-level features.

**Q: Why put caches inside ISP networks?**
A: Most playback is served from close to the viewer. This cuts latency and reduces the cost of sending traffic across the internet.

**Q: How do you decide which titles to pre-fill onto caches?**
A: Predict demand from viewing history, regional popularity, and release schedules. Fill during off-peak hours when network capacity is free.

**Q: What if a title suddenly goes viral?**
A: Increase cache fill for that title, and fall back to higher tiers while the caches warm up.

**Q: How do you handle a failed encode for one chunk?**
A: Retry that chunk only, using an idempotent job. Quality checks catch bad output before publishing.

**Q: How do you protect the source masters?**
A: Store them durably in S3 with versioning and replication, restrict access, and keep them separate from delivery copies.

**Q: How would you support a new device format later?**
A: Add a new encoding profile and re-encode from the stored source. Because masters are kept, new formats don't require the studio to re-upload.

---

## 12. Trade-offs to Discuss

| Decision | Benefit | Cost |
|---|---|---|
| Shot-aligned chunking | Clean boundaries, scene features | More complex analysis step |
| Many variants per title | Smooth playback on any device or network | High storage and encoding cost |
| Proactive cache fill | Fast playback, lower backbone traffic | Predicting demand can be wrong |
| Open Connect in ISPs | Low latency, lower transit costs | Hardware deployment and maintenance |
| Separate control and data planes | Scales delivery independently | More components to coordinate |
| Keeping source masters | Future formats without re-upload | Large storage footprint |

---

## 13. Quick Revision Checklist

- [ ] State functional and non-functional requirements
- [ ] Estimate the work per title (chunks times variants)
- [ ] Walk through the onboarding pipeline from ingest to publish
- [ ] Explain shot detection and scene grouping
- [ ] Explain why chunking enables parallel encoding
- [ ] Describe S3 storage and the key scheme for outputs
- [ ] Explain Open Connect, proactive fill, and steering
- [ ] Explain adaptive bitrate streaming
- [ ] Describe failure handling and idempotent retries
- [ ] Summarize trade-offs and what you would improve

---

## 14. One-Minute Summary

Netflix onboards a title by uploading the source master to S3, splitting it into chunks, and detecting shots (about four seconds each) that get grouped into scenes. Chunks are encoded in parallel into many resolutions and bitrates, checked for quality, and published to the catalog. Delivery relies on Open Connect caches inside ISP networks, filled in advance based on predicted demand. A steering service routes each viewer to the best cache, so most requests never reach the central cloud. This reduces latency and bandwidth cost at scale.
