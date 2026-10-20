# Capacity Planning and Estimation: How Much Data Does YouTube Store Daily? (HLD Notes)

## 1. Why estimation matters

System design interviews often ask for back-of-the-envelope numbers: "How much storage does YouTube need?" or "How many petrol pumps are in Mumbai?" The exact answer is not the point. The interviewer wants to see how you reason.

A good estimate does three things:

- **States the parameters** that drive the result (users, videos per day, size per video, copies).
- **Makes each assumption explicit,** so the interviewer can accept or correct it.
- **Breaks a big unknown into small guessable pieces** and multiplies them together.

Examples of the kind of questions asked:

- Estimate the hardware needed to set up a system like YouTube.
- Estimate the number of petrol pumps in the city of Mumbai.

The result drives real design choices: how many servers, how much storage, whether you need caching, and roughly what the system will cost.

## 2. Storage requirements

The estimate comes to about 1 PB of new original video per day, and about 10 PB per day once copies and formats are included. Here is how to get there, one assumption at a time.

**Step 1: how many videos per day?**

- About 1 billion active users.
- Assume 1 in 1,000 uploads a video each day.
- That gives 1,000,000,000 ÷ 1,000 = 1 million new videos per day.

**Step 2: how big is each video?**

- Assume an average video is 10 minutes long, which is 600 seconds.
- Approach A: assume a 10 minute video is about 1 GB.
- Approach B (frame by frame): 600 s × 24 frames per second = 14,400 frames. At 1 MB per frame that is about 14 GB.
- Approach B overshoots by roughly 15 times, because real video is compressed and frames are far smaller than 1 MB. A normal 10 minute video is about 700 MB, so 1 GB is a fair round figure.

When two methods disagree, say so, revise the weaker assumption, or hope the interviewer corrects you.

**Step 3: raw storage per day**

1 million videos × 1 GB = 1 PB per day. This is the bare minimum, just the original uploads.

**Step 4: redundancy**

Keep 3 copies for fault tolerance and performance: 3 × 1 PB = 3 PB of raw video.

**Step 5: formats and encoding**

Assume one codec (mp4). Each upload is stored at 720p and re-encoded to 480p, 360p, 240p, and 144p, each roughly half the size of the previous.

If X is the original size (1 PB): X + X/2 + X/4 + X/8 + … ≈ 2X. With 3 copies, that is 2X × 3 = 6X = 6 PB of processed video.

| Component | Per day |
| --- | --- |
| Raw uploads, 3 copies | 3 PB |
| Processed formats, 3 copies | 6 PB |
| **Total** | **about 9 PB, rounded to 10 PB** |

**Step 6: hardware and cost**

The talk puts the cost at about 1 million dollars per day, which over a 3 year plan is roughly 1.1 billion dollars (1 million × 365 × 3).

Check on the drive count: the talk says about 100 hard drives for 10 PB, but that would need 100 TB per drive. With typical 10 TB drives, 10 PB needs about 1,000 drives, so treat the drive count as a rough figure.

## 3. Supplementary storage

Video files are the bulk of the storage, but a real system also stores smaller data. In an interview, name these, give each a quick size guess, and show that they are small next to the video.

| Item | What it is | Rough size guess |
| --- | --- | --- |
| Video metadata | Title, description, tags, owner, upload time, format list | A few KB per video |
| Thumbnails | Preview images in a few sizes | Tens to hundreds of KB per video |
| Comments and likes | Text and counters per video | KB to MB per video, grows over time |
| User data | Profiles, subscriptions, watch history | KB per user |
| Search index | Index over titles, tags, captions | Small fraction of metadata size |
| Captions and subtitles | Text tracks per language | Tens of KB per video |

Example: 1 million new videos per day × about 500 KB of metadata and thumbnails = about 500 GB per day. That is 0.5 TB against roughly 10 PB for the video itself, a difference of around 20,000 times.

The takeaway is that supplementary data barely moves the total, but it matters for design: metadata is small, structured, and read very often, so it goes in a database with caching, while video files go in blob or object storage.

## 4. Back of envelope calculations: checking against real numbers

After an estimate, sanity check it against a known figure. The talk uses the upload rate quoted for YouTube, about 3 × 10^4 minutes of video uploaded every minute (roughly 500 hours per minute).

1. Minutes uploaded per day: 3 × 10^4 × 1,440 minutes in a day ≈ 4.3 × 10^7, which the talk rounds to 4.5 × 10^7.
2. Convert to hours: 4.5 × 10^7 ÷ 60 ≈ 7.5 × 10^5 hours of video per day.
3. Encoded size: good encoding fits an hour of film in about 1 GB, so storage is about 7.5 × 10^5 GB, which is roughly 0.75 PB, or about 1 PB per day.

| Method | Result per day |
| --- | --- |
| Our estimate (1 million videos × 1 GB) | 1 PB |
| Real upload rate (4.5 × 10^7 min at 1 GB per hour) | about 0.75 to 1 PB |

The two agree within the same order of magnitude, so the assumptions were reasonable.

**How close is close enough?**

- Off by one order of magnitude (10 times): acceptable for a rough estimate.
- Off by three or more orders (1,000 times): too far. Go back and find what went wrong.

If the numbers disagree, say out loud either which assumption was wrong (for example the video size or the share of users who upload) or which factor you forgot to include. Being able to explain the gap is as valuable as being right.

## 5. YouTube caching estimation

Views are very uneven: a small share of videos gets most of the traffic. So we do not cache everything, only the hot videos, close to the viewers (a CDN or edge cache). Serving them from cache cuts load on the main storage and cuts latency.

*The figures below are my own illustrative assumptions, not numbers from the talk. In an interview, state yours the same way.*

| Step | Assumption or calculation | Result |
| --- | --- | --- |
| Library size after 3 years | 1 million videos per day × 365 × 3 | about 1.1 billion videos |
| Hot share of the library | Assume 1% of videos draw most of the views | about 11 million videos |
| Size per hot video | One popular format, about 1 GB | about 11 PB |
| Regional split | Assume each of about 100 regions caches the 10% of the hot set that its viewers favour | about 1.1 PB per region |

**What this tells us**

- The hot set is far too big for RAM, so video cache lives on disk or SSD at the edge. RAM is for metadata, counters, and the very hottest items.
- Caching a hot 1% instead of everything cuts what each edge site must hold by about 100 times.
- Hot sets change: a new viral video must be pulled into the cache quickly, and old ones evicted (for example with an LRU policy).

**Cache hit ratio.** If the cache serves 80% of requests, origin storage sees only 20% of the traffic, a 5 times reduction. That ratio is the number to defend when sizing the origin servers.

## 6. YouTube video processing estimation

Every upload must be processed: checked, split into chunks, and encoded into each resolution from section 2. Encoding is CPU heavy, so it decides how many servers we need.

*As with caching, the processing speed and server size below are my own assumptions, not figures from the talk.*

| Step | Assumption or calculation | Result |
| --- | --- | --- |
| Video to process per day | 1 million videos × 10 minutes | 10 million video-minutes |
| Formats per video | 720p, 480p, 360p, 240p, 144p | 5 encodes per video |
| Encoding work per day | 10 million × 5 | 50 million encode-minutes |
| Encoding speed | Assume 1 CPU core encodes 1 minute of video in 1 minute | 50 million core-minutes |
| Cores needed, running around the clock | 50 million ÷ 1,440 minutes in a day | about 35,000 cores |
| Servers | At 32 cores per server | about 1,100 servers |

Using the real upload rate from section 4 (4.5 × 10^7 minutes per day) instead of our 10 million: 4.5 × 10^7 × 5 ÷ 1,440 ≈ 156,000 cores, or about 4,900 servers at 32 cores each. Still the same order of magnitude of thousands of servers.

**Design points that follow**

- **Parallelise:** split a video into chunks and encode them on different machines, so a long video finishes quickly.
- **Queue the work:** uploads go into a message queue and workers pull from it, so spikes are absorbed instead of dropping videos.
- **Allow headroom:** traffic peaks above the average, so provision some extra capacity, for example 2 times.
- **Use specialised hardware:** GPUs or encoding chips do the same work with far fewer machines.
- **Order the work:** make the lowest resolution first so the video becomes watchable sooner.

## 7. Conclusion and quick revision

A good estimate is a chain of small, stated assumptions, a clean multiplication, and a sanity check at the end.

**How to run an estimate in an interview**

1. Restate what is being estimated (storage, servers, cost).
2. List the assumptions aloud and round to easy numbers (powers of 10, 1,000 seconds, 1 GB).
3. Multiply step by step, keeping units visible.
4. Add the multipliers people forget: copies for redundancy, extra formats, peak versus average load.
5. Compare with any real figure you know. Within 10 times is fine; off by 1,000 times means find the wrong assumption.

**YouTube numbers at a glance**

| Quantity | Value |
| --- | --- |
| Active users | 1 billion |
| Uploaders per day | 1 in 1,000, so 1 million videos |
| Size per video | about 1 GB (10 minutes) |
| New raw video per day | about 1 PB |
| With 3 copies | about 3 PB |
| Processed formats, 3 copies | about 6 PB |
| Total per day | about 9 to 10 PB |
| Real upload rate check | about 0.75 to 1 PB per day |

**Common slips to avoid**

- Frame arithmetic: 600 s × 24 fps is 14,400 frames, not 150,000, and raw frame size overstates real compressed video.
- Forgetting redundancy and extra encodings, which multiply storage by 9 times here.
- Mixing units (GB, TB, PB) mid-calculation.
