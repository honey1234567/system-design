For a **Senior Software Engineer (SDE-3/L5-L6)** HLD interview, the interviewer is **not** interested in hearing "Client → Load Balancer → App → DB". They already know that.

They want to evaluate whether you understand:

* Distributed systems trade-offs
* Fanout strategies
* Feed generation
* Consistency vs Availability
* Caching challenges
* Hot partition problems
* Celebrity effect
* Timeline ranking
* Distributed counters
* Idempotency
* Scaling storage
* Event-driven architecture
* Failure handling

Twitter (X) is one of the hardest HLD interviews because it contains almost every distributed systems concept.

---

# 1. Requirements

Skip obvious things.

Focus on difficult ones.

### Functional

* Post Tweet
* Delete Tweet
* Follow / Unfollow
* Home Timeline
* User Timeline
* Like
* Retweet
* Reply
* Search hashtag
* Trending hashtags

---

### Non Functional

Very important.

* 500M+ users
* Millions of tweets/min
* Home timeline <200ms
* 99.99% uptime
* Global deployment
* Eventually consistent acceptable
* No duplicate tweets
* Highly available

---

# 2. Biggest Design Decision

The interviewer almost always asks:

> How will you generate Home Timeline?

This is the heart of Twitter.

Two approaches exist.

---

# Option 1

## Pull Model (Fanout on Read)

When user opens app

```
Read all followings

Fetch latest tweets

Merge

Sort

Return
```

Example

Alice follows

```
Bob
Tom
Jerry
```

Open app

Server queries

```
Bob tweets

Tom tweets

Jerry tweets

Merge

Sort

Return
```

Advantages

* Easy write

Disadvantages

Very expensive read.

Imagine

```
User follows

5000 users
```

Need

```
5000 DB reads

merge

sort
```

Impossible.

---

# Option 2

## Push Model (Fanout on Write)

When Bob tweets

Immediately push tweet

into timeline cache of

all followers.

Instead of

```
User opens app

Generate timeline
```

We do

```
Tweet created

↓

Fanout Service

↓

Insert into follower feeds

↓

Ready
```

Read becomes

```
Redis GET
```

Very fast.

---

Interview expects you to know

Twitter uses mostly Push Model.

---

# Problem

Celebrity problem.

Imagine

```
Elon Musk

200 Million followers
```

New tweet

Need

```
Insert

200M timelines
```

Impossible.

This is called

Celebrity Fanout Problem.

---

# Hybrid Fanout

Real Twitter uses hybrid.

Normal user

```
Push
```

Celebrity

```
Pull
```

Meaning

When celebrity tweets

Don't push.

Instead

Store once.

While opening app

Merge celebrity tweets.

Result

```
Regular user

Redis feed

+

Celebrity latest tweets
```

This is one of the most common interview questions.

---

# 3. Architecture

```
            API Gateway

                  │

         Tweet Service

                  │

        Kafka (Tweet Created)

                  │

       Fanout Service

      /             \

Timeline Cache     Search Index

      │

Redis Feed Cache

      │

Timeline Service

      │

Client
```

Kafka decouples tweet creation from expensive downstream work.

---

# 4. Tweet Creation Flow

```
User

↓

Tweet API

↓

Store tweet

↓

Publish Kafka Event

↓

Return Success
```

Don't wait for fanout.

Otherwise

Latency becomes huge.

Fanout happens asynchronously.

---

# 5. Why Kafka?

Suppose

100 downstream services.

Need

* Timeline
* Notification
* Analytics
* Trending
* Search
* Recommendation
* ML models

Without Kafka

Tweet service calls

```
Timeline

↓

Search

↓

Notification

↓

Analytics
```

Latency

Huge.

Instead

```
Tweet stored

↓

Kafka Event

↓

Everyone consumes
```

Loose coupling.

---

# 6. Feed Cache

Most important cache.

Instead of

```
DB

↓

Generate feed
```

Store timeline directly.

Example

Redis

```
User 101

↓

Tweet100

Tweet99

Tweet98
```

Opening app

```
GET Timeline:101
```

O(1)

---

# 7. Timeline Cache Invalidation

Suppose

Bob deletes tweet.

Need remove

from

Millions of feeds.

Hard.

Interview answer

Use

Lazy deletion.

Timeline still contains tweet id.

While fetching

```
Tweet exists?

Yes

Return

No

Skip
```

Eventually

Background cleanup.

---

# 8. Feed Ranking

Don't simply sort by timestamp.

Ranking Service scores tweets.

Example

```
Score

=

Freshness

+

Likes

+

Replies

+

Relationship

+

ML score
```

Timeline Service fetches candidates and the Ranking Service orders them before returning results.

---

# 9. Redis Feed Size

Should we store entire history?

No.

Only latest

```
500

or

1000 tweets
```

Older tweets

Read from DB.

Why?

Otherwise

Redis becomes massive.

---

# 10. Database Choice

Tweet Storage

Use

Wide Column

or

Distributed KV

```
Cassandra

ScyllaDB
```

Why?

Tweet append.

Never updated.

Perfect for LSM trees.

Schema

```
Tweet

TweetID

UserID

Timestamp

Content

MediaURL
```

Partition

```
UserID
```

Cluster

```
Timestamp DESC
```

Efficient user timeline queries.

---

# 11. Why not MySQL?

Problem

```
50 Billion tweets
```

Single machine impossible.

Need

Horizontal scaling.

Cassandra

```
Automatic sharding

Replication

Fast append

High availability
```

---

# 12. Home Timeline Storage

Separate from Tweet table.

```
Redis

User

↓

List

Tweet IDs
```

Only IDs.

Actual tweet fetched separately.

Reduces duplication.

---

# 13. Like Count Problem

Millions click Like simultaneously.

Bad approach

```
UPDATE Tweet

SET Likes=Likes+1
```

Race condition.

---

Better

Atomic increment.

Redis

```
INCR Tweet:100
```

Later flush.

---

# 14. Distributed Counter

Even Redis can become hot.

Solution

Sharded counters.

Instead of

```
Tweet1

Likes
```

Maintain

```
Shard1

Shard2

Shard3

Shard4
```

Final

```
SUM()
```

Google uses this technique.

---

# 15. Trending Hashtags

Huge interview favorite.

Every tweet

Extract hashtags.

Publish Kafka.

Trending Service

Consumes.

Need

Top K.

Don't sort everything.

Use

```
Min Heap

Size K
```

Or

```
Count-Min Sketch
```

Approximate counting is often preferred because exact counting for millions of hashtags is expensive.

---

# 16. Search

Don't search DB.

Need inverted index.

Pipeline

```
Tweet

↓

Kafka

↓

Indexer

↓

Search Engine
```

Typically implemented with a search engine such as Elasticsearch/OpenSearch.

---

# 17. Media

Never store videos in DB.

Store

Object Storage

Return

CDN URL.

---

# 18. Celebrity Hot Partition

Suppose

Partition

```
UserID
```

Elon

```
200M tweets read
```

One partition becomes hot.

Solutions

* Read replicas
* Cache
* CDN for media
* Dedicated shards for hot users
* Separate timeline strategy

---

# 19. Idempotency

Suppose

Kafka retries.

Tweet event processed twice.

Need avoid duplicate fanout.

Solution

Consumer stores processed event IDs (or uses an idempotency key) before applying updates, so repeated deliveries don't duplicate timeline entries.

---

# 20. Ordering Problem

Two tweets

```
A

B
```

Arrive

```
B

A
```

Need ordering.

Use

```
Snowflake IDs

or

Timestamp + sequence
```

Timeline sorts by these IDs rather than arrival time.

---

# 21. Read-after-Write Consistency

User tweets.

Immediately refreshes.

Tweet missing.

Why?

Fanout is asynchronous.

Solutions

* Merge recent tweets from the author's own write path.
* Temporarily query the tweet store for the author's newest posts.
* Accept eventual consistency for followers.

---

# 22. Follow Operation

When Alice follows Bob

Should we backfill years of tweets?

No.

Usually

Last

```
100

or

500 tweets
```

are inserted into Alice's timeline.

---

# 23. Pagination

Never use

```
OFFSET
```

Use cursor pagination.

```
TweetID

↓

Older Tweets

↓

Next Cursor
```

Cursor-based pagination avoids scanning skipped rows and performs well at large scale.

---

# 24. Interview Trade-offs

| Problem            | Naive Solution                     | Senior-Level Solution                                       |
| ------------------ | ---------------------------------- | ----------------------------------------------------------- |
| Home Timeline      | Query on every read                | Hybrid fanout (push for normal users, pull for celebrities) |
| Feed Storage       | Database                           | Redis timeline cache with tweet IDs                         |
| Tweet Distribution | Synchronous calls                  | Kafka event-driven fanout                                   |
| Like Counter       | Database update                    | Sharded distributed counters                                |
| Trending           | Full sort                          | Count-Min Sketch + Top-K heap                               |
| Search             | SQL `LIKE`                         | Search index with inverted index                            |
| Pagination         | OFFSET                             | Cursor-based pagination                                     |
| Feed Ranking       | Timestamp only                     | ML ranking + engagement signals                             |
| Deletes            | Remove from every feed immediately | Lazy deletion with background cleanup                       |
| Fanout             | Always push                        | Push for normal users, pull for celebrities                 |

## Tough follow-up questions interviewers ask

1. **How would you prevent duplicate tweets in follower timelines if Kafka delivers the same event twice?**

   * Use idempotent consumers with event IDs and deduplicate timeline inserts.

2. **What happens if the fanout service crashes halfway through pushing a tweet?**

   * Kafka retains the event, and the consumer resumes from the last committed offset, relying on idempotency for safe retries.

3. **How do you handle a user following 1 million accounts?**

   * Don't merge all followings on every request. Apply limits, precomputed feeds, caching, and selectively pull from active or celebrity accounts.

4. **How do you support global users with low latency?**

   * Use regional deployments, geo-replicated databases, regional Redis caches, and CDNs for media.

5. **How would you redesign the system if tweets could be edited?**

   * Treat edits as versioned updates, propagate invalidation events, and update cached copies or fetch the latest tweet body by ID to avoid stale content.

These are the kinds of questions that distinguish a senior engineer from someone who only knows the standard architecture diagram.
