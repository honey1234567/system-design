****
https://youtu.be/jk3yvVfNvds

Google Maps is one of the **most commonly asked system design interview questions** because it combines:

* Distributed Systems
* Graph Algorithms
* Streaming (real-time traffic)
* Databases
* Caching
* Geo-spatial indexing

Let's understand it as if we're building a **Google Maps for one city first**, then scale it worldwide.

---

# Step 1. What problems does Google Maps solve?

Suppose you're in Delhi.

You ask:

> Take me from Dwarka to Connaught Place.

Google Maps must

1. Show map
2. Search places
3. Find shortest/fastest route
4. Give live navigation
5. Recalculate if traffic changes
6. Update ETA continuously

All these happen together.

---

# Overall Architecture

```
                User Mobile App
                     │
         GPS every 3-5 sec (WebSocket)
                     │
             Load Balancer
                     │
      ┌──────────────┼──────────────┐
      │              │              │
 Map Service    Routing Service   Search Service
      │              │              │
      │              │         Search Index
      │              │
      │       Traffic Cache
      │              │
      │      Road Graph DB
      │
 CDN(Map Tiles)

              ↑
      Traffic Processing
              ↑
      Kafka / MQ
              ↑
 Location Updates
```

This matches the major components in the Google Maps design.([System Design School][1])

---

# Component 1: GPS Location Service

Every few seconds your phone sends

```
{
lat:28.61,
lon:77.23,
speed:45km/hr
}
```

Why every few seconds?

Because Maps needs

* your position
* your speed
* your direction

Instead of HTTP every 3 seconds

```
HTTP
Connect
Send
Disconnect
```

it uses

```
WebSocket

Connect once

GPS
GPS
GPS
GPS
GPS
```

Much cheaper.

---

# Component 2: Message Queue

Millions of users send GPS updates.

Without queue

```
Users
   │
Traffic Service

100 million requests
```

Traffic service crashes.

Instead

```
Users

   │
Kafka

   │
Traffic Service
```

Kafka stores updates.

Consumers process them slowly.

Nothing is lost.

---

# Component 3: Traffic Processing Service

This is the brain.

Suppose 10,000 cars are moving on Road A.

GPS updates

```
50
52
48
51
49 km/hr
```

Average speed

```
50 km/hr
```

Traffic = Green

---

Another road

```
5
6
7
8
```

Average

```
6 km/hr
```

Traffic = Red

Traffic Service updates cache

```
Road A

Current Speed

50

Road B

Current Speed

6
```

This continuously updates travel times.([System Design School][1])

---

# Component 4: Road Database

Google doesn't store roads like

```
Road1

Road2

Road3
```

Instead every intersection becomes a graph node.

Example

```
A ----- B ----- C
        |
        |
        D
```

Stored as

```
Node A

Node B

Node C

Node D

Edges

A-B

B-C

B-D
```

Every edge stores

```
distance

speed limit

current traffic

road type

one-way?
```

Example

```
A->B

Distance

5 km

Traffic speed

20 km/hr
```

So every road segment has a dynamic travel cost.([System Design School][1])

---

# Component 5: Routing Service

This service answers

> Which path should I take?

Suppose

```
A

 / \
5   2

B---C
 1
```

Need

A → B

Possible routes

```
A-B =5

A-C-B

2+1=3
```

Shortest

```
A-C-B
```

Routing service computes this.

---

# Component 6: Cache

Traffic changes every second.

Database is slow.

Instead

```
Redis

Road123

speed=25

Road456

speed=12
```

Routing asks Redis first.

Database only if cache misses.

---

# Component 7: Map Tile Service

World map is divided into tiny images (or vector tiles).

Instead of downloading India

download

```
Tile 100

Tile 101

Tile 102
```

When zooming

```
More tiles
```

Stored in CDN.

This makes maps load almost instantly.([System Design School][1])

---

# Component 8: Search Service

Search

```
Pizza
```

Returns nearby restaurants.

Uses

```
Geo Index

Trie

Search Engine
```

instead of scanning every place.

---

# Biggest Interview Question

## How does Google find the shortest path so quickly?

Imagine India's road network.

```
100 million intersections

200 million roads
```

Running basic Dijkstra every second is too slow.

---

## Beginner approach

Suppose

```
A

|
B

|
C

|
D
```

Need

A→D

Dijkstra visits

```
A

B

C

D
```

Works.

---

Now imagine Delhi.

```
Millions of roads
```

Dijkstra explores everywhere.

Very slow.

---

# Google uses A* (A-Star)

A* asks

> Which roads are moving toward the destination?

Example

```
Destination →

A

 /|\

many roads
```

Dijkstra explores all.

A*

```
Destination →

Only explores roads heading right
```

Much faster because it uses a heuristic (often straight-line distance) to prioritize nodes.([techinterview.org][2])

---

# Even A* is not enough

For

Delhi

→

Mumbai

Millions of nodes.

Still expensive.

---

# Contraction Hierarchies (CH)

This is what large navigation systems commonly use.

Think about a city.

```
House roads

↓

Local roads

↓

Main roads

↓

Highways
```

Long trips almost always use highways.

Instead of exploring

```
Every small street
```

CH precomputes shortcuts.

Example

Normal

```
A

|

B

|

C

|

D
```

Store shortcut

```
A

 \

  D
```

When routing

Instead of

```
A→B→C→D
```

Use

```
A→D shortcut
```

Thousands of road segments become one logical edge.

This is why Google Maps can compute routes in milliseconds instead of seconds.([techinterview.org][3])

---

# Real-Time Rerouting

This is the part interviewers love.

Suppose you're driving.

Original route

```
Home

↓

Road A

↓

Office
```

Traffic suddenly appears.

Road A speed changes

```
60 km/hr

↓

5 km/hr
```

Traffic Service updates

```
Redis

Road A

speed=5
```

Routing Service receives the update and recalculates the route using the new edge weights.

```
Old

Home

↓

Road A

↓

Office

New

Home

↓

Road B

↓

Road C

↓

Office
```

The app updates ETA and displays the new route. This happens because only the **edge weights** (travel times) changed; the road graph itself usually remains the same.([System Design School][1])

---

# Does Google compute the entire route every 3 seconds?

**No.**

Instead it uses several optimizations:

1. Your phone sends a new GPS point every few seconds.
2. If you're still following the planned route, it simply updates your current position and ETA.
3. It recomputes the route only when:

   * you take a wrong turn,
   * traffic changes significantly,
   * a road is closed,
   * a much faster alternative appears.

This avoids millions of unnecessary route computations.

---

# End-to-End Flow

```
Phone sends GPS
        │
        ▼
WebSocket Server
        │
        ▼
Kafka
        │
        ▼
Traffic Processing
        │
        ▼
Redis (latest road speeds)
        │
        ▼
Routing Service
        │
        ▼
A* + Contraction Hierarchies
        │
        ▼
Best Route + ETA
        │
        ▼
Phone
```

---

# Interview Summary (Remember This)

| Component               | Purpose                                                   |
| ----------------------- | --------------------------------------------------------- |
| WebSocket               | Continuous GPS updates                                    |
| Kafka                   | Buffers millions of location updates                      |
| Traffic Processing      | Converts GPS data into road traffic speeds                |
| Graph Database          | Stores intersections as nodes and roads as edges          |
| Redis                   | Fast access to live traffic information                   |
| Routing Service         | Computes shortest/fastest routes                          |
| A*                      | Fast route search toward the destination                  |
| Contraction Hierarchies | Precomputed shortcuts for very fast long-distance routing |
| CDN                     | Delivers map tiles quickly                                |
| Search Service          | Finds places and addresses                                |

**The key interview insight:** Google Maps does **not** search all roads every time. It models the road network as a weighted graph, keeps edge weights (travel times) updated from live traffic, uses fast graph algorithms like **A*** for search, and relies on **Contraction Hierarchies** plus incremental rerouting so route updates stay fast even as traffic changes. ([techinterview.org][3])

[1]: https://systemdesignschool.io/problems/google-map/solution?utm_source=chatgpt.com "Design Google Maps System: A Comprehensive Guide"
[2]: https://www.techinterview.org/post/3233460042/design-google-maps-navigation-system/?utm_source=chatgpt.com "Design Google Maps / Navigation System – techinterview"
[3]: https://www.techinterview.org/post/3233466697/system-design-maps/?utm_source=chatgpt.com "System Design: Maps and Navigation Platform — Routing, ETA, and Real-Time Traffic (2025) – techinterview"

These are the **two most important algorithms** in Google Maps system design interviews.

Interviewers usually ask:

> **"How can Google Maps find a route across millions of roads in under 100 ms?"**

The answer is **not just Dijkstra**. It's a combination of:

1. **A*** → Intelligent search
2. **Contraction Hierarchies (CH)** → Precomputed shortcuts

Let's build the intuition from scratch.

---

# Part 1: Why Dijkstra is Slow?

Suppose we have this graph.

```text
        A
      / | \
     2  5  4
    /   |   \
   B    C    D
   |\       /|
  1| \7   2/ |5
   |  \   /  |
   E---F-----G
      3
```

Suppose we want

```
A → G
```

---

## How Dijkstra Works

It always chooses the node with the **smallest distance from the source**.

Initially

```
Distance

A =0
Everything else =∞
```

Visit A

```
A→B =2

A→C =5

A→D =4
```

Choose smallest

```
B
```

Update neighbors

```
E =3

F =9
```

Choose

```
E
```

Update

```
F =6
```

Choose

```
D
```

Update

```
G =9
```

Choose

```
F
```

Update

```
G=min(9,9)
```

Finally reach G.

---

## Problem

Did we really need to explore

```
B

E

F

C
```

Maybe not.

Many nodes are nowhere near the destination.

For a graph with **100 million intersections**, exploring unnecessary nodes is too expensive.

---

# How A* Thinks

Instead of asking

> Which node is closest to the source?

A* asks

> Which node is closest to BOTH the source and the destination?

This is the key idea.

---

## Formula

A* computes

```
f(n)=g(n)+h(n)
```

where

```
g(n)

Distance travelled so far
```

```
h(n)

Estimated distance remaining
```

```
f(n)

Total estimated cost
```

---

## Beginner Example

Imagine this map.

```text
Home

     A

  /      \

B          C

 \        /

   Office
```

Road lengths

```
A→B =2

A→C =2

B→Office =100

C→Office =3
```

Suppose GPS tells us

```
Straight distance

B→Office =90

C→Office =2
```

These straight-line distances are the heuristic (h(n)).

---

### Step 1

Start

```
A

g=0

h=?
```

---

Visit neighbors

---

Node B

```
g=2

h=90

f=92
```

---

Node C

```
g=2

h=2

f=4
```

Which one should we explore?

```
C
```

because

```
4<92
```

We never waste time exploring B first.

---

## Intuition

Dijkstra says

> Cheapest road so far.

A* says

> Cheapest road so far **and** it seems to head toward the destination.

That single change makes a huge difference.

---

# Real Google Maps Example

Suppose

```
Delhi → Mumbai
```

Dijkstra explores roads

```
Delhi

Noida

Ghaziabad

Meerut

Rohtak

Agra

Jaipur

Lucknow

...
```

Everything nearby.

---

A*

asks

```
Where is Mumbai?
```

Mumbai is southwest.

So roads going

```
North

East
```

are less promising.

It prioritizes roads moving toward the destination.

---

# Where does h(n) come from?

Usually

```
Straight-line distance
```

Example

You are here

```
Delhi
```

Destination

```
Mumbai
```

Even if there is no direct road, the straight-line (Euclidean or great-circle) distance is easy to compute.

This estimate helps guide the search.

---

# Why is A* Correct?

The heuristic should **not overestimate** the true remaining distance.

Example

Actual road

```
10 km
```

Heuristic

```
7 km

✔ Good
```

or

```
10 km

✔ Good
```

or

```
0 km

✔ Good (becomes Dijkstra)
```

But

```
15 km

✘ Bad
```

because it may skip the optimal route.

An admissible (non-overestimating) heuristic guarantees A* still finds the shortest path.

---

# Complexity

Worst case

```
Same as Dijkstra
```

Average

```
Much faster
```

because it expands far fewer nodes.

---

# Part 2: Contraction Hierarchies (CH)

Imagine India.

```
Village roads

↓

Town roads

↓

City roads

↓

Highways
```

When going

```
Delhi → Bangalore
```

Do you really drive through every village?

No.

Almost all long trips use highways.

CH exploits this observation.

---

# Normal Graph

```
A

|

B

|

C

|

D

|

E

|

F
```

To reach F

Need

```
A

↓

B

↓

C

↓

D

↓

E

↓

F
```

Six steps.

---

# CH Preprocessing

Google performs an offline preprocessing step.

It notices

```
B

C

D
```

are intermediate nodes.

It creates a shortcut

```
A

 \

  E
```

The shortcut stores

```
Cost

=

A→B→C→D→E
```

The original roads remain in the graph; the shortcut is an additional edge.

---

Now the graph becomes

```text
A ------E------F
 \            /
  \----------/
```

Route search uses

```
A→E→F
```

instead of

```
A→B→C→D→E→F
```

Far fewer nodes are explored.

---

# Another Example

Without CH

```
Delhi

↓

Gurgaon

↓

Jaipur

↓

Ajmer

↓

Udaipur

↓

Ahmedabad

↓

Mumbai
```

Many nodes.

---

With CH

```
Delhi

↓

Jaipur

↓

Ahmedabad

↓

Mumbai
```

The shortcut "Delhi → Jaipur" represents several local roads.

---

# Why Doesn't Google Create One Giant Shortcut?

Suppose traffic changes.

If the entire route were

```
Delhi→Mumbai
```

as one shortcut,

then

```
Jaipur accident
```

would invalidate the whole shortcut.

Instead, CH builds many **smaller** shortcuts so only affected areas need special handling during routing.

---

# How CH is Built

This happens offline, not while you request a route.

Steps:

1. Rank roads by importance.
2. Contract (temporarily remove) less important intersections one by one.
3. Before removing a node, add shortcuts that preserve shortest paths.
4. Store the resulting hierarchy.

When users search, the algorithm uses these shortcuts for very fast routing.

---

# Why is CH So Fast?

Imagine

```
100 million roads
```

Without CH

```
Explore

50,000 nodes
```

With CH

```
Explore

500 nodes
```

Numbers vary, but the reduction can be dramatic.

---

# Combining A* + CH

A* helps decide **which direction** to search.

CH reduces **how much graph** must be searched.

Together they are much faster than Dijkstra alone.

---

# Example Flow

User asks

```
Delhi → Mumbai
```

Graph already contains shortcuts.

```
Delhi

↓

Shortcut

↓

Jaipur

↓

Shortcut

↓

Ahmedabad

↓

Shortcut

↓

Mumbai
```

A* guides the search toward Mumbai while CH skips thousands of local intersections.

---

# Interview Tricky Questions

## 1. Why not use BFS?

**Answer:** BFS assumes every edge has equal cost. Road networks have different distances, speeds, tolls, and traffic delays, so weighted shortest-path algorithms are needed.

---

## 2. Why not always use Dijkstra?

**Answer:** It guarantees the shortest path but explores too many nodes on very large graphs. A* uses a heuristic to reduce exploration.

---

## 3. What happens if the heuristic is 0?

**Answer:** A* becomes Dijkstra because (f(n)=g(n)).

---

## 4. What happens if the heuristic overestimates?

**Answer:** A* may return a non-optimal path because it may ignore the true shortest route.

---

## 5. Does A* always visit fewer nodes?

**Answer:** Usually yes, but in the worst case (poor heuristic) it can visit nearly as many nodes as Dijkstra.

---

## 6. Why can't Google recompute CH every second as traffic changes?

**Answer:** Building CH is computationally expensive. It is done offline because the road topology changes rarely. Live traffic is handled by updating edge weights and using dynamic routing techniques rather than rebuilding the hierarchy for every traffic update.

---

## 7. Does Google rebuild the entire graph when traffic changes?

**Answer:** No. The graph structure (intersections and roads) usually stays the same. Only the edge weights (travel times) change.

---

## 8. Why doesn't Google compute the route every second?

**Answer:** Continuous recomputation for millions of users would be too expensive. It usually recomputes only if:

* you deviate from the route,
* traffic changes significantly,
* a road closes,
* a substantially better route becomes available.

---

## 9. Why use Contraction Hierarchies if A* is already fast?

**Answer:** A* still searches the original graph. On continental-scale road networks, that's still a lot of nodes. CH shrinks the effective search space with precomputed shortcuts, making route computation much faster.

---

## 10. If an accident occurs on a shortcut road, is the shortcut deleted?

**Answer:** Not necessarily. The shortcut represents an abstract connection. During routing, the algorithm considers current edge weights. If the underlying roads become slow, another route may become cheaper and be selected. The preprocessing is rebuilt only occasionally when the road network itself changes significantly, not for every traffic event.

### A simple way to remember the difference

| Algorithm                        | Think of it as                                                | Main job                                             |
| -------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------- |
| **Dijkstra**                     | "Search everywhere fairly."                                   | Finds shortest path without guessing.                |
| **A***                           | "Head toward the destination."                                | Uses a heuristic to reduce the search area.          |
| **Contraction Hierarchies (CH)** | "Skip unimportant intersections using precomputed shortcuts." | Makes searches on huge road networks extremely fast. |

For system design interviews, a strong summary is:

> **Google Maps models roads as a weighted graph, updates edge weights with live traffic, uses A* to guide the search toward the destination, and relies on Contraction Hierarchies to skip thousands of intermediate intersections, allowing route computation in milliseconds even on continental-scale road networks.**

