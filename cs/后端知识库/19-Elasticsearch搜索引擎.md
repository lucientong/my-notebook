# Elasticsearch 搜索引擎

---

## 📑 目录

### 核心原理
1. [ES核心概念](#1-es核心概念)
2. [倒排索引原理](#2-倒排索引原理)
3. [分词器](#3-分词器)

### 查询与聚合
4. [查询DSL](#4-查询dsl)
5. [聚合分析](#5-聚合分析)

### 架构与性能
6. [集群架构](#6-集群架构)
7. [写入与检索流程](#7-写入与检索流程)
8. [性能优化](#8-性能优化)

### 工程实践与自查
9. [方案对比](#9-方案对比)
10. [Go集成实践](#10-go集成实践)
11. [实战案例](#11-实战案例)
12. [面试题自查](#12-面试题自查)

---

## 1. ES核心概念

### 1.1 基本术语

| ES概念 | MySQL类比 | 说明 |
|--------|----------|------|
| **Index** | Database | 文档的逻辑容器，具有统一的Mapping |
| **Document** | Row | 最小数据单元，JSON格式 |
| **Field** | Column | 文档中的字段 |
| **Mapping** | Schema | 字段类型定义和索引配置 |
| **Shard** | Partition | 索引的物理分片，一个Lucene实例 |
| **Replica** | Slave | 分片的副本，提供高可用和读扩展 |

### 1.2 索引与分片

```
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

### 1.3 文档路由

文档被分配到哪个分片由路由公式决定：

```
shard_num = hash(routing) % number_of_primary_shards
```

默认 `routing = _id`，这也是为什么**主分片数创建后不能修改**的根本原因。

### 1.4 Mapping 定义

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

**字段类型选择**：

| 类型 | 用途 | 是否分词 | 是否可聚合 |
|------|------|----------|-----------|
| `text` | 全文搜索 | ✅ | ❌（需开启fielddata） |
| `keyword` | 精确匹配/聚合/排序 | ❌ | ✅ |
| `integer/long` | 数值范围查询 | ❌ | ✅ |
| `date` | 时间范围/排序 | ❌ | ✅ |
| `nested` | 嵌套对象独立查询 | — | — |
| `object` | 扁平化对象 | — | — |

---

## 2. 倒排索引原理

### 2.1 正排索引 vs 倒排索引 ⭐⭐⭐

**正排索引**（Forward Index）：文档 → 词项
```
Doc1 → ["Elasticsearch", "是", "分布式", "搜索", "引擎"]
Doc2 → ["Elasticsearch", "支持", "全文", "搜索"]
Doc3 → ["Redis", "是", "内存", "数据库"]
```

**倒排索引**（Inverted Index）：词项 → 文档
```
"Elasticsearch" → [Doc1, Doc2]
"搜索"         → [Doc1, Doc2]
"分布式"       → [Doc1]
"Redis"        → [Doc3]
"内存"         → [Doc3]
```

### 2.2 倒排索引结构 ⭐⭐⭐

```
倒排索引 = Term Dictionary + Posting List

┌─────────────────────────────────────────────┐
│              Term Dictionary                  │
│  (所有唯一词项，按字典序排列)                   │
│                                              │
│  Term          → Posting List Pointer        │
│  ─────────────────────────────────           │
│  "database"    → ──→ [3, 7, 15, 42]        │
│  "elastic"     → ──→ [1, 2, 5, 8, 12]      │
│  "redis"       → ──→ [3, 9, 20]            │
│  "search"      → ──→ [1, 2, 4, 8]          │
│  ...                                         │
└─────────────────────────────────────────────┘

Posting List 内容:
┌──────────────────────────────────────────────┐
│  DocID │ TF(词频) │ Position │ Offset        │
│  ──────┼──────────┼──────────┼───────        │
│    1   │    2     │  [3, 15] │ [12:20,45:53] │
│    2   │    1     │  [7]     │ [28:36]       │
│    5   │    3     │  [1,8,12]│ [0:8,...]     │
└──────────────────────────────────────────────┘
```

### 2.3 Term Dictionary 与 FST ⭐⭐⭐

词项数量巨大时，如何快速定位？

**层级结构**：
```
Term Index (FST, 内存)
    ↓
Term Dictionary (磁盘, 分Block)
    ↓
Posting List (磁盘)
```

**FST（Finite State Transducer）**：
- 一种有限状态自动机，类似Trie但更紧凑
- 存储词项的**公共前缀**，大幅压缩内存
- 通过FST可以快速定位到Term Dictionary的某个Block
- 内存占用仅为完整Term Dictionary的1/10~1/20

```
FST 示例（存储 "cat"=5, "car"=3, "do"=7, "dog"=11）:

     c ─── a ─── t → output=5
      \         \
       \         r → output=3
        d ─── o → output=7
               \
                g → output=11
```

### 2.4 Posting List 压缩

**Frame of Reference (FOR) 编码**：

```
原始 Posting List: [73, 300, 302, 332, 343, 372]

Step 1: 差值编码 (Delta Encoding)
[73, 227, 2, 30, 11, 29]

Step 2: 分Block + 位压缩
Block 1: [73, 227, 2]   → max=227, 需要8bit
Block 2: [30, 11, 29]   → max=30,  需要5bit

压缩率: 原始6×32bit=192bit → 压缩后3×8+3×5=39bit
```

**Roaring Bitmaps**（用于Filter场景）：
- 将DocID空间按65536分Block
- 稠密Block用Bitmap（65536 bit = 8KB）
- 稀疏Block用排序数组（2 byte per DocID）
- 适合Bool查询的交集/并集操作

### 2.5 Skip List 跳表加速

Posting List 求交集时使用 Skip List 加速：

```
查询: "elastic" AND "search"

"elastic" posting: 1 → 2 → 5 → 8 → 12 → 15 → 20
                   ↓       ↓       ↓        ↓
Skip:              1 ────→ 5 ────→ 12 ────→ 20  (skip_interval=2)

"search" posting:  1 → 2 → 4 → 8 → 16 → 22

交集过程:
1. elastic=1, search=1 → Match! result=[1]
2. elastic=2, search=2 → Match! result=[1,2]
3. elastic=5, search=4 → search前进
4. elastic=5, search=8 → elastic用skip跳到12
5. elastic=12, search=8 → search前进到16
6. elastic=12, search=16 → elastic前进到15,再20
7. elastic=20, search=16 → search前进到22
8. elastic=20, search=22 → elastic结束

结果: [1, 2]
```

### 2.6 相关性评分 BM25

ES 5.x+ 默认使用 BM25（替代 TF-IDF）：

```
BM25(D, Q) = Σ IDF(qi) × (f(qi,D) × (k1+1)) / (f(qi,D) + k1 × (1-b+b×|D|/avgdl))

其中:
- IDF(qi) = ln((N - n(qi) + 0.5) / (n(qi) + 0.5) + 1)
- f(qi,D) = 词项qi在文档D中的词频
- |D| = 文档长度
- avgdl = 平均文档长度
- k1 = 1.2 (词频饱和参数)
- b = 0.75 (文档长度归一化参数)
```

**与 TF-IDF 的关键区别**：
- BM25 对词频有饱和度控制（k1参数），词频增长到一定程度后收益递减
- BM25 对文档长度的惩罚更合理（b参数控制）

---

## 3. 分词器

### 3.1 分词器组成 ⭐⭐

```
Analyzer = Character Filter + Tokenizer + Token Filter

输入文本 → [Character Filter] → [Tokenizer] → [Token Filter] → 输出词项
           (字符预处理)         (分词切分)     (词项后处理)
```

### 3.2 内置分词器对比

| 分词器 | 说明 | 示例输入 "The Quick-Brown FOX" |
|--------|------|-------------------------------|
| `standard` | Unicode文本分词 | [the, quick, brown, fox] |
| `simple` | 非字母字符处切分 | [the, quick, brown, fox] |
| `whitespace` | 空白字符处切分 | [The, Quick-Brown, FOX] |
| `keyword` | 不分词，整体作为一个Token | [The Quick-Brown FOX] |
| `pattern` | 正则切分 | 可自定义 |

### 3.3 中文分词方案 ⭐⭐

| 分词器 | 模式 | "中华人民共和国国歌" |
|--------|------|---------------------|
| `standard` | 单字切分 | [中,华,人,民,共,和,国,国,歌] |
| `ik_smart` | 最粗粒度 | [中华人民共和国, 国歌] |
| `ik_max_word` | 最细粒度 | [中华人民共和国,中华人民,中华,华人,人民共和国,人民,共和国,共和,国歌] |

**IK分词器配置**：

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

**同义词文件示例** (`synonyms.txt`)：
```
手机,手提电话,移动电话
笔记本,笔记本电脑,laptop
```

### 3.4 自定义分词器

```json
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_strip_filter": {
          "type": "html_strip"
        },
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

## 4. 查询DSL

### 4.1 查询分类 ⭐⭐⭐

```
Query Context  → 计算相关性评分 (_score)
Filter Context → 不计算评分，可缓存，性能更好
```

### 4.2 全文查询

**Match Query**（分词后查询）：

```json
{
  "query": {
    "match": {
      "title": {
        "query": "elasticsearch 入门教程",
        "operator": "and",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

**Multi-Match Query**（多字段查询）：

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

**Match Phrase**（短语查询，要求词序和邻近）：

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

### 4.3 精确查询

**Term Query**（不分词，精确匹配）：

```json
{
  "query": {
    "term": {
      "status": { "value": "published" }
    }
  }
}
```

**Range Query**：

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

### 4.4 Bool组合查询 ⭐⭐⭐

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

**Bool子句语义**：

| 子句 | 含义 | 影响评分 | 缓存 |
|------|------|---------|------|
| `must` | 必须匹配 | ✅ | ❌ |
| `filter` | 必须匹配 | ❌ | ✅ |
| `should` | 应该匹配 | ✅ | ❌ |
| `must_not` | 不能匹配 | ❌ | ✅ |

### 4.5 Nested查询

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

### 4.6 Function Score（自定义评分）

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "手机" } },
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

## 5. 聚合分析

### 5.1 聚合类型 ⭐⭐

| 类型 | 说明 | 常用聚合 |
|------|------|---------|
| Bucket | 分桶（类似GROUP BY） | terms, date_histogram, range, nested |
| Metric | 指标计算 | avg, sum, min, max, cardinality, percentiles |
| Pipeline | 基于其他聚合结果的二次计算 | derivative, moving_avg, bucket_sort |

### 5.2 Bucket聚合

**Terms聚合**：

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

**Date Histogram**：

```json
{
  "size": 0,
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "2025-01",
          "max": "2025-12"
        }
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

### 5.3 Metric聚合

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

### 5.4 Pipeline聚合

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

## 6. 集群架构

### 6.1 节点角色 ⭐⭐⭐

| 角色 | 配置 | 职责 | 资源需求 |
|------|------|------|---------|
| **Master** | `node.roles: [master]` | 集群状态管理、索引创建/删除、分片分配 | 低CPU低内存，高稳定性 |
| **Data** | `node.roles: [data]` | 存储数据、执行CRUD和搜索 | 高磁盘、高内存 |
| **Data Hot/Warm/Cold** | `node.roles: [data_hot]` | 冷热分层存储 | Hot高SSD，Cold大HDD |
| **Coordinating** | `node.roles: []` | 请求路由、结果聚合 | 高CPU高内存 |
| **Ingest** | `node.roles: [ingest]` | 文档预处理Pipeline | 高CPU |

### 6.2 集群健康状态

| 状态 | 含义 | 应对 |
|------|------|------|
| 🟢 Green | 所有主分片和副本都正常分配 | 正常 |
| 🟡 Yellow | 所有主分片正常，部分副本未分配 | 检查节点数量是否足够 |
| 🔴 Red | 部分主分片未分配，存在数据不可用 | 紧急！检查磁盘/内存/节点状态 |

### 6.3 分片分配策略

```json
{
  "cluster.routing.allocation.awareness.attributes": "rack,zone",
  "cluster.routing.allocation.disk.watermark.low": "85%",
  "cluster.routing.allocation.disk.watermark.high": "90%",
  "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
}
```

**分片决策因素**：
- 同一分片的Primary和Replica不能在同一节点
- 磁盘水位线限制
- 感知属性（机架/可用区）均衡分布
- 节点负载均衡（分片数量/磁盘使用）

### 6.4 脑裂防护 ⭐⭐⭐

**ES 7.x+ 自动选举**：

```yaml
# 初始 Master 候选节点列表
cluster.initial_master_nodes: ["master-1", "master-2", "master-3"]

# 最少投票节点数 = master候选节点数/2 + 1（自动计算）
# 3个master节点时，需要至少2个投票才能选举成功
```

**脑裂发生原因**：
1. 网络分区导致master与部分节点失联
2. Master负载过重导致GC暂停，其他节点认为master下线
3. 配置错误（如minimum_master_nodes设置过低）

**防护措施**：
- 专用master节点（不承担数据存储）
- 奇数个master候选节点（3或5个）
- 合理设置discovery超时时间

---

## 7. 写入与检索流程

### 7.1 写入流程 ⭐⭐⭐

```
Client → Coordinating Node → Primary Shard → Replica Shard(s)

详细步骤:
1. Client 发送写请求到任意节点（该节点成为 Coordinating Node）
2. Coordinating Node 根据 routing 计算目标 Primary Shard
3. 转发请求到 Primary Shard 所在节点
4. Primary Shard 执行写入:
   a. 写入 Translog（持久化保障）
   b. 写入 In-Memory Buffer
   c. 返回成功给 Coordinating Node（wait_for_active_shards控制）
5. Primary Shard 并行转发到所有 Replica Shard
6. Replica 完成后返回 Primary
7. Primary 返回结果给 Coordinating Node
8. Coordinating Node 返回给 Client
```

### 7.2 近实时搜索（NRT）⭐⭐⭐

```
写入 → Buffer → (Refresh 1s) → Segment(可搜索) → (Flush 30min) → 磁盘

┌──────────────────────────────────────────────────┐
│                   Lucene Shard                     │
│                                                    │
│  ┌──────────┐   Refresh(1s)    ┌──────────────┐  │
│  │ In-Memory│ ──────────────→  │ New Segment  │  │
│  │  Buffer  │                  │ (searchable) │  │
│  └──────────┘                  └──────────────┘  │
│       │                              │            │
│       │ 同时写入                      │ Flush      │
│       ▼                              ▼            │
│  ┌──────────┐                  ┌──────────────┐  │
│  │ Translog │ ─── Flush ────→ │ Disk Commit  │  │
│  │(顺序写盘) │                  │ (fsync)      │  │
│  └──────────┘                  └──────────────┘  │
└──────────────────────────────────────────────────┘
```

**关键概念**：

| 操作 | 频率 | 作用 | 数据安全 |
|------|------|------|---------|
| **Refresh** | 默认1s | Buffer→Segment，数据变为可搜索 | 不保证（断电丢失Buffer数据） |
| **Flush** | 默认30min或Translog满512MB | Translog清空，Segment持久化 | 保证（fsync到磁盘） |
| **Translog** | 每次写入 | WAL，保证Refresh前的数据不丢失 | 保证（默认每请求fsync） |

### 7.3 Segment Merge

```
小Segment不断产生 → 后台线程定期合并 → 大Segment

合并过程:
1. 选择多个小Segment
2. 将文档合并到新的大Segment
3. 删除已标记删除的文档（真正物理删除）
4. 原子性替换（新Segment可用后删除旧Segment）
```

**Merge策略配置**：

```json
{
  "index.merge.policy.max_merged_segment": "5gb",
  "index.merge.policy.segments_per_tier": 10,
  "index.merge.scheduler.max_thread_count": 1
}
```

### 7.4 检索流程（Query Then Fetch）

```
Phase 1 - Query:
┌────────┐      ┌─────────┐    ┌─────────┐
│Coord   │─────→│ Shard-0 │    │ Shard-1 │
│ Node   │─────→│         │    │         │
│        │      │ Top N   │    │ Top N   │
│        │←─────│ DocIDs  │    │ DocIDs  │
│        │←──────────────────── │         │
│        │                      └─────────┘
│ Merge  │
│ Top N  │
└────────┘

Phase 2 - Fetch:
┌────────┐      ┌─────────┐    ┌─────────┐
│Coord   │─────→│ Shard-0 │    │ Shard-1 │
│ Node   │─────→│ fetch 3 │    │ fetch 2 │
│        │←─────│ docs    │    │ docs    │
│        │←──────────────────── │         │
│ Return │                      └─────────┘
│  5 docs│
└────────┘
```

---

## 8. 性能优化

### 8.1 索引设计优化 ⭐⭐⭐

**1. 合理设置分片数**：
```
每个分片建议: 10GB~50GB
分片数 = 预估数据量 / 单分片大小
例: 1TB数据 → 20~100个分片
```

**2. 时间索引 + 别名**：
```json
// 按月创建索引
PUT logs-2025-04
{
  "aliases": {
    "logs-current": {},
    "logs-search": {}
  }
}

// 写入指向当前月
// 查询指向所有月份（通配符 logs-*）
```

**3. Mapping优化**：
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

### 8.2 查询优化

**1. 用Filter代替Query**（可缓存，不计算分数）：
```json
// ❌ 不推荐
{ "query": { "term": { "status": "active" } } }

// ✅ 推荐
{ "query": { "bool": { "filter": { "term": { "status": "active" } } } } }
```

**2. 避免深分页**：
```json
// ❌ from+size深分页（协调节点需要收集 from+size 条数据）
{ "from": 10000, "size": 10 }

// ✅ search_after（游标翻页）
{
  "size": 10,
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2025-04-20T10:00:00", "doc_12345"]
}

// ✅ scroll（大量数据导出）
POST /index/_search?scroll=5m
{ "size": 1000, "query": { "match_all": {} } }
```

**3. Routing优化**：
```json
// 按用户ID路由，同一用户数据在同一分片
PUT /orders/_doc/order_123?routing=user_456
{
  "user_id": "user_456",
  "amount": 99.9
}

// 查询时指定routing，只查一个分片
GET /orders/_search?routing=user_456
{
  "query": { "term": { "user_id": "user_456" } }
}
```

### 8.3 JVM调优

```yaml
# jvm.options
-Xms16g
-Xmx16g    # 不超过物理内存50%，不超过32GB（指针压缩阈值）

# GC策略（ES 7.x+默认G1）
-XX:+UseG1GC
-XX:G1HeapRegionSize=16m
-XX:InitiatingHeapOccupancyPercent=45
```

**内存分配原则**：
- JVM Heap ≤ 物理内存的50%（剩余给OS PageCache）
- JVM Heap ≤ 30.5GB（确保开启指针压缩 CompressedOops）
- 充足的OS PageCache = Lucene Segment文件的高速缓存

### 8.4 冷热架构 + ILM

```json
// Index Lifecycle Management Policy
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
          "set_priority": { "priority": 50 },
          "allocate": {
            "require": { "data": "warm" }
          }
        }
      },
      "cold": {
        "min_age": "90d",
        "actions": {
          "allocate": {
            "require": { "data": "cold" }
          },
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

### 8.5 写入优化

```json
// 批量写入时的临时优化
PUT /my_index/_settings
{
  "index.refresh_interval": "30s",
  "index.number_of_replicas": 0,
  "index.translog.durability": "async",
  "index.translog.flush_threshold_size": "1gb"
}

// 写入完成后恢复
PUT /my_index/_settings
{
  "index.refresh_interval": "1s",
  "index.number_of_replicas": 1,
  "index.translog.durability": "request"
}
```

---

## 9. 方案对比

### 9.1 搜索引擎对比表

| 维度 | Elasticsearch | Solr | MeiliSearch | Typesense |
|------|--------------|------|-------------|-----------|
| **定位** | 通用搜索+分析 | 企业搜索 | 即时搜索 | 即时搜索 |
| **底层** | Lucene | Lucene | 自研(Rust) | 自研(C++) |
| **部署复杂度** | 高 | 高 | 极低 | 低 |
| **实时性** | 近实时(1s) | 近实时 | 实时 | 实时 |
| **全文搜索** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **聚合分析** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| **扩展性** | 水平扩展 | 水平扩展 | 单机为主 | 集群 |
| **数据规模** | PB级 | PB级 | GB~TB | GB~TB |
| **学习曲线** | 高 | 高 | 极低 | 低 |
| **中文支持** | IK/HanLP | IK/jieba | 内置 | 有限 |
| **适用场景** | 日志/搜索/APM | 企业内容管理 | 站内搜索/电商 | 站内搜索 |
| **开源协议** | SSPL/ELv2 | Apache 2.0 | MIT | GPL-3 |

### 9.2 选型建议

- **大规模日志/APM/SIEM** → Elasticsearch（无替代）
- **电商/内容站内搜索（小规模）** → MeiliSearch（开箱即用）
- **需要完全开源** → OpenSearch（ES分支，Apache 2.0）
- **嵌入式搜索（Go应用）** → Bleve

---

## 10. Go集成实践

### 10.1 客户端初始化

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
        elastic.SetSniff(false), // 容器环境建议关闭嗅探
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

### 10.2 批量写入（BulkProcessor）

```go
func (c *ESClient) NewBulkProcessor(ctx context.Context) (*elastic.BulkProcessor, error) {
    processor, err := c.client.BulkProcessor().
        Name("background-worker").
        Workers(4).
        BulkActions(1000).           // 每1000条触发一次批量请求
        BulkSize(5 << 20).           // 每5MB触发一次
        FlushInterval(time.Second).   // 每秒触发一次
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

// 使用示例
func (c *ESClient) IndexProduct(processor *elastic.BulkProcessor, product *Product) {
    req := elastic.NewBulkIndexRequest().
        Index("products").
        Id(product.ID).
        Doc(product)
    processor.Add(req)
}
```

### 10.3 搜索封装

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

type SearchResult struct {
    Total    int64
    Products []*Product
    Aggs     map[string]interface{}
}

func (c *ESClient) SearchProducts(ctx context.Context, req *SearchRequest) (*SearchResult, error) {
    // 构建Bool查询
    boolQuery := elastic.NewBoolQuery()

    // 全文搜索
    if req.Keyword != "" {
        boolQuery.Must(
            elastic.NewMultiMatchQuery(req.Keyword, "title^3", "description", "tags").
                Type("best_fields").
                MinimumShouldMatch("75%"),
        )
    }

    // Filter条件
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
    if len(req.Brand) > 0 {
        boolQuery.Filter(elastic.NewTermsQueryFromStrings("brand", req.Brand...))
    }

    // 构建搜索
    search := c.client.Search().
        Index("products").
        Query(boolQuery).
        From((req.Page - 1) * req.Size).
        Size(req.Size).
        Highlight(elastic.NewHighlight().
            Field("title").
            PreTags("<em>").
            PostTags("</em>")).
        Aggregation("brands", elastic.NewTermsAggregation().Field("brand").Size(20)).
        Aggregation("price_ranges", elastic.NewRangeAggregation().Field("price").
            AddRange(nil, 100).
            AddRange(100, 500).
            AddRange(500, 1000).
            AddRange(1000, nil))

    // 排序
    switch req.SortBy {
    case "price_asc":
        search.Sort("price", true)
    case "price_desc":
        search.Sort("price", false)
    case "newest":
        search.Sort("created_at", false)
    default:
        search.Sort("_score", false)
    }

    // 执行
    result, err := search.Do(ctx)
    if err != nil {
        return nil, err
    }

    // 解析结果
    searchResult := &SearchResult{
        Total:    result.TotalHits(),
        Products: make([]*Product, 0, len(result.Hits.Hits)),
    }

    for _, hit := range result.Hits.Hits {
        var product Product
        if err := json.Unmarshal(hit.Source, &product); err == nil {
            if hl, ok := hit.Highlight["title"]; ok && len(hl) > 0 {
                product.HighlightTitle = hl[0]
            }
            searchResult.Products = append(searchResult.Products, &product)
        }
    }

    return searchResult, nil
}
```

### 10.4 自动补全（Suggest）

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

## 11. 实战案例

### 设计电商商品搜索系统

**需求**：
- 千万级商品数据
- 支持全文搜索、筛选、聚合
- 搜索响应 < 100ms
- 支持自动补全、搜索建议

**架构**：

```
┌─────────┐    ┌──────────┐    ┌────────────┐    ┌─────────────┐
│ 商品DB  │───→│ Canal/   │───→│ Kafka      │───→│ ES Consumer │
│ (MySQL) │    │ Debezium │    │            │    │ (同步到ES)  │
└─────────┘    └──────────┘    └────────────┘    └──────┬──────┘
                                                         │
┌─────────┐    ┌──────────┐                      ┌──────▼──────┐
│ 搜索API │←──→│ 搜索服务 │←────────────────────→│ ES Cluster  │
│ Gateway │    │ (Go)     │                      │ (3Master+   │
└─────────┘    └──────────┘                      │  6Data)     │
      ↑                                          └─────────────┘
      │
┌─────────┐
│ 前端/APP│
└─────────┘
```

**索引Mapping设计**：

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
      },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" }
    }
  }
}
```

**数据同步Consumer**：

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

---

## 12. 面试题自查

### 搜索治理补充

- ES 不是银弹：写入吞吐受限于Refresh频率，复杂聚合会打满JVM Heap，深分页有天然上限。选型时务必明确是"搜索场景"还是"分析场景"。
- 索引设计是性能的根基：Mapping一旦创建很难修改，字段类型、分词器、Routing策略都需要在设计阶段想清楚。
- 监控先行：集群CPU/Heap/磁盘/GC/查询延迟/Rejection都必须有告警，不要等到Red状态才发现问题。

### 初级高频速刷

**1. text 和 keyword 类型有什么区别？什么时候用哪个？**

text会被分词，适用于全文搜索场景（标题、描述）；keyword不分词，适用于精确匹配、聚合、排序（状态、品牌、ID）。一个常见错误是把需要聚合的字段设为text类型。

**2. ES的近实时搜索是怎么实现的？为什么不是真正的实时？**

文档写入后先进入内存Buffer，需要等Refresh（默认1秒）才会生成可搜索的Segment。这1秒的延迟就是"近实时"的来源。可以手动调用_refresh，但频繁Refresh会产生大量小Segment影响性能。

**3. 为什么ES主分片数设定后不能修改？**

因为文档路由公式是 `hash(routing) % primary_shards`，修改分片数会导致已有文档的路由结果变化，查询时找不到数据。只能通过Reindex迁移到新索引。

**4. ES集群状态Yellow和Red分别代表什么？**

Yellow表示所有主分片正常但部分副本未分配（通常是节点数不够），数据完整但没有冗余保护；Red表示部分主分片未分配，存在数据不可用。Yellow需要关注，Red需要紧急处理。

**5. 什么是倒排索引？和正排索引有什么区别？**

正排索引是文档→词项的映射，倒排索引是词项→文档的映射。搜索时通过倒排索引可以快速定位包含某个词的所有文档，而不需要逐个扫描。

### 中级高频进阶

**6. FST在ES中起什么作用？为什么不直接用HashMap？**

FST是Term Index的数据结构，常驻内存，用于快速定位Term Dictionary中的Block位置。相比HashMap，FST利用公共前缀压缩，内存占用仅为1/10~1/20。百万级词项的HashMap可能需要数GB内存，而FST只需要几百MB。

**7. 如何解决ES深分页问题？**

三种方案：(1) search_after：基于上一页最后一条的排序值翻页，适合向后翻页；(2) scroll：创建快照逐批获取，适合大量数据导出；(3) PIT + search_after：推荐的新方案，兼顾一致性和灵活性。避免使用from+size超过10000条（需要在所有分片上排序from+size条数据）。

**8. ES写入时，Translog的作用是什么？和MySQL的redo log对比？**

Translog是ES的WAL（预写日志），保证Refresh之前的数据在断电时不丢失。类似MySQL的redo log，但有区别：MySQL的redo log是物理日志记录页面变更，Translog是逻辑日志记录操作本身。Flush时清空Translog并将Segment持久化。

**9. 如何设计ES的冷热分离架构？**

使用节点角色标记（data_hot/data_warm/data_cold）+ ILM策略。新数据写入Hot节点（SSD），定时通过ILM策略迁移到Warm节点（合并Segment、降低副本），最终迁移到Cold节点（HDD，只读）。关键是ILM的rollover和allocate配置。

**10. ES聚合查询内存占用过高怎么处理？**

(1) 用doc_values代替fielddata（keyword默认开启）；(2) 限制terms聚合的size；(3) 使用composite聚合分批获取；(4) 增大circuit breaker限制防止OOM；(5) 避免在text字段上做聚合；(6) 考虑pre-aggregation（写入时预计算）。

### 深入问答

#### Q1: 详细解释ES倒排索引从词项查找到文档定位的完整过程

**答**：假设查询词"elasticsearch"：(1) 先在Term Index（FST，内存中）中进行前缀匹配，定位到Term Dictionary的某个Block偏移量；(2) 从磁盘读取该Block，在Block内二分查找定位到精确词项；(3) 从词项元数据获取Posting List的文件偏移量；(4) 读取Posting List获取DocID列表（经过FOR编码压缩）；(5) 如果是Bool查询需要对多个Posting List求交集/并集（利用Skip List加速）；(6) 根据DocID从Stored Fields获取文档内容。整个过程中FST保证了第一步的内存效率，Skip List保证了集合运算效率。

#### Q2: 为什么ES默认只能查前10000条数据？从架构层面解释

**答**：from+size的实现是：Coordinating Node要求每个Shard返回(from+size)条数据，然后在内存中全局排序取Top N。如果有5个分片，from=9990 size=10，每个分片需要返回10000条，Coordinating Node需要在内存中排序50000条数据。from越大，这个代价越高，且大部分数据最终被丢弃。所以设置`index.max_result_window=10000`作为保护。替代方案是search_after，它利用上一页的排序值作为起点，每个分片只需要返回size条数据。

#### Q3: Segment Merge过程中如何保证服务可用？为什么删除文档不是真正删除？

**答**：(1) Merge是后台异步操作，产生新Segment后通过原子操作（更新Commit Point）切换引用，旧Segment随后删除，整个过程对搜索无感知；(2) 删除文档只是在.del文件中标记（Bitmap），查询时过滤掉标记文档。这是因为Segment是不可变的（immutable），不可变带来的好处是：无需加锁、利于OS PageCache、便于压缩。真正的物理删除在Merge时完成——合并时跳过被标记的文档。

#### Q4: 如何设计一个支持亿级商品的搜索系统，保证P99 < 100ms？

**答**：(1) **索引分层**：热门商品独立索引（少量分片SSD），长尾商品按类目分索引；(2) **Routing策略**：按category路由，查询指定类目时只打一个分片；(3) **缓存三层**：请求级缓存（Redis存热门查询结果）→ES Filter Cache→OS PageCache；(4) **查询优化**：Filter前置减少评分计算、禁用_source只返回必要字段、异步获取聚合结果；(5) **数据预处理**：离线计算热度分/排序分写入ES，避免实时复杂function_score；(6) **集群拓扑**：独立Coordinating节点做负载均衡，Data节点按冷热分离。

#### Q5: ES和MySQL如何保证数据最终一致性？同步延迟怎么处理？

**答**：(1) **同步方案**：MySQL Binlog → Canal/Debezium → Kafka → ES Consumer，保证顺序性；(2) **延迟容忍**：搜索场景通常可接受秒级延迟，关键操作（如下单后立即搜索）走MySQL；(3) **补偿机制**：定时全量校验任务对比MySQL和ES数据，发现差异自动修复；(4) **版本控制**：ES文档存储version字段，Consumer处理时用if_seq_no/if_primary_term做乐观锁，避免乱序覆盖；(5) **双写降级**：ES故障时降级为MySQL查询，恢复后重新消费Kafka。

#### Q6: Elasticsearch的分布式一致性模型是什么？和传统数据库有什么区别？

**答**：ES使用Primary-Backup模型：写入只打Primary，Primary成功后并行同步到Replica，通过`wait_for_active_shards`控制一致性级别（默认1=只等Primary）。与传统数据库区别：(1) 不支持传统事务ACID，单文档操作是原子的但跨文档不保证；(2) 读一致性默认是eventual consistency，刚写入可能读不到（需等Refresh）；(3) 没有WAL replay的故障恢复（靠Translog+Peer Recovery）；(4) 脑裂保护靠Master选举而非Paxos/Raft。

#### Q7: 如何监控ES集群健康？哪些指标必须告警？

**答**：必须监控的指标：(1) **集群级**：cluster_health(red/yellow)、pending_tasks、active_shards_percent；(2) **节点级**：JVM Heap Used%(>75%告警)、GC时间和频率、CPU%、磁盘水位线；(3) **索引级**：indexing_rate、search_rate、search_latency(P99)、refresh_time、merge_time；(4) **线程池**：rejected count(search/write/bulk任何rejection都要告警)；(5) **Circuit Breaker**：tripped count(内存熔断)。推荐工具：Elasticsearch自带的_cat API + Prometheus exporter + Grafana。

#### Q8: ES如何处理热点Key导致的分片不均衡？

**答**：(1) **问题识别**：某些分片的请求量/数据量远超其他分片，通过_cat/shards查看；(2) **Routing层面**：避免高基数字段做routing（如热门商品ID），可以用 routing + index sorting 组合；(3) **索引层面**：热门数据独立索引、增加分片数分散压力；(4) **查询层面**：热门查询结果缓存到Redis，减少ES压力；(5) **Coordinating层面**：_preference参数做请求分散，避免总是打到同一个Shard。

#### Q9: Scroll API和Search After各自的适用场景和实现原理？

**答**：**Scroll**：创建一个Point-in-Time快照，每次_scroll请求返回下一批数据。优点是数据一致（快照时间点固定），缺点是占用资源（保持Search Context），不适合实时用户翻页。适合后台数据导出/迁移。**Search After**：基于上一页最后一条的排序值作为下一页起点。优点是无状态不占资源，支持实时数据变化。缺点是只能顺序翻页（不能跳页），需要全局唯一的排序字段（通常加_id作为tiebreaker）。推荐用PIT+search_after替代scroll。

#### Q10: ES中nested类型和object类型有什么区别？各自的性能影响？

**答**：**Object**：扁平化存储，内部字段的对应关系丢失。例如`{specs:[{k:"color",v:"red"},{k:"size",v:"L"}]}`会被存为`specs.k:["color","size"], specs.v:["red","L"]`，查询`color=L`会错误匹配。**Nested**：每个嵌套对象作为独立的隐藏文档（Lucene doc）存储，保持字段关系。性能影响：(1) Nested增加文档数（N个嵌套=N+1个Lucene doc），影响Heap和搜索性能；(2) 更新任何一个嵌套对象需要Reindex整个文档+所有嵌套文档；(3) Nested聚合需要join操作，比普通聚合慢。建议嵌套数量<100。

#### Q11: 如何实现搜索结果的个性化排序？

**答**：(1) **Function Score**：在基础相关性分上叠加业务分数（销量、评分、个性化偏好分），用field_value_factor/script_score；(2) **预计算方案**：离线为每个用户群体计算个性化排序分，写入ES文档，查询时按该字段排序；(3) **Rescore**：ES支持两阶段排序，第一阶段用简单算法（BM25）获取Top 500，第二阶段用复杂模型（LTR plugin）重排Top 50；(4) **外部排序**：ES只做召回和粗排，排序分由独立的Rank Service（Python模型）计算，最终在应用层合并。

#### Q12: BM25的k1和b参数如何调优？什么场景需要调？

**答**：**k1**（默认1.2）：控制词频饱和度。k1越大，高频词的得分增长越慢才饱和。短文本（商品标题）可以调低到0.5-1.0（词频重要性低），长文本（文章正文）保持默认或调高。**b**（默认0.75）：控制文档长度归一化。b=1时长文档惩罚最大，b=0时忽略文档长度。标题搜索建议b=0.3（标题长度差异小），正文搜索保持0.75。调优方法：准备标注数据集，对比不同参数下的NDCG指标，使用网格搜索找最优组合。

#### Q13: ES集群扩容和缩容的最佳实践？

**答**：**扩容**：(1) 新节点加入后自动参与分片Rebalance（可能影响性能）；(2) 建议先设`cluster.routing.allocation.enable: none`，配置好后再开启；(3) 使用`_cluster/reroute`手动迁移特定分片。**缩容**：(1) 先通过`allocation.exclude._ip`标记节点排除，等分片自动迁走；(2) 确认该节点无分片后再下线；(3) 注意Master节点缩容要先调整`voting_configuration`。**滚动重启**：一次只停一个节点，设`cluster.routing.allocation.enable: primaries`防止不必要的Rebalance，重启完恢复。

#### Q14: 如何在ES中实现多租户搜索？

**答**：三种方案：(1) **索引级隔离**：每租户独立索引（`products_tenant_A`），优点是完全隔离、性能可控、独立Mapping，缺点是索引数量膨胀、运维复杂；(2) **Routing隔离**：同一索引按tenant_id做routing，查询必须带routing，优点是索引少，缺点是数据不均衡风险；(3) **Filter隔离**：统一索引+文档级tenant_id字段+强制filter，简单但无法阻止一个大租户影响其他租户。推荐：中小租户用方案3，大租户或合规要求高的用方案1，中间地带用方案2。

#### Q15: ES写入被Reject了怎么排查和解决？

**答**：(1) **确认Rejection类型**：`_cat/thread_pool?v&h=name,active,rejected,queue`查看是write/bulk/search哪个线程池；(2) **根因分析**：Write Rejection通常是磁盘IO跟不上或Merge压力大，Search Rejection通常是查询过重或堆不够；(3) **短期应对**：适当增大队列长度（`thread_pool.write.queue_size`，默认200），但不能无限加大；(4) **长期优化**：写入侧做客户端限流+指数退避重试，优化Bulk大小（5-15MB），减少Refresh频率，升级硬件或增加Data节点；(5) **告警**：Rejection > 0 就应该告警，说明集群已经过载。

#### Q16: 如何从零搭建一个生产级ES集群？给出容量规划方法

**答**：(1) **数据评估**：单条文档大小×文档总数×(1+副本数)×1.5（索引膨胀系数）= 总存储；(2) **分片规划**：总存储/单分片推荐大小(30-50GB)= 主分片数；(3) **节点规划**：每节点建议不超过600-800个分片，JVM 16-30GB，磁盘使用率<85%；(4) **节点角色**：3个专用Master（2C4G）+ N个Data节点（按存储和QPS定）+ 2个Coordinating（按查询并发定）；(5) **网络**：万兆网，节点间延迟<1ms；(6) **最终公式**：Data节点数 = max(总存储/单节点磁盘容量, 目标QPS/单节点QPS能力)。

#### Q17: ES的聚合精度问题是什么？Terms聚合为什么会不准确？

**答**：Terms聚合在分布式环境下有精度问题：每个分片本地计算Top N，Coordinating Node合并各分片结果。如果某个Term只在少数分片中高频出现，可能在该分片的Top N中但在全局排名中被遗漏。解决：(1) 增大`size`（如要Top 10则设size=50）；(2) 增大`shard_size`（默认size*1.5+10）；(3) 对精度要求极高的场景用`"shard_size": 0`（不推荐，内存压力大）；(4) 如果字段基数低（<1000），可以收集所有Bucket。

#### Q18: 如何优化ES的GC问题？什么操作最容易触发长时间GC？

**答**：**易触发长GC的操作**：(1) text字段开启fielddata做聚合（加载到Heap）；(2) 超大Terms聚合（百万级Bucket）；(3) Parent-Child/Nested大量Join；(4) 单次查询from+size过大。**优化**：(1) 使用doc_values替代fielddata；(2) 用G1GC并设合理Heap（≤30.5GB保持指针压缩）；(3) 设Circuit Breaker防止单次请求占用过多内存；(4) 监控`jvm.gc.collectors.old.collection_time`，超过1秒告警；(5) 避免一次加载大量数据到Heap，用composite聚合分页获取。

#### Q19: ES Reindex的原理和最佳实践？什么时候需要Reindex？

**答**：**何时需要**：修改Mapping（字段类型）、修改分片数、修改分词器、数据迁移。**原理**：scroll源索引+bulk写入目标索引，本质是应用层面的数据复制。**最佳实践**：(1) 用`_reindex` API（集群内）或`remote reindex`（跨集群）；(2) 大数据量时设`"slices": "auto"`并行执行；(3) 目标索引先设`refresh_interval: -1, number_of_replicas: 0`加速写入，完成后恢复；(4) 用别名实现零停机切换：`products_v2`写完后原子切换别名；(5) 实测10亿文档reindex约2-4小时（取决于文档大小和硬件）。

#### Q20: ES如何与向量搜索结合？实现混合检索的方案？

**答**：ES 8.x+原生支持`dense_vector`字段和kNN搜索。**混合检索方案**：(1) 文档同时存储text字段（倒排索引）和embedding字段（HNSW向量索引）；(2) 查询时用`knn`子句做向量召回+`query`子句做关键词召回；(3) 通过RRF（Reciprocal Rank Fusion）或线性加权融合两路结果。**配置示例**：`"embedding": {"type": "dense_vector", "dims": 768, "index": true, "similarity": "cosine"}`。**注意**：向量索引内存开销大（每文档dims×4字节），大规模场景考虑用专用向量库（Milvus/Qdrant）做向量召回，ES只做精排。

### 开放式设计题

**D1：设计一个支持千万级商品、百毫秒响应的电商搜索系统，从数据同步到查询优化的全链路你会怎么做？**

**参考思路**：
- 数据链路：MySQL → Canal监听Binlog → Kafka缓冲 → Flink ETL（关联多表拼宽表）→ ES批量写入
- 索引设计：按类目分索引+别名聚合查询、合理设置分片数（单分片30-50GB）、IK分词+同义词+停用词
- 查询优化：Bool Query的Filter前置（利用Cache）、Function Score融合销量/评分/个性化分、search_after替代深分页
- 搜索体验：Suggest自动补全、Did-You-Mean纠错、同义词扩展、高亮标红
- 运维：冷热分离ILM、集群监控（Rejection/GC/延迟）、灰度上线新分词器
- 关键指标：搜索P99<100ms、索引延迟<5s、搜索相关性(NDCG)>0.7

**D2：ES集群出现频繁的GC停顿（Old GC每分钟多次），导致搜索超时，如何排查？**

**参考思路**：
- 确认GC状况：GET _nodes/stats → jvm.gc段 → old_gc.count和old_gc.time
- 内存分析：哪些操作占用最多Heap → 通常是fielddata（text字段聚合）或大Terms聚合
- 常见根因：text字段开启fielddata做聚合（改用keyword子字段）、单次聚合Bucket过多、Parent-Child关系占用、搜索并发过高
- 优化：关闭不必要的fielddata、聚合用composite分页、Circuit Breaker设合理阈值、升级到G1GC、考虑增加节点分散数据
