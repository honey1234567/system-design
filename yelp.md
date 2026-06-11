Elasticsearch is very powerful for **Geo + Text filtering** because it combines:

1. **Full-text search capabilities** (BM25 scoring, fuzzy matching, stemming)
2. **Geospatial indexing** (latitude/longitude search)

This is why systems like food delivery, ride sharing, hotel search, store locators, and job portals often use Elasticsearch.

---

# Problem Example

Suppose a user searches:

> "South Indian restaurant within 5 km"

The system needs to:

* Find restaurants near the user's location
* Match "South Indian" in restaurant name, description, cuisine, reviews
* Rank results by relevance and distance

Traditional SQL becomes slow:

```sql
SELECT *
FROM restaurants
WHERE distance(lat,lng,user_lat,user_lng) < 5
AND cuisine LIKE '%south indian%'
```

For millions of records this becomes expensive.

---

# How Elasticsearch Stores Geo Data

Elasticsearch provides a special field type:

```json
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "location": {
        "type": "geo_point"
      }
    }
  }
}
```

Document:

```json
{
  "name": "A2B South Indian",
  "location": {
    "lat": 28.6139,
    "lon": 77.2090
  }
}
```

Internally Elasticsearch stores geo points using spatial indexing structures.

---

# Geo Filtering

User location:

```text
28.60, 77.20
```

Search within 5 km:

```json
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": {
            "lat": 28.60,
            "lon": 77.20
          }
        }
      }
    }
  }
}
```

This quickly removes all documents outside 5 km.

---

# Text Filtering

Search:

```text
south indian
```

Elasticsearch tokenizes:

```text
south indian
```

into

```text
["south", "indian"]
```

Query:

```json
{
  "match": {
    "name": "south indian"
  }
}
```

---

# Combining Geo + Text

Most common pattern:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "cuisine": "south indian"
          }
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "5km",
            "location": {
              "lat": 28.60,
              "lon": 77.20
            }
          }
        }
      ]
    }
  }
}
```

Execution:

```text
1. Apply geo filter
2. Apply text match
3. Rank remaining results
```

---

# Distance-Based Ranking

Suppose results:

```text
Restaurant A → 500m away
Restaurant B → 2km away
Restaurant C → 4km away
```

We can boost nearby restaurants.

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "cuisine": "south indian"
        }
      },
      "functions": [
        {
          "gauss": {
            "location": {
              "origin": "28.60,77.20",
              "scale": "2km"
            }
          }
        }
      ]
    }
  }
}
```

Ranking becomes:

```text
A score = 10
B score = 8
C score = 5
```

Nearby restaurants appear first.

---

# Real System Example: Uber

Driver document:

```json
{
  "driver_id": 123,
  "location": {
    "lat": 28.61,
    "lon": 77.20
  },
  "vehicle": "SUV"
}
```

Passenger requests SUV.

Query:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "vehicle": "SUV"
          }
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "3km",
            "location": {
              "lat": 28.60,
              "lon": 77.19
            }
          }
        }
      ]
    }
  }
}
```

Returns nearby SUV drivers instantly.

---

# Real System Example: Zomato/Swiggy

Restaurant document:

```json
{
  "name": "A2B",
  "cuisine": "South Indian",
  "rating": 4.5,
  "location": {
    "lat": 28.61,
    "lon": 77.20
  }
}
```

User searches:

```text
best dosa near me
```

Elasticsearch combines:

```text
Text Match:
- dosa
- south indian

Geo Match:
- within delivery radius

Ranking:
- distance
- rating
- popularity
```

---

# Why Elasticsearch is Better than SQL Here

| Feature          | SQL       | Elasticsearch   |
| ---------------- | --------- | --------------- |
| Full-text search | Limited   | Excellent       |
| Fuzzy search     | Hard      | Built-in        |
| Geo distance     | Expensive | Optimized       |
| Ranking          | Manual    | Built-in        |
| Typo tolerance   | Difficult | Easy            |
| Millions of docs | Slower    | Designed for it |

---

# HLD Interview Perspective

In systems like:

* Uber Technologies
* Swiggy
* Zomato
* Airbnb
* LinkedIn

Elasticsearch is typically used as a **search layer**, not the source of truth.

```text
                User Search
                     |
                     v
              Search API
                     |
                     v
             Elasticsearch
         (Geo + Text Index)
                     |
          -------------------
          |                 |
          v                 v
      Restaurant DB     Cache
      (MySQL/Postgres) (Redis)
```

Flow:

```text
1. Data stored in MySQL
2. Kafka captures updates
3. Elasticsearch indexes data
4. User search hits Elasticsearch
5. Elasticsearch returns matching IDs
6. Details fetched from DB/Cache
```

This architecture allows sub-second searches over tens or hundreds of millions of records while supporting complex geo + text queries.

**Elasticsearch** and **PostGIS** can both solve geo-search problems, but they are designed for different purposes.

| Feature                    | Elasticsearch | PostGIS                                      |
| -------------------------- | ------------- | -------------------------------------------- |
| Primary purpose            | Search engine | Geospatial database extension for PostgreSQL |
| Text search                | Excellent     | Good (PostgreSQL Full-Text Search)           |
| Geo queries                | Good          | Excellent                                    |
| Complex spatial operations | Limited       | Very powerful                                |
| Aggregations               | Fast          | Good                                         |
| ACID transactions          | No            | Yes                                          |
| Source of truth            | Usually No    | Yes                                          |
| Ranking by relevance       | Excellent     | Limited                                      |
| Large-scale search         | Excellent     | Good                                         |
| Joins                      | Poor          | Excellent                                    |

---

# When to Use Elasticsearch

Suppose a user searches:

> "Italian restaurants near Connaught Place"

Requirements:

* Full-text search on restaurant name, cuisine, reviews
* Typo tolerance ("Itallian")
* Ranking by relevance
* Geo filtering

Elasticsearch is ideal:

```json
{
  "bool": {
    "must": {
      "match": {
        "cuisine": "italian"
      }
    },
    "filter": {
      "geo_distance": {
        "distance": "5km",
        "location": {
          "lat": 28.63,
          "lon": 77.22
        }
      }
    }
  }
}
```

Why?

* BM25 relevance scoring
* Fuzzy matching
* Autocomplete
* Fast search across millions of documents

This is why companies like Zomato and Swiggy commonly use Elasticsearch as their search layer.

---

# When to Use PostGIS

Suppose you need:

* Find all houses inside a polygon
* Find nearest road
* Check if a driver entered a geofence
* Calculate route intersections
* Find all delivery zones overlapping an area

Example:

```sql
SELECT *
FROM delivery_zones
WHERE ST_Contains(
    zone_polygon,
    ST_Point(77.20, 28.61)
);
```

Or:

```sql
SELECT *
FROM drivers
ORDER BY location <-> ST_Point(77.20, 28.61)
LIMIT 10;
```

PostGIS shines because it supports advanced spatial functions:

* ST_Contains
* ST_Intersects
* ST_Within
* ST_Distance
* ST_Buffer
* ST_Union

These are difficult or impossible to do efficiently in Elasticsearch.

---

# Example: Uber Driver Search

### Elasticsearch Approach

Store:

```json
{
  "driverId": 123,
  "location": {
    "lat": 28.61,
    "lon": 77.20
  }
}
```

Search:

```json
{
  "geo_distance": {
    "distance": "3km"
  }
}
```

Good for:

* Quick nearby-driver lookup
* Search ranking

Not ideal for:

* Complex route geometry calculations

---

### PostGIS Approach

Store:

```sql
CREATE TABLE drivers (
    id BIGINT,
    location GEOGRAPHY(POINT)
);
```

Query:

```sql
SELECT id
FROM drivers
ORDER BY location <-> ST_Point(77.20,28.61)
LIMIT 20;
```

Good for:

* Exact nearest-neighbor searches
* Spatial analytics
* Geofencing

---

# Real-World Architecture

Many systems use **both**.

```text
                    User Search
                         |
                         v
                  Elasticsearch
              (Text + Geo Search)
                         |
                  Restaurant IDs
                         |
                         v
                    PostgreSQL
                      + PostGIS
                 (Source of Truth)
```

### Why both?

PostGIS:

```text
Stores
- Restaurants
- Delivery zones
- Polygons
- Geofences
```

Elasticsearch:

```text
Provides
- Full-text search
- Autocomplete
- Ranking
- Geo filtering
```

---

# Delivery App Example

A food-delivery platform might use:

### PostGIS

To answer:

```text
Can this restaurant deliver to this address?
```

Query:

```sql
ST_Contains(delivery_polygon, user_location)
```

### Elasticsearch

To answer:

```text
Best pizza near me
```

Query:

```text
Text Search + Geo Filter + Ranking
```

---

# HLD Interview Rule of Thumb

Use **PostGIS** when:

* Geography is the primary problem.
* You need polygons, routes, geofencing, spatial joins, containment checks.
* Database is the source of truth.

Use **Elasticsearch** when:

* Search is the primary problem.
* You need relevance ranking, autocomplete, typo tolerance, and geo+text search.

Use **both** when building large systems like ride-sharing, food delivery, real estate, maps, or local business search:

```text
PostGIS  -> geo calculations, geofences, spatial analytics
Elasticsearch -> search, ranking, autocomplete, discovery
```

A common interview answer is:

> "PostGIS is a geospatial database. Elasticsearch is a search engine with geospatial capabilities. If I need complex spatial computation, I choose PostGIS. If I need geo-aware search and ranking at scale, I choose Elasticsearch. In large production systems, I often use both."

A **geospatial database** is a database designed to store, index, and query **location-based data** such as points, lines, polygons, routes, and geographic regions.

Instead of storing only normal data:

```text
User
 ├── id
 ├── name
 └── email
```

a geospatial database can also store:

```text
Restaurant
 ├── id
 ├── name
 └── location (latitude, longitude)

Delivery Zone
 └── polygon

Road
 └── line string
```

---

# Why Normal Databases Are Not Enough

Suppose you store:

```sql
Restaurant
-----------
id
name
latitude
longitude
```

and want to find restaurants within 5 km.

Without geospatial support:

```sql
SELECT *
FROM restaurants
WHERE latitude BETWEEN x AND y
AND longitude BETWEEN a AND b;
```

Problems:

* Inaccurate on Earth's curved surface
* Slow for millions of records
* Cannot perform advanced geo operations

A geospatial database solves this efficiently using specialized spatial indexes.

---

# Types of Geospatial Objects

### 1. Point

Represents a single location.

Example:

```text
Restaurant
ATM
Driver
User
```

```text
(28.6139, 77.2090)
```

📍 Delhi

---

### 2. LineString

Represents a path.

Example:

```text
Road
Metro route
Flight path
```

```text
A ---------- B
```

---

### 3. Polygon

Represents an area.

Example:

```text
Delivery zone
City boundary
Geofence
```

```text
+--------+
|        |
| Area   |
|        |
+--------+
```

---

### 4. MultiPolygon

Multiple areas.

Example:

```text
Country with islands
Multiple service zones
```

---

# Common Geospatial Queries

## 1. Find Nearby Objects

Example:

> Find restaurants within 5 km

```sql
SELECT *
FROM restaurants
WHERE ST_DWithin(
    location,
    user_location,
    5000
);
```

---

## 2. Nearest Neighbor Search

Example:

> Find nearest driver

```sql
SELECT *
FROM drivers
ORDER BY location <-> user_location
LIMIT 1;
```

Used by ride-sharing systems.

---

## 3. Point in Polygon

Example:

> Can restaurant deliver to this address?

```sql
SELECT *
FROM delivery_zones
WHERE ST_Contains(
    zone_polygon,
    user_location
);
```

---

### Visual

```text
Delivery Zone

+------------+
|            |
|      X     |
|            |
+------------+
```

If X is inside → delivery available.

---

## 4. Intersection

Example:

> Does route intersect a restricted area?

```sql
ST_Intersects(route, restricted_zone)
```

Used in logistics and maps.

---

## 5. Distance Calculation

```sql
SELECT ST_Distance(
    point1,
    point2
);
```

Used for:

* Driver matching
* Delivery ETA
* Store locators

---

# How It Works Internally

A geospatial database does **not** scan every row.

Instead it creates spatial indexes such as:

* R-tree
* GiST
* QuadTree
* Geohash
* S2 Cells

Example:

```text
Earth

+----+----+
| A  | B  |
+----+----+
| C  | D  |
+----+----+
```

Locations are indexed by regions.

Query:

```text
Find nearby restaurants
```

The database checks only relevant regions instead of all records.

---

# Popular Geospatial Databases

### PostGIS

Most popular.

Supports:

* Points
* Polygons
* Spatial joins
* Geofencing
* Route calculations

Used heavily in mapping and logistics.

---

### MongoDB

Supports:

```text
2D indexes
2dsphere indexes
```

Good for moderate geo workloads.

---

### Elasticsearch

Provides:

```text
Geo distance search
Geo bounding box
Geo shapes
```

Excellent for geo + text search.

---

### Google BigQuery GIS

Used for large-scale spatial analytics.

---

# Real-World Examples

## Uber

Driver:

```text
Point
```

Operations:

```text
Nearest driver
Distance calculation
Geofencing
```

---

## Zomato / Swiggy

Restaurant:

```text
Point
```

Delivery Area:

```text
Polygon
```

Operations:

```text
Can deliver?
Nearby restaurants
```

---

## Google Maps

Road:

```text
LineString
```

City:

```text
Polygon
```

Operations:

```text
Route generation
Traffic analysis
Shortest path
```

---

# HLD Interview Perspective

When designing systems like:

* Uber Technologies
* DoorDash
* Swiggy
* Zomato
* Google Maps

you typically use:

```text
PostGIS
   ↓
Store locations, polygons, routes

Redis Geo
   ↓
Fast nearby lookups

Elasticsearch
   ↓
Geo + text search
```

### Quick Interview Definition

> A geospatial database is a database that can store and efficiently query geographic data such as points, lines, and polygons. It provides specialized spatial indexes and functions to perform operations like nearest-neighbor search, distance calculation, geofencing, containment, and intersection, which are difficult and inefficient in traditional relational databases.

**CDC (Change Data Capture)** is a technique used to capture changes (INSERT, UPDATE, DELETE) happening in a database and stream those changes to other systems in near real time.

Instead of repeatedly querying the database:

```text
SELECT * FROM orders
WHERE updated_at > last_poll_time;
```

CDC automatically detects changes and publishes them.

---

# Why CDC is Needed

Imagine an e-commerce system:

```text
                PostgreSQL
                     |
              Orders Table
```

When an order is created, multiple systems need updates:

* Elasticsearch (search index)
* Redis (cache)
* Analytics warehouse
* Notification service
* Recommendation engine

Without CDC:

```text
Application
   ├── Update DB
   ├── Update Cache
   ├── Update Elasticsearch
   ├── Update Kafka
   └── Update Analytics
```

Problems:

* Tight coupling
* Failures create inconsistencies
* Difficult to maintain

With CDC:

```text
Application
      |
      v
 PostgreSQL
      |
      v
 CDC Tool
      |
      v
    Kafka
      |
 -------------
 |     |     |
 v     v     v
ES   Redis Analytics
```

The application only writes to the database.

---

# How CDC Works

Every database maintains a transaction log.

Examples:

| Database   | Log                   |
| ---------- | --------------------- |
| PostgreSQL | WAL (Write Ahead Log) |
| MySQL      | Binlog                |
| SQL Server | Transaction Log       |
| Oracle     | Redo Log              |

Example update:

```sql
UPDATE users
SET name='Shreya'
WHERE id=100;
```

The database writes:

```text
UPDATE users
id=100
old_name=John
new_name=Shreya
```

to its transaction log.

A CDC tool reads this log continuously.

---

# CDC Flow

```text
User
  |
  v
Application
  |
  v
PostgreSQL
  |
  | WAL
  v
Debezium
  |
  v
Kafka
  |
  +--> Elasticsearch
  +--> Redis
  +--> Data Warehouse
```

Very common architecture in modern systems.

---

# Example Using Debezium

Suppose this row exists:

```text
users
----------------
id=1
name=John
```

Update:

```sql
UPDATE users
SET name='Shreya'
WHERE id=1;
```

Debezium emits:

```json
{
  "before": {
    "id": 1,
    "name": "John"
  },
  "after": {
    "id": 1,
    "name": "Shreya"
  },
  "op": "u"
}
```

where:

```text
u = update
c = create
d = delete
```

---

# CDC vs Polling

### Polling

```text
Every 5 seconds:
SELECT * FROM users
WHERE updated_at > last_time
```

Problems:

* DB load
* Delay
* Can miss changes
* Inefficient

---

### CDC

```text
Database Change
      ↓
Immediate Event
      ↓
Consumers
```

Benefits:

* Near real-time
* Low overhead
* Reliable
* Scalable

---

# CDC in Elasticsearch Sync

A very common HLD interview question.

Without CDC:

```text
App
 ├── PostgreSQL
 └── Elasticsearch
```

What if ES update fails?

```text
DB updated
ES failed
```

Now data is inconsistent.

---

With CDC:

```text
App
 |
 v
PostgreSQL
 |
 v
Debezium
 |
 v
Kafka
 |
 v
Elasticsearch Consumer
```

Database remains the source of truth.

Elasticsearch is updated asynchronously.

---

# CDC in Microservices

Suppose:

```text
Order Service
Inventory Service
Payment Service
```

Order placed:

```sql
INSERT INTO orders ...
```

CDC emits:

```json
{
  "orderId": 123
}
```

Consumers receive it:

```text
Inventory Service
    ↓
Reserve Stock

Payment Service
    ↓
Charge Payment

Analytics
    ↓
Update Dashboard
```

This follows the Event-Driven Architecture pattern.

---

# Popular CDC Tools

### Debezium

Most popular open-source CDC solution.

Supports:

* PostgreSQL
* MySQL
* SQL Server
* Oracle
* MongoDB

---

### AWS Database Migration Service

AWS-managed CDC.

---

### Oracle GoldenGate

Enterprise CDC solution.

---

# CDC vs Event Sourcing

People often confuse them.

### CDC

```text
Database
    ↓
Read transaction logs
    ↓
Generate events
```

Events are generated after the fact.

---

### Event Sourcing

```text
Application
    ↓
Store events directly
    ↓
Database state rebuilt from events
```

Events are the source of truth.

Example:

```text
OrderCreated
OrderPaid
OrderShipped
```

---

# HLD Interview Perspective

For systems like:

* Uber Technologies
* Netflix
* Amazon
* Airbnb

a common architecture is:

```text
              PostgreSQL
                    |
                 WAL/Binlog
                    |
                Debezium
                    |
                  Kafka
         -----------------------
         |         |           |
         v         v           v
      Redis   Elasticsearch  Data Lake
```

A concise interview definition:

> CDC (Change Data Capture) is a mechanism that captures INSERT, UPDATE, and DELETE operations from a database's transaction log and streams them to downstream systems in near real time. It is commonly used to keep caches, search indexes, analytics platforms, and microservices synchronized with the source database without adding extra load to the application.
A **Yelp HLD (High-Level Design)** interview question is essentially:

> Design a system where users can search nearby businesses (restaurants, cafes, salons, hospitals, etc.), view details, ratings, reviews, photos, and receive geo-based recommendations.

This combines:

* Geo-spatial search
* Full-text search
* Reviews
* Ranking
* Caching
* Search indexing

---

# 1. Requirements Gathering

### Functional Requirements

Users should be able to:

1. Search nearby businesses
2. Search by text

```text
Pizza near me
Best cafe in Delhi
Gym near airport
```

3. View business details
4. Add reviews
5. Add ratings
6. Upload photos
7. Sort by:

   * Distance
   * Rating
   * Popularity

---

### Non-Functional Requirements

* Low latency (<100ms search)
* High availability
* Eventual consistency acceptable
* Geo search support
* Scale to millions of businesses

---

# 2. Capacity Estimation

Assume:

```text
100M users
20M businesses
1B reviews
```

Search traffic:

```text
50K searches/sec
```

Review traffic:

```text
5K writes/sec
```

Searches greatly exceed writes.

Therefore:

```text
Read Heavy System
```

---

# 3. Core APIs

### Search API

```http
GET /search
```

Request:

```json
{
  "query":"pizza",
  "lat":28.61,
  "lon":77.20,
  "radius":"5km"
}
```

---

### Business Details

```http
GET /business/{id}
```

---

### Add Review

```http
POST /review
```

```json
{
  "businessId":123,
  "rating":5,
  "comment":"Amazing food"
}
```

---

# 4. Database Design

## Business Table

```sql
Business
---------
business_id
name
category
address
latitude
longitude
avg_rating
review_count
```

---

## Review Table

```sql
Review
--------
review_id
business_id
user_id
rating
comment
created_at
```

---

## User Table

```sql
User
------
user_id
name
```

---

# Why Not Search Directly in MySQL?

Query:

```sql
SELECT *
FROM business
WHERE category='pizza'
AND distance < 5km
```

For 20M businesses this becomes expensive.

Need specialized search engine.

---

# 5. High-Level Architecture

```text
                User
                  |
                  v
            Load Balancer
                  |
        -------------------
        |                 |
        v                 v
   Search Service   Review Service
        |
        v
 Elasticsearch
        |
        v
 PostgreSQL
```

---

# 6. Search Architecture

This is the most important part.

Search requires:

```text
Geo Search
+
Text Search
```

Example:

```text
best pizza near me
```

Need:

```text
pizza     -> text filter
near me   -> geo filter
```

Elasticsearch is ideal.

---

## Elasticsearch Document

```json
{
  "businessId":123,
  "name":"Dominos Pizza",
  "category":"Pizza",
  "rating":4.5,
  "location":{
     "lat":28.61,
     "lon":77.20
  }
}
```

---

# 7. Geo Search

User:

```text
lat=28.61
lon=77.20
radius=5km
```

ES query:

```json
{
  "geo_distance": {
      "distance":"5km"
  }
}
```

Returns only nearby businesses.

---

# 8. Text Search

Query:

```text
pizza
```

ES performs:

```text
Tokenization
Stemming
BM25 Ranking
Fuzzy Search
```

Examples:

```text
pizza
piza
pizzza
```

still work.

---

# 9. Geo + Text Search

Combined query:

```text
Find pizza shops
within 5km
```

ES:

```text
Text Match
AND
Geo Filter
```

Result:

```text
Pizza Hut
Dominos
La Pinoz
```

---

# 10. Ranking

Businesses should not be sorted only by distance.

Bad:

```text
200m away
Rating 1.5
```

should not beat

```text
500m away
Rating 4.9
```

Ranking score:

```text
Final Score =
Text Relevance
+ Rating Score
+ Popularity
+ Distance Score
```

Example:

```text
Score =
0.4 * BM25
+0.3 * Rating
+0.2 * Reviews
+0.1 * Distance
```

---

# 11. Review Service

User adds review:

```text
5 stars
Amazing pizza
```

Flow:

```text
User
 |
 v
Review Service
 |
 v
PostgreSQL
```

---

# 12. Review Aggregation

Avoid:

```sql
AVG(rating)
```

over millions of reviews every request.

Store precomputed values:

```sql
avg_rating
review_count
```

inside Business table.

Update asynchronously.

---

# 13. CDC Pipeline

Whenever a review is added:

```text
PostgreSQL
     |
     v
   CDC
     |
     v
   Kafka
```

Consumers update:

```text
Elasticsearch
Analytics
Cache
```

Common stack:

```text
PostgreSQL
     |
Debezium
     |
Kafka
```

---

# 14. Elasticsearch Sync

```text
Business Updated
      |
      v
PostgreSQL
      |
      v
CDC
      |
      v
Kafka
      |
      v
Elasticsearch Consumer
```

Keeps search index updated.

---

# 15. Caching Layer

Frequently viewed businesses:

```text
Starbucks
Dominos
McDonalds
```

stored in:

Redis

```text
Business Cache
Review Cache
Search Cache
```

---

# 16. Geo-Spatial Indexing

For extremely large scale:

```text
20M+
businesses
```

Use:

### Geohash

Earth divided into cells.

```text
9q8
9q9
9qb
```

Nearby locations fall into similar cells.

---

### S2 Cells

Used by:

Google

More accurate spherical partitioning.

---

# 17. Photo Storage

Never store photos in DB.

Store in object storage.

```text
User Upload
      |
      v
Object Storage (S3)
      |
      v
CDN
```

Possible choice:

Amazon Web Services object storage service.

---

# 18. Scaling

### Database

```text
Primary
   |
Replicas
```

Reads:

```text
Business details
Reviews
```

go to replicas.

---

### Elasticsearch

```text
Index
  |
Shards
```

Example:

```text
20 shards
3 replicas
```

---

# 19. End-to-End Search Flow

```text
User searches:
"pizza near me"

       |
       v
Search API
       |
       v
Redis Cache
       |
 cache miss
       |
       v
Elasticsearch
   (geo + text)
       |
 business IDs
       |
       v
Business Service
       |
 PostgreSQL/Cache
       |
       v
Results
```

---

# Follow-Up Questions Interviewers Ask

### Why Elasticsearch over PostgreSQL?

Answer:

```text
Full-text search
Geo filtering
Ranking
Autocomplete
Fuzzy matching
```

---

### Why CDC?

Answer:

```text
Keep Elasticsearch and cache
synchronized with database
without application coupling.
```

---

### Why Redis?

Answer:

```text
Popular business details
review counts
hot searches
```

---

### PostGIS vs Elasticsearch?

Answer:

```text
PostGIS:
Spatial computation

Elasticsearch:
Geo + text search
```

---

# Final Production Architecture

```text
                 Users
                    |
             API Gateway
                    |
      --------------------------------
      |                              |
      v                              v
 Search Service              Review Service
      |                              |
      v                              v
 Elasticsearch              PostgreSQL
      |                              |
      |                        Debezium CDC
      |                              |
      |                              v
      |                            Kafka
      |                              |
      |------------------------------|
                     |
                     v
                  Consumers
                     |
        ----------------------------
        |            |             |
        v            v             v
 Elasticsearch    Redis      Analytics
```

This design demonstrates most of the concepts interviewers look for in Yelp-style systems: **geo-search, Elasticsearch, PostGIS/geospatial concepts, Redis caching, Kafka, CDC, ranking, sharding, replication, and eventual consistency.**
