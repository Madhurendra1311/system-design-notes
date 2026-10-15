# Instagram System Design: News Feed Interview Prep

Based on the walkthrough description you shared. The four features aren't named in that description, so the requirements below are the standard set for this problem. Check them against the video.

---

## 1. Problem Statement

Design the server side of Instagram so that users can:

1. Upload photos (with captions)
2. Follow other users
3. View a personalized news feed of posts from people they follow
4. Like and comment on posts

---

## 2. Clarifying Requirements

**Functional**
- Upload and retrieve photos
- Follow and unfollow users
- Generate a home feed ordered by recency (or relevance)
- Like and comment on posts

**Non-functional**
- Highly available. The feed should load even if some services are down.
- Low latency feed reads, since users open the app often
- Read-heavy: many more feed views than uploads
- Eventual consistency is acceptable. A new post can appear a few seconds late.

**Scale assumptions to state**
- Hundreds of millions of daily active users
- Tens of millions of photos uploaded per day
- Most users follow hundreds of accounts, but a few "celebrity" accounts have millions of followers

---

## 3. Entity Relationship Diagram

Start with the data model, then move to services.

| Entity | Key columns |
|---|---|
| **User** | user_id (PK), username, email, bio, profile_pic_url, created_at |
| **Photo** | photo_id (PK), user_id (FK), image_url, caption, created_at |
| **Follow** | follower_id, followee_id (composite PK), created_at |
| **Like** | user_id, photo_id (composite PK), created_at |
| **Comment** | comment_id (PK), photo_id (FK), user_id (FK), text, created_at |

Relationships:
- One User has many Photos (1:N)
- Users follow each other through Follow, a many-to-many self-relation
- Likes and Comments connect Users and Photos (M:N and 1:N)

Interview point: the Follow table is queried in both directions. "Who do I follow?" drives feed generation, and "Who follows me?" drives fan-out on write. Index both.

---

## 4. Microservice Architecture

Each service owns one responsibility and its own data store.

| Service | Responsibility | Storage |
|---|---|---|
| API Gateway | Routing, auth, rate limiting | None |
| User service | Profiles, authentication | Relational DB |
| Photo upload service | Accepts uploads, writes to object storage, emits events | Object store (S3-like) |
| Metadata service | Photo records, captions, timestamps | Relational or wide-column DB |
| Follow service | Follow graph | Graph-friendly store or sharded relational DB |
| Feed service | Builds and serves home feeds | Cache (Redis) plus feed store |
| Engagement service | Likes and comments | Wide-column DB (high write volume) |
| Notification service | Push notifications for likes, follows, comments | Message queue plus push providers |

Supporting infrastructure: load balancers in front of each service, a CDN for images, and a message queue (Kafka or similar) connecting upload, feed, and notification services.

---

## 5. Photo Upload Flow

1. Client uploads the image to the photo upload service.
2. The service stores the original image in object storage and gets back a URL.
3. Image is resized into several resolutions. This can run asynchronously in workers.
4. Metadata (photo_id, user_id, URL, caption) is written to the metadata DB.
5. An event "PhotoCreated" goes onto the queue for the feed and notification services.
6. Images are served through the CDN, so the feed service only passes URLs.

---

## 6. News Feed Generation

This is the core of the design. There are two main approaches.

### Option A: Fan-out on read (pull)
- When a user opens the feed, fetch the list of followees, then fetch recent posts from each and merge them.
- **Pros:** cheap writes, no wasted work for inactive users.
- **Cons:** slow reads for users following many accounts, heavy load on the database at read time.

### Option B: Fan-out on write (push)
- When a user posts, push the photo_id into the feed cache of every follower.
- Each user's feed is precomputed and read with one fast lookup.
- **Pros:** very fast reads, which matter most for a read-heavy system.
- **Cons:** a celebrity with 10 million followers creates 10 million writes for one post.

### Recommended: Hybrid
- Use fan-out on write for normal users.
- Use fan-out on read for celebrity accounts above a follower threshold.
- When building a feed, merge the precomputed list with recent posts from followed celebrities.

Interview point: this trade-off is the most common follow-up. Be ready to explain why hybrid handles both the typical case and the celebrity case.

---

## 7. Caching and Load Balancing

- **Load balancers** distribute requests across stateless service instances.
- **Feed cache (Redis):** stores each user's precomputed list of recent photo_ids, capped at a few hundred entries.
- **Metadata cache:** hot photo records and user profiles cached to reduce DB load.
- **CDN:** serves image bytes close to users. This is the biggest bandwidth saver.
- **Cache invalidation:** new posts are appended to feed caches. Deleted posts are lazily removed or filtered at read time.

Read path for a feed:
1. Load balancer routes the request to the feed service.
2. Feed service reads the precomputed photo_id list from Redis.
3. Metadata is fetched from cache, with DB fallback on a miss.
4. The response includes CDN URLs, so the client fetches images directly.

---

## 8. Scaling and Partitioning

- **Shard the photo metadata** by user_id, so one user's posts are stored together.
- **Shard the follow graph** by follower_id for fast "who do I follow" queries.
- **Engagement counts** (likes, comments) can be approximate and updated asynchronously to avoid hot rows on viral posts.
- **Replication:** each shard has replicas for read scaling and failover.

---

## 9. Likely Follow-Up Questions

**Q: Why fan-out on write instead of read?**
A: The system is read-heavy. Precomputing feeds moves work to the write path, where it happens once per post, so feed reads stay fast.

**Q: How do you handle celebrities?**
A: Hybrid approach. Don't fan out to millions of followers. Pull celebrity posts at read time and merge them into the precomputed feed.

**Q: How do you order the feed?**
A: Default to reverse chronological order using timestamps. Ranking can be added as a separate service that reorders candidates.

**Q: How do you keep the feed consistent when a user unfollows someone?**
A: Accept brief inconsistency. Filter out unfollowed users at read time, or remove their posts from the cache asynchronously.

**Q: How do you store and serve images?**
A: Object storage for originals, a CDN for delivery, and multiple resolutions generated by background workers.

**Q: What happens if the feed service goes down?**
A: Stateless instances behind a load balancer restart or fail over. The feed cache is replicated. Users might see a stale feed rather than an error.

**Q: How would you scale likes on a viral photo?**
A: Batch and aggregate counter updates, use sharded counters, and accept approximate counts shown to users.

**Q: Why use a message queue?**
A: It decouples upload from feed fan-out and notifications, absorbs traffic spikes, and lets consumers retry on failure.

**Q: How would you add search or explore?**
A: A separate indexing pipeline consumes the same events and builds search indexes, without touching the core upload path.

---

## 10. Trade-offs to Discuss

| Decision | Benefit | Cost |
|---|---|---|
| Fan-out on write | Fast feed reads | Expensive for celebrities, write amplification |
| Fan-out on read | Cheap writes | Slow reads for heavy followers |
| Hybrid feed | Balances both | More complex logic |
| Eventual consistency for feeds | High availability, lower latency | Brief staleness |
| CDN for images | Low latency, lower origin load | Cache invalidation for edited images |
| Sharded counters for likes | Handles viral posts | Counts are approximate |

---

## 11. Quick Revision Checklist

- [ ] State functional and non-functional requirements and scale assumptions
- [ ] Draw the ER diagram with keys and relationships
- [ ] List microservices and their data stores
- [ ] Walk through the photo upload flow
- [ ] Compare fan-out on read vs write and justify hybrid
- [ ] Explain caching layers and the feed read path
- [ ] Describe sharding for metadata and the follow graph
- [ ] Mention failure handling and eventual consistency
- [ ] Summarize trade-offs and what you would improve

---

## 12. One-Minute Summary

Start with an ER diagram of users, photos, follows, likes, and comments. Split the system into microservices: upload, metadata, follow, feed, engagement, and notifications, with a load balancer in front and a CDN for images. Store images in object storage and serve them through a CDN. Build feeds with a hybrid of fan-out on write for normal users and fan-out on read for celebrities, and cache precomputed feeds in Redis. Shard data by user, and accept eventual consistency for feeds and counts.
