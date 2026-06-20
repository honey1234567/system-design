https://systemdesignschool.io/problems/realtime-monitoring-system/solution?utm_source=neetcode

# Real-Time Monitoring System HLD (Beginner Friendly)

A real-time monitoring system continuously collects metrics/logs from servers and applications, detects issues, and alerts engineers immediately.

Examples:

* CPU usage monitoring
* Memory monitoring
* API latency monitoring
* Error rate monitoring
* Database health monitoring

Think of it as a hospital heart monitor for your infrastructure.

---

# 1. Requirements

### Functional Requirements

1. Collect metrics from servers
2. Store metrics
3. Query metrics
4. Visualize metrics on dashboards
5. Generate alerts
6. Support near real-time updates

### Non-Functional Requirements

* High availability
* Scalable to millions of metrics/sec
* Low latency
* Fault tolerant

---

# 2. High-Level Architecture

```text
+------------+
| Application|
| Servers    |
+-----+------+
      |
      v
+------------+
| Monitoring |
| Agent      |
+-----+------+
      |
      v
+------------+
| Kafka      |
+-----+------+
      |
      v
+-------------------+
| Stream Processor  |
| (Flink/Spark)     |
+-----+-------------+
      |
      +----------------+
      |                |
      v                v

+------------+   +------------+
| Time Series|   | Alert      |
| Database   |   | Engine     |
+------+-----+   +------+-----+
       |                |
       |                v
       |         Email/Slack/SMS
       |
       v
+-------------+
| Dashboard   |
| Grafana     |
+-------------+
```

---

# 3. Step-by-Step Flow

Suppose a server CPU suddenly becomes 95%.

### Step 1: Agent Collects Metrics

Monitoring agent runs on each machine.

Examples:

* CPU = 95%
* Memory = 80%
* Disk = 70%

```json
{
  "server":"server-1",
  "cpu":95,
  "memory":80,
  "timestamp":"10:00:01"
}
```

Popular agents:

* Prometheus Node Exporter
* Telegraf
* Fluent Bit

---

# 4. Why Kafka?

Without Kafka:

```text
Agent -> Database
```

Problem:

If database crashes:

```text
Metrics Lost
```

With Kafka:

```text
Agent -> Kafka -> Consumers
```

Kafka acts as a buffer.

Benefits:

* Handles traffic spikes
* Prevents data loss
* Decouples producers and consumers
* Allows multiple consumers

Example:

10,000 servers send metrics simultaneously.

Kafka absorbs traffic while consumers process at their own speed.

---

# 5. Stream Processing Layer

Tools:

* Apache Flink
* Apache Spark

Responsibilities:

### Aggregation

Raw data:

```text
CPU:
90
92
95
91
93
```

Calculate:

```text
Average CPU = 92.2%
```

### Windowing

Compute metrics every:

* 1 minute
* 5 minutes
* 15 minutes

### Anomaly Detection

Example:

Normal CPU = 40%

Current CPU = 95%

Generate anomaly event.

---

# 6. Time Series Database

Why not MySQL?

Monitoring data looks like:

```text
(timestamp, metric, value)
```

Example:

```text
10:00 -> CPU -> 90
10:01 -> CPU -> 92
10:02 -> CPU -> 95
```

Millions of such entries arrive every minute.

Time-series databases are optimized for this.

Examples:

* Prometheus
* InfluxDB

Data model:

```text
Metric:
CPU

Tags:
server=server1
region=india

Value:
95

Timestamp:
10:00
```

---

# 7. Alert Engine

Rules:

```text
IF CPU > 90%
FOR 5 minutes
THEN Alert
```

Example flow:

```text
Metric arrives
      |
      v
Rule Evaluation
      |
      v
Threshold Breached
      |
      v
Create Alert
```

Notification channels:

* Email
* Slack
* SMS
* PagerDuty

---

# 8. Dashboard Layer

Popular tool:

* Grafana

Dashboard queries TSDB.

Example:

```text
CPU Usage
|
|      *
|    * *
|  *   *
| *    *
+-------------> Time
```

Engineers can:

* View CPU
* View Memory
* View Network
* View Errors
* Zoom into time ranges

---

# 9. Database Schema

### Metrics Table

```text
Metric
------
metric_name
server_id
value
timestamp
```

Example:

```text
cpu
server1
95
10:00
```

In practice TSDB stores this much more efficiently.

---

# 10. Scaling the System

### Agent Scaling

Every server runs its own agent.

```text
Server1 -> Agent1
Server2 -> Agent2
Server3 -> Agent3
```

No central bottleneck.

---

### Kafka Scaling

Use partitions.

```text
Topic: Metrics

Partition-1
Partition-2
Partition-3
Partition-4
```

More consumers can read in parallel.

---

### Processing Scaling

```text
Kafka
  |
  +--> Flink Worker 1
  +--> Flink Worker 2
  +--> Flink Worker 3
```

Add workers horizontally.

---

### Database Scaling

Partition by:

```text
Metric Type
```

or

```text
Time Range
```

Example:

```text
January Data
February Data
March Data
```

---

# 11. Handling Failures

### Kafka Failure

Replication:

```text
Broker1
Broker2
Broker3
```

Data exists on multiple brokers.

---

### Processor Failure

Checkpointing in Flink:

```text
State Saved
Every 30 seconds
```

Processor can resume from checkpoint.

---

### Database Failure

Use replicas:

```text
Primary
   |
Replica1
Replica2
```

---

# 12. Most Common Interview Questions

### Why Kafka?

* Buffering
* Durability
* Decoupling
* Scaling

### Why Flink?

* Real-time processing
* Window aggregation
* Stateful computation
* Event-time processing

### Why TSDB instead of MySQL?

* Optimized for timestamp data
* Better compression
* Faster range queries

### Why Grafana?

* Visualization
* Alerting
* Easy integrations

### What happens if traffic suddenly becomes 100x?

1. Agents continue publishing.
2. Kafka absorbs spike.
3. Consumer lag increases temporarily.
4. Add more Flink consumers.
5. Scale TSDB cluster.

---

# Beginner Interview Version (2-minute Answer)

> Monitoring agents collect metrics from servers and applications. Metrics are sent to Kafka, which acts as a durable buffer. Stream processors such as Flink aggregate and analyze metrics in real time. Processed metrics are stored in a time-series database like Prometheus or InfluxDB. Grafana dashboards visualize the data, while an alert engine continuously evaluates rules and sends notifications when thresholds are breached. Kafka, Flink, and the TSDB can all be scaled horizontally to handle millions of metrics per second.

In a real-time monitoring system, **Apache Flink** is the brain that processes the incoming stream of metrics.

Without Flink:

```text
Server -> Kafka -> Database -> Dashboard
```

You can store and visualize data, but you cannot easily perform real-time calculations, aggregations, or anomaly detection.

With Flink:

```text
Server -> Kafka -> Flink -> Database
                  |
                  +-> Alert Engine
```

## What Flink Does

### 1. Real-Time Aggregation

Servers may send CPU metrics every second:

```text
10:00:01 -> 90%
10:00:02 -> 92%
10:00:03 -> 95%
10:00:04 -> 91%
```

Instead of storing every value, Flink can calculate:

```text
1-minute average CPU = 92%
```

and store the aggregated result.

---

### 2. Windowing

A "window" means processing data over a period of time.

Example:

```text
CPU values for last 5 minutes
```

Flink can calculate:

* Average CPU in last 5 min
* Max CPU in last 5 min
* Request count in last 1 min

Example:

```text
Window = 1 minute

CPU:
90
95
92
93

Result:
Avg = 92.5
```

This is one of Flink's most important features.

---

### 3. Alert Generation

Suppose requirement:

```text
Alert if CPU > 90%
for 5 continuous minutes
```

Flink maintains state:

```text
Minute 1 -> 92
Minute 2 -> 94
Minute 3 -> 95
Minute 4 -> 93
Minute 5 -> 91
```

After 5 minutes:

```text
Generate Alert
```

Without a stream processor, this logic becomes difficult to implement efficiently.

---

### 4. Anomaly Detection

Normal traffic:

```text
100 requests/sec
```

Suddenly:

```text
10,000 requests/sec
```

Flink can detect:

```text
Current Traffic >> Historical Average
```

and emit an anomaly event.

---

### 5. Top-K Calculations

Example:

```text
Top 10 APIs with highest latency
Top 10 servers with highest CPU
Top 10 services with most errors
```

Flink continuously maintains these rankings.

This is why Flink is often used in interview questions like Spotify Top-K, Trending Hashtags, or Monitoring Systems.

---

### 6. Data Enrichment

Incoming metric:

```json
{
  "serverId": 101,
  "cpu": 95
}
```

Flink can join it with metadata:

```json
{
  "serverId": 101,
  "service": "payment",
  "region": "india"
}
```

Output:

```json
{
  "serverId": 101,
  "service": "payment",
  "region": "india",
  "cpu": 95
}
```

---

### 7. Stateful Processing

Flink remembers previous events.

Example:

```text
User logged in
User clicked product
User purchased item
```

Flink can track the entire sequence.

This "memory" is called **state**.

---

## Why Not Do This in the Application Server?

You could write custom code:

```text
Kafka Consumer
      |
Custom Logic
      |
Database
```

Problems:

* Hard to scale
* Hard to recover after crashes
* Complex state management
* Windowing logic becomes difficult
* Checkpointing must be built manually

Flink already provides:

* Distributed processing
* State management
* Fault tolerance
* Checkpointing
* Windowing
* Event-time processing

---

## Interview Answer

> Flink is used to process streaming data in real time. It consumes events from Kafka, performs aggregations, window-based computations, anomaly detection, alert generation, and Top-K calculations. It maintains state, supports fault tolerance through checkpointing, and scales horizontally, making it ideal for real-time monitoring systems.
A **Time Series Database (TSDB)** is a database optimized for storing data that changes over time.

In monitoring systems, almost every record has:

```text
Timestamp + Metric + Value
```

Example:

```text
10:00:01 CPU = 80%
10:00:02 CPU = 82%
10:00:03 CPU = 85%
10:00:04 CPU = 90%
```

This is called **time-series data**.

---

# Why Not Use MySQL/PostgreSQL?

You can store it in MySQL:

```sql
Metric(
  server_id,
  metric_name,
  value,
  timestamp
)
```

But monitoring systems generate huge volumes:

```text
10,000 servers
100 metrics/server
Every 10 seconds
```

That's:

```text
100,000 metrics every 10 sec
10 million+ metrics/day
```

Traditional databases struggle with:

* Massive write rates
* Time-range queries
* Data retention
* Compression

---

# Common Monitoring Queries

Engineers rarely ask:

```sql
SELECT * FROM metric WHERE id=123
```

Instead they ask:

```text
Show CPU usage of server-1
for last 24 hours
```

or

```text
Show average latency
for last 5 minutes
```

or

```text
Show error count trend
for past week
```

TSDBs are optimized for exactly these queries.

---

# Example

Suppose CPU data:

```text
10:00 -> 80
10:01 -> 85
10:02 -> 90
10:03 -> 95
```

A TSDB can efficiently answer:

```text
Average CPU last hour
Maximum CPU today
CPU trend last week
```

---

# Why TSDB is Fast

## 1. Time-Based Indexing

Normal DB:

```text
Index on ID
```

TSDB:

```text
Index on Timestamp
```

Since most queries are time-based:

```text
Last 5 min
Last 1 hour
Last 24 hour
```

retrieval becomes very fast.

---

## 2. Compression

Monitoring data often looks like:

```text
80
81
80
82
81
80
```

Values don't change much.

TSDB uses specialized compression.

Example:

Instead of storing:

```text
80,81,80,82,81,80
```

it stores compressed deltas.

Result:

```text
70%-90% less storage
```

---

## 3. Retention Policies

Monitoring data grows forever.

Example:

```text
1 billion metrics/day
```

Keeping everything forever is expensive.

TSDB can automatically delete old data.

Example:

```text
Keep raw data for 30 days
Keep aggregated data for 1 year
Delete older data
```

No custom cleanup jobs needed.

---

## 4. Downsampling

Raw data:

```text
Every second CPU metric
```

After 30 days you may not need second-level precision.

TSDB automatically converts:

```text
1-second metrics
```

to

```text
1-minute averages
```

saving huge amounts of storage.

---

# Role of TSDB in Monitoring Architecture

```text
Servers
   |
Agents
   |
Kafka
   |
Flink
   |
TSDB
   |
Grafana
```

### Kafka

Temporary buffer.

### Flink

Processes and aggregates streams.

### TSDB

Stores historical metrics efficiently.

### Grafana

Reads data from TSDB and displays graphs.

---

# Why Flink + TSDB Together?

Many beginners ask:

> If TSDB can aggregate data, why do we need Flink?

Because they solve different problems.

### Flink

Real-time computation:

```text
CPU > 90%
Generate alert immediately
```

```text
Top 10 busiest servers
```

```text
Anomaly detection
```

### TSDB

Historical storage:

```text
Show CPU graph for last 7 days
```

```text
Average latency yesterday
```

```text
Memory trend last month
```

Think of it as:

```text
Kafka   = Buffer
Flink   = Processor
TSDB    = Storage
Grafana = Visualization
```

---

# Interview Answer

> A Time Series Database is designed specifically for timestamped data such as CPU, memory, latency, and request counts. It provides high write throughput, efficient time-range queries, compression, retention policies, and downsampling. In a monitoring system, Flink processes metrics in real time, while the TSDB stores historical metrics and serves dashboards and analytical queries.

Yes, **Flink can keep state**, but it is **not a replacement for a Time Series Database**.

This is a very common interview follow-up question.

## What Flink Stores

Flink stores **state needed for computation**.

Example:

```text
CPU values in last 5 minutes
```

To calculate:

```text
Average CPU over 5 minutes
```

Flink keeps these values in its state.

Or:

```text
Server-1 CPU > 90% for 5 mins?
```

Flink stores:

```text
Current count = 4 minutes
```

This state is used for processing logic.

---

## What TSDB Stores

TSDB stores historical data for querying.

Example:

```text
CPU usage of Server-1
for last 30 days
```

or

```text
Latency trend
for last 6 months
```

or

```text
Show graph in Grafana
```

This is long-term storage.

---

## Why Not Store Everything in Flink?

Imagine:

```text
100,000 servers
100 metrics/server
every second
```

That's:

```text
10 million metrics/sec
```

Now suppose management asks:

```text
Show CPU graph for last year
```

If Flink stored all historical data:

* State would become enormous
* Recovery would be very slow
* Checkpoints would become huge
* Expensive memory/storage requirements

Flink was never designed to be queried like a database.

---

## Think of Flink State Like RAM

Suppose you're calculating:

```text
Average CPU for last 5 minutes
```

You only need:

```text
Last 5 minutes data
```

After calculation:

```text
Old values can be discarded
```

This is operational state.

---

## Think of TSDB Like Hard Disk

TSDB keeps:

```text
Last 1 day
Last 1 week
Last 1 month
Last 1 year
```

and supports queries like:

```sql
Show avg CPU
between Jan 1 and Jan 31
```

Flink cannot efficiently answer such ad-hoc analytical queries.

---

## Real Architecture

```text
Metrics
   |
Kafka
   |
Flink
   |
   +----> Alert Engine
   |
   +----> TSDB
             |
         Grafana
```

Flink:

```text
Process
Aggregate
Detect anomalies
Generate alerts
```

TSDB:

```text
Store
Query
Visualize
Retain history
```

---

## Could Flink Store History?

Technically yes.

Flink state can be backed by:

* Memory
* RocksDB
* Checkpoints in object storage

But using Flink as your historical database would be like:

> Using application server memory as your primary database.

It works for small amounts of data but becomes impractical at scale.

---

### Interview Answer

> Flink maintains state for stream processing, such as windows, counters, and aggregations, but it is not optimized for long-term storage or ad-hoc queries. A Time Series Database stores historical metrics efficiently, supports retention policies, compression, and fast time-range queries. Therefore, Flink processes data while the TSDB serves as the system of record for historical monitoring data.
