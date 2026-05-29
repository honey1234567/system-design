https://www.youtube.com/watch?v=KOvDWEGVXec

redis sorted set internally also creates heap(min heap)

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
