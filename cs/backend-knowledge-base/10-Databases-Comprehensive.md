# Databases Comprehensive

Language: English | [中文](../后端知识库/10-数据库综合.md)

---

## Table of Contents

1. [Relational Databases](#1-relational-databases)
2. [Columnar Databases](#2-columnar-databases)
3. [Time-Series Databases](#3-time-series-databases)
4. [Vector Databases](#4-vector-databases)
5. [Document Databases](#5-document-databases)
6. [NewSQL Databases](#6-newsql-databases)
7. [Search Engine Databases](#7-search-engine-databases)
8. [Database Selection](#8-database-selection)
9. [Practical Multi-Database Architecture](#9-practical-multi-database-architecture)
10. [Interview Self-Check](#10-interview-self-check)

---

## 1. Relational Databases

### 1.1 PostgreSQL

PostgreSQL is a feature-rich relational database with strong SQL support, MVCC, extensions, JSONB, full-text search, GIS, and advanced indexing.

Common strengths:

- Strong SQL standards support.
- Rich index types: B-Tree, Hash, GIN, GiST, BRIN.
- JSONB for semi-structured data.
- Extensions such as PostGIS and TimescaleDB.
- Good correctness and transaction semantics.

Use PostgreSQL when you need relational modeling, strong consistency, flexible queries, and advanced database features.

### 1.2 Oracle

Oracle is common in enterprise systems with strict requirements around transactions, reliability, security, and long-term support.

Strengths:

- Mature enterprise features.
- Strong optimizer.
- Advanced partitioning and RAC options.
- Rich backup/recovery ecosystem.

Trade-off: high cost and operational complexity.

### 1.3 SQL Server

SQL Server is common in Microsoft ecosystems.

Strengths:

- Good integration with Windows/.NET.
- Mature tooling.
- Enterprise BI ecosystem.

---

## 2. Columnar Databases

### 2.1 Columnar Storage Principles

Row store:

```text
row1: user_id, amount, status, created_at
row2: user_id, amount, status, created_at
```

Column store:

```text
user_id column
amount column
status column
created_at column
```

Columnar databases are efficient for analytical queries because they scan only required columns and compress similar values well.

### 2.2 ClickHouse

ClickHouse is a high-performance OLAP database.

Strengths:

- Fast aggregation over large datasets.
- Columnar storage and vectorized execution.
- Compression.
- Partitioning and sorting keys.
- Distributed tables.

Good use cases:

- Metrics analytics.
- Log analytics.
- User behavior analysis.
- Real-time dashboards.

Trade-offs:

- Not designed for high-frequency small OLTP updates.
- Schema and sorting key design are critical.
- Distributed query cost needs careful control.

### 2.3 HBase

HBase is a distributed wide-column database built on HDFS.

Use cases:

- Very large sparse tables.
- High write throughput.
- Key-range access.

Trade-offs:

- Operational complexity.
- Query model is limited compared with SQL databases.

---

## 3. Time-Series Databases

### 3.1 Time-Series Data

Time-series data is append-heavy and time-indexed.

Examples:

- Metrics.
- IoT sensor data.
- Traces.
- Financial ticks.

Typical operations:

- Downsampling.
- Retention policy.
- Time-window aggregation.
- Tag-based filtering.

### 3.2 InfluxDB

InfluxDB is designed for time-series workloads with retention policies, tags, fields, and time-based queries.

### 3.3 TimescaleDB

TimescaleDB is a PostgreSQL extension for time-series data.

Strengths:

- SQL compatibility.
- Hypertables.
- Continuous aggregates.
- PostgreSQL ecosystem.

### 3.4 Prometheus

Prometheus is a monitoring-oriented time-series system.

Strengths:

- Pull model.
- PromQL.
- Service discovery.
- Alerting ecosystem.

Trade-off: it is a monitoring system first, not a general-purpose analytics database.

---

## 4. Vector Databases

### 4.1 Principles

Vector databases store embeddings and perform approximate nearest neighbor search.

Core concepts:

- Embedding.
- Similarity metric: cosine, inner product, L2 distance.
- ANN index: HNSW, IVF, PQ.
- Metadata filtering.

Use cases:

- Semantic search.
- RAG.
- Recommendation.
- Image/audio similarity.

### 4.2 Milvus

Milvus is an open-source vector database.

Strengths:

- Multiple index types.
- Distributed architecture.
- Hybrid vector + scalar filtering.
- Common in AI/RAG systems.

### 4.3 Pinecone and Weaviate

Pinecone is managed and easy to operate. Weaviate provides vector search with schema and hybrid search features.

Selection depends on operational preference, cost, scale, and integration.

---

## 5. Document Databases

### 5.1 MongoDB

MongoDB stores JSON-like BSON documents.

Strengths:

- Flexible schema.
- Natural mapping to application objects.
- Secondary indexes.
- Aggregation pipeline.
- Sharding.

Good use cases:

- Content management.
- Product catalogs.
- Event-like documents.
- Rapidly evolving schemas.

Trade-offs:

- Flexible schema can become inconsistent without governance.
- Cross-document transactions exist but should not be overused.
- Query/index design is still required.

### 5.2 CouchDB and DocumentDB

CouchDB emphasizes replication and offline-first scenarios. DocumentDB is a managed document-compatible service in cloud ecosystems.

---

## 6. NewSQL Databases

### 6.1 TiDB

TiDB is a distributed SQL database compatible with MySQL protocol.

Strengths:

- Horizontal scalability.
- Distributed transactions.
- HTAP capabilities with TiFlash.
- MySQL compatibility.

Trade-offs:

- More moving parts than MySQL.
- Distributed transaction latency.
- Hotspot and region design matter.

### 6.2 CockroachDB

CockroachDB is a distributed SQL database with strong consistency and PostgreSQL-like interface.

Good for multi-region and fault-tolerant SQL workloads, with trade-offs in latency and operational complexity.

---

## 7. Search Engine Databases

### 7.1 Elasticsearch

Elasticsearch is optimized for full-text search and log analytics.

It is not a replacement for transactional databases.

Use it for:

- Inverted index search.
- Fuzzy search.
- Aggregations.
- Log and observability search.

Keep source-of-truth data in OLTP storage and sync searchable projections into Elasticsearch.

---

## 8. Database Selection

### 8.1 OLTP vs OLAP

| Workload | Goal | Typical Database |
|----------|------|------------------|
| OLTP | transactions, point reads/writes | MySQL, PostgreSQL |
| OLAP | analytics over large data | ClickHouse, BigQuery |
| Time-series | time-window metrics | Prometheus, InfluxDB, TimescaleDB |
| Search | text and relevance | Elasticsearch |
| Vector | semantic similarity | Milvus, Pinecone, Weaviate |
| Document | flexible documents | MongoDB |

### 8.2 Selection Decision Tree

Ask:

1. Is this source-of-truth transactional data?
2. Is the query mostly point lookup, range scan, aggregation, full-text search, or vector similarity?
3. What are the consistency requirements?
4. What is write/read volume?
5. How will data be retained, archived, and recovered?
6. What operational skill does the team have?

### 8.3 Multi-Database Collaboration

Common architecture:

```text
OLTP database as source of truth
-> CDC / message queue
-> Elasticsearch for search
-> ClickHouse for analytics
-> Redis for cache
-> vector database for semantic retrieval
```

Data consistency is usually eventual across derived stores.

---

## 9. Practical Multi-Database Architecture

Example: user activity platform.

| Data | Storage |
|------|---------|
| User profile | MySQL/PostgreSQL |
| Hot cache | Redis |
| Searchable content | Elasticsearch |
| Event analytics | ClickHouse |
| Metrics | Prometheus |
| Embeddings | Milvus |

Design concerns:

- Source of truth must be explicit.
- CDC or event pipeline must be observable.
- Backfill and replay must be possible.
- Derived stores need repair jobs.
- Query services should degrade gracefully if non-critical stores fail.

---

## 10. Interview Self-Check

### Q1: How do you choose between MySQL and PostgreSQL?

**Answer:** Both are strong OLTP databases. MySQL is common for high-throughput web services and has a large ecosystem. PostgreSQL offers richer SQL features, extensions, JSONB, advanced indexing, and often better complex-query flexibility.

### Q2: Why is ClickHouse fast for analytics?

**Answer:** It uses columnar storage, compression, vectorized execution, sparse indexes, partitioning, and sorted data layout, so analytical queries scan less data and aggregate efficiently.

### Q3: Is Elasticsearch a database?

**Answer:** It stores data, but it is best treated as a search/indexing system rather than the source of truth for transactional data.

### Q4: What is a vector database used for?

**Answer:** It stores embedding vectors and supports approximate nearest neighbor search for semantic search, RAG, recommendation, and similarity search.

### Q5: How do you keep MySQL and Elasticsearch consistent?

**Answer:** Use CDC or reliable events, make sync idempotent, monitor lag, support replay/backfill, and treat MySQL as the source of truth.

### Q6: What is HTAP?

**Answer:** Hybrid Transactional/Analytical Processing means one system supports transactional and analytical workloads, often with separate storage/compute paths.

### Q7: When should you use MongoDB?

**Answer:** When document structure maps naturally to business data and schema flexibility is useful. Still design indexes and validation rules carefully.

### Q8: What makes time-series databases different?

**Answer:** They optimize for append-heavy time-indexed data, retention, downsampling, tag filtering, and window aggregation.

### Q9: What is the main risk of multi-database architecture?

**Answer:** Data consistency and operational complexity. Every derived store needs sync, monitoring, replay, and repair.

### Q10: How do you answer database selection questions in interviews?

**Answer:** Start from workload and consistency requirements, then compare query pattern, data model, scale, latency, operational cost, and failure handling.

### Senior Interview Follow-Ups

### Q11: How do you prevent multi-database architecture from becoming inconsistent and unmaintainable?

**Answer:** Define one source of truth first. Every derived store should have an owner, sync pipeline, replay path, lag metric, reconciliation job, and documented degradation behavior. Writes should go through an outbox, CDC, or reliable event stream instead of ad hoc dual writes. For high-risk domains, add version checks, idempotent consumers, audit logs, and periodic repair from the source of truth.

### Q12: How would you choose between PostgreSQL JSONB, MongoDB, and Elasticsearch for semi-structured data?

**Answer:** Use PostgreSQL JSONB when the data still belongs to a relational transaction boundary and needs SQL joins or constraints. Use MongoDB when document shape maps naturally to the domain and schema evolution is frequent. Use Elasticsearch when the main requirement is search, filtering, relevance, or log exploration, not source-of-truth transactions. The key trade-off is query flexibility versus consistency, operational complexity, and index governance.

### Q13: What is the rollback plan when a new database or derived index goes wrong?

**Answer:** Keep the old read path until the new store has been backfilled, validated, and shadow-tested. Use feature flags or routing percentages for gradual traffic migration. Track correctness metrics, lag, error rate, and query latency. If the new store fails, route reads back to the source of truth or old index, pause consumers if needed, fix/replay from a durable offset, and only cut over again after reconciliation passes.
