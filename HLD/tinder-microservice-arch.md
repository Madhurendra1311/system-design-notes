We design the system architecture of Tinder. Designing these apps starts with clarifying the system requirements. In an interview, Tinder has multiple requirements.

When designing the system, we will be drawing logical components and defining their interactions with other services based on the microservice architecture. We assume that any internal details can be handled with our prior knowledge of system design concepts. This includes load balancing, message queues, databases, etc...

The Tinder system has four requirements: storing profiles, recommendations, noting matches and chatting with matches. Storing profiles is trivial except for the image storage, where we argue on BLOB vs File storage. The distributed file architecture seems best when storing images.

Direct Messaging or chatting with matches can be done using the XMPP protocol, which uses web sockets to have peer-to-peer communications between client and server. Each connection is built over TCP, ensuring that the connection is maintained. The session microservice can send messages to the receiver based on connection to user mappings.

Our system is decoupled as much as possible. We try to maintain accept and reject information on the client. On swiping left or right, the client can note the action and avoid showing the same user again, perhaps using bloom filters.

The server has a validation engine called the matcher microservice, which notes matches and allows or disallows chat between two users.

The final requirement of recommendations needs city-wise partitioning of the user data. This is achieved using NoSQL databases like Cassandra or Amazon Dynamo. The other option is to use relational databases with horizontal partitioning. The concept is now referred to as sharding.

# Tinder System Design: Interview Prep

Based on Gaurav Sen's walkthrough of designing Tinder's architecture. The design is illustrative and not how Tinder is actually built.

---

## 1. Problem Statement

Design the backend for a dating app where users:

1. Store and manage profiles (including photos)
2. Receive recommendations of potential matches
3. Swipe left (reject) or right (accept)
4. Get matched when both users swipe right
5. Chat with their matches

---

## 2. Clarifying Requirements

Before drawing anything, confirm scope and assumptions with the interviewer.

**Functional requirements**
- Create and edit a profile with photos
- Show recommended profiles, filtered by location (city)
- Record swipe actions
- Create a match when two users both accept each other
- Allow real-time chat only between matched users

**Non-functional requirements**
- Scalable: many users, with traffic concentrated in dense cities
- Low latency for recommendations and chat
- Availability over strict consistency for recommendations
- Decoupled services so each can scale independently

**Assumptions to state**
- Internal details such as load balancers, message queues, and caches can be taken as known concepts.
- The system uses a microservice architecture.

---

## 3. High-Level Architecture

The system is split into these components:

| Component | Responsibility |
|---|---|
| Client app | Shows profiles, records swipes locally, holds the "seen" set |
| Profile service | Stores user profile data |
| Image storage | Stores profile photos in distributed file storage |
| Recommendation service | Suggests profiles, partitioned by city |
| Swipe handling | Accept and reject decisions, mostly kept on the client |
| Matcher service | Validates swipes, records matches, and decides whether chat is allowed |
| Session service | Tracks which connection belongs to which user and routes messages |
| Chat service | Real-time messaging over XMPP |

---

## 4. Component Deep Dives

### 4.1 Profile Storage and Image Storage

- Profile data (name, age, bio, preferences) is structured, so it goes in a database.
- Images are large binary files. Two options:
  - **BLOB storage** inside the database: simple, but it bloats the database and makes it slow to back up and query.
  - **File storage** (distributed file system or object storage such as S3): better for large media, scales well, and serves files through CDNs.
- **Recommended choice:** distributed file storage for images. The database stores only a reference (URL or key) to each image.

### 4.2 Recommendations

- The recommendation service needs to find candidates near the user, so user data is **partitioned by city**.
- Two ways to partition:
  - **NoSQL databases** such as Cassandra or DynamoDB, which partition data natively using a partition key such as city.
  - **Relational databases with horizontal partitioning (sharding)**, where rows are split across servers by a key such as city.
- Interview point: a city-based partition key means a dense city can become a **hot partition**. Mention mitigation, such as splitting very large cities by geographic cell or by user ID range.

### 4.3 Swipes and Avoiding Repeat Profiles

- The design keeps accept and reject decisions **on the client** to reduce server load and keep the system decoupled.
- The client records each swipe and avoids showing the same profile again.
- A **Bloom filter** is a compact probabilistic set that can track seen profiles cheaply.
  - **No false negatives:** a profile already seen is always detected, so it is never shown again.
  - **Possible false positives:** an unseen profile may be wrongly marked as seen and hidden. This is acceptable for a dating feed.
- Trade-off to mention: the client must send the accept action to the server so matching can be checked. Otherwise the server can't tell whether two users have both accepted.

### 4.4 Matcher Service

- Receives swipe events from clients.
- Checks whether the other user has already swiped right on the current user.
- If both have, it records a match and allows chat between the two users.
- If not, it stores the right swipe and waits for the reverse action.
- Acts as the validation engine that controls who can chat with whom.

### 4.5 Chat Using XMPP

- Chat uses the **XMPP** protocol over **WebSockets**, which runs on **TCP**. TCP keeps the connection reliable and ordered.
- The walkthrough describes peer-to-peer communication. In practice, XMPP messages are routed through the server, so clients don't connect to each other directly. This is worth stating correctly in the interview.
- The **session service** maintains a mapping from **user ID to active connection**. To deliver a message, it looks up the recipient's connection and pushes the message.
- If the recipient is offline, store the message and deliver it when they reconnect.
- Chat is allowed only if the matcher service has recorded a match.

---

## 5. End-to-End Flows

**Viewing recommendations**
1. Client requests recommendations for the user's city.
2. Recommendation service reads from the city partition.
3. Client filters out seen profiles (Bloom filter) and displays the rest.
4. Images are loaded from file storage.

**Swiping**
1. User swipes right on profile B.
2. Client records the swipe locally and marks B as seen.
3. Client sends the accept to the matcher service.
4. Matcher checks whether B has already accepted A.
5. If yes, it creates a match. If not, it stores the accept and waits.

**Chatting**
1. User A sends a message to B.
2. Matcher confirms A and B are matched.
3. Session service looks up B's connection.
4. Message is pushed to B over the open connection. If B is offline, it is stored.

---

## 6. Likely Follow-Up Questions

**Q: Why store images in file storage and not the database?**
A: Images are large and rarely change, so object or distributed file storage is cheaper and faster to serve, often through a CDN. The database keeps only a reference.

**Q: How do you avoid showing the same profile twice?**
A: Track seen profiles on the client with a Bloom filter. It is space-efficient and never misses a seen profile. Its only error is occasionally hiding an unseen profile.

**Q: Why keep swipe decisions on the client?**
A: It reduces server writes and keeps services decoupled. The server only needs the accept signal to detect matches.

**Q: How do you scale the recommendation service?**
A: Partition by city using a NoSQL store like Cassandra or DynamoDB, or shard a relational database. Watch for hot partitions in large cities.

**Q: How does chat stay real-time?**
A: A persistent WebSocket connection over TCP. The session service maps users to connections so messages can be pushed immediately.

**Q: What happens if the recipient is offline?**
A: Store the message and deliver it when the user reconnects. Optionally send a push notification.

**Q: Why not let the matcher check every swipe in real time against the whole user base?**
A: The matcher only needs to check reverse swipes for a specific pair, which is a fast lookup. Keeping it narrow limits load.

**Q: Where would you add a message queue?**
A: Between the swipe handler and the matcher, to absorb spikes and decouple the two services. Also useful for push notifications.

**Q: What would you do differently in production?**
A: Add caching for hot profiles, rate limiting on swipes to prevent bots, fraud and abuse detection, geo-indexing for location queries, and multi-region deployment.

---

## 7. Trade-offs to Discuss

| Decision | Benefit | Cost |
|---|---|---|
| Client-side swipe state | Less server load, looser coupling | Client can be tampered with; server must verify matches |
| Bloom filter for seen profiles | Compact, fast | Rare false positives hide unseen profiles |
| File storage for images | Scales, cheap, CDN-friendly | Extra service to manage |
| NoSQL partitioned by city | Scales horizontally | Hot partitions in dense cities |
| XMPP over WebSockets | Real-time, reliable | Persistent connections use server resources |
| Matcher as a separate service | Clear validation point | Extra network hop per swipe |

---

## 8. Quick Revision Checklist

- [ ] State functional and non-functional requirements first
- [ ] Draw the components and their responsibilities
- [ ] Explain BLOB vs file storage and pick file storage for images
- [ ] Explain city-based partitioning and the hot-partition risk
- [ ] Explain the Bloom filter and its false-positive trade-off
- [ ] Walk through the matcher logic for a match
- [ ] Explain how chat routes messages through the session service
- [ ] Mention scaling, caching, and failure handling
- [ ] Acknowledge the design's limitations and what you would improve

---

## 9. One-Minute Summary

Tinder is split into microservices. Profiles go in a database, and images go in distributed file storage. Recommendations come from a city-partitioned NoSQL or sharded store. Swipes are tracked on the client with a Bloom filter to avoid repeats. The matcher service checks for mutual right swipes and creates matches. Chat runs over XMPP on WebSockets, and the session service routes messages to the right connection, but only between matched users.
