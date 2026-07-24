# Elasticsearch Search Engine

Language: English | [中文](../后端知识库/19-Elasticsearch搜索引擎.md)

---

## Table of Contents

### Core Principles
1. [Core Elasticsearch Concepts](#1-core-elasticsearch-concepts)
2. [Inverted Index Principles](#2-inverted-index-principles)
3. [Analyzers](#3-analyzers)

### Querying and Aggregation
4. [Query DSL](#4-query-dsl)
5. [Aggregations](#5-aggregations)

### Architecture and Performance
6. [Cluster Architecture](#6-cluster-architecture)
7. [Write and Search Flow](#7-write-and-search-flow)
8. [Performance Optimization](#8-performance-optimization)

### Engineering Practice and Interview Review
9. [Solution Comparison](#9-solution-comparison)
10. [Go Integration Practice](#10-go-integration-practice)
11. [Practical Case](#11-practical-case)
12. [Interview Self-Check](#12-interview-self-check)

---

## 1. Core Elasticsearch Concepts

### 1.1 Basic Terms

| Elasticsearch Concept | MySQL Analogy | Explanation |
|-----------------------|---------------|-------------|
| **Index** | Database / table-like logical container | Logical container for documents with a mapping |
| **Document** | Row | Smallest data unit, stored as JSON |
| **Field** | Column | A field inside a document |
| **Mapping** | Schema | Field type definitions and index settings |
| **Shard** | Partition | Physical shard of an index; backed by a Lucene index |
| **Replica** | Slave / read replica | Copy of a shard for high availability and read scaling |

### 1.2 Index and Shards

```text
Index: products (5 Primary Shards, 1 Replica)
┌─────────────────────────────────────────────────┐
│  Node-1         Node-2         Node-3           │
│  ┌─────┐       ┌─────┐       ┌─────┐          │
│  │ P0  │       │ P1  │       │ P2  │          │
│  │ R1  │       │ R2  │       │ R0  │          │
│  └─────┘       └─────┘       └─────┘          │
│  ┌─────┐       ┌─────┐                         │
│  │ P3  │       │ P4  │                         │
│  │ R4  │       │ R3  │                         │
│  └─────┘       └─────┘                         │
└─────────────────────────────────────────────────┘
```

### 1.3 Document Routing

Elasticsearch decides the target primary shard with:

```text
shard_num = hash(routing) % number_of_primary_shards
```

By default, `routing = _id`. This is the core reason why the number of primary shards cannot be changed after an index is created: changing the shard count would change document routing results.

### 1.4 Mapping Definition

```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "brand": {
        "type": "keyword"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "description": {
        "type": "text",
        "index": false
      },
      "created_at": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "tags": {
        "type": "keyword"
      },
      "specs": {
        "type": "nested",
        "properties": {
          "key": { "type": "keyword" },
          "value": { "type": "keyword" }
        }
      }
    }
  }
}
```

**Field type selection**:

| Type | Use Case | Analyzed? | Aggregatable? |
|------|----------|-----------|---------------|
| `text` | Full-text search | Yes | No, unless fielddata is enabled |
| `keyword` | Exact match, aggregation, sorting | No | Yes |
| `integer/long` | Numeric range query | No | Yes |
| `date` | Time range query and sorting | No | Yes |
| `nested` | Independent queries on nested objects | N/A | N/A |
| `object` | Flattened object | N/A | N/A |

---

## 2. Inverted Index Principles

### 2.1 Forward Index vs Inverted Index ⭐⭐⭐

**Forward index** maps document to terms:

```text
Doc1 -> ["Elasticsearch", "is", "distributed", "search", "engine"]
Doc2 -> ["Elasticsearch", "supports", "full-text", "search"]
Doc3 -> ["Redis", "is", "in-memory", "database"]
```

**Inverted index** maps term to documents:

```text
"Elasticsearch" -> [Doc1, Doc2]
"search"        -> [Doc1, Doc2]
"distributed"   -> [Doc1]
"Redis"         -> [Doc3]
"in-memory"     -> [Doc3]
```

Search engines use inverted indexes to quickly locate all documents containing a term instead of scanning every document.

### 2.2 Inverted Index Structure ⭐⭐⭐

```text
Inverted Index = Term Dictionary + Posting List

┌─────────────────────────────────────────────┐
│              Term Dictionary                │
│  All unique terms, sorted lexicographically │
│                                             │
│  Term          -> Posting List Pointer      │
│  ─────────────────────────────────          │
│  "database"    -> -> [3, 7, 15, 42]         │
│  "elastic"     -> -> [1, 2, 5, 8, 12]       │
│  "redis"       -> -> [3, 9, 20]             │
│  "search"      -> -> [1, 2, 4, 8]           │
│  ...                                        │
└─────────────────────────────────────────────┘

Posting List:
┌──────────────────────────────────────────────┐
│  DocID │ TF       │ Position │ Offset        │
│  ──────┼──────────┼──────────┼───────        │
│    1   │    2     │  [3, 15] │ [12:20,45:53] │
│    2   │    1     │  [7]     │ [28:36]       │
│    5   │    3     │  [1,8,12]│ [0:8,...]     │
└──────────────────────────────────────────────┘
```

### 2.3 Term Dictionary and FST ⭐⭐⭐

When there are millions of terms, the engine needs a compact way to locate terms quickly.

```text
Term Index (FST, in memory)
    |
    v
Term Dictionary (on disk, block-based)
    |
    v
Posting List (on disk)
```

**FST, or Finite State Transducer**:

- A finite-state automaton similar to Trie but more compact.
- Compresses common prefixes of terms.
- Helps locate the block in the Term Dictionary quickly.
- Often uses far less memory than loading the full Term Dictionary.

```text
FST example storing "cat"=5, "car"=3, "do"=7, "dog"=11:

     c --- a --- t -> output=5
      \         \
       \         r -> output=3
        d --- o -> output=7
               \
                g -> output=11
```

### 2.4 Posting List Compression

**Frame of Reference encoding**:

```text
Original Posting List: [73, 300, 302, 332, 343, 372]

Step 1: Delta Encoding
[73, 227, 2, 30, 11, 29]

Step 2: Block + bit packing
Block 1: [73, 227, 2] -> max=227, requires 8 bits
Block 2: [30, 11, 29] -> max=30,  requires 5 bits

Compression: 6*32 bits = 192 bits -> 3*8 + 3*5 = 39 bits
```

**Roaring Bitmaps** are often used for filter scenarios:

- Split the DocID space into blocks of 65,536.
- Dense blocks use bitmaps.
- Sparse blocks use sorted arrays.
- Efficient for intersection and union in boolean queries.

### 2.5 Skip List Acceleration

Posting list intersection uses skip lists to avoid scanning every element.

```text
Query: "elastic" AND "search"

"elastic" posting: 1 -> 2 -> 5 -> 8 -> 12 -> 15 -> 20
                   |       |       |        |
Skip:              1 ----> 5 ----> 12 ----> 20

"search" posting:  1 -> 2 -> 4 -> 8 -> 16 -> 22

Result: [1, 2]
```

### 2.6 BM25 Relevance Scoring

Elasticsearch 5.x+ uses BM25 by default instead of TF-IDF.

```text
BM25(D, Q) = Σ IDF(qi) * (f(qi,D) * (k1+1)) / (f(qi,D) + k1 * (1-b+b*|D|/avgdl))

Where:
- IDF(qi) = ln((N - n(qi) + 0.5) / (n(qi) + 0.5) + 1)
- f(qi,D) = term frequency of qi in document D
- |D| = document length
- avgdl = average document length
- k1 = 1.2 by default, controls term-frequency saturation
- b = 0.75 by default, controls document length normalization
```

Key differences from TF-IDF:

- BM25 caps the benefit of term frequency through `k1`.
- BM25 handles document length normalization more reasonably through `b`.

---

## 3. Analyzers

### 3.1 Analyzer Pipeline ⭐⭐

```text
Analyzer = Character Filter + Tokenizer + Token Filter

Input text -> [Character Filter] -> [Tokenizer] -> [Token Filter] -> Output terms
              preprocessing          splitting      post-processing
```

### 3.2 Built-in Analyzer Comparison

| Analyzer | Description | Example: "The Quick-Brown FOX" |
|----------|-------------|--------------------------------|
| `standard` | Unicode-aware text segmentation | [the, quick, brown, fox] |
| `simple` | Splits on non-letter characters | [the, quick, brown, fox] |
| `whitespace` | Splits on whitespace only | [The, Quick-Brown, FOX] |
| `keyword` | Treats the whole input as one token | [The Quick-Brown FOX] |
| `pattern` | Regex-based splitting | Customizable |

### 3.3 Chinese Tokenization ⭐⭐

| Analyzer | Mode | Example: "中华人民共和国国歌" |
|----------|------|-----------------------------|
| `standard` | Single-character split | [中, 华, 人, 民, 共, 和, 国, 国, 歌] |
| `ik_smart` | Coarse-grained | [中华人民共和国, 国歌] |
| `ik_max_word` | Fine-grained | [中华人民共和国, 中华人民, 中华, 华人, 人民共和国, 人民, 共和国, 共和, 国歌] |

**IK analyzer configuration**:

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_ik_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "my_synonym", "my_stopword"]
        }
      },
      "filter": {
        "my_synonym": {
          "type": "synonym",
          "synonyms_path": "analysis/synonyms.txt"
        },
        "my_stopword": {
          "type": "stop",
          "stopwords_path": "analysis/stopwords.txt"
        }
      }
    }
  }
}
```

### 3.4 Custom Analyzer

```json
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_strip_filter": { "type": "html_strip" },
        "emoji_filter": {
          "type": "pattern_replace",
          "pattern": "[\\p{So}]",
          "replacement": ""
        }
      },
      "tokenizer": {
        "my_ngram": {
          "type": "ngram",
          "min_gram": 2,
          "max_gram": 3,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "autocomplete_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip_filter", "emoji_filter"],
          "tokenizer": "my_ngram",
          "filter": ["lowercase"]
        }
      }
    }
  }
}
```

---

## 4. Query DSL

### 4.1 Query Context vs Filter Context ⭐⭐⭐

```text
Query Context  -> computes relevance score (_score)
Filter Context -> does not compute score, can be cached, usually faster
```

Use filter context for exact constraints such as status, category, tenant ID, and time ranges.

### 4.2 Full-Text Queries

**Match query**:

```json
{
  "query": {
    "match": {
      "title": {
        "query": "elasticsearch tutorial",
        "operator": "and",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

**Multi-match query**:

```json
{
  "query": {
    "multi_match": {
      "query": "elasticsearch guide",
      "type": "best_fields",
      "fields": ["title^3", "description", "content"],
      "tie_breaker": 0.3
    }
  }
}
```

**Match phrase query**:

```json
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "distributed search",
        "slop": 2
      }
    }
  }
}
```

### 4.3 Exact Queries

**Term query**:

```json
{
  "query": {
    "term": {
      "status": { "value": "published" }
    }
  }
}
```

**Range query**:

```json
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 500
      },
      "created_at": {
        "gte": "2025-01-01",
        "lt": "now/d",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

### 4.4 Bool Query ⭐⭐⭐

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "elasticsearch" } }
      ],
      "must_not": [
        { "term": { "status": "deleted" } }
      ],
      "should": [
        { "match": { "tags": "tutorial" } },
        { "match": { "tags": "beginner" } }
      ],
      "filter": [
        { "range": { "price": { "lte": 100 } } },
        { "term": { "category": "tech" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

| Clause | Meaning | Affects Score? | Cache Friendly? |
|--------|---------|----------------|-----------------|
| `must` | Must match | Yes | No |
| `filter` | Must match | No | Yes |
| `should` | Should match | Yes | No |
| `must_not` | Must not match | No | Yes |

### 4.5 Nested Query

```json
{
  "query": {
    "nested": {
      "path": "specs",
      "query": {
        "bool": {
          "must": [
            { "term": { "specs.key": "color" } },
            { "term": { "specs.value": "red" } }
          ]
        }
      }
    }
  }
}
```

### 4.6 Function Score

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "phone" } },
      "functions": [
        {
          "field_value_factor": {
            "field": "sales",
            "modifier": "log1p",
            "factor": 0.1
          }
        },
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "7d",
              "decay": 0.5
            }
          }
        },
        {
          "filter": { "term": { "is_promoted": true } },
          "weight": 2
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

---

## 5. Aggregations

### 5.1 Aggregation Types ⭐⭐

| Type | Description | Common Aggregations |
|------|-------------|---------------------|
| Bucket | Grouping, similar to `GROUP BY` | terms, date_histogram, range, nested |
| Metric | Metric calculation | avg, sum, min, max, cardinality, percentiles |
| Pipeline | Secondary calculation based on aggregation results | derivative, moving_avg, bucket_sort |

### 5.2 Bucket Aggregation

**Terms aggregation**:

```json
{
  "size": 0,
  "aggs": {
    "brand_distribution": {
      "terms": {
        "field": "brand",
        "size": 20,
        "order": { "_count": "desc" },
        "min_doc_count": 10
      },
      "aggs": {
        "avg_price": {
          "avg": { "field": "price" }
        }
      }
    }
  }
}
```

**Date histogram**:

```json
{
  "size": 0,
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM",
        "min_doc_count": 0
      },
      "aggs": {
        "total_revenue": {
          "sum": { "field": "amount" }
        }
      }
    }
  }
}
```

### 5.3 Metric Aggregation

```json
{
  "size": 0,
  "aggs": {
    "price_stats": {
      "extended_stats": { "field": "price" }
    },
    "unique_brands": {
      "cardinality": {
        "field": "brand",
        "precision_threshold": 1000
      }
    },
    "price_percentiles": {
      "percentiles": {
        "field": "price",
        "percents": [50, 75, 90, 95, 99]
      }
    }
  }
}
```

### 5.4 Pipeline Aggregation

```json
{
  "size": 0,
  "aggs": {
    "monthly_sales": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total": { "sum": { "field": "amount" } }
      }
    },
    "max_monthly_sales": {
      "max_bucket": {
        "buckets_path": "monthly_sales>total"
      }
    },
    "sales_derivative": {
      "derivative": {
        "buckets_path": "monthly_sales>total"
      }
    }
  }
}
```

---

## 6. Cluster Architecture

### 6.1 Node Roles ⭐⭐⭐

| Role | Configuration | Responsibility | Resource Need |
|------|---------------|----------------|---------------|
| **Master** | `node.roles: [master]` | Cluster state, index creation/deletion, shard allocation | Low CPU/memory, high stability |
| **Data** | `node.roles: [data]` | Store data, execute CRUD and search | High disk and memory |
| **Data Hot/Warm/Cold** | `node.roles: [data_hot]` | Tiered storage | Hot on SSD, cold on HDD/object storage |
| **Coordinating** | `node.roles: []` | Request routing and result merging | High CPU/memory |
| **Ingest** | `node.roles: [ingest]` | Preprocess documents through pipelines | High CPU |

### 6.2 Cluster Health

| Status | Meaning | Response |
|--------|---------|----------|
| Green | All primary shards and replicas are assigned | Normal |
| Yellow | All primary shards are assigned, but some replicas are not | Check node count and allocation settings |
| Red | Some primary shards are unassigned; data may be unavailable | Urgent investigation |

### 6.3 Shard Allocation

```json
{
  "cluster.routing.allocation.awareness.attributes": "rack,zone",
  "cluster.routing.allocation.disk.watermark.low": "85%",
  "cluster.routing.allocation.disk.watermark.high": "90%",
  "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
}
```

Shard allocation considers:

- Primary and replica of the same shard should not be on the same node.
- Disk watermarks prevent unsafe allocation.
- Awareness attributes distribute shards across racks or zones.
- Balancing tries to spread shard count and disk usage.

### 6.4 Split-Brain Protection ⭐⭐⭐

In Elasticsearch 7.x+, master election is handled by the new cluster coordination subsystem.

```yaml
cluster.initial_master_nodes: ["master-1", "master-2", "master-3"]

# The voting quorum is calculated automatically.
# With 3 master-eligible nodes, at least 2 votes are required.
```

Common causes of split-brain-like incidents:

1. Network partition between master and part of the cluster.
2. Heavy master load or long GC pause.
3. Incorrect legacy configuration, such as too low `minimum_master_nodes` in older ES versions.

Mitigation:

- Use dedicated master nodes.
- Use an odd number of master-eligible nodes, usually 3 or 5.
- Tune discovery and timeout settings carefully.

---

## 7. Write and Search Flow

### 7.1 Write Flow ⭐⭐⭐

```text
Client -> Coordinating Node -> Primary Shard -> Replica Shard(s)

Detailed steps:
1. Client sends a write request to any node; that node becomes the coordinating node.
2. The coordinating node calculates the target primary shard based on routing.
3. The request is forwarded to the node holding the primary shard.
4. The primary shard writes to:
   a. Translog, for durability.
   b. In-memory buffer.
5. The primary forwards the operation to replica shards.
6. Replicas finish and respond to the primary.
7. The primary responds to the coordinating node.
8. The coordinating node responds to the client.
```

### 7.2 Near Real-Time Search ⭐⭐⭐

```text
Write -> Buffer -> (Refresh 1s) -> Segment(searchable) -> (Flush) -> Disk

┌──────────────────────────────────────────────────┐
│                   Lucene Shard                   │
│                                                  │
│  ┌──────────┐   Refresh(1s)    ┌──────────────┐ │
│  │ In-Memory│ ──────────────-> │ New Segment  │ │
│  │  Buffer  │                  │ (searchable) │ │
│  └──────────┘                  └──────────────┘ │
│       │                              │           │
│       │ also write                   │ Flush     │
│       v                              v           │
│  ┌──────────┐                  ┌──────────────┐ │
│  │ Translog │ ─── Flush ────-> │ Disk Commit  │ │
│  │(WAL)     │                  │ (fsync)      │ │
│  └──────────┘                  └──────────────┘ │
└──────────────────────────────────────────────────┘
```

| Operation | Frequency | Purpose | Durability |
|-----------|-----------|---------|------------|
| **Refresh** | Default 1s | Buffer -> searchable segment | Not a disk fsync guarantee |
| **Flush** | Default periodic or translog threshold | Clear translog and commit segments | Durable |
| **Translog** | Every write | WAL before refresh | Durable by configured policy |

### 7.3 Segment Merge

```text
Small segments are continuously created -> background merge -> larger segments

Merge process:
1. Select multiple small segments.
2. Merge live documents into a new large segment.
3. Drop documents marked as deleted.
4. Atomically switch references after the new segment is ready.
```

Segments are immutable. This improves concurrency, compression, and OS page cache efficiency, but creates merge cost.

### 7.4 Search Flow: Query Then Fetch

```text
Phase 1 - Query:
Coordinating node sends query to all target shards.
Each shard returns Top N DocIDs and scores.
The coordinating node merges global Top N.

Phase 2 - Fetch:
The coordinating node requests full documents from the shards that own selected DocIDs.
The final result is returned to the client.
```

This explains why deep pagination is expensive: each shard may need to return `from + size` candidates before the coordinating node can merge the global page.

---

## 8. Performance Optimization

### 8.1 Index Design ⭐⭐⭐

**Shard sizing**:

```text
Recommended shard size: 10GB-50GB
Number of shards = estimated data size / target shard size
Example: 1TB data -> roughly 20-100 shards
```

**Time-based indices and aliases**:

```json
PUT logs-2025-04
{
  "aliases": {
    "logs-current": {},
    "logs-search": {}
  }
}
```

**Mapping optimization**:

```json
{
  "mappings": {
    "dynamic": "strict",
    "_source": { "excludes": ["large_field"] },
    "properties": {
      "status": { "type": "keyword", "doc_values": true },
      "description": { "type": "text", "norms": false },
      "internal_id": { "type": "keyword", "index": false }
    }
  }
}
```

### 8.2 Query Optimization

**Use filter context when scoring is not needed**:

```json
{ "query": { "bool": { "filter": { "term": { "status": "active" } } } } }
```

**Avoid deep pagination**:

```json
{
  "size": 10,
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2025-04-20T10:00:00", "doc_12345"]
}
```

For large exports, use scroll or, in newer versions, point-in-time plus `search_after`.

**Routing optimization**:

```json
PUT /orders/_doc/order_123?routing=user_456
{
  "user_id": "user_456",
  "amount": 99.9
}
```

If queries always include a tenant ID or user ID, routing can reduce the number of shards hit. The trade-off is potential hot shards.

### 8.3 JVM Tuning

```yaml
# jvm.options
-Xms16g
-Xmx16g

# ES 7.x+ defaults to G1 in many distributions
-XX:+UseG1GC
-XX:G1HeapRegionSize=16m
-XX:InitiatingHeapOccupancyPercent=45
```

Memory principles:

- JVM heap should usually be no more than 50% of physical memory.
- Keep heap at or below roughly 30.5GB to preserve compressed ordinary object pointers.
- Leave enough memory for OS page cache, which caches Lucene segment files.

### 8.4 Hot/Warm/Cold Architecture and ILM

```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50gb"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "90d",
        "actions": {
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

### 8.5 Write Optimization

```json
PUT /my_index/_settings
{
  "index.refresh_interval": "30s",
  "index.number_of_replicas": 0,
  "index.translog.durability": "async",
  "index.translog.flush_threshold_size": "1gb"
}
```

After bulk loading, restore normal settings:

```json
PUT /my_index/_settings
{
  "index.refresh_interval": "1s",
  "index.number_of_replicas": 1,
  "index.translog.durability": "request"
}
```

Bulk write tuning should always be paired with backpressure, retry with exponential backoff, and monitoring of rejected thread-pool tasks.

---

## 9. Solution Comparison

### 9.1 Search Engine Comparison

| Dimension | Elasticsearch | Solr | MeiliSearch | Typesense |
|-----------|---------------|------|-------------|-----------|
| **Positioning** | General search + analytics | Enterprise search | Instant search | Instant search |
| **Foundation** | Lucene | Lucene | Rust implementation | C++ implementation |
| **Deployment complexity** | High | High | Very low | Low |
| **Freshness** | Near real-time | Near real-time | Real-time | Real-time |
| **Full-text search** | Excellent | Excellent | Good | Good |
| **Aggregations** | Excellent | Good | Limited | Limited |
| **Scalability** | Horizontal | Horizontal | Mostly single-node | Clustered |
| **Data scale** | PB-level | PB-level | GB-TB | GB-TB |
| **Chinese support** | IK/HanLP | IK/jieba | Built-in | Limited |
| **Best for** | Logs, search, APM, analytics | Enterprise content | Site search, e-commerce | Site search |
| **License** | SSPL/ELv2 | Apache 2.0 | MIT | GPL-3 |

### 9.2 Selection Guide

- **Large-scale logs, APM, SIEM** -> Elasticsearch or OpenSearch.
- **Small to medium e-commerce or content search** -> MeiliSearch for ease of use.
- **Fully open-source requirement** -> OpenSearch.
- **Embedded search in Go applications** -> Bleve.

---

## 10. Go Integration Practice

### 10.1 Client Initialization

```go
package es

import (
    "context"
    "log"
    "time"

    "github.com/olivere/elastic/v7"
)

type ESClient struct {
    client *elastic.Client
}

func NewESClient(urls []string) (*ESClient, error) {
    client, err := elastic.NewClient(
        elastic.SetURL(urls...),
        elastic.SetSniff(false), // disable sniffing in containerized environments
        elastic.SetHealthcheck(true),
        elastic.SetHealthcheckInterval(30*time.Second),
        elastic.SetRetrier(elastic.NewBackoffRetrier(
            elastic.NewExponentialBackoff(100*time.Millisecond, 5*time.Second),
        )),
        elastic.SetGzip(true),
    )
    if err != nil {
        return nil, err
    }
    return &ESClient{client: client}, nil
}
```

### 10.2 Bulk Write with BulkProcessor

```go
func (c *ESClient) NewBulkProcessor(ctx context.Context) (*elastic.BulkProcessor, error) {
    processor, err := c.client.BulkProcessor().
        Name("background-worker").
        Workers(4).
        BulkActions(1000).
        BulkSize(5 << 20).
        FlushInterval(time.Second).
        Stats(true).
        After(func(executionId int64, requests []elastic.BulkableRequest, response *elastic.BulkResponse, err error) {
            if err != nil {
                log.Printf("Bulk error: %v", err)
            }
            if response != nil && response.Errors {
                for _, item := range response.Failed() {
                    log.Printf("Failed: index=%s id=%s error=%s",
                        item.Index, item.Id, item.Error.Reason)
                }
            }
        }).
        Do(ctx)
    return processor, err
}

func (c *ESClient) IndexProduct(processor *elastic.BulkProcessor, product *Product) {
    req := elastic.NewBulkIndexRequest().
        Index("products").
        Id(product.ID).
        Doc(product)
    processor.Add(req)
}
```

### 10.3 Search Wrapper

```go
type SearchRequest struct {
    Keyword  string
    Category string
    MinPrice float64
    MaxPrice float64
    Brand    []string
    Page     int
    Size     int
    SortBy   string
}

func (c *ESClient) SearchProducts(ctx context.Context, req *SearchRequest) (*SearchResult, error) {
    boolQuery := elastic.NewBoolQuery()

    if req.Keyword != "" {
        boolQuery.Must(
            elastic.NewMultiMatchQuery(req.Keyword, "title^3", "description", "tags").
                Type("best_fields").
                MinimumShouldMatch("75%"),
        )
    }
    if req.Category != "" {
        boolQuery.Filter(elastic.NewTermQuery("category", req.Category))
    }
    if req.MinPrice > 0 || req.MaxPrice > 0 {
        rangeQuery := elastic.NewRangeQuery("price")
        if req.MinPrice > 0 {
            rangeQuery.Gte(req.MinPrice)
        }
        if req.MaxPrice > 0 {
            rangeQuery.Lte(req.MaxPrice)
        }
        boolQuery.Filter(rangeQuery)
    }

    result, err := c.client.Search().
        Index("products").
        Query(boolQuery).
        From((req.Page - 1) * req.Size).
        Size(req.Size).
        Aggregation("brands", elastic.NewTermsAggregation().Field("brand").Size(20)).
        Do(ctx)
    if err != nil {
        return nil, err
    }

    return parseSearchResult(result), nil
}
```

For production systems, prefer `search_after` for deep pagination and use application-level request validation to limit query cost.

### 10.4 Suggest

```go
func (c *ESClient) Suggest(ctx context.Context, prefix string) ([]string, error) {
    suggest := elastic.NewCompletionSuggester("product-suggest").
        Text(prefix).
        Field("suggest").
        Size(10).
        SkipDuplicates(true)

    result, err := c.client.Search().
        Index("products").
        Suggester(suggest).
        Do(ctx)
    if err != nil {
        return nil, err
    }

    suggestions := make([]string, 0)
    if s, ok := result.Suggest["product-suggest"]; ok {
        for _, entry := range s {
            for _, option := range entry.Options {
                suggestions = append(suggestions, option.Text)
            }
        }
    }
    return suggestions, nil
}
```

---

## 11. Practical Case

### Designing an E-Commerce Product Search System

**Requirements**:

- Tens of millions of products.
- Full-text search, filtering, and aggregation.
- Search response under 100ms.
- Autocomplete and search suggestions.

**Architecture**:

```text
┌────────────┐    ┌──────────────┐    ┌────────┐    ┌─────────────┐
│ Product DB │ -> │ Canal/Debezium│ -> │ Kafka  │ -> │ ES Consumer │
│  (MySQL)   │    │ Binlog CDC    │    │        │    │ sync to ES  │
└────────────┘    └──────────────┘    └────────┘    └──────┬──────┘
                                                            │
┌────────────┐    ┌──────────────┐                    ┌──────▼──────┐
│ API Gateway│ <- │ Search Service│ <----------------> │ ES Cluster  │
└────────────┘    │    (Go)       │                    │ 3M + 6Data  │
                  └──────────────┘                    └─────────────┘
```

**Mapping design**:

```json
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "product_synonym"]
        },
        "product_search_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": ["lowercase", "product_synonym"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_id": { "type": "keyword" },
      "title": {
        "type": "text",
        "analyzer": "product_analyzer",
        "search_analyzer": "product_search_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "category_path": { "type": "keyword" },
      "brand": { "type": "keyword" },
      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "sales": { "type": "integer" },
      "rating": { "type": "float" },
      "status": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "specs": {
        "type": "nested",
        "properties": {
          "name": { "type": "keyword" },
          "value": { "type": "keyword" }
        }
      },
      "suggest": {
        "type": "completion",
        "analyzer": "product_analyzer"
      }
    }
  }
}
```

**Data synchronization consumer**:

```go
func (c *Consumer) HandleMessage(msg *kafka.Message) error {
    var event ProductEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return err
    }

    switch event.Type {
    case "INSERT", "UPDATE":
        doc := c.buildESDoc(event.Product)
        c.bulkProcessor.Add(
            elastic.NewBulkIndexRequest().
                Index("products").
                Id(event.Product.ID).
                Doc(doc),
        )
    case "DELETE":
        c.bulkProcessor.Add(
            elastic.NewBulkDeleteRequest().
                Index("products").
                Id(event.Product.ID),
        )
    }
    return nil
}
```

Production considerations:

- Binlog CDC should preserve order for the same product ID.
- The ES consumer must handle retries and idempotent upserts.
- Search fallback can use MySQL for critical operations when ES is unavailable.
- Periodic reconciliation should compare MySQL and ES to repair drift.
- Search relevance should be evaluated with labeled data, not only by subjective examples.

---

## 12. Interview Self-Check

### Search Governance Notes

- Elasticsearch is not a silver bullet. Complex aggregations can exhaust heap, frequent refreshes increase segment pressure, and deep pagination has a natural cost ceiling.
- Index design is the foundation of performance. Mapping, analyzer, routing, shard count, and field types must be decided carefully before production traffic.
- Monitoring comes first: CPU, heap, disk, GC, search latency, indexing latency, thread-pool rejection, and circuit breakers all need alerts.

### Quick Questions

### Q1: What is the difference between `text` and `keyword`?

**Answer:** `text` is analyzed and is suitable for full-text search, such as titles and descriptions. `keyword` is not analyzed and is suitable for exact match, aggregation, sorting, IDs, status, and brand fields. A common mistake is aggregating on a `text` field instead of a `keyword` subfield.

### Q2: How does near real-time search work in Elasticsearch?

**Answer:** Writes first enter an in-memory buffer and translog. A refresh, by default every second, creates a searchable segment from the buffer. This delay is why ES is near real-time rather than strictly real-time. Frequent manual refreshes create many small segments and hurt performance.

### Q3: Why cannot the number of primary shards be changed after index creation?

**Answer:** Document routing depends on `hash(routing) % primary_shards`. Changing the number of primary shards would change the target shard for existing documents, making data lookup inconsistent. The usual solution is to create a new index and reindex.

### Q4: What do yellow and red cluster health mean?

**Answer:** Yellow means all primary shards are available but some replicas are unassigned, so data is available but redundancy is reduced. Red means some primary shards are unassigned, so data may be unavailable and the issue is urgent.

### Q5: What is an inverted index?

**Answer:** A forward index maps documents to terms, while an inverted index maps terms to documents. Search engines use inverted indexes to find matching documents quickly without scanning all documents.

### Q6: What is the role of FST in Elasticsearch?

**Answer:** FST is used in the term index to locate blocks in the term dictionary efficiently. Compared with a hash map, it compresses common prefixes and uses much less memory, which is critical when the term dictionary is huge.

### Q7: How do you solve deep pagination?

**Answer:** Avoid `from + size` for deep pages because each shard must return many candidates and the coordinating node must merge them. Use `search_after` for user pagination, scroll for batch export, or PIT plus `search_after` for consistent deep pagination.

### Q8: What is the role of Translog? How is it different from MySQL redo log?

**Answer:** Translog is Elasticsearch's write-ahead log. It protects writes before they are refreshed into searchable segments and committed. MySQL redo log records physical page changes, while Translog records operations. Flush clears Translog after segments are committed.

### Q9: How would you design hot/warm/cold architecture?

**Answer:** Use node roles such as `data_hot`, `data_warm`, and `data_cold` with ILM policies. New data is written to hot nodes on SSD. Older data is moved to warm nodes with segment merge and possibly fewer shards. Cold data is stored on cheaper storage and usually queried less frequently.

### Q10: How do you handle high memory usage from aggregations?

**Answer:** Use `doc_values` instead of fielddata, avoid aggregating on `text` fields, limit terms aggregation size, use composite aggregation for pagination, tune circuit breakers, and pre-aggregate data at write time when possible.

### Deep-Dive Questions

### Q11: Explain the full lookup path from a query term to matching documents.

**Answer:** For a term such as `elasticsearch`, ES first uses the in-memory FST term index to locate the target block in the term dictionary. It then reads the block and finds the exact term. From the term metadata, it locates the posting list, reads the compressed DocID list, and uses skip lists to intersect or union posting lists for boolean queries. Finally, it fetches stored fields or `_source` for the selected DocIDs.

### Q12: Why does Elasticsearch limit default pagination to the first 10,000 results?

**Answer:** With `from + size`, each shard returns `from + size` candidates, and the coordinating node merges all candidates in memory. If there are 5 shards and `from=9990,size=10`, the coordinator may merge 50,000 hits to return only 10. The `index.max_result_window=10000` default protects the cluster from excessive memory and CPU cost.

### Q13: How does Segment Merge maintain service availability? Why are deleted documents not removed immediately?

**Answer:** Merge runs in the background. It creates a new segment, copies live documents, skips deleted documents, and atomically switches references when ready. Deleted documents are initially marked in `.del` files because segments are immutable. Physical deletion happens during merge.

### Q14: How would you design a product search system for hundreds of millions of products with P99 under 100ms?

**Answer:** I would separate hot and long-tail products, use category-based routing where possible, optimize mappings and analyzers, put filters in filter context, return only necessary fields, cache hot queries in Redis, precompute ranking signals, and use dedicated coordinating nodes. I would also monitor P99 latency, heap, GC, rejection, and query cost, and evaluate relevance with labeled data.

### Q15: How do you keep Elasticsearch eventually consistent with MySQL?

**Answer:** A common pipeline is MySQL Binlog -> Canal/Debezium -> Kafka -> ES Consumer. The consumer performs idempotent upserts or deletes. Search use cases usually accept seconds of delay; critical reads can fall back to MySQL. Periodic reconciliation compares MySQL and ES and repairs drift. Version fields or sequence checks prevent stale updates from overwriting newer data.

### Q16: What is Elasticsearch's distributed consistency model?

**Answer:** ES uses a primary-replica model. Writes go to the primary shard and are replicated to replicas. `wait_for_active_shards` controls how many shard copies must be active. ES does not provide general multi-document ACID transactions. Single-document operations are atomic, while search visibility is near real-time and eventually consistent after refresh.

### Q17: What metrics should be monitored in an ES cluster?

**Answer:** Monitor cluster health, pending tasks, active shard percentage, JVM heap usage, GC frequency and duration, CPU, disk watermarks, indexing rate, search rate, P99 latency, refresh and merge time, search/write/bulk thread-pool rejections, and circuit breaker trips.

### Q18: How do you handle shard imbalance or hot shards?

**Answer:** First identify hot shards using `_cat/shards`, node stats, and query/indexing metrics. Then adjust routing, avoid low-cardinality or hot-key routing, split hot data into a dedicated index, increase shards before data growth if necessary, cache hot query results, and distribute requests using coordinating nodes and `_preference` carefully.

### Q19: What are Scroll API and Search After used for?

**Answer:** Scroll creates a point-in-time snapshot and retrieves data in batches, which is useful for exports and migrations but consumes search context resources. `search_after` uses the last sort values from the previous page to fetch the next page and is better for user-facing deep pagination. PIT plus `search_after` is the modern choice for consistent pagination.

### Q20: What is the difference between `nested` and `object`?

**Answer:** `object` fields are flattened, so relationships between fields inside the same array element can be lost. `nested` stores each nested object as a hidden Lucene document, preserving field relationships. The cost is more Lucene documents, slower joins/aggregations, and full document reindexing when nested data changes.

### Q21: How would you implement personalized ranking?

**Answer:** Start with full-text recall and base relevance, then use `function_score` to combine sales, rating, freshness, or preference scores. For more advanced systems, precompute segment-level ranking features, use rescore or LTR for top candidates, or let ES handle recall and coarse ranking while a separate ranking service performs model-based reranking.

### Q22: How do you tune BM25's `k1` and `b`?

**Answer:** `k1` controls term-frequency saturation. Lower values are often better for short fields like product titles. `b` controls length normalization. Lower `b` reduces the penalty for longer documents. Tuning should be based on labeled relevance data and metrics such as NDCG, not just manual examples.

### Q23: What are best practices for scaling an ES cluster?

**Answer:** For scale-out, add nodes carefully and watch shard relocation cost. For scale-in, exclude a node from allocation, wait for shards to move away, and then remove it. For rolling restarts, restart one node at a time and temporarily reduce unnecessary allocation movement. Dedicated master nodes should be handled especially carefully.

### Q24: How would you design multi-tenant search?

**Answer:** There are three common options: index-level isolation, routing-level isolation, and filter-level isolation. Index-level isolation gives the strongest boundary but may create too many indices. Routing by `tenant_id` reduces index count but risks uneven data distribution. Filter-level isolation is simplest but one large tenant can affect others. The choice depends on tenant size, compliance, and operational cost.

### Q25: How do you troubleshoot ES write rejection?

**Answer:** First check thread-pool rejection metrics, especially write, bulk, and search pools. Write rejection often means disk IO, merge pressure, or too-large bulk requests. Short-term mitigations include client-side throttling and backoff. Long-term fixes include optimizing bulk size, lowering refresh frequency during ingestion, adding data nodes, improving disk, and monitoring rejection count as an overload signal.

### Q26: How would you plan capacity for a production ES cluster?

**Answer:** Estimate total storage as document size times document count times replica factor times index expansion factor. Then choose shard count based on a target shard size of roughly 30-50GB. Plan data nodes by storage, QPS, heap, disk watermarks, and shard count. A common topology uses 3 dedicated masters, N data nodes, and coordinating nodes for high query concurrency.

### Q27: Why can terms aggregation be inaccurate in distributed ES?

**Answer:** Each shard computes local top terms and the coordinating node merges them. A globally important term may be missing from a shard's local top N and therefore be undercounted. Increasing `size` and `shard_size` improves accuracy at the cost of memory. For paginated bucket retrieval, use composite aggregation.

### Q28: What operations commonly trigger long GC pauses?

**Answer:** Common causes include enabling fielddata on `text` fields for aggregation, huge terms aggregations, parent-child or heavy nested joins, deep pagination, and queries that load large structures into heap. Fixes include using `keyword` with doc values, composite aggregation, circuit breakers, reasonable heap sizing, and query cost controls.

### Q29: When do you need Reindex? What are the best practices?

**Answer:** Reindex is needed when changing mappings, shard count, analyzers, or moving data. Best practices include creating a new index, disabling refresh and replicas during bulk load if safe, using sliced reindex for parallelism, monitoring throughput and failures, and using aliases for zero-downtime cutover.

### Q30: How does Elasticsearch support hybrid search with vectors?

**Answer:** ES 8.x supports `dense_vector` and kNN search. A document can store both text fields for inverted-index search and embeddings for vector recall. Hybrid search combines keyword and vector results using linear weighting or RRF. Vector indexes have high memory cost, so very large-scale vector recall may be better handled by Milvus, Qdrant, or another vector database, with ES used for keyword search and reranking.

### Open-Ended Design Questions

### D1: Design an e-commerce search system for tens of millions of products with sub-100ms response time.

**Reference approach:**

- Data pipeline: MySQL -> Binlog CDC -> Kafka -> ETL or enrichment -> ES bulk indexing.
- Index design: category-aware index strategy, aliases, reasonable shard size, Chinese analyzers, synonyms, stopwords, and `keyword` subfields.
- Query optimization: bool filter first, function score for sales/rating/freshness, `search_after` instead of deep pagination, return only required fields.
- Search experience: autocomplete, suggestions, highlighting, synonym expansion, and typo tolerance where needed.
- Operations: ILM hot/warm/cold tiers, monitoring for rejection/GC/latency, controlled rollout for analyzer changes.
- Key metrics: search P99 below 100ms, indexing delay below target, and relevance metrics such as NDCG.

### D2: The ES cluster has frequent Old GC pauses and search timeouts. How would you troubleshoot?

**Reference approach:**

- Confirm GC with `_nodes/stats` and JVM GC metrics.
- Identify heap-heavy operations: fielddata on text fields, huge terms aggregations, parent-child joins, nested explosion, or deep pagination.
- Check query logs, slow logs, rejection counts, circuit breaker trips, and heap usage by node.
- Fix field mappings, use keyword/doc values, paginate aggregations with composite, limit query cost, and tune circuit breakers.
- If workload is legitimate, add data nodes, separate coordinating nodes, or redesign heavy analytics into pre-aggregated data.
