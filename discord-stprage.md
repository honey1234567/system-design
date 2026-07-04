This is one of the best real-world system design articles because it explains **how a system evolves when scale breaks your original design**.

Below is a beginner-friendly summary that covers **all the important engineering concepts** without skipping anything.

---

# Big Picture

## Problem

Discord stores **trillions of chat messages**.

Back in 2017 they had:

* Billions of messages
* 12 Cassandra nodes

By 2022 they had:

* Trillions of messages
* 177 Cassandra nodes

The database was becoming increasingly difficult to operate.

The article is **not about how to design a chat system from scratch**.

Instead, it answers:

> "What happens when your database that worked for years suddenly starts failing because your company became huge?"

([Discord][1])

---

# Original Architecture

```
User
   │
   ▼
Discord API
   │
   ▼
Cassandra Database
```

Messages were stored in Cassandra.

Why Cassandra?

* Distributed database
* Easy horizontal scaling
* High write throughput
* Fault tolerant

At that time it worked well.

---

# How Messages Were Stored

Every Discord message has

* Message ID
* Channel ID
* Timestamp
* Content

Discord uses **Snowflake IDs**.

A Snowflake ID contains time information.

That means

```
Higher ID
↓

Newer message
```

So they don't need an extra timestamp index.

Messages naturally stay ordered.

---

# How Cassandra Stores Messages

Instead of storing all messages together, Discord partitions them.

Partition Key

```
Channel ID
+
Time Bucket
```

Example

```
General Channel

Bucket 1
------------
Message A
Message B
Message C

Bucket 2
------------
Message D
Message E
```

A bucket is simply a fixed time window.

For example

```
Jan 1
Jan 2
Jan 3
```

or

```
One Hour
```

depending on implementation.

This prevents one partition from becoming infinitely large.

---

# Why Partition by Channel?

Imagine

Channel A

```
100 messages/day
```

Channel B

```
20 million messages/day
```

If every message of Channel B goes to one partition,

that partition becomes huge.

This becomes the biggest problem.

---

# Hot Partition Problem

Suppose everyone is watching FIFA World Cup Final.

A server has

```
1 million users
```

Someone posts

```
@everyone
```

Now

```
100,000 people
```

open that message.

Every request goes to

```
Same channel

↓

Same partition

↓

Same Cassandra node
```

One node suddenly receives

```
100,000 reads
```

Other nodes

```
Idle
```

This is called a

# Hot Partition

One partition receives enormous traffic while others don't.

Result

* High latency
* Timeouts
* Node overload
* Cascading failures

([Discord][1])

---

# Why Cassandra Started Struggling

As Discord grew,

some servers had

```
50 friends
```

while others had

```
500,000 members
```

Large servers generated enormous traffic.

So

```
Large Server

↓

One Channel

↓

One Partition

↓

One Node

↓

Overloaded
```

Even though the cluster had

177 nodes,

only a few were overloaded.

---

# What Happens During Hot Partition?

Imagine

100,000 requests arrive.

Database can process

```
5000/sec
```

Remaining

```
95,000
```

wait.

Now more requests arrive.

Queue grows.

Latency becomes

```
50 ms

↓

200 ms

↓

1 second

↓

Timeout
```

Now clients retry.

Retries generate even more traffic.

Eventually the database melts down.

This is called **cascading latency**.

---

# First Thought

"We'll migrate to ScyllaDB."

But they realized

> A new database alone won't solve hot partitions.

Even ScyllaDB would suffer if all requests hit the same partition.

So they solved the root cause first.

---

# Introducing Data Services

Instead of

```
API

↓

Database
```

They added a middle layer.

```
Client

↓

API

↓

Data Service

↓

Database
```

This became one of the biggest improvements.

---

# Why Data Service?

Because many users ask for the same data simultaneously.

Example

100,000 users open the same announcement.

Without Data Service

```
100,000 API Requests

↓

100,000 Database Reads
```

Terrible.

---

# Request Coalescing

This is the key idea.

Instead of

```
Request 1

↓

Database

Request 2

↓

Database

Request 3

↓

Database
```

they do

```
Request 1

↓

Database

Request 2

↓

Wait

Request 3

↓

Wait

Database returns

↓

All receive same response
```

One database query.

Thousands of users.

Huge savings.

This technique is called **request coalescing**.

---

# Is This Caching?

No.

Many beginners confuse them.

Caching

```
Database once

↓

Save result

↓

Future requests

↓

Cache
```

Coalescing

```
Database query still happens

BUT

duplicate requests are merged while the query is running.
```

So

```
1000 concurrent requests

↓

1 query
```

instead of

```
1000 queries
```

---

# Consistent Hash Routing

Now another problem.

Suppose Data Service has

```
10 instances
```

Without routing

```
Request 1

↓

Server 1

Request 2

↓

Server 4

Request 3

↓

Server 8
```

Each server creates its own database query.

No coalescing.

So Discord routes using

```
Channel ID
```

Every request for

```
Channel 123
```

always reaches

```
Data Service 7
```

Now

```
1000 users

↓

Same service

↓

One query
```

Much better.

([Discord][1])

---

# Why Rust?

Discord wrote the Data Service in **Rust**.

Reasons:

* Very fast (close to C/C++)
* Memory safe
* Excellent concurrency support
* Strong async ecosystem (Tokio)
* Drivers for Cassandra and ScyllaDB
* Reliable once compiled

Rust made it easier to build highly concurrent services safely. ([Discord][1])

---

# Why Move to ScyllaDB?

Discord had already migrated most other databases to **ScyllaDB** by 2020.

Benefits they observed:

* Better performance
* Lower operational overhead
* Better scalability
* Efficient resource usage

One missing feature initially was fast reverse queries (reading in the opposite sort order), but after improvements from the ScyllaDB team, it met Discord's needs. ([Discord][1])

---

# Migration Requirements

They needed to:

* Migrate trillions of messages
* Zero downtime
* No data loss
* Keep serving users during migration

---

# First Migration Plan

Idea:

```
Old messages

↓

Cassandra

New messages

↓

ScyllaDB
```

Then migrate historical data later.

They also planned to **dual-write** new messages to both databases during the transition.

However, the migration tools estimated about **three months** to complete, which was too slow. ([Discord][1])

---

# Rust Migrator

Instead of relying only on the existing migration approach, they extended their Rust data-service library into a migration tool.

The migrator:

* Reads token ranges from Cassandra
* Tracks progress locally with SQLite checkpoints
* Streams data into ScyllaDB efficiently

The estimated migration time dropped from about **3 months to roughly 9 days**. ([Discord][1])

---

# Migration Speed

Peak throughput reached about:

**3.2 million messages per second**

At the end, progress appeared stuck at **99.9999%** because some Cassandra ranges contained huge numbers of **tombstones** (markers left after deletions) that hadn't been compacted. After compacting those ranges, the migration finished. ([Discord][1])

---

# Data Validation

Discord didn't simply trust the migration.

For a small percentage of reads they:

1. Read from Cassandra.
2. Read from ScyllaDB.
3. Compared the results automatically.

Once the outputs matched consistently, they switched production traffic to ScyllaDB. ([Discord][1])

---

# Results

The improvements were substantial:

| Metric                        | Cassandra | ScyllaDB |
| ----------------------------- | --------- | -------- |
| Nodes                         | 177       | 72       |
| Storage per node              | ~4 TB     | ~9 TB    |
| Historical message read (p99) | 40–125 ms | ~15 ms   |
| Message insert (p99)          | 5–70 ms   | ~5 ms    |

The new system also handled massive spikes, such as traffic during the **2022 FIFA World Cup Final**, without the operational issues that previously required frequent intervention. ([Discord][1])

---

# Key System Design Lessons

1. **A database alone won't solve scaling issues** if the traffic pattern (such as hot partitions) remains unchanged.
2. **Hot partitions** are one of the biggest challenges in distributed databases.
3. **Request coalescing** can dramatically reduce duplicate database work during traffic spikes.
4. **Consistent hashing** helps route related requests to the same service instance, making coalescing effective.
5. **Migrate carefully** using dual writes, validation, and incremental rollout to avoid downtime.
6. **Measure before replacing**—Discord first identified the true bottlenecks, then redesigned the architecture around them.
7. **Operational simplicity matters**: reducing on-call incidents and maintenance effort was as important as improving raw performance.

---

## Interview Takeaways

If you're asked about this article in a system design interview, the main concepts to mention are:

* Cassandra partitioning using **Channel ID + Time Bucket**
* Snowflake IDs for chronological ordering
* Hot partitions and cascading latency
* Data Service layer between API and database
* Request coalescing (different from caching)
* Consistent hash routing
* Rust for high-performance concurrent services
* Migration from Cassandra to ScyllaDB
* Dual writes and automated data validation
* Scaling from billions to trillions of messages with zero downtime

These are the core engineering ideas that made Discord's redesign successful. ([Discord][1])

[1]: https://discord.com/blog/how-discord-stores-trillions-of-messages?utm_source=chatgpt.com "How Discord Stores Trillions of Messages"
