# Big Data Processing Fundamentals

Language: English | [中文](../后端知识库/14-大数据处理基础.md)

---

## Table of Contents

1. [Big Data Basics](#1-big-data-basics)
2. [Batch Processing](#2-batch-processing)
3. [Stream Processing](#3-stream-processing)
4. [Data Storage](#4-data-storage)
5. [Log Analytics Practice](#5-log-analytics-practice)
6. [Alert Analysis Practice](#6-alert-analysis-practice)
7. [Interview Self-Check](#7-interview-self-check)

---

## 1. Big Data Basics

### 1.1 The 4Vs

Big data is often described by:

- Volume: large amount of data.
- Velocity: fast data generation and ingestion.
- Variety: structured, semi-structured, and unstructured data.
- Veracity: data quality and uncertainty.

Backend engineers should focus less on the slogan and more on pipeline reliability, data correctness, latency, cost, and governance.

### 1.2 Lambda Architecture

Lambda architecture separates batch and real-time paths.

```text
Raw data
  -> batch layer -> accurate historical views
  -> speed layer -> low-latency incremental views
  -> serving layer -> query/API
```

Pros:

- Combines accuracy and low latency.
- Batch layer can recompute results.

Cons:

- Two code paths.
- Higher maintenance cost.
- Consistency between batch and streaming results is hard.

Modern systems often prefer Kappa architecture when streaming replay is reliable enough.

---

## 2. Batch Processing

### 2.1 MapReduce

MapReduce splits computation into map and reduce phases.

```text
Input splits -> Map -> Shuffle/Sort -> Reduce -> Output
```

Strengths:

- Scales to very large data.
- Fault-tolerant.
- Simple programming model.

Weaknesses:

- High disk IO.
- High latency.
- Not good for iterative algorithms.

### 2.2 Spark

Spark improves performance through memory computing and DAG scheduling.

Core concepts:

- Driver.
- Executor.
- RDD/DataFrame/Dataset.
- DAG.
- Stage.
- Task.
- Shuffle.

Performance topics:

- Avoid unnecessary shuffle.
- Use partitioning wisely.
- Cache only reused data.
- Watch data skew.
- Tune executor memory and parallelism.

### 2.3 Hive

Hive provides SQL-like data warehouse access on top of distributed storage.

Use cases:

- Offline reporting.
- ETL.
- Data warehouse modeling.

Modern engines may use Spark SQL, Trino, or Presto for faster interactive queries.

---

## 3. Stream Processing

### 3.1 Stream Processing Concepts

Important terms:

- Event time: when event actually happened.
- Processing time: when system processes it.
- Watermark: progress estimate for event time.
- Window: group events by time/rule.
- State: data remembered across events.
- Checkpoint: fault-tolerant state snapshot.

### 3.2 Flink Core Concepts ⭐⭐⭐

Flink is a stateful stream processing engine.

Strengths:

- Event-time processing.
- Stateful operators.
- Exactly-once state consistency.
- Checkpoints and savepoints.
- Low latency.
- Stream-batch unified model.

Flink job structure:

```text
Source -> Transformation -> Sink
```

### 3.3 Windows and Watermarks

Window types:

- Tumbling window.
- Sliding window.
- Session window.

Watermarks handle late events. A good interview answer should mention the trade-off between waiting longer for late data and producing results quickly.

### 3.4 Kafka Streams

Kafka Streams is a lightweight stream processing library embedded in applications.

Use it when:

- Processing is closely tied to Kafka topics.
- The topology is not too complex.
- You want simpler deployment than a Flink cluster.

---

## 4. Data Storage

### 4.1 ClickHouse

ClickHouse is well-suited for log analytics, metrics aggregation, and user behavior analysis.

Design points:

- Choose partition key by data lifecycle, often date.
- Choose sorting key by common filters.
- Avoid high-frequency updates.
- Control small parts and merge pressure.

### 4.2 Elasticsearch

Elasticsearch is useful for full-text search and log exploration.

Compared with ClickHouse:

- Elasticsearch is stronger for text search and flexible filtering.
- ClickHouse is stronger for heavy analytical aggregation.

Many platforms use both.

---

## 5. Log Analytics Practice

Typical pipeline:

```text
Application logs
-> agent collector
-> Kafka
-> stream processing / parsing
-> Elasticsearch for search
-> ClickHouse for analytics
-> dashboard and alerting
```

Key concerns:

- Backpressure.
- Schema evolution.
- Duplicate events.
- Late events.
- PII masking.
- Retention policy.
- Query cost.

---

## 6. Alert Analysis Practice

### 6.1 Alert Deduplication

Massive alerts need deduplication and grouping.

Strategies:

- Fingerprint by service, instance, rule, severity, and labels.
- Suppress repeated alerts within a time window.
- Group correlated alerts.
- Escalate based on duration and impact.

### 6.2 Root Cause Analysis

RCA inputs:

- Metrics.
- Logs.
- Traces.
- Deploy events.
- Dependency topology.
- Change records.

Practical approach:

```text
detect symptom -> identify blast radius -> correlate changes -> inspect dependencies -> confirm root cause -> mitigate -> postmortem
```

---

## 7. Interview Self-Check

### Q1: What is the difference between batch and stream processing?

**Answer:** Batch processing handles bounded datasets with higher latency and strong recomputation ability. Stream processing handles unbounded data continuously with lower latency and state management.

### Q2: What is shuffle?

**Answer:** Shuffle redistributes data across nodes by key. It is expensive because it involves network IO, serialization, disk spill, and often data skew.

### Q3: Why is Spark faster than MapReduce?

**Answer:** Spark builds a DAG, keeps intermediate data in memory when possible, and avoids writing every stage to disk like classic MapReduce.

### Q4: What is event time?

**Answer:** Event time is when the event occurred, unlike processing time, which is when the system sees it. Event time is important for correct windowed analytics.

### Q5: What is a watermark?

**Answer:** A watermark estimates event-time progress and tells the system when it can close windows while still tolerating late data.

### Q6: What does exactly-once mean in Flink?

**Answer:** It usually means state is updated exactly once with respect to checkpoints and replay. End-to-end exactly-once also requires compatible sinks or idempotent/transactional writes.

### Q7: How do you handle data skew?

**Answer:** Detect hot keys, add salting, split heavy keys, pre-aggregate, tune partitioning, or redesign the key distribution.

### Q8: ClickHouse vs Elasticsearch?

**Answer:** Use ClickHouse for analytical aggregation over structured data. Use Elasticsearch for full-text search and exploratory log search.

### Q9: What is Lambda architecture?

**Answer:** It combines batch and speed layers to provide accurate historical views and low-latency real-time views, at the cost of maintaining two paths.

### Q10: How do you design a reliable log pipeline?

**Answer:** Use durable buffering like Kafka, idempotent processing, schema governance, backpressure, retries, DLQ, monitoring for lag/drop rate, and clear retention policies.

### Senior Interview Follow-Ups

### Q11: How do you handle late, duplicate, and out-of-order events in stream processing?

**Answer:** Use event time instead of processing time, define watermark delay based on product tolerance, and decide whether late events update previous results or go to a side output. Deduplicate with stable event IDs, source offsets, or business keys. For out-of-order data, keep keyed state until the allowed lateness expires. The trade-off is correctness versus latency, state size, and operational cost.

### Q12: A Flink job has rising checkpoint duration and Kafka lag. How would you debug it?

**Answer:** Check source lag, backpressure graph, busy time, checkpoint alignment time, state size, RocksDB metrics, sink latency, GC, and network shuffle. Common causes include slow sinks, data skew, too much keyed state, insufficient parallelism, large checkpoints, or unstable storage. Short-term mitigation may throttle input or scale parallelism; long-term fixes include resharding keys, optimizing state TTL, using async sinks, and tuning checkpoint/storage settings.

### Q13: How do you design replay and backfill safely?

**Answer:** Make processing deterministic and idempotent, separate historical backfill traffic from online streams, record input offsets or snapshot versions, and write to a temporary table/index before cutover when possible. Backfill should have rate limits, observability, and rollback. For derived stores, compare counts, checksums, and sampled business records before declaring the result correct.
