For **System Design interviews**, most problems are actually combinations of a small set of core concepts and techniques.

If you master these concepts, you can solve 80-90% of HLD questions.

---

# 1. Caching

## Used In

* Netflix
* YouTube
* Instagram
* Amazon
* Twitter Feed

### Problem

Database is slow.

```text
User
  |
  v
Database
```

Every request hits DB.

---

### Solution

```text
User
  |
Cache (Redis)
  |
Database
```

---

### Concepts

* Cache Aside
* Write Through
* Write Back
* Cache Invalidation
* TTL

Interview Questions:

* Why Redis?
* What if cache crashes?
* Cache stampede?
* Cache warming?

---

# 2. Message Queue

## Used In

* Uber
* Spotify
* WhatsApp
* Payment Systems

### Problem

Services tightly coupled.

```text
Order Service
   |
Payment Service
```

---

### Solution

```text
Order Service
      |
    Kafka
      |
Payment Service
```

---

### Concepts

* Async Processing
* Event Driven Architecture
* Retry
* Dead Letter Queue
* Back Pressure

Questions:

* Kafka vs RabbitMQ?
* Why not REST?
* How ordering works?

---

# 3. Stream Processing

## Used In

* Spotify Top Songs
* Fraud Detection
* Real Time Analytics

### Tools

* Apache Flink
* Apache Spark

---

### Problem

Need realtime results.

```text
1 million events/sec
```

---

### Concepts

* Windowing
* Watermarks
* Event Time
* State Management
* Checkpointing

Questions:

```text
Top K last hour?
Top K last day?
```

---

# 4. Top K Pattern

## Used In

* Spotify Top Songs
* YouTube Trending
* Google Search

### Techniques

### Min Heap

```text
Top 100
```

Store only 100 elements.

Complexity:

```text
O(N log K)
```

---

### Redis ZSET

Leaderboard.

```redis
ZADD
ZINCRBY
ZREVRANGE
```

---

Questions:

```text
Why heap?
Why ZSET?
```

---

# 5. Rate Limiting

## Used In

* APIs
* Login Systems
* Payment APIs

### Problem

User sends:

```text
10000 req/sec
```

---

### Techniques

#### Token Bucket

```text
Tokens available = 100
```

Consume token per request.

---

#### Leaky Bucket

Smooth traffic.

---

#### Sliding Window

Most common interview answer.

---

Storage:

```text
Redis
```

Questions:

```text
Distributed rate limiting?
```

---

# 6. Sharding

## Used In

* Instagram
* Facebook
* Twitter

### Problem

Single DB too large.

---

### Solution

```text
User1 -> DB1
User2 -> DB2
User3 -> DB3
```

---

### Techniques

#### Hash Based

```java
userId % N
```

---

#### Consistent Hashing

Used in:

* Cache
* Distributed DB

Questions:

```text
What happens when new node added?
```

---

# 7. Replication

## Used In

Almost every system.

---

### Master Slave

```text
       Master
      /   |   \
Replica Replica Replica
```

Writes:

```text
Master
```

Reads:

```text
Replicas
```

---

Questions:

```text
Replication lag?
```

---

# 8. Partitioning

## Used In

Kafka
Cassandra
DynamoDB

### Problem

One machine cannot handle all data.

---

### Solution

```text
Partition1
Partition2
Partition3
```

Questions:

```text
Hot partition?
```

---

# 9. Search Pattern

## Used In

* Google
* Amazon
* LinkedIn

---

### Problem

SQL LIKE slow.

```sql
LIKE '%iphone%'
```

---

### Solution

Use:

Elasticsearch

---

Concepts:

* Inverted Index
* Full Text Search
* Ranking

Questions:

```text
Why not MySQL?
```

---

# 10. Notification Pattern

## Used In

* WhatsApp
* Instagram
* Facebook

---

### Components

```text
Notification Service
      |
Kafka
      |
Push Workers
```

---

Questions:

```text
Millions of notifications?
```

---

# 11. WebSocket Pattern

## Used In

* Google Docs
* WhatsApp
* Stock Market

---

### Problem

Need realtime updates.

---

### Solution

```text
Client <----> WebSocket Server
```

Questions:

```text
Why not polling?
Why not SSE?
```

---

# 12. Concurrency Control

## Used In

* Google Docs
* Banking
* Ticket Booking

---

### Techniques

#### Optimistic Locking

```text
Version Check
```

#### Pessimistic Locking

```text
Lock Row
```

#### OT

Google Docs

#### CRDT

Distributed Editing

Questions:

```text
Why OT?
Why not locking?
```

---

# 13. Leaderboard Pattern

## Used In

* Gaming
* Spotify
* Fantasy Apps

---

### Data Structure

Redis ZSET

```redis
ZINCRBY
```

Questions:

```text
Global ranking?
Country ranking?
```

---

# 14. Time Series Pattern

## Used In

* Monitoring
* Metrics
* Stock Market

---

### Databases

* InfluxDB
* Prometheus

Questions:

```text
Store billions of metrics?
```

---

# 15. File Storage Pattern

## Used In

* Google Drive
* Dropbox
* S3

---

### Concepts

Chunking

```text
100 MB
```

↓

```text
10 chunks
```

---

Questions:

```text
Resume upload?
```

---

# 16. Recommendation Pattern

## Used In

* Netflix
* Spotify
* YouTube

---

Pipeline:

```text
Events
  |
Kafka
  |
Feature Store
  |
ML Model
```

Questions:

```text
Real-time recommendations?
```

---

# 17. Feed Generation Pattern

## Used In

* Facebook
* Twitter
* Instagram

---

### Fan Out On Write

```text
Post
 |
Push to followers
```

Fast reads.

---

### Fan Out On Read

```text
Generate feed when user opens app
```

Fast writes.

Questions:

```text
Celebrity problem?
```

---

# 18. Distributed Lock Pattern

## Used In

* Inventory
* Payment

Tools:

* Redis
* ZooKeeper

Questions:

```text
Double booking prevention?
```

---

# Most Important Mapping for Interviews

| Problem              | Core Concepts                     |
| -------------------- | --------------------------------- |
| URL Shortener        | Hashing, Cache, DB                |
| WhatsApp             | WebSocket, Kafka, Storage         |
| Google Docs          | WebSocket, OT/CRDT, Redis         |
| Spotify Top K        | Kafka, Flink, Heap, Redis ZSET    |
| Uber                 | Geospatial Index, Kafka, Redis    |
| Twitter Feed         | Fanout, Cache, DB                 |
| YouTube              | CDN, Object Storage, Queue        |
| Netflix              | CDN, Cache, Recommendation        |
| Ticket Booking       | Locking, Transactions             |
| Payment System       | Idempotency, Queue, Saga          |
| Search Engine        | Crawling, Indexing, Elasticsearch |
| Notification Service | Kafka, Push Workers               |
| Leaderboard          | Redis ZSET, Top K                 |
| Monitoring System    | Time Series DB                    |
| Google Drive         | Chunking, Object Storage          |

A strong HLD candidate usually thinks in terms of these reusable building blocks: **Cache → Queue → Stream Processing → Storage → Search → Realtime Communication → Consistency Control → Scaling Techniques (Sharding, Replication, Partitioning).** Once you know these patterns, most interview designs become a matter of assembling the right pieces rather than inventing a new architecture.
