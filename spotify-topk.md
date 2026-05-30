https://www.youtube.com/watch?v=KOvDWEGVXec

redis sorted set internally also creates heap(min heap)

Excellent question. This is where many system design candidates get confused.

**Short answer:**

**No, Redis ZSET does not use a Min Heap internally.**
Redis ZSET uses:

```text
Hash Table
    +
Skip List
```

So why did we discuss Min Heap?

Because **Top-K is an algorithmic problem first**, and there are multiple ways to solve it.

---

# Approach 1: Min Heap (Classic Top-K Algorithm)

Suppose you have:

```text
100 Million songs
```

Need:

```text
Top 100 songs
```

You are processing a stream:

```text
A=100
B=200
C=50
D=300
...
```

Keep only K elements:

```text
MinHeap size = 100
```

Complexity:

```text
Insert = O(log K)
Memory = O(K)
```

This is commonly used in:

* Coding interviews
* Streaming algorithms
* Flink operators
* Spark jobs

---

## Example

Need Top 3:

```text
A=100
B=90
C=80
D=200
```

Heap:

```text
80
90
100
```

New value:

```text
200
```

Since:

```text
200 > 80
```

Remove 80.

Heap becomes:

```text
90
100
200
```

Top 3 maintained.

---

# Approach 2: Redis ZSET

Redis stores all songs.

Example:

```text
songA = 1000
songB = 800
songC = 700
...
100 million songs
```

Internally:

```text
HashMap
+
Skip List
```

Skip list remains sorted:

```text
1000
 |
800
 |
700
 |
500
```

Now:

```redis
ZREVRANGE top_songs 0 99
```

returns top 100 directly.

No heap needed.

---

# Why Not Use Min Heap in Redis?

Imagine:

```text
100 Million songs
```

If Redis keeps only Top 100:

```text
songA
songB
songC
...
```

What happens when songX suddenly becomes popular?

Current:

```text
songX = 50
```

After viral event:

```text
songX = 1,000,000
```

Redis needs the full count history.

Therefore Redis stores:

```text
ALL songs
```

not just top K.

---

# Then Where Is Min Heap Actually Used?

### Case 1: Flink Computing Top-K

Suppose Flink receives:

```text
166K events/sec
```

For a 1-hour window:

```text
100 million songs
```

After aggregation:

```text
songA = 10000
songB = 5000
...
```

To find top 100:

```java
PriorityQueue<SongCount>
```

(Min Heap)

is very common.

Because:

```text
O(N log K)
```

instead of:

```text
O(N log N)
```

---

### Case 2: Google Search Top Queries

```text
top 100 searches
```

Heap.

---

### Case 3: YouTube Trending

```text
top 100 videos
```

Heap.

---

### Case 4: Streaming Analytics

Flink operators often maintain:

```text
State
   +
Priority Queue (Min Heap)
```

for Top-K windows.

---

# In Spotify HLD, What Happens?

A realistic flow is:

```text
Kafka
   |
   v
Flink
```

Flink counts:

```text
songA = 10000
songB = 5000
songC = 3000
```

Then Flink maintains:

```text
Top100 Heap
```

Result:

```text
Top100 songs
```

Then writes final leaderboard to:

```text
Redis ZSET
```

or

```text
Redis Hash
```

for serving.

---

# Why Use Redis If Flink Already Has Top-K?

Because Flink is a compute engine.

You don't want millions of users asking:

```text
Give me Top 100 songs
```

from Flink.

Instead:

```text
Flink
  |
  v
Redis
```

Redis acts as a cache/read layer.

---

# What do large companies typically do?

A common production pattern is:

```text
Kafka
   |
   v
Flink
   |
   +--> Min Heap / Top-K State
   |
   +--> Redis Leaderboard
   |
   +--> Data Lake
```

So:

* **Min Heap** = computation algorithm for finding Top-K efficiently.
* **Redis ZSET** = storage/query layer for serving Top-K to users.
* **Redis itself does not use Min Heap internally**; it uses a **Skip List + Hash Table** implementation.


<img width="654" height="381" alt="image" src="https://github.com/user-attachments/assets/ba6d0d4a-f29d-424f-b67b-9396e5e91fef" />

A Redis Sorted Set (ZSET) stores data like:

```text
key -> {
   member1 : score1,
   member2 : score2
}
```

Internally:

* members are unique
* sorted by score
* allows range queries efficiently

Very useful for:

* leaderboards
* rankings
* timestamps
* priority queues
* rate limiting

---

# Example ZSET

```bash
ZADD leaderboard 100 alice
ZADD leaderboard 200 bob
ZADD leaderboard 150 charlie
```

Stored as:

```text
leaderboard:
bob -> 200
charlie -> 150
alice -> 100
```

---

# 1. Extract all data using key

## Command

```bash
ZRANGE leaderboard 0 -1
```

Output:

```text
alice
charlie
bob
```

(default ascending by score)

---

# With scores

```bash
ZRANGE leaderboard 0 -1 WITHSCORES
```

Output:

```text
alice 100
charlie 150
bob 200
```

---

# Descending order

```bash
ZREVRANGE leaderboard 0 -1 WITHSCORES
```

Output:

```text
bob 200
charlie 150
alice 100
```

---

# 2. Filter by score

Very common.

---

## Example

Get users with score between 120 and 180:

```bash
ZRANGEBYSCORE leaderboard 120 180
```

Output:

```text
charlie
```

---

# Inclusive / Exclusive filtering

## Inclusive

```bash
ZRANGEBYSCORE leaderboard 100 150
```

includes both.

---

## Exclusive

```bash
ZRANGEBYSCORE leaderboard (100 (200
```

means:

```text
score > 100 AND score < 200
```

---

# With scores

```bash
ZRANGEBYSCORE leaderboard 100 200 WITHSCORES
```

---

# 3. Filter top N users

## Top 2

```bash
ZREVRANGE leaderboard 0 1 WITHSCORES
```

Output:

```text
bob 200
charlie 150
```

Complexity:

```text
O(logN + M)
```

Efficient even for millions of entries.

---

# 4. Filter by rank

## Get rank 1 to 3

```bash
ZRANGE leaderboard 0 2
```

---

# Reverse rank

```bash
ZREVRANGE leaderboard 0 2
```

---

# 5. Extract using lexicographical filtering

If all scores same.

Example:

```bash
ZADD names 0 apple
ZADD names 0 banana
ZADD names 0 cat
```

---

## Query

```bash
ZRANGEBYLEX names [a [c
```

Output:

```text
apple
banana
cat
```

---

# Internal implementation

Redis ZSET internally uses:

## 1. HashMap

```text
member -> score
```

For O(1) lookup.

---

## 2. Skip List

Maintains sorted order.

Allows:

* rank queries
* score filtering
* range extraction

Complexity:

```text
O(logN)
```

---

# Real-world example: Chat system

Store online users by timestamp.

```bash
ZADD active_users 1716970000 user1
ZADD active_users 1716970100 user2
```

---

## Get users active in last 5 mins

```bash
ZRANGEBYSCORE active_users 1716969800 +inf
```

---

# Real-world example: Rate limiter

Store request timestamps.

```bash
ZADD req_user123 1716971000 req1
```

Remove old requests:

```bash
ZREMRANGEBYSCORE req_user123 0 1716970000
```

Count active requests:

```bash
ZCARD req_user123
```

---

# Java example using Jedis

```java
Jedis jedis = new Jedis("localhost");

jedis.zadd("leaderboard", 100, "alice");
jedis.zadd("leaderboard", 200, "bob");

Set<Tuple> result =
    jedis.zrevrangeWithScores("leaderboard", 0, 1);

for (Tuple t : result) {
    System.out.println(
        t.getElement() + " " + t.getScore()
    );
}
```

---

# Spring Boot example

Using Spring Boot + RedisTemplate:

```java
ZSetOperations<String, String> zset =
    redisTemplate.opsForZSet();

zset.add("leaderboard", "alice", 100);

Set<String> top =
    zset.reverseRange("leaderboard", 0, 2);
```

---

# Interview follow-up questions

## Why ZSET instead of LIST?

Because LIST:

* not sorted
* no score filtering
* no rank queries

---

## Why not HASH?

HASH:

* no ordering
* no range queries

---

## Why ZSET perfect for leaderboard?

Supports:

* sorted ranking
* top K
* range filtering
* score updates

efficiently.

---

# Time complexities

| Operation     | Complexity  |
| ------------- | ----------- |
| ZADD          | O(logN)     |
| ZRANGE        | O(logN + M) |
| ZRANGEBYSCORE | O(logN + M) |
| ZREM          | O(logN)     |
| ZRANK         | O(logN)     |

Where:

```text
M = returned elements
```

When Redis starts handling:

* millions of users
* huge caches
* realtime systems
* leaderboards
* sessions
* pub/sub
* streams

then managing and scaling data becomes a major system design topic.

---

# 1. How to manage multiple types of data in Redis?

Redis is not just a key-value store.

It supports multiple data structures.

---

# Common Redis data structures

| Use Case          | Redis Type  |
| ----------------- | ----------- |
| User cache        | STRING      |
| Sessions          | HASH        |
| Leaderboard       | ZSET        |
| Queue             | LIST        |
| Rate limiter      | ZSET        |
| Realtime pub/sub  | PUBSUB      |
| Event streaming   | STREAM      |
| Counters          | STRING/INCR |
| Presence tracking | SET         |
| Geo location      | GEO         |

---

# Example key organization

Very important in production.

Use namespacing.

```text id="5n4vks"
user:1001:profile
user:1001:session
leaderboard:game1
chat:room:45
rate_limit:user:1001
```

Benefits:

* avoids collisions
* easier debugging
* easier eviction
* easier sharding

---

# 2. Problem when Redis grows

Suppose:

```text id="0u1lnr"
100 million users
```

Single Redis instance becomes bottleneck.

Problems:

* RAM full
* CPU saturation
* network bottleneck
* single point of failure

---

# 3. How Redis scales?

Mainly 3 approaches:

1. Vertical scaling
2. Replication
3. Sharding/Clustering

---

# 4. Vertical scaling (simple but limited)

Increase:

* RAM
* CPU

Example:

```text id="s0jxly"
16 GB → 128 GB
```

Problem:

* expensive
* hardware limit exists

Not enough at internet scale.

---

# 5. Replication

Redis supports:

```text id="9hb5n5"
Primary → Replica
```

---

# Architecture

```text id="t1zl2z"
        Primary
        /     \
   Replica1  Replica2
```

---

# Benefits

## Read scaling

Reads distributed to replicas.

---

## High availability

If primary dies:

* replica promoted

Using:

* Redis Sentinel
  OR
* Redis Cluster

---

# But replication alone doesn't solve

```text id="il7d8v"
memory scaling
```

Because all replicas store full dataset.

---

# 6. Real scaling solution = Sharding

Most important concept.

---

# What is sharding?

Split data across multiple Redis nodes.

Example:

```text id="5mbeqg"
User 1-1M    → Redis A
User 1M-2M   → Redis B
User 2M-3M   → Redis C
```

Each node stores partial data.

---

# Benefits

## Horizontal scaling

Can scale almost infinitely.

---

## Memory distributed

100 GB data:

```text id="v4h10w"
10 nodes × 10 GB
```

---

# 7. How sharding works?

Two common methods:

---

# Method 1: Client-side hashing

Application decides shard.

Example:

```java id="fw03zc"
shard = hash(userId) % 4
```

---

# Problem

Adding new shard causes:

```text id="i0i2vw"
massive rehashing
```

---

# Better solution:

Consistent hashing.

---

# 8. Consistent hashing

Used heavily in distributed systems.

---

# Idea

Map:

* keys
* servers

on hash ring.

Benefits:

* minimal data movement
* easy scaling

---

# Example

```text id="4qbnyd"
hash(user123) → shard 2
```

Add new node:
only small portion migrates.

---

# 9. Redis Cluster (industry solution)

Redis has built-in clustering.

---

# Redis Cluster architecture

Redis uses:

```text id="xajp5j"
16384 hash slots
```

---

# Example

```text id="oqrjzj"
Node A → slots 0-5000
Node B → slots 5001-10000
Node C → slots 10001-16383
```

---

# Key mapping

```text id="y97lhg"
slot = CRC16(key) % 16384
```

Redis automatically routes requests.

---

# Advantages

## Automatic sharding

---

## Automatic failover

---

## High availability

---

## Horizontal scaling

---

# 10. Redis Cluster with replicas

Production architecture:

```text id="m1c5ib"
Primary A → Replica A1
Primary B → Replica B1
Primary C → Replica C1
```

---

# If primary fails

Replica promoted automatically.

---

# 11. Biggest Redis scaling problem = RAM

Redis is memory-first.

RAM expensive.

---

# Solutions

---

# Solution 1: Eviction policies

When memory full:
Redis evicts keys.

Policies:

* LRU
* LFU
* TTL based

---

# Example

```text id="0o3qha"
allkeys-lru
```

Removes least recently used keys.

---

# Solution 2: TTLs

Cache should expire.

```bash id="r8i9ud"
SET user:1 data EX 300
```

Auto remove after 5 mins.

---

# Solution 3: Hot/cold separation

Keep:

* hot data in Redis
* cold data in DB

Example:

```text id="s0mdzb"
Redis → recent chats
S3/DB → old chats
```

---

# 12. Multi-tier caching architecture

Very common interview topic.

---

# Example

```text id="3xjlwm"
Client
   ↓
CDN
   ↓
Application Cache
   ↓
Redis
   ↓
Database
```

---

# Why?

Reduces DB load massively.

---

# 13. Redis persistence

Redis is in-memory,
but supports persistence.

---

# RDB snapshots

Periodic dump.

```text id="m7ylu5"
snapshot.rdb
```

Fast recovery.

---

# AOF (Append Only File)

Logs every write.

Safer.

---

# Production setup

Usually:

```text id="jlwmso"
AOF + periodic snapshots
```

---

# 14. Redis bottlenecks

---

# Problem 1: Hot keys

Example:

```text id="djlwm6"
celebrity_profile
```

Millions hit same key.

---

# Solutions

## Replication

---

## Local app cache

---

## CDN

---

## Request coalescing

---

# Problem 2: Big keys

Huge:

* hash
* zset
* list

causes:

* blocking
* latency spikes

---

# Solution

Split keys.

Example:

```text id="smbj0i"
chat:room1:part1
chat:room1:part2
```

---

# Problem 3: Single-threaded nature

Redis core is mostly single-threaded.

CPU-heavy commands dangerous.

Avoid:

```text id="pwjlwm"
KEYS *
```

Use:

```text id="8tktlw"
SCAN
```

---

# 15. Redis scaling architecture (real-world)

Example architecture:

```text id="qfjlwm"
                Load Balancer
                      |
               App Servers
                      |
            -------------------
            |        |        |
        Redis Cluster
            |        |
         Replica   Replica
                      |
                  Kafka
                      |
                   Database
```

---

# 16. How companies use Redis

## Netflix

* distributed caching

---

## Uber

* geospatial indexing
* realtime tracking

---

## Twitter

* timeline cache

---

## Discord

* presence tracking

---

# 17. Best practices

---

## Use small values

Avoid huge JSON blobs.

---

## Add TTLs

Prevent stale data.

---

## Use pipelining

Batch commands.

---

## Avoid blocking operations

Never use:

```text id="5rjlwm"
KEYS *
```

---

## Use Redis Cluster

for large-scale production.

---

## Monitor carefully

Metrics:

* memory
* latency
* evictions
* replication lag
* hot keys

Use:

* Prometheus
* Grafana

---

# 18. Interview-level scaling answer

If interviewer asks:
“How would you scale Redis?”

Good answer:

```text id="s7xjlwm"
1. Start with single Redis
2. Add replication for HA/read scaling
3. Add TTL + eviction
4. Move to Redis Cluster for sharding
5. Use consistent hashing
6. Handle hot keys carefully
7. Separate hot/cold data
8. Persist critical data asynchronously
```
In HLD/system design, Apache Flink is commonly used for **Top-K problems on streaming data** because databases or Redis alone become insufficient at very large scale and realtime requirements.

Examples:

* Top 10 trending hashtags
* Top 100 videos
* Top searched queries
* Top products sold in last 5 mins
* Top active users
* Live cricket leaderboard
* Fraud detection rankings

---

# 1. What is the actual problem?

Suppose events arrive continuously:

```text id="r7yk1s"
view(video1)
view(video2)
view(video1)
view(video3)
...
millions/sec
```

Now requirement:

```text id="l5v27t"
Get Top 10 videos in last 5 minutes
in realtime
```

This is not simple DB querying anymore.

---

# 2. Why normal DB query is bad?

Naive SQL:

```sql id="6mt6kx"
SELECT video_id, COUNT(*)
FROM views
WHERE time > NOW() - 5 min
GROUP BY video_id
ORDER BY COUNT(*) DESC
LIMIT 10;
```

Problems at scale:

* scanning huge data repeatedly
* expensive aggregation
* high latency
* cannot handle millions events/sec efficiently

---

# 3. Why Redis alone is not enough?

Redis ZSET works well for:

* smaller scale
* approximate realtime ranking

Example:

```bash id="ewcn2f"
ZINCRBY trending 1 video1
```

Then:

```bash id="jlwmkq"
ZREVRANGE trending 0 9
```

Works great initially.

---

# But problems appear

---

## Problem 1: Sliding window

Requirement:

```text id="0olrdy"
Top K in last 5 mins only
```

Redis does not naturally manage:

* event-time windows
* watermarking
* late events
* window expiration

You must build everything manually.

---

## Problem 2: Huge scale

Suppose:

```text id="89xl1v"
10 million events/sec
```

Redis becomes:

* memory bottleneck
* write bottleneck

---

## Problem 3: Distributed aggregation

Top-K requires:

```text id="9vyl6s"
global aggregation
```

Across multiple machines.

Very difficult manually.

---

# 4. Why Flink solves this?

Apache Flink is designed exactly for:

```text id="db8qoz"
continuous distributed stream computation
```

---

# Flink advantages for Top-K

| Capability              | Why important                   |
| ----------------------- | ------------------------------- |
| Streaming processing    | Continuous realtime computation |
| Windowing               | Last 5 min, 1 hour etc          |
| Stateful processing     | Maintains counts efficiently    |
| Distributed aggregation | Parallel scaling                |
| Event-time support      | Correct ordering                |
| Fault tolerance         | No data loss                    |
| Checkpointing           | Recovery                        |
| Exactly-once processing | Accurate counts                 |

---

# 5. Typical architecture

```text id="1wiyrm"
Producers
   ↓
Kafka
   ↓
Flink
   ↓
Redis / DB / Dashboard
```

---

# Flow

## Step 1: Events enter Kafka

Example:

```json id="6jsa6f"
{
  "video": "v1",
  "timestamp": 123456
}
```

---

## Step 2: Flink consumes stream

Flink:

* groups by video_id
* maintains counters
* computes windows

---

## Step 3: Compute Top-K

Example:

```text id="v6d65k"
Top 10 videos every 10 sec
```

---

## Step 4: Push result

Store result in:

* Redis
* ElasticSearch
* Dashboard cache

---

# 6. Sliding window example

Requirement:

```text id="l8omnl"
Top 10 hashtags in last 5 mins
updated every 30 sec
```

Flink can do:

```text id="cf8zcb"
Window size = 5 min
Slide interval = 30 sec
```

This is extremely hard to implement correctly manually.

---

# 7. Why Flink better than Spark Streaming?

Common interview follow-up.

---

# Spark Streaming

Uses:

```text id="f3f9fv"
micro-batches
```

Latency:

```text id="mtzjlwm"
seconds
```

---

# Flink

True streaming:

```text id="3jlwmc"
event-by-event processing
```

Latency:

```text id="5xjlwm"
milliseconds
```

Better for realtime Top-K.

---

# 8. How Flink computes Top-K internally?

---

# Step 1: KeyBy

```text id="jlwm92"
group by item_id
```

---

# Step 2: Window

```text id="1fjlwm"
5-minute sliding window
```

---

# Step 3: Aggregate

Maintain counts incrementally.

---

# Step 4: Heap/Priority Queue

Maintain:

```text id="jlwmn3"
Top K elements only
```

Complexity:

```text id="gjlwm4"
O(log K)
```

instead of sorting all items.

---

# 9. Why not sort entire dataset?

Suppose:

```text id="jlwm0x"
100 million unique hashtags
```

Sorting all repeatedly is expensive.

---

# Better approach

Maintain min-heap size K.

Example:

```text id="9tjlwm"
Top 10 only
```

Very memory efficient.

---

# 10. Event time vs processing time

Very important Flink concept.

---

# Problem

Events may arrive late.

Example:

```text id="jlwmcd"
Event generated at 10:01
Arrives at 10:05
```

---

# Flink supports watermarking

Allows:

* late event handling
* accurate windowing

Redis alone cannot do this elegantly.

---

# 11. Fault tolerance

Suppose Flink node crashes.

Without checkpointing:

```text id="jlwmre"
counts lost
```

---

# Flink solution

Checkpoint state periodically to:

* HDFS
* S3
* RocksDB

Recovery resumes correctly.

---

# 12. Exactly-once semantics

For Top-K:
double counting is dangerous.

Flink supports:

```text id="jlwmk1"
exactly-once guarantees
```

using:

* checkpoints
* Kafka offsets
* transactional sinks

---

# 13. Where Redis still fits?

Redis is often final serving layer.

Architecture:

```text id="jlwmx0"
Kafka
   ↓
Flink
   ↓
Redis ZSET
   ↓
API/dashboard
```

---

# Why?

Flink computes.
Redis serves fast reads.

---

# 14. Example: Twitter Trending Hashtags

Likely architecture:

```text id="jlwmr7"
Tweet events
   ↓
Kafka
   ↓
Flink aggregation
   ↓
Top-K hashtags
   ↓
Redis cache
   ↓
API
```

---

# 15. Example: YouTube Trending Videos

Requirements:

* realtime
* regional trends
* time windows
* millions/sec

Perfect Flink use case.

---

# 16. Interview answer (best concise form)

If interviewer asks:
“Why Flink needed for Top-K?”

Strong answer:

```text id="jlwmz8"
Redis or DB can store rankings,
but Flink is needed for scalable realtime stream processing.

Flink efficiently computes distributed aggregations,
sliding windows, event-time processing,
late event handling, and fault-tolerant Top-K
over millions of streaming events/sec.

Typically:
Kafka → Flink → Redis
where Flink computes Top-K
and Redis serves low-latency reads.
```
<img width="508" height="274" alt="image" src="https://github.com/user-attachments/assets/acd59935-bfa0-4e21-8424-7cf90a360ee6" />

# Spotify Top-K Songs System Design (Beginner → Advanced HLD)

This is a very common system design interview question because it tests:

* Real-time data ingestion
* Stream processing
* Top-K algorithms
* Distributed systems
* Scalability
* Event processing

Let's design:

> "Show Top 100 songs globally, per country, and per city in near real time."

---

# 1. Requirements

## Functional Requirements

When a user plays a song:

```text
User A → Song X
User B → Song X
User C → Song Y
```

System should:

* Count plays
* Update rankings
* Show Top 100 songs

Examples:

```text
Global Top 100

1. Song X
2. Song Y
3. Song Z
```

Country:

```text
Top 100 India
Top 100 USA
```

City:

```text
Top 100 Delhi
Top 100 Mumbai
```

---

## Non Functional Requirements

### Scale

Spotify:

```text
600M+ users
100M+ songs
Millions of streams/minute
```

Assume:

```text
10 million song plays/minute
```

which is

```text
166K events/sec
```

Need:

* High throughput
* Low latency
* Fault tolerance

---

# 2. High Level Architecture

```text
                Users
                   |
                   v
          +----------------+
          | Load Balancer  |
          +----------------+
                   |
                   v
          +----------------+
          | Play Service   |
          +----------------+
                   |
                   v
          +----------------+
          | Kafka          |
          +----------------+
                   |
        -----------------------
        |         |          |
        v         v          v

  Analytics   Billing   Recommendation

        |
        v

   Flink / Spark
        |
        v

     Redis
        |
        v

      API
        |
        v

    Mobile App
```

---

# 3. Why Kafka?

Every song play becomes an event.

Example:

```json
{
  "userId":"123",
  "songId":"song_567",
  "country":"India",
  "city":"Delhi",
  "timestamp":"..."
}
```

Instead of directly updating DB:

```text
Play Service
   |
   +--> Kafka
```

Benefits:

### Decoupling

Many consumers can use same event.

```text
Kafka
 |
 +-- Recommendation
 +-- Analytics
 +-- Billing
 +-- Top K
```

### Reliability

Kafka persists data.

Even if consumer dies:

```text
Consumer restart
→ replay events
```

---

# 4. Why Not Update DB Directly?

Bad approach:

```text
Play Service
     |
     +--> MySQL update
```

For every play:

```sql
UPDATE songs
SET count = count + 1
```

At

```text
166K updates/sec
```

database becomes bottleneck.

---

# 5. Stream Processing Layer

We need real-time ranking.

Use:

* Apache Flink
* Spark Streaming

Most companies use:

Apache Flink

because latency is very low.

---

# 6. What Flink Does

Input:

```text
Song A
Song B
Song A
Song A
Song C
```

Flink continuously counts.

Output:

```text
Song A = 3
Song B = 1
Song C = 1
```

No batch processing.

Everything happens live.

---

# 7. Top-K Problem

Suppose:

```text
100 Million songs
```

Need:

```text
Top 100 songs
```

Naive:

```text
Sort all songs
```

Complexity:

```text
O(N log N)
```

Huge.

Not feasible every second.

---

# 8. Min Heap Solution

Keep only K songs.

Example:

```text
K = 3
```

Counts:

```text
A=100
B=90
C=80
D=200
E=150
```

Maintain Min Heap.

---

### Step 1

```text
A=100

Heap

100
```

---

### Step 2

```text
A=100
B=90
```

Heap:

```text
 90
 /
100
```

---

### Step 3

```text
A=100
B=90
C=80
```

Heap:

```text
   80
  /  \
100  90
```

---

### Step 4

New:

```text
D=200
```

Compare with root.

```text
200 > 80
```

Remove 80.

Heap:

```text
100
90
200
```

---

Final:

```text
Top 3
200
100
90
```

Complexity:

```text
O(log K)
```

instead of

```text
O(log N)
```

Huge improvement.

---

# 9. Why Redis?

Users constantly ask:

```text
GET /top100
```

Cannot query Flink every time.

Store result in Redis.

```text
Flink
  |
  v
Redis
```

---

Redis Sorted Set (ZSET):

```text
Key:
top_songs_global
```

```text
Score = play count

Member = song id
```

Example:

```text
songA 1000
songB 800
songC 600
```

---

# 10. Redis ZSET Commands

Insert:

```redis
ZADD top_songs_global 1000 songA
```

Update:

```redis
ZINCRBY top_songs_global 1 songA
```

Get Top 10:

```redis
ZREVRANGE top_songs_global 0 9 WITHSCORES
```

Output:

```text
songA 1000
songB 800
songC 600
```

Complexity:

```text
O(log N)
```

---

# 11. How Redis Finds Top K So Fast?

Internally:

```text
HashMap
+
Skip List
```

Structure:

```text
songA -> 1000
songB -> 800
songC -> 600
```

Skip List keeps sorted order.

```text
1000 -> 800 -> 600
```

Top K:

```text
Read last K nodes
```

Very fast.

---

# 12. Country Wise Top K

Need:

```text
Top India
Top USA
Top Japan
```

Create separate ZSET.

```text
top_india
top_usa
top_japan
```

When event arrives:

```json
{
 "song":"A",
 "country":"India"
}
```

Update:

```redis
ZINCRBY top_india 1 A
```

---

# 13. City Wise Top K

Keys:

```text
top_delhi
top_mumbai
top_bangalore
```

or

```text
top_city:delhi
top_city:mumbai
```

---

# 14. Why Flink Needed If Redis Can Do Top K?

Most asked interview question.

Many people say:

```text
Directly increment Redis.
```

Works at small scale.

Problem:

```text
10 million events/minute
```

Need:

### Windowing

Top songs:

```text
Last 1 hour
Last 24 hours
Last week
```

Redis cannot efficiently compute event-time windows at massive scale.

---

### Late Events

Suppose:

```text
Event generated at 1:00 PM
Arrives at 1:10 PM
```

Flink handles:

```text
Watermarks
Event Time
```

Redis doesn't.

---

### Aggregation

Need:

```text
Global
Country
City
Genre
Artist
```

Flink computes all in one pipeline.

---

### Fault Tolerance

Flink checkpoints state.

```text
Kafka offset
+
Current counts
```

Can recover exactly.

---

# 15. Flink Internal Flow

```text
Kafka Event

{
 song:A
 country:India
 city:Delhi
}
```

↓

Partition by songId

```text
keyBy(songId)
```

↓

Count plays

```text
A = 100
```

↓

Update leaderboard

↓

Write to Redis

---

# 16. Partitioning in Kafka

Without partitioning:

```text
Single consumer
```

Bottleneck.

Use:

```text
Partition by songId
```

Example:

```text
Song A -> P1
Song B -> P2
Song C -> P3
```

Consumers:

```text
C1 reads P1
C2 reads P2
C3 reads P3
```

Parallel processing.

---

# 17. Hot Song Problem

Imagine:

```text
Taylor Swift new release
```

90% traffic.

All events:

```text
song123
```

go to one partition.

Hot partition.

---

Solution: Key Salting

Instead of:

```text
song123
```

Use:

```text
song123_1
song123_2
song123_3
...
song123_10
```

Spread across partitions.

Later aggregate:

```text
sum(song123_*)
```

---

# 18. Storage Layer

Historical analytics stored in:

* Apache Hive
* Apache Iceberg
* Apache Hudi

Flow:

```text
Kafka
  |
  +--> Flink
  |
  +--> Data Lake
```

Used for:

```text
Top songs of 2025
Monthly reports
Analytics
```

---

# Final HLD

```text
                 Users
                    |
                    v
             +-------------+
             | API Gateway |
             +-------------+
                    |
                    v
             +-------------+
             | Play Service|
             +-------------+
                    |
                    v
             +-------------+
             | Kafka       |
             +-------------+
                    |
          ---------------------
          |         |         |
          v         v         v

      Billing   Reco     Top-K

                         |
                         v
                 +---------------+
                 | Apache Flink  |
                 +---------------+
                         |
                         v
                 +---------------+
                 | Redis ZSET    |
                 +---------------+
                         |
                         v
                 +---------------+
                 | Ranking API   |
                 +---------------+
                         |
                         v
                     Users

                         |
                         v

                Data Lake/Hive
```


