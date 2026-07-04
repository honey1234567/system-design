This article is one of the best beginner-friendly chat system design guides because it focuses on **why each component exists**, not just drawing boxes. The core philosophy is:

https://systemdesignschool.io/problems/chatapp/solution?utm_source=neetcode

> **A chat system is a durable log of messages with a real-time delivery layer on top.** ([System Design School][1])

Let's go through every important concept in simple language.

---

# 1. First Understand the Problem

When someone says **"Design WhatsApp/Discord/Messenger"**, don't think:

```
User A
   │
Server
   │
User B
```

That only works if both users are online.

Real chat applications must support:

* User online
* User offline
* Group chat
* Message history
* Read receipts
* Ordering
* Billions of users
* No message loss

These requirements completely change the architecture. ([System Design School][1])

---

# 2. Requirements

## Functional Requirements

The article considers these features:

### Send message

```
A
 │
 ▼
Server
 │
 ▼
B
```

---

### Group Chat

```
A

↓

Group

↓

B
C
D
E
```

---

### Offline Messages

Suppose

```
A sends

B offline
```

Message should still arrive after B reconnects.

---

### Message History

If you reinstall WhatsApp,

you still see old chats.

So messages must be stored permanently.

---

### Read Receipts

Need to know

```
✓ Sent

✓✓ Delivered

Blue Tick
Read
```

---

### Presence

Need to know

```
Online

Last seen

Typing...
```

Notice

Presence is NOT message data.

It is temporary information.

---

# 3. Non Functional Requirements

These are extremely important in interviews.

---

## Durability

Once sender gets

```
Message Sent
```

it should never disappear.

This is the highest priority.

---

## Low Latency

Users expect

```
Hello

↓

Few milliseconds

↓

Friend receives
```

Not

```
30 seconds later
```

---

## Ordering

Messages should appear

```
Hi

How are you?

Good Morning
```

Never

```
Good Morning

Hi

How are you?
```

Ordering is required.

---

## Availability

If network disconnects,

app should reconnect automatically.

---

## Massive Scale

Need to support

Millions of users

Millions of messages

Hundreds of millions of connections

---

# 4. Biggest Idea of Entire Article

The article repeatedly emphasizes

> **Persist first. Deliver later.**

Meaning

Wrong

```
Receive message

↓

Deliver

↓

Store
```

Suppose server crashes.

Message delivered?

Maybe.

Stored?

No.

Lost forever.

---

Correct

```
Receive

↓

Store

↓

Acknowledge sender

↓

Deliver
```

Now

Even if receiver is offline,

message is already safe.

This is the golden rule.

([System Design School][1])

---

# 5. Persistent Connections

Normal HTTP

```
Open connection

↓

Request

↓

Close
```

Terrible for chat.

---

Instead

Use

```
WebSocket
```

Connection remains open.

```
Phone
      │
──────┼────────
Open Connection
──────┼────────
```

Server can instantly push messages.

No polling.

---

# 6. Gateway Servers

Imagine

100 Million users.

One server cannot maintain

100 Million TCP connections.

Instead

```
Gateway 1

Gateway 2

Gateway 3

Gateway 4
```

Each gateway maintains around

```
100,000 connections
```

Total

```
1000 Gateways

↓

100 Million users
```

Gateways only maintain connections.

They don't own messages.

---

# 7. Connection Registry

Problem

Suppose

```
Alice

↓

Gateway 1

Bob

↓

Gateway 729
```

How does Gateway 1 know where Bob is?

Need

```
Connection Registry

User

↓

Gateway Mapping
```

Example

```
Alice → Gateway 1

Bob → Gateway 729

John → Gateway 11
```

Usually stored in Redis or another in-memory system because it's temporary and rebuilt on reconnect. ([System Design School][1])

---

# 8. Message Service

Gateway shouldn't talk directly to every other gateway.

Imagine

```
1000 gateways
```

Every gateway connecting to every other gateway becomes unmanageable.

Instead

```
Gateway

↓

Message Service

↓

Gateway
```

Message Service becomes the central owner of message processing.

---

# 9. Internal Message Bus

The message service publishes work to an internal broker.

```
Gateway

↓

Message Service

↓

Kafka / PubSub

↓

Recipient Gateway
```

Each gateway listens to its own topic.

Example

```
gateway-25

↓

Topic gateway.25
```

If Bob is connected there,

only Gateway 25 receives Bob's messages.

This avoids every gateway seeing every message. ([System Design School][1])

---

# 10. Durable Message Storage

Every message is stored.

```
Conversation

↓

Messages

↓

Permanent Database
```

Database could be

* Cassandra
* ScyllaDB
* DynamoDB
* RocksDB-backed systems

The key point is durability.

---

# 11. Conversation Log

The article treats a chat as

```
Conversation

↓

Ordered Log
```

Example

```
Conversation

1 Hello

2 Hi

3 Bye

4 Thanks
```

A conversation is just an append-only ordered log.

Very important interview concept.

---

# 12. Per-Conversation Ordering

Many beginners ask

Why not global ordering?

Imagine

```
Alice chatting with Bob

John chatting with Mike
```

No one cares which conversation happened first.

Need ordering only inside

```
Conversation A
```

or

```
Conversation B
```

Therefore every conversation has

```
server_seq

1

2

3

4
```

This avoids expensive global ordering. ([System Design School][1])

---

# 13. Client Message ID

Suppose

```
Send

↓

ACK lost

↓

Client retries
```

Server receives

same message twice.

Solution

```
client_msg_id
```

Example

```
UUID

123456
```

If duplicate arrives

Ignore it.

---

This provides

```
At Least Once

+

Deduplication

=

Exactly Once Experience
```

The network may deliver twice, but users only see one copy. ([System Design School][1])

---

# 14. Offline Delivery

Suppose

```
Bob offline
```

Message already stored.

When Bob reconnects

```
Sync

↓

Give messages

After sequence 55
```

Bob receives everything missed.

No data loss.

---

# 15. Sync API

Instead of downloading the whole chat,

client asks

```
Give me messages

after sequence

102
```

Server returns

```
103

104

105
```

Fast and bandwidth-efficient.

---

# 16. Read Receipts

Three states

```
Sent

↓

Delivered

↓

Read
```

Each updates metadata for the user's cursor.

---

# 17. Presence

Presence means

```
Online

Offline

Last Seen

Typing
```

Very important point

Presence is

NOT durable.

If Redis crashes

Users reconnect

Presence rebuilds automatically.

Unlike messages,

presence isn't permanently stored. ([System Design School][1])

---

# 18. Heartbeats

How does server know user is still online?

Every few seconds

```
Phone

↓

Heartbeat

↓

Server
```

If heartbeats stop

```
TTL expires

↓

Offline
```

This is lightweight and self-healing.

---

# 19. Fan-Out on Write

Small groups

```
A

↓

Server

↓

B

↓

C

↓

D
```

One message becomes

multiple deliveries.

Cheap for small groups.

---

# 20. Fan-Out on Read

Large groups

Imagine

```
50,000 members
```

Writing

50,000 copies

for every message

is expensive.

Instead

Store once.

```
Shared Log

↓

Member 1 reads

Member 2 reads

Member 3 reads
```

This is

Fan-Out on Read.

Very common interview question. ([System Design School][1])

---

# 21. High-Level Architecture

```
Client
   │
WebSocket
   │
Gateway Server
   │
Message Service
   │
───────────────
│             │
Connection     Message Store
Registry
│
Message Bus
│
Recipient Gateway
│
Recipient Client
```

Each component has a single responsibility:

* **Gateway** → Maintains WebSocket connections.
* **Connection Registry** → Maps users to gateway servers.
* **Message Service** → Validates, persists, sequences, and routes messages.
* **Durable Store** → Permanent conversation history.
* **Message Bus** → Efficient communication between services and gateways.

---

# 22. Design Principles to Remember

If asked to design a chat application, these are the key ideas interviewers expect:

1. **Persist before acknowledging or delivering** to guarantee durability.
2. Use **WebSockets** for low-latency, bidirectional communication.
3. Scale connections with a **gateway fleet**.
4. Maintain a **connection registry** (`user → gateway`) in memory.
5. Use a **message service** as the central coordinator instead of gateway-to-gateway communication.
6. Keep messages in a **durable per-conversation log**.
7. Order messages with a **per-conversation sequence number**, not a global sequence.
8. Support **at-least-once delivery with deduplication** using `client_msg_id` for an exactly-once user experience.
9. Deliver live to online users and **sync missed messages** after reconnect.
10. Use **fan-out on write** for small groups and **fan-out on read** (shared log) for very large groups.
11. Keep **presence and routing** in ephemeral in-memory storage, separate from the durable message path. ([System Design School][1])

[1]: https://systemdesignschool.io/problems/chatapp/solution?utm_source=chatgpt.com "Chat / Messenger System Design | System Design Interview"

This is one of the **most commonly asked interview questions**.

The important thing to understand is:

> **WebSocket is not the only possible choice.** It's simply the **best fit** for most real-time chat applications because it provides a persistent, bidirectional, low-latency connection.

Let's compare all the options.

---

# What does a chat application need?

Suppose Alice sends:

```
Hi Bob
```

Bob should receive it immediately.

This means the server must be able to **push data to the client at any time**.

This requirement eliminates many traditional approaches.

---

# 1. HTTP Request-Response

Normal websites work like this:

```
Client
   │ Request
   ▼
Server
   │ Response
   ▼
Client
```

Example:

```
Browser → GET /profile
Server → Profile JSON
```

Then the connection closes.

### Problem

Suppose Bob is waiting for a message.

```
Bob opens chat

↓

Waiting...
```

Server cannot suddenly send:

```
Alice: Hello
```

because there is **no active connection**.

Bob would never receive the message unless he makes another request.

---

# 2. Polling

Solution:

Ask repeatedly.

```
Every 5 seconds

Client
↓

Any new messages?

↓

Server

↓

No

↓

5 sec later

↓

Any new messages?
```

### Example

```
12:00:00
No message

12:00:05
No message

12:00:10
No message

12:00:15
Hello!
```

### Problems

If Alice sends at:

```
12:00:11
```

Bob only gets it at:

```
12:00:15
```

Latency:

```
4 seconds
```

Also imagine:

```
100 million users

↓

Every 5 seconds

↓

HTTP request
```

Even if nobody is chatting, your servers process **millions of unnecessary requests**.

---

# 3. Long Polling

A better approach.

Instead of replying immediately:

```
Client

↓

Any message?

↓

Server waits...
```

Server keeps the request open.

```
Client

↓

Waiting...

↓

Alice sends message

↓

Server responds immediately
```

Then client opens another request.

### Better than polling

Much lower latency.

### Problems

Still repeatedly creating HTTP requests.

```
Open request

↓

Receive message

↓

Close

↓

Open another request

↓

Close
```

This creates extra CPU and network overhead.

---

# 4. Server-Sent Events (SSE)

SSE keeps one HTTP connection open.

```
Server

↓

Push message

↓

Client
```

Good.

### Problem

Communication is only one-way.

```
Server
↓

Client
```

If client wants to send a message:

```
Need another HTTP POST
```

So chat becomes:

```
POST /send

+

SSE receive
```

Two different communication channels.

---

# 5. WebSocket

Now look at WebSocket.

```
Client
⇅
Server
```

One connection.

Both sides can send anytime.

```
Alice types

↓

Server immediately receives

↓

Server immediately sends to Bob

↓

Bob receives
```

No reopening.

No polling.

No extra requests.

---

# Why is WebSocket perfect for chat?

Because chat needs **continuous two-way communication**.

```
Client
⇅
Server
```

Examples:

* Send message
* Receive message
* Typing indicator
* Read receipt
* Online status
* Delivery acknowledgment
* Voice call signaling
* Video call signaling

Everything happens on the same connection.

---

# Connection Comparison

### HTTP

```
Open

↓

Request

↓

Response

↓

Close
```

For 100 messages:

```
100 TCP connections
```

---

### Long Polling

```
Open

↓

Wait

↓

Message

↓

Close

↓

Open again
```

Still many connections over time.

---

### WebSocket

```
Open once

↓

Message

↓

Message

↓

Message

↓

Message

↓

Close after hours
```

One connection may stay open for hours.

---

# Why is WebSocket faster?

Imagine Bob receives:

```
100 messages
```

### HTTP

```
Open TCP

↓

TLS handshake

↓

Headers

↓

Response

↓

Close
```

Repeated many times.

---

### WebSocket

```
Connection already exists

↓

Tiny frame

↓

Delivered
```

No repeated handshakes.

Lower latency.

Less bandwidth.

---

# What happens inside WhatsApp?

```
Phone

⇅ WebSocket

Gateway Server

↓

Message Service

↓

Database
```

When Alice sends:

```
Hello
```

Flow:

```
Alice

↓

WebSocket

↓

Gateway

↓

Store message

↓

Find Bob's gateway

↓

Push immediately

↓

Bob
```

No polling involved.

---

# But doesn't keeping millions of WebSockets open consume memory?

Yes.

Suppose:

```
50 million users online
```

One server cannot maintain:

```
50 million sockets
```

Instead:

```
Gateway 1
100K users

Gateway 2
100K users

Gateway 3
100K users

...

Gateway 500
100K users
```

This is why chat systems use **many gateway servers**.

Each gateway only manages a fraction of the connections.

---

# Why not use gRPC?

gRPC also supports **bidirectional streaming**, making it technically capable of powering chat.

However:

* Browsers have limited native support for gRPC streaming.
* WebSockets are universally supported across browsers and mobile platforms.
* The ecosystem for browser-based real-time messaging is more mature with WebSockets.

Many internal microservices use **gRPC**, while the client (browser/mobile app) communicates with the gateway using **WebSocket**.

---

# Can HTTP/2 or HTTP/3 replace WebSocket?

HTTP/2 and HTTP/3 improve performance by multiplexing requests over fewer connections, but they still use the **request-response** model by default.

They don't automatically provide a persistent, bidirectional messaging channel like WebSocket.

---

# Interview Answer (30 seconds)

> We use **WebSocket** because chat applications require **low-latency, bidirectional, persistent communication**. Unlike HTTP polling or long polling, WebSocket establishes a single long-lived connection that allows both the client and server to send messages at any time. This reduces connection overhead, minimizes latency, saves bandwidth, and efficiently supports features like instant messaging, typing indicators, presence updates, and read receipts. It's not the only possible solution—SSE or gRPC streaming can work in specific scenarios—but WebSocket is the most widely adopted protocol for internet-facing real-time chat applications due to its broad client support and efficient communication model.

