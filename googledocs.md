A High-Level Design (HLD) for something like Google Docs is mostly about **real-time collaborative editing at internet scale** while keeping latency low and data consistent.

## Requirements

### Functional

1. Create/edit documents
2. Real-time collaboration (multiple users edit simultaneously)
3. Auto-save
4. Version history
5. Share with permissions (view/edit/comment)
6. Comments and suggestions
7. Search documents
8. Offline editing + sync later

### Non-functional

* Low latency (<100–200 ms for collaboration feel)
* High availability
* Scalability (millions of users)
* Durability
* Conflict resolution
* Security and access control

---

## High-level architecture

```text
                  +----------------+
Users(Web/Mobile) | Browser Client |
                  +--------+-------+
                           |
                    Load Balancer
                           |
          +----------------+----------------+
          |                                 |
   API Gateway                       WebSocket Gateway
          |                                 |
          |                                 |
+---------+--------+             +----------+---------+
| Auth Service     |             | Collaboration Svc |
| User Service     |             | (real-time edits) |
+------------------+             +----------+---------+
                                             |
                                   +---------+---------+
                                   |
                           Conflict Resolution
                          (OT / CRDT engine)
                                   |
              +--------------------+------------------+
              |                                       |
      Document Service                    Presence Service
              |                                       |
              |                                Active users
              |
      +-------+----------+
      |                  |
Metadata DB         Document Storage
(PostgreSQL)          (Blob/S3)
      |
      |
Version Service
      |
Search Index
(Elasticsearch)
```

---

## Major components

### 1. Client application

Browser does more work than many people expect.

Maintains:

* Current document state
* Cursor positions
* Pending local edits
* Local cache
* Offline changes queue

Example:

User types:

```text
Hello
```

Client converts it into operation:

```json
{
   "operation":"insert",
   "position":5,
   "character":"o"
}
```

Instead of sending the whole document:

```text
Hell -> Hello
```

send only the delta:

```text
Insert('o', position=5)
```

This reduces bandwidth.

---

### 2. WebSocket Gateway

HTTP is not ideal for continuous updates.

Real-time edits use:

```text
Browser
   ↓
WebSocket connection
   ↓
Collaboration Service
```

Benefits:

* Persistent connection
* Low latency
* Bidirectional communication

---

### 3. Collaboration Service

Core of the system.

Responsibilities:

* Receive edits
* Order edits
* Broadcast edits
* Handle user sessions
* Maintain document rooms

Example:

Users:

```text
User A edits Doc123
User B edits Doc123
User C edits Doc123
```

Server creates:

```text
Room: Doc123
```

All edits flow through this room.

---

### 4. Conflict resolution

Main challenge:

Two users edit same location simultaneously.

Example:

Document:

```text
Hello
```

User A:

```text
Insert "X" at position 2
```

User B:

```text
Delete character at position 2
```

Without coordination:

```text
HeXllo
or
Hello
or
HeXlo
```

Different clients may diverge.

Two popular approaches:

### Operational Transformation (OT)

Used historically by Google Docs.

Flow:

```text
A → operation1
B → operation2

Server transforms:

operation2 relative to operation1
```

Example:

```text
A: Insert X at position 2
B: Insert Y at position 2

Transform:

A → position 2
B → position 3
```

Result:

```text
HeXYllo
```

---

### CRDT

Alternative approach.

Each operation gets unique IDs.

Operations merge automatically.

Good for:

* Offline editing
* Peer-to-peer collaboration

Tradeoff:

* More metadata
* Higher memory usage

---

## Document storage

Split into:

### Metadata DB

Stores:

```text
doc_id
owner
permissions
created_time
last_modified
```

Use:

```text
PostgreSQL
```

---

### Document content storage

Large content:

```text
document_body
images
attachments
```

Use:

```text
Object storage:

S3/GCS
```

---

## Version history

Instead of storing full document every keystroke:

Bad:

```text
Version1 = full doc
Version2 = full doc
Version3 = full doc
```

Use delta storage:

```text
V1

+ Insert "Hello"
+ Delete "a"
+ Insert "world"
```

Reconstruction:

```text
Base + deltas
```

Checkpoint occasionally:

```text
V100 full snapshot
```

Otherwise replay becomes expensive.

---

## Auto-save flow

```text
User typing
    ↓
Local buffer
    ↓
Every few seconds
    ↓
Send batch of operations
    ↓
Persist in storage
```

Instead of:

```text
save on every keystroke
```

Batching reduces DB writes.

---

## Search system

Documents indexed asynchronously.

```text
Document saved
      ↓
Kafka event
      ↓
Search Indexer
      ↓
Elasticsearch
```

Search query:

```text
"meeting notes"
```

returns:

```text
Doc1
Doc2
Doc3
```

---

## Scaling strategy

### Collaboration service

Problem:

One server cannot hold all active documents.

Partition by:

```text
hash(document_id)
```

Example:

```text
Doc1 → Server 2
Doc2 → Server 5
Doc3 → Server 1
```

---

### Use cache

```text
Redis
```

For:

* Active documents
* User sessions
* Presence info

---

### Message queue

```text
Kafka
```

For:

* Version events
* Search indexing
* Analytics
* Notifications

---

## End-to-end flow

```text
User A types "H"
       |
       v
Client sends delta
       |
       v
WebSocket Gateway
       |
       v
Collaboration Service
       |
       v
OT engine resolves conflicts
       |
       +----> Broadcast to User B
       |
       +----> Broadcast to User C
       |
       v
Persist operation
       |
       v
Version Service
       |
       v
Storage
```

---

## Common interviewer follow-up: "Why not send entire document every edit?"

Because:

```text
Document = 10 MB
Typing speed = 5 chars/sec
Millions of users
```

Network usage explodes.

Delta approach:

```text
Insert("H",pos=25)
```

might be only a few bytes.

---

For a system design interview, the key discussion areas are usually:

1. Real-time communication → WebSockets
2. Conflict resolution → OT vs CRDT
3. Persistence → snapshot + delta
4. Scale → partitioning + cache + Kafka
5. Reliability → auto-save + versioning

Those five areas are where most interview depth comes from.

Good question. We do **not strictly need a separate WebSocket Gateway**, and Google Docs does **not use only WebSockets for everything**.

## Why a separate WebSocket Gateway?

You could build:

```text
Client
   |
API Service + WebSocket handling
```

for a small system.

But at Google Docs scale, separating it becomes useful because **WebSocket traffic behaves very differently from normal HTTP traffic**.

### HTTP API traffic

Characteristics:

* Short-lived request/response
* Stateless
* Easy to scale horizontally

Example:

```text
POST /createDocument
GET /documents/123
PUT /share
```

Flow:

```text
Browser
   ↓
API Gateway
   ↓
Document service
```

Connection ends after response.

---

### WebSocket traffic

Characteristics:

* Long-lived connection
* Server maintains state
* Millions of concurrent connections
* Continuous messages in both directions

Example:

```text
User typing:

Insert(H,pos=5)
Insert(e,pos=6)
Insert(l,pos=7)
```

Flow:

```text
Browser
   ↓
Persistent WebSocket connection
```

Connection may stay alive for hours.

---

If API servers handled both:

```text
                +-------------+
Users --------->| API Servers |
                +-------------+
```

Problems:

1. API servers become busy maintaining persistent connections
2. Memory usage grows
3. Connection management becomes complex
4. Scaling APIs and real-time traffic independently becomes difficult

---

Instead:

```text
                    +---------------+
Users  ---> LB ---->| API Gateway   |
                    +---------------+
                            |
                +------------------+
                | Document Service |
                +------------------+


                    +------------------+
Users ---> LB ----->| WebSocket Gateway|
                    +------------------+
                             |
                     Collaboration Svc
```

Now you can scale independently:

```text
API servers: 100
WebSocket servers: 5000
```

if collaboration traffic spikes.

---

## Why only WebSocket in the previous design?

Actually we are **not using only WebSockets**.

Google Docs usually uses a mix.

### HTTP/REST

Used for:

* Login
* Create document
* Fetch document
* Share document
* Permissions
* Version history
* Comments
* Search

Example:

```text
GET /docs/123
POST /share
```

These are request-response operations.

---

### WebSocket

Used for:

* Real-time edits
* Cursor movement
* Presence updates
* Typing indicators

Example:

```text
Insert(H,pos=5)
Delete(pos=10)
Cursor(userA,15)
```

---

## Real Google Docs-like flow

```text
User opens document
      |
      v
HTTP request
GET /document/123
      |
      v
Document loaded

      |
      v
Browser opens WebSocket

      |
      v
Real-time collaboration starts

Typing
Cursor movement
Live updates

      |
      v
Auto-save via background API
```

---

## Why not polling?

Polling:

```text
Client:
"Any changes?"
"Any changes?"
"Any changes?"
```

every second.

Problems:

* Huge network waste
* Delay between edits
* Millions of unnecessary requests

---

## Why not Server-Sent Events (SSE)?

SSE:

```text
Server ---> Client
```

only one direction.

Google Docs needs:

```text
Client ---> Server
Server ---> Client
```

because users constantly send edits.

So SSE alone is insufficient.

---

Think of it like this:

* **HTTP API Gateway** → receptionist handling individual requests
* **WebSocket Gateway** → conference bridge keeping thousands of calls open
* **Collaboration Service** → moderator coordinating everyone's edits

That's why large collaborative systems split them.

The core problem both algorithms solve is:

> Multiple users edit the same document at the same time and all clients must eventually see the same final document.

Example:

```text
Initial document: "Hello"
```

User A and User B edit simultaneously.

---

# 1. Operational Transformation (OT)

OT does **not merge states**. It **transforms incoming operations relative to other operations** so that everyone reaches the same result.

Think of it as:

```text
"My operation was created on an older version,
so adjust it according to newer operations."
```

---

## Example 1: Two inserts at same position

Initial:

```text
Hello
```

Simultaneous edits:

```text
User A: Insert X at position 2
User B: Insert Y at position 2
```

Operations:

```text
A → Insert(X,2)
B → Insert(Y,2)
```

Server receives:

```text
A first
B second
```

Without OT:

Apply A:

```text
HeXllo
```

Apply B directly:

```text
HeYXllo
```

Different users might end up with different ordering.

---

OT transforms B against A.

Transformation rule:

```text
If two inserts happen at same position,
shift later operation right
```

Transform:

```text
B: Insert(Y,2)
→ Insert(Y,3)
```

Now execute:

Apply A:

```text
HeXllo
```

Apply transformed B:

```text
HeXYllo
```

All users receive:

```text
HeXYllo
```

---

## Example 2: Insert vs Delete

Initial:

```text
Hello
```

Operations:

```text
User A: Delete position 1 ('e')
User B: Insert X at position 4
```

A:

```text
Delete(1)
```

B:

```text
Insert(X,4)
```

Apply A:

```text
Hllo
```

Document became shorter.

B was created assuming:

```text
Hello
```

Position 4 no longer means the same thing.

Transform B:

```text
Insert(X,4)
→ Insert(X,3)
```

Final:

```text
HllXo
```

---

## Simplified OT algorithm

```text
Receive operation O2

for each previously applied operation O1:

      O2 = Transform(O2,O1)

Apply O2
Broadcast O2
```

Pseudo-code:

```java
Operation transform(Operation current,
                    Operation previous){

    if(previous.type=="insert"
       && previous.pos<=current.pos){

          current.pos++;
    }

    return current;
}
```

---

## OT characteristics

Advantages:

* Small metadata
* Efficient network usage
* Historically used by Google Docs

Disadvantages:

* Transformation logic becomes complicated
* Hard with many concurrent users
* Difficult for offline support

---

# 2. CRDT (Conflict-free Replicated Data Type)

CRDT takes a different approach.

Instead of transforming operations:

```text
Adjust operation → OT
```

CRDT says:

```text
Give every element a unique identity
and merge automatically
```

No central transformation is required.

---

## Example

Initial:

```text
Hello
```

Represent internally as:

```text
H(1)
e(2)
l(3)
l(4)
o(5)
```

Numbers are unique IDs.

---

Two users insert simultaneously:

```text
User A inserts X after e

User B inserts Y after e
```

Generate IDs:

```text
X(2.1)
Y(2.2)
```

Internal structure:

```text
H(1)
e(2)
X(2.1)
Y(2.2)
l(3)
l(4)
o(5)
```

Final:

```text
HeXYllo
```

Everyone reaches the same result because IDs determine ordering.

---

## Offline example

Initial:

```text
Hello
```

User A offline:

```text
Insert X
```

User B offline:

```text
Delete e
```

After reconnect:

A sends:

```text
Insert(X,id=2.1)
```

B sends:

```text
Delete(id=2)
```

Merge:

```text
H
X
l
l
o
```

Result:

```text
HXllo
```

No transformation logic needed.

---

## Simplified CRDT algorithm

Insert:

```java
newNode = Node(
      value='X',
      id=generateUniqueID(),
      parent=existingNodeID
)
```

Merge:

```java
sort(nodes by id)
```

Deletion:

Usually:

```java
mark node deleted
```

instead of physically removing.

Example:

```text
H(1)
e(2) [deleted]
X(2.1)
l(3)
```

---

## Why tombstones?

If you physically remove:

```text
e(2)
```

another user may still reference:

```text
Insert after id=2
```

So CRDT often keeps deleted entries:

```text
e(2)[deleted]
```

called **tombstones**.

---

## OT vs CRDT

| Feature                    |                      OT |                    CRDT |
| -------------------------- | ----------------------: | ----------------------: |
| Approach                   |    Transform operations |            Merge states |
| Central server             |        Usually required |            Not required |
| Offline support            |                    Hard |               Excellent |
| Metadata                   |                   Small |                   Large |
| Complexity                 | Complex transform rules | Complex data structures |
| Storage                    |                 Smaller |                  Larger |
| Google Docs (historically) |                     Yes |                      No |
| Local-first apps           |           Less suitable |                  Better |

---

Visual summary:

```text
OT

A ---> Server
          |
B ---> Transform()
          |
          v
     Same result


CRDT

A ---> add unique IDs
B ---> add unique IDs
          |
          v
       Merge()
```

The intuition:

* **OT:** “Move my edit so it still makes sense.”
* **CRDT:** “Give every edit an identity so merging becomes deterministic.”

For a system like Google Docs, the **client** is what runs on the user's device, and the **server** is the backend infrastructure in the cloud handling collaboration, storage, permissions, synchronization, and persistence.

# What is client vs server here?

## Client (browser/mobile app)

Runs on your laptop or phone.

Responsibilities:

* Render document UI
* Capture typing/cursor movement
* Keep local document state
* Buffer edits
* Maintain WebSocket connection
* Show other users' cursors
* Cache for offline usage

Example:

You type:

```text
Hello
```

The client does not send:

```text
Entire document = "Hello"
```

It sends:

```text
Insert("H",position=0)
Insert("e",position=1)
```

---

## Server (Google cloud backend)

Responsibilities:

* Authenticate users
* Manage permissions
* Receive edits
* Resolve conflicts (OT/CRDT)
* Broadcast edits
* Persist document
* Store versions
* Index for search

---

# Complete HLD

```text
                                 GOOGLE DOCS HLD

 ┌──────────────────────────────── CLIENT SIDE ───────────────────────────────┐
 │                                                                             │
 │   User Browser / Mobile App                                                │
 │                                                                             │
 │  ┌────────────────────────────────────────────────────┐                     │
 │  │ UI Layer                                           │                     │
 │  │                                                     │                     │
 │  │ - Text editor                                       │                     │
 │  │ - Toolbar                                           │                     │
 │  │ - Comments                                           │                    │
 │  │ - Cursor rendering                                   │                    │
 │  └────────────────────────────────────────────────────┘                     │
 │                                                                             │
 │  ┌────────────────────────────────────────────────────┐                     │
 │  │ Local State Manager                                │                     │
 │  │                                                     │                     │
 │  │ - Current document                                  │                     │
 │  │ - Pending edits                                     │                     │
 │  │ - Offline cache                                     │                     │
 │  │ - Cursor position                                   │                     │
 │  └────────────────────────────────────────────────────┘                     │
 │                                                                             │
 │                     │                                                       │
 └─────────────────────┼───────────────────────────────────────────────────────┘
                       │
                       │ HTTP + WebSocket
                       ▼

                 ┌─────────────────┐
                 │ Load Balancer   │
                 └────────┬────────┘
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │

        ▼                                   ▼

┌──────────────────┐              ┌────────────────────┐
│ API Gateway      │              │ WebSocket Gateway  │
│                  │              │                    │
│ Short requests   │              │ Persistent         │
│                  │              │ connections        │
└────────┬─────────┘              └─────────┬──────────┘
         │                                  │
         │                                  │
         ▼                                  ▼

┌──────────────────┐              ┌────────────────────┐
│ Auth Service     │              │ Collaboration Svc  │
│                  │              │                    │
│ Login            │              │ Live document room │
│ Permissions      │              │ Edit processing    │
└────────┬─────────┘              └─────────┬──────────┘
         │                                  │
         │                                  ▼
         │                      ┌─────────────────────┐
         │                      │ OT / CRDT Engine    │
         │                      │                     │
         │                      │ Conflict handling   │
         │                      └─────────┬───────────┘
         │                                │
         │                                │
         ▼                                ▼

                 ┌────────────────────────────┐
                 │ Document Service           │
                 │                            │
                 │ Save/read documents        │
                 └──────────┬─────────────────┘
                            │
           ┌────────────────┴──────────────────┐
           │                                   │

           ▼                                   ▼

┌───────────────────┐              ┌───────────────────┐
│ Metadata DB       │              │ Document Storage  │
│                   │              │                   │
│ docId             │              │ content           │
│ owner             │              │ images            │
│ permissions       │              │ attachments       │
└────────┬──────────┘              └────────┬──────────┘
         │                                  │
         │                                  │
         ▼                                  ▼

┌──────────────────┐               ┌──────────────────┐
│ Version Service  │               │ Cache (Redis)    │
│                  │               │                  │
│ Snapshots        │               │ Active docs      │
│ Delta history    │               │ Sessions         │
└────────┬─────────┘               └────────┬─────────┘
         │                                  │
         └──────────────┬───────────────────┘
                        │
                        ▼

                ┌──────────────────┐
                │ Kafka/Event Bus  │
                └────────┬─────────┘
                         │
                         ▼

               ┌────────────────────┐
               │ Search Indexer     │
               │                    │
               │ Elasticsearch      │
               └────────────────────┘
```

# End-to-end flow when typing one character

Suppose document contains:

```text
Hell
```

You type:

```text
o
```

Flow:

```text
1. Client captures keypress

2. Client creates operation:
   Insert("o",position=4)

3. Sent through WebSocket

4. WebSocket Gateway receives

5. Collaboration Service receives

6. OT/CRDT resolves conflicts

7. Broadcast to other users

8. Persist operation

9. Save to storage

10. Other clients update screen
```

Result:

```text
Hello
```

---

# Why both HTTP and WebSocket?

HTTP:

```text
Login
Create document
Share document
Get history
```

WebSocket:

```text
Typing
Cursor movement
Live collaboration
Presence updates
```

A common interview question is: *"Why not everything through WebSocket?"*

Because:

* Login/search/history are request-response operations
* Real-time editing needs persistent low-latency communication
* Scaling those workloads independently is easier.
In Google Docs-like systems, these three solve **different bottlenecks**:

* **Redis → speed**
* **Kafka → decoupling + reliable event flow**
* **Elasticsearch → fast text search**

They are not interchangeable.

---

# 1. Why Redis is needed

Without cache:

```text
User
   ↓
Document Service
   ↓
Database
```

Imagine document `Doc123` has **50 people collaborating**.

Every keystroke:

```text
Insert(H,10)
Insert(e,11)
Insert(l,12)
```

If each edit hits DB:

```text
50 users × 5 chars/sec
= 250 writes/sec
```

Now millions of documents exist.

Database becomes overloaded.

---

With Redis:

```text
                    ┌─────────┐
User edits ───────► │ Redis   │
                    │ Active  │
                    │ document│
                    └────┬────┘
                         │
                         ▼
                  Database (async)
```

Flow:

1. Load document into Redis
2. Read/write from Redis (memory)
3. Periodically persist to DB

Redis stores:

```text
Doc123:
{
  content:"Hello...",
  users:[A,B,C],
  cursorA:20,
  cursorB:55
}
```

Use cases:

* Active document content
* User sessions
* Presence ("5 users viewing")
* Cursor positions
* Temporary locks

Why?

Memory access:

```text
Redis: ~sub-millisecond
DB: milliseconds to tens of ms
```

---

# 2. Why Kafka is needed

Without Kafka:

```text
Document Service
    |
    ├──> Search Service
    |
    ├──> Analytics Service
    |
    ├──> Notification Service
    |
    └──> Version Service
```

Problem:

If one service becomes slow:

```text
Search Service down
```

Document save may fail or get delayed.

Everything becomes tightly coupled.

---

With Kafka:

```text
Document Service
       |
       | Publish event
       ▼

   ┌───────────┐
   │ Kafka     │
   └─────┬─────┘
         │
 ┌───────┼─────────┬───────────┐
 ▼       ▼         ▼           ▼

Search  Version  Analytics Notifications
Svc     Svc       Svc         Svc
```

Event example:

```json
{
 "event":"document_updated",
 "docId":"123",
 "user":"A",
 "timestamp":"..."
}
```

Consumers process independently.

Benefits:

### Decoupling

Document service doesn't care who consumes.

### Reliability

If Search service is down:

```text
Kafka stores events
```

Search can process later.

### Scaling

```text
1000 events/sec
10000 events/sec
100000 events/sec
```

Add more consumers.

---

Google Docs examples:

Events:

```text
Document updated
Comment added
Document shared
Version created
```

---

# 3. Why Elasticsearch is needed

Suppose DB contains:

```text
Doc1: "Meeting notes for AI team"

Doc2: "Budget planning"

Doc3: "Weekly AI architecture discussion"
```

User searches:

```text
AI notes
```

Using SQL:

```sql
SELECT *
FROM documents
WHERE content LIKE '%AI%'
```

Problems:

* Slow for huge text
* Doesn't rank relevance
* Poor typo handling
* Weak full-text features

---

Elasticsearch creates an index:

```text
AI → Doc1, Doc3
notes → Doc1
planning → Doc2
```

(search engine inverted index)

Query:

```text
AI notes
```

returns:

```text
1. Doc1 (best match)
2. Doc3
```

Can support:

* Fuzzy search

```text
Aritecture
```

→ returns:

```text
Architecture
```

* Ranking
* Autocomplete
* Filters

---

# Complete visual flow

```text
                 User typing
                      |
                      ▼

             Collaboration Service
                      |
                      ▼

                 Redis Cache
                      |
                      ▼

              Document Service
                      |
         Publish "document updated"
                      |
                      ▼

                   Kafka
          ┌────────┼───────────┐
          ▼        ▼           ▼

   Search Svc  Version Svc Analytics

          ▼
  Elasticsearch

```

---

Think of them like this:

```text
Redis
= working desk
(keep things immediately needed)

Kafka
= conveyor belt
(move work between teams)

Elasticsearch
= library index
(find documents quickly)
```

Database alone can store everything, but at Google Docs scale it becomes difficult to achieve **low latency + high throughput + powerful search** using only one system.


These are the kinds of follow-up questions interviewers commonly ask after a Google Docs HLD discussion. I've grouped them by topic because interviews often go progressively deeper.


# Estimation questions (very common)

1. Assume **100M daily users**, estimate:

   * QPS
   * WebSocket connections
   * Storage needed
   * Network bandwidth

2. Average document size = 500 KB:

   * How much storage for 1B documents?

3. Users type:

   * 5 characters/sec
   * 1M active users

   Estimate:

```text
operations/sec
network traffic
server count
```

---

# Very common "deep dive" question

Interviewers often end with:

> "Design Google Docs for 100 million concurrent users."

That usually opens discussion on:

```text
Document sharding
Hot document handling
Geo-replication
Distributed OT/CRDT
Connection management
Caching strategy
Fault tolerance
```

For Google Docs-style collaboration, **optimistic locking and pessimistic locking alone are usually not enough**. They work well for database rows, but real-time text editing has different behavior.

## Why pessimistic lock is a poor fit

Pessimistic locking:

```text
User A opens document
      ↓
Acquire lock
      ↓
User B tries editing
      ↓
Blocked
```

Result:

```text
User A: typing...
User B: "Document locked"
```

This becomes frustrating for collaborative editing.

Imagine 50 people editing:

```text
A edits paragraph 1
B edits paragraph 2
C edits title
```

With document-level lock:

```text
Only one person edits at a time
```

You lose the whole point of Google Docs.

You could try **fine-grained locking**:

```text
Paragraph1 → Lock
Paragraph2 → Lock
Line3 → Lock
```

But then new problems appear:

* lock management complexity
* deadlocks
* users waiting for locks
* constant lock acquire/release

---

## Why optimistic locking alone is insufficient

Optimistic locking works like:

```text
Document version = 10

User A reads version 10
User B reads version 10
```

Both edit.

A saves:

```text
Version 10 → 11
```

B saves:

```text
Version mismatch detected
```

B gets:

```text
Conflict detected
Please refresh
```

This is fine for forms:

```text
Edit profile
Edit employee record
```

But terrible for live typing.

Imagine typing:

```text
H
e
l
l
o
```

and every few keystrokes:

```text
Conflict detected
```

---

## Why OT/CRDT works better

Instead of:

```text
Reject conflicting edits
```

they do:

```text
Transform/Merge conflicting edits
```

Example:

```text
Initial: Hello

A: Insert X at position 2
B: Insert Y at position 2
```

OT:

```text
Transform B → position 3
```

Result:

```text
HeXYllo
```

No user interruption.

---

## What does GitHub use?

[GitHub](https://github.com?utm_source=chatgpt.com) primarily uses **Git's version-control model**, which is different from Google Docs.

Git typically follows something closer to:

```text
Read
Edit locally
Commit
Merge later
```

Example:

```text
main branch

User A → change line 10
User B → change line 10
```

Later:

```text
git merge
```

Possible outcomes:

### No conflict

```text
A changed line 10
B changed line 20

Auto merge
```

### Conflict

```text
<<<<<<< HEAD
Hello World
=======
Hello AI
>>>>>>> branch
```

Human resolves it.

---

Git's model behaves somewhat like **optimistic concurrency**:

```text
Assume conflicts are rare
Resolve later
```

because code development usually tolerates:

* delayed synchronization
* explicit merges
* manual conflict resolution

---

Google Docs and GitHub solve different problems:

| Feature                    |              Google Docs |      Git/GitHub |
| -------------------------- | -----------------------: | --------------: |
| Sync                       |                Real time |    Commit based |
| Conflict handling          |                OT / CRDT | Merge algorithm |
| User waits?                |                       No |       Sometimes |
| Manual conflict resolution |                     Rare |          Common |
| Offline support            |                      Yes |             Yes |
| Main goal                  | Continuous collaboration |  Source control |

So if Google Docs used Git-style locking/merging:

```text
User A types "Hello"
User B types "World"

Conflict:
Please resolve manually
```

That would feel very awkward for normal document editing.

