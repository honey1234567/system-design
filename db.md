Yes, you **can store large text in a normal database**, but whether you *should* depends on the use case.

## Option 1: Store Large Text in Database

Example:

```sql
CREATE TABLE documents (
    id BIGINT PRIMARY KEY,
    title VARCHAR(255),
    content TEXT
);
```

or

```sql
content LONGTEXT
```

in MySQL.

### Advantages

✅ ACID transactions

✅ Easy querying

✅ Backup and replication handled by DB

✅ Good for KBs to a few MBs of text

### Problems

If documents become very large (10MB, 100MB, 1GB):

* Database size grows rapidly
* Indexes become larger
* Replication becomes slower
* Backups become huge
* Read/write performance degrades
* Expensive storage on DB servers

Imagine storing 100 million documents averaging 5MB each:

```text
100M × 5MB = 500 TB
```

Keeping 500 TB in PostgreSQL/MySQL is extremely expensive.

---

# Option 2: Store Text in Object Storage

Store file in:

* Amazon S3
* Google Cloud Storage
* Azure Blob Storage

Database only stores metadata:

```sql
documents
----------
id
title
s3_url
created_at
```

Actual text:

```text
s3://bucket/documents/123.txt
```

### Advantages

✅ Very cheap

✅ Virtually unlimited storage

✅ Easy horizontal scaling

✅ Better for large files

✅ CDN integration

### Disadvantages

❌ Cannot run SQL queries on content directly

❌ Need extra network call

❌ Slightly higher latency

---

# Which Is Faster?

### Small Text (<100KB)

Database wins.

```text
DB Query
   ↓
Content Returned
```

Single operation.

Latency:

```text
5-20 ms
```

---

### Large Documents (10MB+)

Object storage usually wins.

```text
DB Query
   ↓
Get Metadata
   ↓
Fetch From S3
```

Latency:

```text
20-100 ms
```

But databases struggle when millions of huge documents are stored.

---

# Real Systems

## Google Docs

Document metadata:

```text
Document ID
Owner
Permissions
Version
```

stored in database.

Actual document content:

```text
Object Storage
```

Snapshots are often stored in object storage.

---

## Slack

Messages:

```text
Database
```

Attachments:

```text
Object Storage (S3)
```

---

## YouTube

Metadata:

```text
Database
```

Videos:

```text
Object Storage
```

Storing videos in a relational database would be impractical.

---

# HLD Interview Rule

A common interview guideline:

| Size               | Storage                    |
| ------------------ | -------------------------- |
| Few KB             | Database                   |
| Hundreds KB        | Database or Object Storage |
| MBs to GBs         | Object Storage             |
| Images/Videos/PDFs | Object Storage             |
| Metadata           | Database                   |

---

# Example: Google Docs Design

Instead of:

```sql
documents
---------
id
content LONGTEXT
```

use:

```sql
documents
---------
id
owner
snapshot_path
version
```

```text
snapshot_path
=
s3://docs/123/v45.snapshot
```

The database remains small and fast, while object storage holds the large document snapshots.

### Interview Answer

If asked *"Why not store everything in DB?"*:

> Databases are optimized for structured data, transactions, and indexing. Large blobs or documents increase storage cost, replication time, backup size, and query latency. Object storage is cheaper, infinitely scalable, and optimized for storing large immutable content. Therefore, we store metadata in the database and large content in object storage.

This is one of the most common HLD interview follow-ups:

> "Traffic suddenly becomes 100x. Database is the bottleneck. App servers cannot scale further. What do you do?"

The answer depends on the bottleneck.

---

# Step 1: Identify Bottleneck

```text
Users
  ↓
Load Balancer
  ↓
App Servers
  ↓
Database
```

Monitor:

* CPU
* Memory
* Disk IOPS
* Network
* Query latency
* Connection count

Ask:

> Is DB CPU high?
>
> Is disk saturated?
>
> Too many reads?
>
> Too many writes?
>
> Too many connections?

Different bottlenecks require different solutions.

---

# Case 1: Read Traffic is High (Most Common)

Suppose:

```text
100,000 QPS

95% Reads
5% Writes
```

Example:

* Yelp search
* Instagram feed
* Netflix homepage

---

## Solution 1: Redis Cache

Before:

```text
App
 ↓
DB
```

After:

```text
App
 ↓
Redis
 ↓
DB
```

```text
Cache Hit → Redis
Cache Miss → DB
```

If cache hit ratio:

```text
95%
```

Then:

```text
100k requests

95k → Redis
5k → DB
```

DB load drops dramatically.

---

## Solution 2: CDN

For images/videos:

```text
User
 ↓
CDN
 ↓
Origin
```

Example:

* Profile photos
* Product images
* Static assets

CDN removes traffic from both app servers and DB.

---

## Solution 3: Read Replicas

```text
          Master
             |
    ----------------
    |              |
Replica1      Replica2
```

Writes:

```text
Master
```

Reads:

```text
Replicas
```

Example:

```text
Master 10k QPS
Replica1 10k QPS
Replica2 10k QPS
```

Total:

```text
30k QPS
```

Common in:

* MySQL
* PostgreSQL

---

# Case 2: Write Traffic is High

Harder problem.

Example:

* WhatsApp messages
* Google Docs edits
* Payment transactions

---

## Solution 1: Queue Writes

Instead of:

```text
User
 ↓
DB
```

Do:

```text
User
 ↓
Kafka
 ↓
Consumers
 ↓
DB
```

Now traffic spikes are absorbed.

```text
100k requests/sec
```

Kafka stores them.

Consumers process gradually.

---

## Solution 2: Batch Writes

Bad:

```text
1 DB write per event
```

Good:

```text
1000 events
 ↓
single batch write
```

Example:

```sql
INSERT INTO logs VALUES (...1000 rows...)
```

Far more efficient.

---

# Case 3: Too Many DB Connections

Example:

```text
500 App Servers
```

Each opens:

```text
100 connections
```

Total:

```text
50,000 connections
```

DB dies.

---

## Solution: Connection Pooling

Use:

* PgBouncer
* HikariCP
* ProxySQL

```text
Apps
 ↓
Connection Pool
 ↓
DB
```

50,000 logical connections become:

```text
500 actual DB connections
```

---

# Case 4: Single Database Machine Reached Limit

Vertical scaling exhausted.

Need horizontal scaling.

---

## Sharding

Before:

```text
DB1
```

After:

```text
User 1-1M  → DB1
User 1M-2M → DB2
User 2M-3M → DB3
```

Shard key:

```text
user_id % N
```

Now load is distributed.

---

# Case 5: Hot Records

Classic interview question.

Example:

```text
Taylor Swift profile
```

or

```text
World Cup Final score
```

Millions of users hit same row.

---

## Solution

Cache aggressively.

```text
Redis
```

or

```text
Materialized View
```

or

```text
Precomputed data
```

Never let every request hit DB.

---

# If App Servers Cannot Scale

Sometimes CPU is already maxed.

Then reduce work per request.

---

## Move Computation Offline

Bad:

```text
Request
 ↓
Calculate Top K
 ↓
Return
```

Good:

```text
Kafka
 ↓
Flink
 ↓
Precompute Top K
 ↓
Redis
```

Request becomes:

```text
GET Top K
```

from Redis.

---

# Real Example: Instagram Feed

Naive:

```text
Open App
 ↓
Generate Feed
 ↓
Read millions of rows
```

Peak traffic kills DB.

Instead:

```text
Post Created
 ↓
Kafka
 ↓
Feed Service
 ↓
Redis
```

Feed already prepared.

User request:

```text
Redis Lookup
```

Milliseconds.

---

# Interview Framework

Whenever asked:

> "Database is bottleneck during peak traffic, what will you do?"

Answer in this order:

```text
1. Add Cache (Redis)
2. Add CDN (static content)
3. Add Read Replicas
4. Use Connection Pooling
5. Queue Writes (Kafka)
6. Batch Writes
7. Shard Database
8. Precompute Heavy Queries
9. Eventually move to distributed storage
```

A strong HLD candidate first tries **cache → replicas → queues**, and only then moves to **sharding**, because sharding significantly increases operational complexity.

This is a very important HLD topic. Scaling traffic and handling concurrency are **different problems**.

* **Scaling problem** → Can system handle 1M requests/sec?
* **Concurrency problem** → What if 1000 users modify the same data at the same time?

Even if your DB can handle millions of requests, concurrency bugs can still corrupt data.

---

# Example: Bank Account

Initial balance:

```text
Balance = 1000
```

Two requests arrive simultaneously:

```text
Request A: Withdraw 100
Request B: Withdraw 200
```

Both read:

```text
Balance = 1000
```

Then:

```text
A writes 900
B writes 800
```

Final balance:

```text
800
```

Expected:

```text
700
```

This is called a **Lost Update** problem.

---

# Concurrency at App Server Layer

Suppose:

```text
LB
 ↓
App1
App2
App3
```

User updates the same document.

```text
Request A → App1
Request B → App2
```

Each server has its own memory.

```text
App1 thinks value = X
App2 thinks value = X
```

Both update.

Result:

```text
Data corruption
```

---

# Solution 1: Database Transactions

```sql
BEGIN;

UPDATE account
SET balance = balance - 100
WHERE id = 1;

COMMIT;
```

Database ensures atomicity.

---

# Solution 2: Row Locking

```sql
SELECT *
FROM account
WHERE id=1
FOR UPDATE;
```

Flow:

```text
Request A acquires lock
Request B waits
```

Then:

```text
A commits
B proceeds
```

No lost updates.

---

# Pessimistic Locking

Assume conflict will happen.

```text
Acquire lock first
Then update
```

Example:

```sql
SELECT * FROM seats
WHERE seat_id=1
FOR UPDATE;
```

Used in:

* Payments
* Inventory
* Seat booking

---

# Problems with Pessimistic Locking

```text
High contention
Threads waiting
Deadlocks
Reduced throughput
```

Example:

```text
1000 users
1 concert seat
```

999 users wait.

---

# Optimistic Locking

Assume conflicts are rare.

Table:

```text
id
balance
version
```

Before update:

```text
balance=1000
version=5
```

Update:

```sql
UPDATE account
SET balance=900,
    version=6
WHERE id=1
AND version=5;
```

Only one request succeeds.

Others fail and retry.

---

# Why Optimistic Locking Scales Better

No waiting.

```text
Read
Compute
Try update
```

If conflict:

```text
Retry
```

Great for:

* User profiles
* Documents
* Product edits

---

# Google Docs Example

Two users edit same document.

Without concurrency control:

```text
User A writes line 10
User B writes line 10
```

One edit disappears.

Google Docs uses:

* OT (Operational Transformation)
* CRDT

instead of database locks.

---

# Concurrency in Distributed App Servers

Suppose:

```text
App1
App2
App3
```

All try to process:

```text
Order #123
```

Problem:

```text
Duplicate processing
```

Example:

```text
Payment charged twice
Email sent twice
```

---

# Solution: Distributed Lock

Use Redis.

```text
SET order_123 locked NX EX 30
```

Meaning:

```text
Acquire lock only if absent
Expire after 30s
```

Only one server gets lock.

---

# Example: Inventory System

Stock:

```text
iPhone = 1
```

Two buyers.

Without lock:

```text
Buyer A sees stock=1
Buyer B sees stock=1
```

Both buy.

Final:

```text
Stock = -1
```

Overselling.

---

# Better Solution: Atomic DB Update

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id=1
AND stock > 0;
```

Check affected rows.

```text
1 row updated → success
0 rows updated → sold out
```

No lock needed.

This is a favorite interview answer.

---

# Issues If You Don't Handle Concurrency

## 1. Lost Updates

```text
1000 → 900
1000 → 800
```

Expected:

```text
700
```

Got:

```text
800
```

---

## 2. Double Payment

```text
Charge #1
Charge #2
```

Customer billed twice.

---

## 3. Overselling Inventory

```text
Stock = 1

User A buys
User B buys
```

Both succeed.

---

## 4. Duplicate Messages

```text
Kafka retry
```

Same event processed twice.

---

## 5. Dirty Reads

Transaction A:

```text
Balance=1000 → 500
```

Not committed.

Transaction B reads:

```text
500
```

A rolls back.

B saw invalid data.

---

## 6. Deadlocks

Transaction A:

```text
Lock Row1
Wait Row2
```

Transaction B:

```text
Lock Row2
Wait Row1
```

Both stuck.

---

# Interview Cheat Sheet

| Scenario                                 | Best Solution             |
| ---------------------------------------- | ------------------------- |
| Bank transfer                            | Transaction + row lock    |
| Seat booking                             | Pessimistic lock          |
| Inventory                                | Atomic update             |
| User profile update                      | Optimistic lock           |
| Google Docs                              | OT / CRDT                 |
| Multiple app servers processing same job | Distributed lock          |
| Kafka duplicate events                   | Idempotency key           |
| Payment processing                       | Transaction + idempotency |

---

A strong HLD answer usually separates concurrency handling into **three layers**:

```text
1. Application Layer
   - Optimistic Lock
   - Distributed Lock
   - Idempotency

2. Database Layer
   - Transactions
   - Row Locks
   - Isolation Levels

3. Distributed System Layer
   - Kafka Ordering
   - Leader Election
   - OT/CRDT
```

Many candidates discuss only database locks, but senior-level interviews often expect you to explain how concurrency is handled across **multiple application servers, caches, queues, and databases simultaneously**.
