Ticketmaster is one of the most frequently asked **Senior SDE / System Design** interview questions because it combines many difficult concepts:

* High traffic
* Read-heavy + write-heavy workload
* Inventory management
* Concurrency
* Distributed locking
* Queue systems
* Payment
* Notifications
* Search
* Scalability

Unlike Netflix or YouTube, **Ticketmaster's biggest challenge is preventing two people from buying the same seat.**

---

# 1. First understand the business

Imagine you are booking movie tickets.

```
Movie: Avengers

Hall

A1 A2 A3 A4 A5
B1 B2 B3 B4 B5
C1 C2 C3 C4 C5
```

Suppose

* Seat A1 is available.

At exactly **10:00:00**

```
User A clicks A1

AND

User B clicks A1
```

Only ONE should get it.

This is the hardest problem.

Everything else is comparatively easier.

---

# 2. Functional Requirements

Interviewer first expects this.

### User can

* Search events
* Search concerts
* View seat map
* Select seats
* Reserve seats temporarily
* Pay
* Receive ticket

Admin

* Create event
* Add venue
* Set pricing
* Cancel event

---

# 3. Non Functional Requirements

Interviewers love these.

### High Availability

Ticketmaster cannot go down during Taylor Swift ticket sales.

---

### Consistency

Seat should never be sold twice.

Very important.

---

### Scalability

Millions of users.

---

### Low latency

Seat map should load quickly.

---

# 4. High Level Architecture

```
                Users
                   |
             Load Balancer
                   |
             API Gateway
                   |
      ----------------------------
      |     |      |      |
 Search Seat Payment Notification
      |      |        |
      ------------------
             |
      Booking Service
             |
      Seat Reservation
             |
          Database
```

Each service is independent.

---

# 5. Services

Let's understand one by one.

---

## Search Service

Responsible for

```
Search concert

Search by city

Search by artist

Search by date
```

Search is mostly read-heavy.

Uses

```
Elasticsearch/OpenSearch
```

instead of SQL.

Example

```
Search

Taylor Swift

↓

Instant results
```

---

## Event Service

Stores

```
Event Name

Venue

Date

Time

Pricing

Seat Layout
```

Database

```
PostgreSQL
```

---

## Seat Service

Stores

```
Seat A1

Seat A2

Seat B1
```

Status

```
Available

Reserved

Booked
```

Example

| Seat | Status    |
| ---- | --------- |
| A1   | Available |
| A2   | Booked    |
| A3   | Reserved  |

---

## Booking Service

Most important service.

Responsible for

```
Reserve seat

Confirm booking

Release seat

Generate ticket
```

---

## Payment Service

Talks to

```
Stripe

Razorpay

PayPal
```

Only after successful payment

Booking becomes permanent.

---

## Notification Service

Sends

```
Email

SMS

Push Notification
```

---

# 6. Database Design

Simple version

### Event

```
EventID

Name

VenueID

Time

Price
```

---

Venue

```
VenueID

Name

Location
```

---

Seat

```
SeatID

VenueID

Row

Column
```

---

Booking

```
BookingID

UserID

SeatID

Status

PaymentID
```

---

# 7. User Flow

Suppose booking starts.

```
User opens app

↓

Search Concert

↓

Select Event

↓

View Seats

↓

Choose A1

↓

Reserve Seat

↓

Payment

↓

Booking Confirmed

↓

Email Ticket
```

---

# 8. Biggest Interview Question

## How do we prevent double booking?

Example

```
Seat A1

Available
```

Two requests arrive.

```
User A

User B
```

Both reach server.

Without protection

```
Both succeed.

BAD
```

Need concurrency control.

---

# Method 1

## Database Lock

```
BEGIN

SELECT A1

LOCK ROW

UPDATE status

COMMIT
```

Only one transaction proceeds.

Second waits.

Very safe.

But slower.

---

# Method 2

## Optimistic Locking

Seat table

```
SeatID

Status

Version
```

Initially

```
Version=5
```

User A

```
Reads version=5
```

User B

```
Reads version=5
```

A updates

```
Version=6
```

Now B tries

```
Expected Version=5

Current Version=6

FAIL
```

B refreshes.

Very scalable.

---

# Method 3

## Distributed Lock (Redis)

```
SETNX SeatA1
```

Only one request acquires lock.

Others fail immediately.

Very common.

---

# Method 4 (Best)

Seat Reservation

Instead of directly booking

Reserve first.

```
Seat A1

↓

Reserved

↓

Timer starts

↓

5 minutes
```

If payment succeeds

```
Booked
```

Else

```
Available again
```

This is exactly what users experience on many ticketing sites.

---

# 9. Reservation Workflow

```
User

↓

Reserve A1

↓

Booking Service

↓

Seat status

Reserved

↓

Redis TTL = 5 min

↓

Payment

↓

Booked
```

If timer expires

```
Reserved

↓

Expired

↓

Available
```

---

# 10. Why Redis?

SQL is slow for temporary reservations.

Redis provides

```
Fast

TTL support

Automatic expiration
```

Example

```
SET SeatA1 Reserved EX 300
```

Automatically expires in 300 seconds.

---

# 11. Booking States

```
Available

↓

Reserved

↓

Booked
```

or

```
Available

↓

Reserved

↓

Expired

↓

Available
```

---

# 12. Why Queue?

During famous concerts

```
10 million users

10 seconds
```

Server crashes.

Instead

```
User

↓

Virtual Queue

↓

Booking
```

Example

```
Position

14562

Estimated wait

18 minutes
```

This protects backend services from sudden traffic spikes.

---

# 13. Virtual Waiting Room

```
Users

      |
      v
+----------------+
| Waiting Queue  |
+----------------+
      |
      v
+----------------+
| Booking APIs   |
+----------------+
      |
      v
+----------------+
| Booking DB     |
+----------------+
```

Only a limited number of users (for example, 5,000 at a time) are allowed to proceed to the booking system. Others wait in the queue.

---

# 14. Caching

Popular data

```
Events

Seat Layout

Venue

Pricing
```

Cache

```
Redis
```

instead of database.

---

# 15. CDN

Images

```
Artist photos

Venue maps

Event banners
```

Served from CDN.

Not from application servers.

---

# 16. Payment Failure

Suppose

```
Seat Reserved

↓

Payment Failed
```

System should

```
Release Seat
```

Otherwise seat remains blocked forever.

---

# 17. Microservices

```
                API Gateway
                     |
   ----------------------------------------
   |          |         |        |         |
Search     Event     Booking   Payment  Notification
   |          |         |        |         |
 Elasticsearch PostgreSQL Booking DB  Payment Gateway
                            |
                          Redis
```

Each service can scale independently based on demand.

---

# 18. How does Ticketmaster differ from Netflix or YouTube?

| Feature            | Ticketmaster                          | Netflix/YouTube                                       |
| ------------------ | ------------------------------------- | ----------------------------------------------------- |
| Primary workload   | Booking transactions                  | Video streaming                                       |
| Main challenge     | Prevent double booking                | Deliver video with low buffering                      |
| Consistency        | Strong consistency for seat inventory | Eventual consistency is often acceptable for metadata |
| Critical data      | Seat inventory                        | Video files and metadata                              |
| Peak traffic       | Flash sales (concerts, sports)        | Continuous streaming                                  |
| Key infrastructure | Redis locks, reservations, queues     | CDN, adaptive bitrate streaming, edge caching         |

---

# 19. Typical Interview Flow (45–60 minutes)

A strong interview answer usually progresses in this order:

1. Clarify requirements (events, seat selection, payment).
2. Estimate scale (users, events, requests per second).
3. Draw the high-level architecture.
4. Explain the booking flow.
5. Deep dive into concurrency (locking, optimistic locking, reservations).
6. Discuss scaling (caching, queues, databases, sharding).
7. Cover failures (payment failure, reservation expiry, service crashes).
8. Mention trade-offs (consistency vs availability, lock contention vs throughput).

---

# Key Interview Takeaways

If you remember just these concepts, you'll be able to explain Ticketmaster confidently:

* **Search** uses a search engine (like Elasticsearch/OpenSearch) because users search by artist, city, and date.
* **Booking Service** is the core component that manages reservations and confirmations.
* **Seat inventory** is the source of truth and requires **strong consistency**.
* **Temporary reservations with expiration (TTL)** prevent seats from being held indefinitely.
* **Concurrency control** (row locks, optimistic locking, or distributed locks) prevents double booking.
* **Virtual waiting rooms and queues** protect the system during flash sales.
* **Redis** is commonly used for fast temporary reservations, locks, caching, and TTL.
* **Asynchronous messaging** can notify other services (ticket generation, email, analytics) after a booking is confirmed.
* **CDNs and caches** reduce load by serving static assets and frequently accessed event information quickly.

For interviews, the most important deep-dive topic is **"How do you guarantee that a seat is never sold to two users at the same time?"** If you can explain reservation states, locking strategies, and expiration handling clearly, you've covered the core challenge that interviewers are usually evaluating.

This is the **most important part** of the Ticketmaster interview. Many candidates can draw the architecture, but interviewers spend **70–80% of the interview** discussing **concurrency** and **scaling**.

Let's dive into it step by step as if you're in an interview.

---

# Part 1: Understanding the Problem

Imagine Ticketmaster opens booking for a Taylor Swift concert.

```
10,000 seats

10 million users

Booking opens at 10:00 AM
```

Exactly at 10:00

```
          Seat A1

             |
   ---------------------
   |        |         |
 User1   User2    User3
```

All three click **Book A1** at the same millisecond.

Only ONE should succeed.

The other two should receive

```
Seat unavailable.
```

Question:

**How do we make sure this happens?**

---

# Why is this difficult?

Suppose our code is

```java
Seat seat = seatRepository.findById("A1");

if(seat.isAvailable()){

    seat.setStatus(BOOKED);

    seatRepository.save(seat);
}
```

Looks correct.

But internally this happens.

Time

```
10:00:00.001

User A

Read seat

Available
```

Before A updates...

```
10:00:00.002

User B

Read seat

Available
```

Before B updates...

```
10:00:00.003

User C

Read seat

Available
```

Then

```
User A updates

Booked
```

Then

```
User B updates

Booked
```

Then

```
User C updates

Booked
```

Three successful bookings.

Impossible in real life.

This is called a **Race Condition**.

---

# What is Race Condition?

A race condition happens when multiple requests try to modify the same resource at the same time, and the final result depends on the order in which they execute.

Example

```
Bank balance = ₹1000

User A withdraws ₹1000

User B withdraws ₹1000
```

Both read

```
Balance = ₹1000
```

Both succeed.

Final balance becomes

```
-₹1000
```

Same problem.

Ticketmaster has identical issue.

---

# Solution 1 — Pessimistic Locking

Think of a hotel room.

The receptionist locks the room record while checking someone in.

Nobody else can modify it.

Database does the same.

SQL

```sql
BEGIN;

SELECT *
FROM Seat
WHERE seat_id='A1'
FOR UPDATE;

UPDATE Seat
SET status='BOOKED'
WHERE seat_id='A1';

COMMIT;
```

---

### Timeline

```
User A

Lock Seat A1

--------------------

User B

Wait...

--------------------

User C

Wait...
```

After A commits

```
User B wakes up

Reads

Already booked

Fail
```

---

Advantages

✔ Very safe

✔ Easy

✔ Database guarantees correctness

---

Disadvantages

```
Thousands of users

↓

Thousands of locks

↓

Database slows down
```

High contention.

---

# Solution 2 — Optimistic Locking

Instead of locking,

we detect conflicts.

Seat table

```
Seat

SeatID

Status

Version
```

Example

```
Seat A1

Available

Version = 7
```

---

Both users read

```
Version = 7
```

User A updates

```
UPDATE Seat

SET status='BOOKED',

version=8

WHERE

seatId='A1'

AND version=7
```

Database

```
Rows updated = 1
```

Success.

---

User B

Runs

```
WHERE version=7
```

Database

Current Version

```
8
```

Rows updated

```
0
```

Booking fails.

---

Interviewer likes hearing

> We don't lock rows. We simply retry when version mismatches.

---

Advantages

Very scalable.

No waiting.

---

Disadvantages

During flash sales

```
10000 users

↓

9999 retries
```

Many failures.

---

# Solution 3 — Redis Distributed Lock

Suppose Booking Service has many instances.

```
           Load Balancer

          /     |      \

 Booking1 Booking2 Booking3
```

All three can process the same seat.

We need a lock shared across all servers.

Redis provides this.

```
SET seat:A1 user123 NX EX 300
```

Meaning

```
Only create key

if not exists

Expire after 300 sec
```

---

Timeline

```
Booking1

SETNX

Success

Lock acquired
```

Booking2

```
SETNX

Failed
```

Booking3

```
Failed
```

Only one continues.

---

After booking

```
DEL seat:A1
```

Lock released.

---

Advantages

Very fast.

Works across multiple servers.

---

Problems

Suppose

```
Booking Server crashes.
```

Lock never deleted.

Seat remains locked forever.

Solution

TTL

```
EX 300
```

Redis removes it automatically after 5 minutes.

---

# Solution 4 — Reservation (Used by Ticketmaster)

This is the industry standard.

Instead of directly booking

```
Available

↓

Booked
```

We insert a middle state.

```
Available

↓

Reserved

↓

Booked
```

---

Flow

User clicks

```
Book
```

Immediately

```
Seat

Reserved
```

Timer starts

```
5 minutes
```

User pays.

If payment succeeds

```
Booked
```

Else

```
Expired

↓

Available
```

---

Why?

Imagine payment gateway takes

```
40 seconds.
```

Without reservation

Someone else may buy the same seat.

Reservation guarantees

```
Seat belongs to you

for 5 minutes.
```

---

# Complete Reservation Timeline

```
User

↓

Reserve Seat

↓

Redis Lock

↓

Seat Status = RESERVED

↓

TTL = 5 min

↓

Payment
```

Payment success

```
Seat

BOOKED
```

Payment failed

```
Seat

AVAILABLE
```

---

# What if Payment Success but Server Crashes?

Classic interview question.

```
Payment completed

↓

Booking server crashed

↓

Seat still RESERVED
```

Bad.

Solution

Use **idempotent operations** and **event-driven processing**.

```
Payment Service

↓

Publishes

Payment Success Event

↓

Booking Service

↓

Marks BOOKED
```

If Booking Service crashes, the event stays in the message queue until it is processed.

---

# Scaling Deep Dive

Concurrency ensures correctness.

Scaling ensures millions of users can use the system simultaneously.

---

# Traffic Estimation

Suppose

```
10 million users
```

Open app simultaneously.

Even if only

```
20%

book tickets
```

That's

```
2 million active users.
```

Suppose each sends

```
5 requests
```

```
Search

Seat Map

Availability

Reserve

Payment
```

Total

```
10 million requests
```

in a few minutes.

One server cannot handle this.

---

# Horizontal Scaling

Instead of

```
1 Server
```

We use

```
Load Balancer

↓

--------------------

Server1

Server2

Server3

Server4

Server5
```

Load Balancer distributes requests.

---

# Stateless Services

Booking servers should not keep user session in memory.

Bad

```
Server1

stores

Reservation
```

If Server1 crashes

Reservation disappears.

Good

```
Reservation

Redis

Database
```

Any server can continue processing.

---

# Database Bottleneck

Millions of reads

```
SELECT Event
```

hit the database.

Database becomes slow.

Solution

Cache.

```
Users

↓

Redis

↓

Database
```

Frequently accessed data such as event details, venue information, and seat layouts are served from cache.

---

# But Should Seat Availability Be Cached?

Interviewers often ask this.

Suppose cache says

```
Seat A1

Available
```

Database

```
Booked
```

Cache is stale.

Wrong answer.

Therefore:

```
Event Details

Cache ✓

Artist Info

Cache ✓

Images

Cache ✓

Seat Availability

Use carefully
```

Usually, the booking operation always checks the database (or another strongly consistent inventory store) before confirming a reservation.

---

# Virtual Waiting Room

Biggest scaling trick.

Instead of allowing

```
10 million

↓

Booking API
```

we do

```
10 million

↓

Queue

↓

5000 users

↓

Booking
```

Only

```
5000
```

users are inside booking system.

Others wait.

Benefits

```
Stable servers

No overload

Fairness
```

---

# Message Queue

Some work doesn't need to happen immediately.

```
Booking Complete

↓

Email

↓

Analytics

↓

Invoice

↓

Recommendation
```

Don't block the user.

Instead

```
Booking Service

↓

Kafka/RabbitMQ

↓

Email Service

↓

Analytics

↓

Notification
```

User gets response immediately while background services process events.

---

# Database Scaling

Eventually one database becomes a bottleneck.

### Read Replicas

```
          Primary

        /    |    \

Replica1 Replica2 Replica3
```

Writes go to the primary database.

Reads such as event information and booking history can go to replicas.

---

### Sharding

If one database cannot store all bookings:

```
Shard 1

Events A–F

Shard 2

Events G–M

Shard 3

Events N–Z
```

Or shard by

```
Venue

City

Region
```

This distributes load across multiple databases.

---

# Complete Scalable Architecture

```
                     Users
                        |
                 Global Load Balancer
                        |
                  API Gateway
                        |
        +-------------------------------+
        |       |         |             |
     Search   Booking   Payment   Notification
        |         |         |             |
 Elasticsearch   Redis     Payment API    Kafka
                  |
            Booking Service
                  |
             Primary Database
                  |
          Read Replicas / Shards
```

---

# Interview Summary

If an interviewer asks:

> "How would you design Ticketmaster to handle millions of users without double booking?"

A strong answer is:

1. Use a **virtual waiting room** to control the number of users entering the booking system.
2. Make booking services **stateless** and scale them horizontally behind a load balancer.
3. Use **temporary seat reservations** with a TTL (for example, 5 minutes).
4. Protect seat inventory using **optimistic locking**, **pessimistic locking**, or a **distributed Redis lock**, depending on the trade-offs.
5. Store booking state in a **strongly consistent database** rather than relying solely on cache.
6. Use **Redis** for temporary reservations, locks, and caching of non-critical data.
7. Process non-critical tasks asynchronously with a **message queue**.
8. Scale the database using **read replicas** for read-heavy traffic and **sharding** when a single database is no longer sufficient.

### One subtle interview insight

Many candidates think **Redis locks alone guarantee correctness**. They don't. Redis helps coordinate concurrent requests across application servers, but the **database must still enforce consistency** (through transactions, unique constraints, version checks, or row locking). The safest production systems combine these techniques rather than relying on a single mechanism.
This is the **most important part** of the Ticketmaster interview. Many candidates can draw the architecture, but interviewers spend **70–80% of the interview** discussing **concurrency** and **scaling**.

Let's dive into it step by step as if you're in an interview.

---

# Part 1: Understanding the Problem

Imagine Ticketmaster opens booking for a Taylor Swift concert.

```
10,000 seats

10 million users

Booking opens at 10:00 AM
```

Exactly at 10:00

```
          Seat A1

             |
   ---------------------
   |        |         |
 User1   User2    User3
```

All three click **Book A1** at the same millisecond.

Only ONE should succeed.

The other two should receive

```
Seat unavailable.
```

Question:

**How do we make sure this happens?**

---

# Why is this difficult?

Suppose our code is

```java
Seat seat = seatRepository.findById("A1");

if(seat.isAvailable()){

    seat.setStatus(BOOKED);

    seatRepository.save(seat);
}
```

Looks correct.

But internally this happens.

Time

```
10:00:00.001

User A

Read seat

Available
```

Before A updates...

```
10:00:00.002

User B

Read seat

Available
```

Before B updates...

```
10:00:00.003

User C

Read seat

Available
```

Then

```
User A updates

Booked
```

Then

```
User B updates

Booked
```

Then

```
User C updates

Booked
```

Three successful bookings.

Impossible in real life.

This is called a **Race Condition**.

---

# What is Race Condition?

A race condition happens when multiple requests try to modify the same resource at the same time, and the final result depends on the order in which they execute.

Example

```
Bank balance = ₹1000

User A withdraws ₹1000

User B withdraws ₹1000
```

Both read

```
Balance = ₹1000
```

Both succeed.

Final balance becomes

```
-₹1000
```

Same problem.

Ticketmaster has identical issue.

---

# Solution 1 — Pessimistic Locking

Think of a hotel room.

The receptionist locks the room record while checking someone in.

Nobody else can modify it.

Database does the same.

SQL

```sql
BEGIN;

SELECT *
FROM Seat
WHERE seat_id='A1'
FOR UPDATE;

UPDATE Seat
SET status='BOOKED'
WHERE seat_id='A1';

COMMIT;
```

---

### Timeline

```
User A

Lock Seat A1

--------------------

User B

Wait...

--------------------

User C

Wait...
```

After A commits

```
User B wakes up

Reads

Already booked

Fail
```

---

Advantages

✔ Very safe

✔ Easy

✔ Database guarantees correctness

---

Disadvantages

```
Thousands of users

↓

Thousands of locks

↓

Database slows down
```

High contention.

---

# Solution 2 — Optimistic Locking

Instead of locking,

we detect conflicts.

Seat table

```
Seat

SeatID

Status

Version
```

Example

```
Seat A1

Available

Version = 7
```

---

Both users read

```
Version = 7
```

User A updates

```
UPDATE Seat

SET status='BOOKED',

version=8

WHERE

seatId='A1'

AND version=7
```

Database

```
Rows updated = 1
```

Success.

---

User B

Runs

```
WHERE version=7
```

Database

Current Version

```
8
```

Rows updated

```
0
```

Booking fails.

---

Interviewer likes hearing

> We don't lock rows. We simply retry when version mismatches.

---

Advantages

Very scalable.

No waiting.

---

Disadvantages

During flash sales

```
10000 users

↓

9999 retries
```

Many failures.

---

# Solution 3 — Redis Distributed Lock

Suppose Booking Service has many instances.

```
           Load Balancer

          /     |      \

 Booking1 Booking2 Booking3
```

All three can process the same seat.

We need a lock shared across all servers.

Redis provides this.

```
SET seat:A1 user123 NX EX 300
```

Meaning

```
Only create key

if not exists

Expire after 300 sec
```

---

Timeline

```
Booking1

SETNX

Success

Lock acquired
```

Booking2

```
SETNX

Failed
```

Booking3

```
Failed
```

Only one continues.

---

After booking

```
DEL seat:A1
```

Lock released.

---

Advantages

Very fast.

Works across multiple servers.

---

Problems

Suppose

```
Booking Server crashes.
```

Lock never deleted.

Seat remains locked forever.

Solution

TTL

```
EX 300
```

Redis removes it automatically after 5 minutes.

---

# Solution 4 — Reservation (Used by Ticketmaster)

This is the industry standard.

Instead of directly booking

```
Available

↓

Booked
```

We insert a middle state.

```
Available

↓

Reserved

↓

Booked
```

---

Flow

User clicks

```
Book
```

Immediately

```
Seat

Reserved
```

Timer starts

```
5 minutes
```

User pays.

If payment succeeds

```
Booked
```

Else

```
Expired

↓

Available
```

---

Why?

Imagine payment gateway takes

```
40 seconds.
```

Without reservation

Someone else may buy the same seat.

Reservation guarantees

```
Seat belongs to you

for 5 minutes.
```

---

# Complete Reservation Timeline

```
User

↓

Reserve Seat

↓

Redis Lock

↓

Seat Status = RESERVED

↓

TTL = 5 min

↓

Payment
```

Payment success

```
Seat

BOOKED
```

Payment failed

```
Seat

AVAILABLE
```

---

# What if Payment Success but Server Crashes?

Classic interview question.

```
Payment completed

↓

Booking server crashed

↓

Seat still RESERVED
```

Bad.

Solution

Use **idempotent operations** and **event-driven processing**.

```
Payment Service

↓

Publishes

Payment Success Event

↓

Booking Service

↓

Marks BOOKED
```

If Booking Service crashes, the event stays in the message queue until it is processed.

---

# Scaling Deep Dive

Concurrency ensures correctness.

Scaling ensures millions of users can use the system simultaneously.

---

# Traffic Estimation

Suppose

```
10 million users
```

Open app simultaneously.

Even if only

```
20%

book tickets
```

That's

```
2 million active users.
```

Suppose each sends

```
5 requests
```

```
Search

Seat Map

Availability

Reserve

Payment
```

Total

```
10 million requests
```

in a few minutes.

One server cannot handle this.

---

# Horizontal Scaling

Instead of

```
1 Server
```

We use

```
Load Balancer

↓

--------------------

Server1

Server2

Server3

Server4

Server5
```

Load Balancer distributes requests.

---

# Stateless Services

Booking servers should not keep user session in memory.

Bad

```
Server1

stores

Reservation
```

If Server1 crashes

Reservation disappears.

Good

```
Reservation

Redis

Database
```

Any server can continue processing.

---

# Database Bottleneck

Millions of reads

```
SELECT Event
```

hit the database.

Database becomes slow.

Solution

Cache.

```
Users

↓

Redis

↓

Database
```

Frequently accessed data such as event details, venue information, and seat layouts are served from cache.

---

# But Should Seat Availability Be Cached?

Interviewers often ask this.

Suppose cache says

```
Seat A1

Available
```

Database

```
Booked
```

Cache is stale.

Wrong answer.

Therefore:

```
Event Details

Cache ✓

Artist Info

Cache ✓

Images

Cache ✓

Seat Availability

Use carefully
```

Usually, the booking operation always checks the database (or another strongly consistent inventory store) before confirming a reservation.

---

# Virtual Waiting Room

Biggest scaling trick.

Instead of allowing

```
10 million

↓

Booking API
```

we do

```
10 million

↓

Queue

↓

5000 users

↓

Booking
```

Only

```
5000
```

users are inside booking system.

Others wait.

Benefits

```
Stable servers

No overload

Fairness
```

---

# Message Queue

Some work doesn't need to happen immediately.

```
Booking Complete

↓

Email

↓

Analytics

↓

Invoice

↓

Recommendation
```

Don't block the user.

Instead

```
Booking Service

↓

Kafka/RabbitMQ

↓

Email Service

↓

Analytics

↓

Notification
```

User gets response immediately while background services process events.

---

# Database Scaling

Eventually one database becomes a bottleneck.

### Read Replicas

```
          Primary

        /    |    \

Replica1 Replica2 Replica3
```

Writes go to the primary database.

Reads such as event information and booking history can go to replicas.

---

### Sharding

If one database cannot store all bookings:

```
Shard 1

Events A–F

Shard 2

Events G–M

Shard 3

Events N–Z
```

Or shard by

```
Venue

City

Region
```

This distributes load across multiple databases.

---

# Complete Scalable Architecture

```
                     Users
                        |
                 Global Load Balancer
                        |
                  API Gateway
                        |
        +-------------------------------+
        |       |         |             |
     Search   Booking   Payment   Notification
        |         |         |             |
 Elasticsearch   Redis     Payment API    Kafka
                  |
            Booking Service
                  |
             Primary Database
                  |
          Read Replicas / Shards
```

---

# Interview Summary

If an interviewer asks:

> "How would you design Ticketmaster to handle millions of users without double booking?"

A strong answer is:

1. Use a **virtual waiting room** to control the number of users entering the booking system.
2. Make booking services **stateless** and scale them horizontally behind a load balancer.
3. Use **temporary seat reservations** with a TTL (for example, 5 minutes).
4. Protect seat inventory using **optimistic locking**, **pessimistic locking**, or a **distributed Redis lock**, depending on the trade-offs.
5. Store booking state in a **strongly consistent database** rather than relying solely on cache.
6. Use **Redis** for temporary reservations, locks, and caching of non-critical data.
7. Process non-critical tasks asynchronously with a **message queue**.
8. Scale the database using **read replicas** for read-heavy traffic and **sharding** when a single database is no longer sufficient.

### One subtle interview insight

Many candidates think **Redis locks alone guarantee correctness**. They don't. Redis helps coordinate concurrent requests across application servers, but the **database must still enforce consistency** (through transactions, unique constraints, version checks, or row locking). The safest production systems combine these techniques rather than relying on a single mechanism.

Below is the **beginner-friendly version** of the Ticketmaster system design from System Design School. I'll explain it as if you've never designed a large-scale system before. The core idea of the design is to **separate read operations (searching) from write operations (booking)** and make booking safe so that one seat is never sold twice. ([System Design School][1])

---

# Step 1: What is Ticketmaster?

Ticketmaster is simply an online ticket booking platform.

Users can:

* Search events
* View seats
* Reserve seats
* Pay
* Get tickets

The difficult part is **booking**, not searching.

---

# Step 2: What is the biggest problem?

Suppose there is only **one seat left**.

```text
Seat A1

Available
```

At exactly 10:00 AM

```text
User A clicks Book

        AND

User B clicks Book
```

Only one should get the seat.

This is the main system design challenge.

---

# Step 3: High-Level Architecture

```text
                Users
                   |
            Load Balancer
                   |
             API Gateway
                   |
      --------------------------
      |          |             |
 Search Service Booking Service Payment Service
      |          |             |
    Cache     Message Queue    Payment Gateway
      |          |
   Database   Seat Database
```

Each service has one job.

---

# Step 4: Search Flow (Read Path)

Searching is simple because users only **read** data.

```text
User

↓

Search "Coldplay"

↓

Search Service

↓

Cache

↓

Database

↓

Show Events
```

### Why cache?

Imagine 1 million people searching the same concert.

Without cache

```text
Everyone

↓

Database
```

Database becomes slow.

Instead

```text
Users

↓

Redis Cache

↓

Database (only if needed)
```

Cache is much faster.

---

# Step 5: Booking Flow (Write Path)

Booking changes data, so it is harder.

```text
User

↓

Click Book

↓

Booking Service

↓

Message Queue

↓

Booking Consumer

↓

Database

↓

Payment

↓

Ticket
```

Notice that booking requests first go to a **queue** instead of directly to the database. ([System Design School][1])

---

# Step 6: Why use a Queue?

Imagine booking opens.

```text
10 Million Users

↓

Book Now
```

Without a queue

```text
10 Million Requests

↓

Database
```

The database crashes.

Instead

```text
10 Million Requests

↓

Queue

↓

Process One by One

↓

Database
```

The queue acts like people waiting in line at a ticket counter.

---

# Step 7: What does the Queue solve?

Suppose three users want Seat A1.

Without queue

```text
A

B

C

↓

Database Together
```

Chaos.

With queue

```text
Queue

↓

A

↓

B

↓

C
```

Now requests are processed in order.

---

# Step 8: Seat Reservation

Don't directly mark the seat as booked.

Instead

```text
Available

↓

Reserved

↓

Booked
```

Example

```text
You clicked Book.

↓

Seat Reserved

↓

You have 5 minutes

↓

Pay
```

If payment succeeds

```text
Reserved

↓

Booked
```

If payment fails

```text
Reserved

↓

Available Again
```

This is called a **temporary hold**.

---

# Step 9: Why both Cache and Database?

The design updates both.

```text
Booking

↓

Cache

↓

Database
```

Why?

### Cache

Fast.

Shows latest seat status quickly.

### Database

Permanent storage.

Source of truth.

If the cache crashes,

the database still has the correct booking information. ([System Design School][1])

---

# Step 10: Payment Flow

```text
Reserve Seat

↓

Payment Service

↓

Stripe / Razorpay

↓

Payment Success

↓

Booking Confirmed

↓

Ticket Generated
```

If payment fails

```text
Release Seat

↓

Available Again
```

---

# Step 11: Why use a Message Queue after Payment?

Some work can happen later.

```text
Booking Done

↓

Queue

↓

Email

↓

SMS

↓

Analytics
```

The user gets a fast response while background tasks continue asynchronously.

---

# Step 12: Why separate Search and Booking?

Think of a shopping mall.

**Search Service**

Like asking,

> "Where is Store 5?"

Just gives information.

**Booking Service**

Like buying the product.

Actually changes inventory.

Separating them allows each service to scale independently.

---

# Step 13: Complete User Journey

```text
User

↓

Search Concert

↓

Search Service

↓

Select Seat

↓

Booking Service

↓

Queue

↓

Reserve Seat

↓

Payment

↓

Booking Confirmed

↓

Email Ticket
```

---

# Step 14: Simple Analogy

Imagine a restaurant.

```text
Customers

↓

Receptionist

↓

Waiting Line

↓

Table

↓

Payment

↓

Receipt
```

In Ticketmaster:

| Restaurant   | Ticketmaster    |
| ------------ | --------------- |
| Receptionist | Booking Service |
| Waiting Line | Message Queue   |
| Table        | Seat            |
| Bill         | Payment         |
| Receipt      | Ticket          |

---

# Beginner Interview Summary (Remember These 7 Points)

1. **Search Service** → Finds events and seats.
2. **Cache (Redis)** → Makes searching fast.
3. **Booking Service** → Handles reservations.
4. **Message Queue** → Processes booking requests in order and prevents the database from being overloaded.
5. **Database** → Stores the permanent booking record.
6. **Payment Service** → Confirms payment before final booking.
7. **Seat states** → `Available → Reserved → Booked` (or `Reserved → Available` if payment fails).

---

## The one sentence interview answer

> "Ticketmaster separates searching from booking. Search requests are served quickly using a cache, while booking requests go through a queue, temporarily reserve the seat, complete payment, and then permanently book it in the database. This design prevents double booking and allows the system to handle millions of users during flash sales." ([System Design School][1])

[1]: https://systemdesignschool.io/problems/ticketmaster/solution?utm_source=chatgpt.com "Design Ticketmaster: A Comprehensive Guide"
