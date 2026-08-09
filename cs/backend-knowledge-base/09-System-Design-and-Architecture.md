# System Design and Architecture

Language: English | [中文](../后端知识库/09-系统设计与架构.md)

> **Interview / Production**: clarify constraints before naming components. This doc keeps the **system-design selection** view. Gateway plugins, Sentinel/Hystrix details, and microservice implementation of rate limit / circuit break / distributed transactions → [12-Microservices-Architecture](./12-Microservices-Architecture.md).

---

## Table of Contents

### Methodology
0. [System Design Method & Back-of-envelope](#0-system-design-method--back-of-envelope)

### Stability & Resilience
1. [Rate Limiting](#1-rate-limiting) (selection; details in *12*)
2. [Circuit Breaking & Degradation](#2-circuit-breaking--degradation) (selection; details in *12*)

### Distributed Consistency
3. [Distributed Transactions](#3-distributed-transactions) (selection table; details in *12*)
4. [Distributed Consistency](#4-distributed-consistency)
5. [High Availability](#5-high-availability) (SLO / multi-active)

### Scale Capabilities
6. [Idempotency](#6-idempotency)
7. [Multi-level Cache & Penetration / Breakdown / Avalanche](#7-multi-level-cache--penetration--breakdown--avalanche)
8. [Message Queue Selection](#8-message-queue-selection-kafka--rabbitmq--rocketmq)
9. [Read/Write Scale & Sharding](#9-readwrite-scale--sharding) (DB detail → *10*)

### Component Deep Dive
10. [Distributed ID (Snowflake / Segment)](#10-distributed-id-snowflake--segment)
11. [Distributed Lock & RedLock](#11-distributed-lock--redlock)
12. [Microservices Overview](#12-microservices-overview)

### Cases & Self-Check
13. [System Design Cases](#13-system-design-cases)
14. [Interview Self-Check](#14-interview-self-check)
15. [Summary](#summary)

---

## 0. System Design Method & Back-of-envelope

Walk a fixed cadence shallow → deep. Do not dump a component zoo first.

### 0.1 Seven-step cadence

| Step | Do | Output |
|------|----|--------|
| **1. Clarify** | Functional / non-functional: DAU, peak QPS, R/W ratio, consistency, latency, compliance | Boundaries & trade-offs |
| **2. Estimate** | Back-of-envelope: QPS, storage, bandwidth, machine order | Order of magnitude (re-estimate if off by 10×) |
| **3. API** | Core endpoints, error codes, idempotency keys | Contract |
| **4. Data** | Entities, PKs, indexes, shard keys, cache keys | Data model |
| **5. Design** | Single box → stateful split → async → multi-tier cache | One architecture you can narrate |
| **6. Deep Dive** | Hot keys, consistency, failure, pagination, TTL | Where interviewers dig |
| **7. Scale** | Sharding, multi-active, degradation, capacity & observability | Evolution path |

### 0.2 Estimation cheat sheet (orders of magnitude)

| Quantity | Approx | Note |
|----------|--------|------|
| Seconds / day | ≈ 8.64×10⁴ ≈ **10⁵** | QPS ≈ daily requests / 10⁵ |
| Seconds / year | ≈ 3.15×10⁷ ≈ **π×10⁷** | |
| Same-DC RTT | 0.5–2 ms | |
| Cross-region | 20–200 ms+ | Budget for multi-active |
| Sequential disk | 100–500 MB/s class | Random IOPS separate |
| 1 GbE | ≈ 125 MB/s | Do not only count CPU |
| 1 char | ≈ 1–4 B (encoding) | URL / text estimates |
| 6-char Base62 | 62⁶ ≈ **56.8B** | Collision + capacity together |

**Example**: 100M daily PV → avg QPS ≈ 1000; peak/avg 10× → peak ≈ 10k QPS. Estimate R/W and hotspots before deciding on shard / CDN / queue.

### 0.3 Answer template (open mouth ready)

```
1) Goals & constraints (consistency / latency / cost)
2) Capacity estimate (QPS, storage, bandwidth)
3) High-level architecture (edge → service → store → async)
4) Deep dive on 1–2 hardest paths
5) Failure & degradation (dependency down?)
6) Evolution (MVP → multi-active)
```

---

## 1. Rate Limiting

> **Split with *12***: this chapter = algorithm selection & distributed quota for system design. Gateway plugins / Sentinel detail → [12 § resilience / rate limit](./12-Microservices-Architecture.md).

### 1.1 Why rate limit?

**Scenes**: flash sale / hot events; abuse (DDoS, crawlers); protect downstream from cascade.

**Without it**: CPU/mem exhaustion, DB pool exhaustion, whole-system collapse.

### 1.2 Common algorithms

#### Fixed window counter

Count requests in a fixed window. Simple; **boundary burst** problem (99 near end of window1 + 99 at start of window2 ≈ 2× limit in one second).

#### Sliding window

Split the window into smaller buckets; count across the sliding range. Fixes boundary burst; more memory.

#### Token bucket (recommended)

Tokens refill at a fixed rate; requests consume tokens; excess tokens discarded when full. Allows controlled **burst**. Go: `golang.org/x/time/rate`.

**Hand-written token bucket — must use float64** ⭐⭐⭐

```go
// tokens/rate MUST be float64. int64(elapsed.Seconds()*rate) truncates
// sub-second elapsed to 0 → almost no refill under high QPS.
type TokenBucket struct {
    capacity float64
    tokens   float64
    rate     float64 // token/s
    lastTime time.Time
    mu       sync.Mutex
}

func (tb *TokenBucket) AllowN(n float64) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    now := time.Now()
    elapsed := now.Sub(tb.lastTime).Seconds()
    tb.tokens += elapsed * tb.rate // float accumulate — no integer truncation
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    tb.lastTime = now
    if tb.tokens >= n {
        tb.tokens -= n
        return true
    }
    return false
}
```

**Keys**: lazy refill (no ticker); float math; capacity = burst.

#### Token bucket vs leaky bucket

| | Token bucket | Leaky bucket |
|--|--------------|--------------|
| Idea | Tokens arrive at fixed rate; requests consume variably | Requests enter variably; leave at fixed rate |
| Burst | ✅ allowed (spend stored tokens) | ❌ smoothed (queue + fixed drain) |
| Curve | jagged peaks OK | constant outflow |
| Typical | API gateway | DB write protection / MQ consume |

#### Leaky bucket

Queue + fixed drain. Protects downstream smoothness; increases wait under burst.

### 1.3 Distributed rate limiting

**Shared quota across instances.**

**Option 1: Redis + Lua fixed window** (simple)

> ⚠️ The script below is a **fixed-window counter** (`INCR` + `EXPIRE`), **not** a token bucket.
> Token bucket → case “Distributed Rate Limiter” with `HMSET tokens/last_refill` Lua.

```lua
-- Fixed window: KEYS[1]=key  ARGV[1]=limit  ARGV[2]=window_seconds
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end
if current > limit then return 0 else return 1 end
```

| Redis shape | Essence | Use |
|-------------|---------|-----|
| `INCR`+`EXPIRE` | Fixed window | Cheap quota (boundary spike) |
| ZSET timestamps | Sliding window | Precise user/IP limits |
| Hash(`tokens`,`last_refill`) | Token bucket | API burst |

**Option 2: Nginx** `limit_req_zone` + `burst=... nodelay`.

Trade-off: strong global accuracy costs latency/availability; local limiting is fast but approximate. Fail-open vs fail-close is a product decision (limiters often fail-open + local fallback).

---

## 2. Circuit Breaking & Degradation

> **Split with *12***: state machine + degradation selection here; Hystrix/Sentinel detail → [12](./12-Microservices-Architecture.md).

### 2.1 Circuit breaker

```
Closed → Open → Half-Open → Closed
```

- **Closed**: pass traffic; track failure ratio.
- **Open**: fail fast when threshold exceeded.
- **Half-Open**: allow a few probes; success → Closed, failure → Open.

Metrics: error rate, timeout rate, slow-call rate, volume. Library example: `sony/gobreaker`.

### 2.2 Degradation

Keep the critical path by shedding non-essential work: hide recommendations, serve cache, async writes, default payloads. Prefer config-center feature switches. Good degradation is **product-aware**, decided before the incident.

---

## 3. Distributed Transactions

> **Split with *12***: selection table here; 2PC/TCC/Saga/outbox implementation → [12 Distributed Transactions](./12-Microservices-Architecture.md).

### 3.1 2PC

Prepare → commit/rollback. Sync blocking, coordinator SPOF, partition pain. **Not recommended** for most internet backends.

### 3.2 Comparison

| Approach | Idea | Pros | Cons | When |
|----------|------|------|------|------|
| **2PC** | Coordinator two phases | Strong consistency | Blocking, SPOF | ❌ avoid |
| **TCC** | Try reserve → Confirm → Cancel | No long DB locks; **business-level reservation** | Invasive (3 APIs); Confirm/Cancel need idempotency & empty-rollback | Fund freeze / explicit hold |
| **Saga** | Local txs + compensations | Long workflows | Eventual; complex compensations | Order pipelines |
| **Local message / outbox** | Same local tx writes business + outbox | Reliable, practical | Scan/relay latency | ✅ default recommendation |

### 3.3 TCC core

```
Try (business-level strong isolation / reserve)
  → Confirm (commit)
  → Cancel (release)

Transfer example:
Try:     freeze payer funds
Confirm: deduct frozen; credit payee
Cancel:  unfreeze payer
```

**Clarify**: TCC is **business-layer resource reservation & isolation**, not “DB linearizability.” Confirm/Cancel still need idempotency, empty rollback, and anti-hanging.

### 3.4 Saga

Orchestration (central coordinator) vs choreography (events). Eventual consistency + compensations.

### 3.5 Local message table (recommended)

Local DB transaction writes business row + outbox → async relay → idempotent consumer. Avoids XA while keeping reliable publish. See *12* Outbox/Inbox.

---

## 4. Distributed Consistency

### 4.1 CAP

| Letter | Engineering meaning (Brewer / Gilbert–Lynch) |
|--------|-----------------------------------------------|
| **C**onsistency | **Linearizability**: after a successful write, any subsequent read on a non-failed node sees that write (or a newer value) |
| **A**vailability | **Every non-failed node that receives a request returns a non-error response** (need not be the latest value) |
| **P**artition tolerance | System continues under message loss/delay between nodes. Partition is reality — **P ≠ “still available”** |

**Correct statement**: distributed systems must face partitions (**P**). **Under partition, trade C vs A** — not a casual “pick any two of three.”

Under partition:

- **CP**: refuse possibly inconsistent ops (ZooKeeper, etcd, HBase).
- **AP**: non-failed nodes keep answering; accept staleness / eventual consistency (Cassandra, Dynamo-style defaults).
- **CA**: only meaningful if you assume no partition. In WAN / multi-node reality, **CA is not an architecture tag you choose**.

#### Why C + A + P cannot hold together ⭐⭐⭐

Partition N1↔N2. Client writes v1 to N1. Client reads N2:

1. Keep linearizability (**C**) + tolerate partition (**P**) → N2 must refuse → sacrifice **A**.
2. Keep availability (**A**) + tolerate partition (**P**) → N2 returns v0 → sacrifice **C**.

Colloquial “choose two” is shorthand; say **“C/A trade-off under partition.”**

#### Cassandra CAP label (do not mis-tag)

Cassandra’s product posture is **AP + tunable consistency**. `W+R>N` gives quorum read-your-writes on a healthy cluster; raising W/R **reduces availability under partition**. **Do not** relabel the whole system “CP” because of one W/R setting (e.g. `W=3,R=1`).

#### CP vs AP selection cheat

| Scene | Lean | Note |
|-------|------|------|
| Ledger / config center | CP | Wrong value unacceptable |
| Likes / counters | AP | Brief inaccuracy OK |
| Cart | AP | UX; merge conflicts |
| **Flash-sale stock** | **Business correctness ≠ CAP-CP** | Redis atomic deduct + MQ eventual DB; not a ZooKeeper-style CP cluster for the whole path |
| DNS | AP | Prefer availability |

### 4.2 BASE (relative to ACID)

BASE is **not** “CAP extended.” It is a large-scale internet engineering philosophy **relative to ACID**: trade immediate consistency for availability and partition-friendly scale within acceptable bounds.

| | Meaning |
|--|---------|
| **B**asically Available | Degrade / partial outage over total blackout |
| **S**oft State | Intermediate states OK (“paying”, “stock held”) |
| **E**ventually Consistent | Replicas converge after writes stop |

Vs CAP: BASE often lands on **AP + compensation/reconciliation**; critical paths can still be CP.

### 4.3 Consistent hashing

Hash ring + nodes + clockwise ownership. Node join/leave remaps only neighbors. **Virtual nodes** reduce imbalance.

### 4.4 Raft ⭐⭐⭐

Roles: Leader (sole write entry), Follower, Candidate.

**Election**: election timeout → Candidate, term++, RequestVote; majority → Leader; randomized timeouts avoid split votes; vote only if candidate log is at least as up-to-date.

**Log replication**: client → Leader append → parallel AppendEntries → majority ack → commit → reply; followers catch up; log matching on (index, term).

Used by etcd / Consul / TiKV. Easier story than classic multi-proposer Paxos.

---

## 5. High Availability

### 5.1 Failure-domain isolation

Contain blast radius: process / host / AZ / region / dependency / tenant. Thread-pool / bulkhead isolation so payment does not starve on notify pool exhaustion.

### 5.2 Timeout & retry

Timeouts at every hop. Retry only idempotent ops (or with idempotency key). Exponential backoff + jitter; retry budget; circuit break. Bad retries amplify outages.

### 5.3 SLO, availability, “nines”

| Availability | ~ yearly downtime | Engineering read |
|--------------|-------------------|------------------|
| 99% (2 nines) | ~3.65 days | Internal tools |
| 99.9% (3) | ~8.8 hours | Common online baseline |
| 99.99% (4) | ~52 minutes | Core payment paths |
| 99.999% (5) | ~5 minutes | Multi-active + drills |

`SLO` = success / latency SLIs. Availability = **domains + redundancy + drills + change discipline**, not just more boxes.

**Rough product**: serial dependency availability ≈ product of deps. Five critical deps at 99.9% each ≈ **~99.5%** end-to-end.

### 5.4 Multi-active (same-city dual / geo multi)

```
Goal: unit closed-loop — most requests finish in-unit; cross-unit is exception
Routing: GSLB / hash(user_id) → unit; avoid request drift
Data: strong consistency in-unit; async + conflict (LWW / business arbiter) across units
Failover: traffic cutover + reconciliation; care about RPO/RTO, not “zero loss” slogans
Constraint: global uniqueness (phone, ID) needs a center or merge strategy
```

Open design D01 at the end covers global multi-active e-commerce.

---

## 6. Idempotency

### 6.1 Why

Timeouts, retries, at-least-once MQ, double-clicks → same business intent runs twice. Payment / stock / coupons without idempotency = money loss.

**Definition**: one execution vs many → same observable business result. Better: **return the first success result**, not only “duplicate.”

### 6.2 Common patterns

| Pattern | How | Use |
|---------|-----|-----|
| Idempotency key | Client `Idempotency-Key`; server SETNX / unique row | HTTP writes |
| Business unique key | Order / payment UNIQUE | DB last line of defense |
| State machine | `created→paid`; illegal transition → current state | Orders |
| One-time token | Issue token; validate+delete on submit | Form anti-dup |
| Optimistic lock | `UPDATE ... WHERE version=?` | Stock / balance |

### 6.3 Landing

```
Write API: idempotency key (edge) + unique index (store) + return first result
MQ: consumer dedupe table / business key; assume at-least-once
Distributed tx: TCC Confirm/Cancel and Saga compensations MUST be idempotent
```

---

## 7. Multi-level Cache & Penetration / Breakdown / Avalanche

### 7.1 Layers

```
Browser/CDN → edge local cache → Redis → DB
```

CDN/local = ultra-low latency, hard consistency; Redis = shared hot; DB = authority + must be rate-limited when pierced.

### 7.2 Three classics

| Problem | Symptom | Fixes |
|---------|---------|-------|
| **Penetration** | Query missing keys → DB | Bloom filter; cache null (short TTL) |
| **Breakdown** | Hot key expires → stampede | Single-flight rebuild; logical expire + async refresh |
| **Avalanche** | Mass expire / cache cluster down | TTL jitter; multi-tier degrade; rate limit + isolation |

### 7.3 Update strategies (one-liners)

- **Cache Aside**: miss → load DB → fill; update DB then **delete** cache (most common).
- Read/Write Through: cache proxies persistence.
- Delayed double-delete / binlog subscribe: shrink dirty windows under concurrency.

---

## 8. Message Queue Selection (Kafka / RabbitMQ / RocketMQ)

### 8.1 Comparison (system-design lens)

| | Kafka | RabbitMQ | RocketMQ |
|--|-------|----------|----------|
| Role | High-throughput log/stream | General messaging | Business messaging (Alibaba ecosystem) |
| Throughput | Extremely high | Mid-high | High |
| Model | Topic + Partition | Exchange + Queue | Topic + Queue |
| Ordering | Per partition | Per queue (scale weak) | Mature ordered messages |
| Tx / business | Stream-oriented tx | Ack / tx modes | **Transactional (half) messages** friendly |
| Delay | DIY / external | TTL+DLX | Built-in delay levels |
| Typical | Analytics, event bus | Complex routing, jobs | Orders, payment, e-commerce peak |

### 8.2 Selection mantra

```
Extreme throughput / streaming → Kafka
Flexible routing / enterprise integration → RabbitMQ
Transactional messages + e-commerce semantics (delay, order) → RocketMQ
```

Common design points regardless of pick: delivery semantics (often at-least-once), idempotent consume, backlog, DLQ, replay, couple with rate limit / breaker. Deep impl → *12* / *08*.

---

## 9. Read/Write Scale & Sharding

### 9.1 Scale ladder

```
Vertical scale → read replicas → sharding → unitization / distributed SQL
```

| Move | Solves | Introduces |
|------|--------|------------|
| Read/write split | Read throughput | Replication lag; sticky primary for read-your-writes |
| Vertical split | Fat tables / domain | Cross-DB tx / JOIN |
| Horizontal shard | Capacity & write bottleneck | Cross-shard page, global index, reshard |

### 9.2 Shard key & pagination (system view)

- Shard key: high cardinality, write spread, queries carry the key when possible.
- Cross-shard page: avoid deep `OFFSET`; seek / business cursor; prefer single-shard.
- Engine detail (Mongo shard key, TiDB, …) → [10-Databases-Comprehensive](./10-Databases-Comprehensive.md).

---

## 10. Distributed ID (Snowflake / Segment)

### Snowflake

64-bit layout:

```
1 bit sign (0) | 41-bit ms timestamp | 10-bit worker | 12-bit sequence
                 (~69y from epoch)     (≤1024 nodes)   (≤4096 / ms / node)
```

Pros: roughly time-ordered, no central store at generate time, high QPS, timestamp recoverable.

**Clock skew**: NTP step-back, leap second, VM restore. Detect `now < lastTimestamp`. Strategies: refuse / spin if skew ≤ few ms / borrow next ms / clock-sequence bits.

Worker ID: config, ZK sequential node, DB auto-inc mod 1024, hash(ip:port) (watch collisions).

### Segment (Leaf-Segment)

Batch-claim ranges from DB (`max_id += step` with optimistic `version`); dual-buffer preload when current segment ~10% left. Strictly increasing; restart wastes unused IDs; long DB outage still hurts after buffers drain.

### Comparison

| Scheme | Order | Perf | Deps | Notes |
|--------|-------|------|------|-------|
| UUID v4 | unordered | ★★★★★ | none | Random PK → page splits |
| Snowflake | trend ↑ | ★★★★★ | worker alloc | Better MySQL PK than UUID |
| Leaf-Segment | strict ↑ | ★★★★ | MySQL | Orders needing monotonic IDs |
| Redis INCR | strict ↑ | ★★★★ | Redis HA | Fine if Redis already core |
| DB auto-inc | strict ↑ | ★★ | MySQL | Low QPS only |

**Interview**: Snowflake as PK beats UUID because B+Tree appends trend-ordered keys; UUID random inserts cause page splits.

---

## 11. Distributed Lock & RedLock

### Redis single-node lock

```redis
SET lock_key <random_value> NX PX 30000
```

Unlock with Lua: `GET` equals token then `DEL` — never bare `DEL` (expired holder must not delete the next owner’s lock).

**Watchdog (Redisson)**: if lease unspecified, renew every `lease/3`. Process crash → watchdog dies → lock expires → no deadlock.

### RedLock

Acquire on majority of independent Redis masters within TTL budget; validity = TTL − acquisition time − clock drift. On failure, unlock all.

### Kleppmann vs Antirez

| | Kleppmann | Antirez |
|--|-----------|---------|
| Model | Async; do not trust clocks | Semi-sync; bounded drift |
| GC pause | Fatal for correctness locks | Detectable during acquire |
| Recommend | Fencing token + storage CAS | RedLock under ops assumptions |

**Two uses of locks**: efficiency (dup work OK if rare fail) vs correctness (must be safe). Correctness → ZK/etcd + fencing; efficiency → single Redis OK.

**Fencing token**: storage rejects writes with token < max seen.

### etcd lock

Lease + KeepAlive; **Revision** as natural fencing token; Watch previous key to avoid herd. Prefer for correctness + K8s-native ops.

| | Redis | ZK | etcd |
|--|-------|----|------|
| Model | AP-ish (async repl) | CP (ZAB) | CP (Raft) |
| Fencing | not native | zxid | Revision |
| Perf | ★★★★★ | ★★★ | ★★★★ |

**Interview template**: classify efficiency vs correctness; Redis for efficiency; ZK/etcd for correctness; if asked RedLock, state the Kleppmann/Antirez split.

---

## 12. Microservices Overview

> **Overview only.** Split, communication, discovery, tracing, config → full treatment in [12-Microservices-Architecture.md](./12-Microservices-Architecture.md).
> Rate limit / circuit break / distributed tx: §§1–3 here keep **selection**; implementation SSOT is *12*.

| Topic | Point | Detail |
|-------|-------|--------|
| Split | Capability, cohesion, data ownership | *12* |
| Comm | REST/gRPC sync; MQ async | *12* |
| Discovery | Consul / etcd / Nacos | *12* |
| Resilience | Breaker / degrade / bulkhead | *12* |
| Tracing | OpenTelemetry / Jaeger | *12* |

Do not split by noun layers; split by ownership boundaries.

---

## 13. System Design Cases

### Case: Short URL service

Follow §0: Clarify → Estimate → API → Data → Design → Deep Dive.

#### Clarify

Long→short code; 301/302 redirect; optional expiry, custom codes, click stats. Write QPS ≪ read QPS; read latency sensitive; short + globally unique.

#### Estimate (sample)

| Assumption | Order |
|------------|-------|
| New links / day | 1e7 |
| Redirects / day | 1e9 → avg ≈ 1e4 QPS; 10× peak → ~1e5 |
| Code length | 6 Base62 → ~56.8B space |
| Row size | ~0.5–1KB → yearly TB-class with hot/cold |

#### API

```
POST /api/v1/links  {long_url, expire_at?} → {short_code}
GET  /{short_code} → redirect long_url
```

#### Code generation

| Scheme | Pros | Collision |
|--------|------|-----------|
| **Distributed ID + Base62** (preferred) | Unique, ordered | Essentially none |
| Hash(long)+truncate Base62 | Stable map for same URL | **Must retry on conflict** |

#### 301 vs 302

| | 301 | 302 |
|--|-----|-----|
| Cache | Browser/CDN may pin | More origin hits; better stats/change |
| Use | Immutable forever | Expiry / stats / replace long URL (common) |

Production often **302** (or 307). Immutable + traffic save → 301 + CDN.

#### Storage & path

```
User → CDN → LB → Short-link service
                    ├─ Redis (hot redirects)
                    └─ MySQL (authority, short_code PK)
```

Read: Redis → miss DB → fill (TTL ≤ remaining TTL). Lazy expire + sweeper. Abuse: auth + rate limit + domain blacklist.

---

### Case: Flash sale

#### Requirements

Million-class peak QPS; read≫write; limited stock; **never oversell**. Correctness can be eventual to DB — **whole path need not be CAP-CP**.

#### Funnel architecture

```
User → CDN static → Nginx rate limit + local cache   (~1M QPS peak)
         → Layer1 edge: captcha + token bucket (~1000 QPS)
              **filters ~99.9%** (1M → 1k is 99.9%, not “90%”)
         → Layer2 Redis pre-deduct (Lua atomic)
         → Layer3 Kafka async order
         → Layer4 Order service: create order + MySQL optimistic lock
```

Model: **Redis atomic hold + MQ eventual persist**. Do not tag the whole chain CP.

#### Redis Lua (prevent negative stock)

```lua
-- KEYS[1]=stock  KEYS[2]=bought set  ARGV[1]=user  ARGV[2]=qty
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return -1 end
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock == nil then return -2 end
local qty = tonumber(ARGV[2])
if qty == nil or qty <= 0 then return -3 end
if stock < qty then return 0 end   -- refuse before DECRBY (no negative)
redis.call('DECRBY', KEYS[1], qty)
redis.call('SADD', KEYS[2], ARGV[1])
return 1
```

#### MySQL optimistic lock (read version first)

```sql
SELECT version FROM sku_stock WHERE sku_id = ?;
UPDATE sku_stock SET stock = stock - 1, version = version + 1
 WHERE sku_id = ? AND stock > 0 AND version = ?;
-- affected==0 → conflict / sold out; order_id UNIQUE for idempotency
```

Kafka publish failure → **rollback Redis** (`INCR` + `SREM`). Hot stock: shard one stock into N keys.

Degrade ladder: close non-core → busy on Redis miss → sync DB at tiny QPS if Kafka down → static “too hot” page.

---

### Case: Feed timeline

#### Push / pull / hybrid

- **Push (write fanout)**: write each follower inbox — fast read, write amp for celebrities.
- **Pull (read fanout)**: merge followees’ outboxes — fast write, slow read.
- **Hybrid (recommended)**: push for normal authors; pull celebrities at read time.

#### Architecture sketch

Write: persist post → outbox ZSET → if followers < threshold, Kafka fanout to inboxes; else outbox only.
Read: inbox + celebrity outboxes → merge → hydrate content.

#### Cursor pagination (critical)

**Do not** `OFFSET` inbox then stitch celebrity posts — missing/dup pages. Use **cursor = (score, post_id)** on the **merged** stream; open-interval score + post_id tie-break.

Cold start: on follow normal user, async backfill last N into inbox (rate-limited); celebrities: no backfill, pull outbox on read. Unfollow: async inbox cleanup OK eventual. Cap inbox size (`ZREMRANGEBYRANK`).

---

### Case: Distributed job scheduler (classic)

#### Clarify

Cron / delay / one-shot; millions of jobs; multi-tenant. Exactly-once is unrealistic → **at-least-once + idempotent handlers**. Auto failover; observability of success/fail/timeout/shard progress.

#### Architecture

```
API/console → Scheduler HA (etcd leader election) → sharded trigger queue / MQ
                                                    → Worker fleet claim & run
                                                    → exec log + alert + retry
```

#### Deep dive

| Hard part | Approach |
|-----------|----------|
| Scheduler HA | etcd/ZK election; lease loss → release |
| Sharding | `hash(job_id) % shard` or time-wheel shards; no single full-table scan |
| Exactly-once | Unique `(job_id, schedule_time)` + business idempotency |
| Long jobs | Lease heartbeat; timeout reclaim; cancel |
| Misfire | fire immediately / skip / fire latest only |

Same interview muscles as short-link / flash sale: **election, sharding, idempotency, failover**.

---

### Case: Distributed rate-limit service

#### Goals

Shared quota across instances; multi-dimension (user/IP/API); token bucket + sliding window; P99 decide < 5ms; fail-open when Redis down (often + local fallback). Consistency: brief imprecision OK (**AP-leaning**).

#### Flow

Local hot “already limited” reject → Redis Lua atomic decide → allow/deny. Config center pushes rules.

#### Distributed token bucket Lua (lazy refill)

```lua
-- KEYS[1]=key  ARGV=capacity, rate, now_sec_float, requested
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])
local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])
if tokens == nil then tokens = capacity; last_refill = now end
local elapsed = math.max(0, now - last_refill)
tokens = math.min(capacity, tokens + elapsed * rate)
local allowed = 0
if tokens >= requested then tokens = tokens - requested; allowed = 1 end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', KEYS[1], math.ceil(capacity / rate) * 2)
return allowed
```

Sliding window: `ZREMRANGEBYSCORE` + `ZCARD` + `ZADD` with unique member. Prefer Redis `TIME` or NTP for clock skew.

Selection: API burst → token bucket; precise user quota → sliding window; protect DB writes → leaky bucket.

---

## 14. Interview Self-Check

> Numbering: `Q01+` concepts; `D01+` design prompts. Each maps to a section + follow-up.

### Architecture review add-ons

Answer what bottleneck a component removes **and** what complexity it adds. Cover traffic model, consistency goal, failure model, observability, change/rollback.

---

### Basics (Q01–Q08)

**Q01｜§0｜High concurrency / availability / consistency?**
Peak capacity vs survive faults vs agreed consistency level.
**Follow-up**: How do you align trade-offs with product?

**Q02｜§0｜Why back-of-envelope?**
Order of magnitude before components; 10× off → different architecture.
**Follow-up**: 100M daily PV, peak/avg 10 — peak QPS?

**Q03｜§6｜Idempotent API? Why payment?**
Same intent → same result; retries otherwise lose money. Prefer return first result.
**Follow-up**: Idempotency key vs DB unique constraint roles?

**Q04｜§5｜SPOF? Why failure domains?**
One failure kills the chain; isolation bounds blast radius.
**Follow-up**: Five serial 99.9% deps → end-to-end?

**Q05｜§5.3｜99.9% vs 99.99% downtime?**
~8.8h vs ~52min. Pair SLO with SLI + error budget.
**Follow-up**: Publish policy when error budget is spent?

**Q06｜§1/§2｜Isolation vs rate limit vs breaker?**
Isolation = local spread; limit = total traffic; breaker = stop hammering a sick dep.
**Follow-up**: Map to bulkhead / pools in *12*?

**Q07｜§9｜Cache / MQ / sharding each fix?**
Hot reads / decouple+peak / capacity+write.
**Follow-up**: Why is cross-shard pagination dangerous?

**Q08｜§5｜Why are canary & rollback architecture?**
Change fails; small blast + rollback.
**Follow-up**: How to make DB changes rollback-safe?

---

### Intermediate (Q09–Q20)

**Q09｜§1｜Algorithms? Token vs leaky?**
Fixed/sliding/token/leaky. Token allows burst; leaky smooths outflow.
**Follow-up**: Why must hand-written token bucket avoid integer truncation of `elapsed*rate`?

**Q10｜§1.3｜Distributed limit? What is that Lua?**
Redis+Lua shared quota. `INCR+EXPIRE` = **fixed window**, not token bucket; token bucket uses tokens/last_refill (case §13).
**Follow-up**: Fail-open vs fail-close cost?

**Q11｜§2｜Breaker states? Breaker vs degrade?**
Closed→Open→Half-Open. Breaker protects downstream; degrade protects your critical path.
**Follow-up**: How many half-open probes?

**Q12｜§3｜Tx options? Is TCC “strong consistency”?**
2PC/TCC/Saga/outbox. TCC = **business reservation/isolation**; Confirm/Cancel still idempotent — not DB linearizability.
**Follow-up**: Why default to local message/outbox?

**Q13｜§4.1｜Why not “have all three”?**
Must face partition; under partition trade linearizability vs non-failed response. Say **C/A under P**, not casual three-choose-two.
**Follow-up**: Precise C/A/P? Can Cassandra `W=3,R=1` be labeled CP?

**Q14｜§4.2｜BASE vs CAP/ACID?**
BA/Soft/Eventual; philosophy **vs ACID**, not “CAP+”.
**Follow-up**: Which paths stay CP?

**Q15｜§4.3｜Consistent hash + vnodes?**
Ring remap neighbors; vnodes fix skew.
**Follow-up**: How many vnodes?

**Q16｜§4.4｜Raft election & commit?**
Timeout → candidate; majority; random timeout; majority ack commits.
**Follow-up**: Why minority cannot commit? Relation to ZK quorum?

**Q17｜§7｜Penetration / breakdown / avalanche?**
Missing key / hot expire / mass fail. Bloom+null / single-flight / TTL jitter+limit.
**Follow-up**: Why Redis alone can still avalanche?

**Q18｜§8｜Kafka / Rabbit / Rocket?**
Stream→Kafka; routing→Rabbit; tx+e-com→RocketMQ.
**Follow-up**: At-least-once + idempotency?

**Q19｜§10｜Snowflake + clock skew?**
1+41+10+12; wait/refuse/borrow; or segment.
**Follow-up**: Why Snowflake PK > UUID?

**Q20｜§11｜RedLock controversy? Lock choice?**
Efficiency vs correctness; Kleppmann: clocks/GC; fencing; Antirez: ops assumptions. Correctness → ZK/etcd.
**Follow-up**: etcd Revision as fencing token?

---

### Cases & design (Q21–Q26 + D01–D04)

**Q21｜§13 flash sale｜Funnel & consistency model?**
CDN→edge (~**99.9%** filtered 1M→1k)→Redis Lua (`stock < qty`)→MQ→DB optimistic lock (read `version` first). Model = atomic hold + eventual DB — **not whole-path CP**.
**Follow-up**: Kafka fail → Redis rollback? Hot-key stock shards?

**Q22｜§13 Feed｜Hybrid & pagination pitfall?**
Push normals, pull celebs; **score+id cursor on merged stream**; no inbox offset then stitch. Cold start: backfill normals, not celebs.
**Follow-up**: Dirty inbox after unfollow?

**Q23｜§13 short link｜Code gen? Collision? 301/302?**
Snowflake+Base62 preferred; hash truncate must retry. 302 for stats/change; 301 for cache.
**Follow-up**: Cache TTL vs link expiry?

**Q24｜§13 rate-limit service｜Local + Redis?**
Local hot reject + Redis atomic; Redis down fail-open + local.
**Follow-up**: ZSET sliding-window memory cost?

**Q25｜§13 scheduler｜No loss / no dup?**
Leader scheduler + shard trigger + at-least-once + unique exec key / business idempotency; lease for long jobs.
**Follow-up**: Misfire strategies?

**Q26｜§5.4 / D01｜Multi-active tension?**
Unit closed-loop vs global consistency; async + conflict; cutover via RPO/RTO.
**Follow-up**: Global unique phone numbers?

---

### Open design prompts

**D01｜§5.4｜Global multi-active e-commerce**
Unit routing; in-unit strong / cross-unit eventual; conflict; cutover + reconciliation.
**Follow-up**: Payment callback lands in wrong unit?

**D02｜§5｜P99 50ms → 500ms**
Layered triage + traces + change diff + rollback verify.
**Follow-up**: Large P99−P50 gap means?

**D03｜§13｜IM online messaging (or deepen scheduler)**
Stateful conn layer + routing + store/sync + multi-device read; group fanout vs read amplify.
**Follow-up**: Online catch-up cursor?

**D04｜§13 flash sale｜Whiteboard from zero**
Estimate peak & stock; no-oversell + latency; four-layer filter; Lua + idempotency; degrade levels + drills.
**Follow-up**: Only DB row locks — cost?

---

### Senior synthesis (Q27–Q32)

**Q27｜§6｜Why return first result on idempotent retry?**
Caller needs the business outcome, not “you duplicated.”
**Follow-up**: Store first response how?

**Q28｜§2/§5｜Why define degrade in requirements?**
Degrade is product trade-off; inventing under fire causes loss.
**Follow-up**: One path that must never degrade?

**Q29｜§0/§5｜Four capacity steps?**
Peak estimate → single-box load test → headroom instances → scale thresholds + rollback.
**Follow-up**: Cross-check limit thresholds with capacity plan?

**Q30｜§1/§5｜Retry budget?**
Count, wall clock, backoff+jitter, retryable whitelist, couple to breaker.
**Follow-up**: How retries cause avalanche?

**Q31｜§4｜Decision frame when C vs A conflict?**
Money/compliance → stronger consistency; stats/rec → eventual; write latency & repair SLO.
**Follow-up**: Why cart ≠ debit?

**Q32｜Whole doc｜Five review questions first?**
Traffic model, consistency, failure model, observability, change/scale.
**Follow-up**: Structure a failure postmortem (context→metrics→root→verify redo).

---

## Summary

System design exam path (shallow → deep):

1. **Method**: Clarify → Estimate → API → Data → Design → Deep Dive → Scale
2. **Stability**: rate limit / break+degrade / timeout-retry / bulkhead (**selection here; impl in *12***)
3. **Consistency & tx**: C/A under partition; BASE vs ACID; TCC reservation; outbox
4. **Scale**: idempotency, cache triad, MQ pick, replicas & shards
5. **Deep components**: Snowflake/segment, Redis lock & RedLock debate, etcd fencing
6. **Cases**: short link, flash sale, Feed, job scheduler, distributed limiter — same template for estimate & failure

---
