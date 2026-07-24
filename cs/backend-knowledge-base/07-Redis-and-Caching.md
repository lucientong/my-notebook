# Redis and Caching - In-Depth Guide

Language: English | [中文](../后端知识库/07-Redis与缓存.md)

---

## Table of Contents

### Data Structures and Cache Consistency
1. [Redis Data Structures](#1-redis-data-structures)
2. [Three Classic Cache Problems](#2-three-classic-cache-problems)
3. [Distributed Locks](#3-distributed-locks)

### Persistence and High Availability
4. [Persistence](#4-persistence)
5. [High Availability Architecture](#5-high-availability-architecture)

### Practice and Interview Review
6. [Practical Cases](#6-practical-cases)
7. [Redis 7.0+ Features](#7-redis-70-features)
8. [Interview Self-Check](#8-interview-self-check)

---

## 1. Redis Data Structures

### 1.1 Five Basic Data Types

| Type | Internal Encoding | Use Cases | Common Commands |
|------|-------------------|-----------|-----------------|
| **String** | SDS / integer encoding | cache, counter, distributed lock | `SET`, `GET`, `INCR`, `SETNX` |
| **List** | quicklist, listpack in newer versions | queue, timeline | `LPUSH`, `RPOP`, `LRANGE` |
| **Hash** | listpack / hashtable | object cache, shopping cart | `HSET`, `HGET`, `HGETALL` |
| **Set** | intset / hashtable | deduplication, common friends | `SADD`, `SMEMBERS`, `SINTER` |
| **ZSet** | listpack or skiplist + hashtable | leaderboard, delay queue | `ZADD`, `ZRANGE`, `ZRANK` |

### 1.2 Advanced Data Types

| Type | Meaning | Use Cases |
|------|---------|-----------|
| **Bitmap** | Bit-level storage | sign-in records, online status |
| **HyperLogLog** | Approximate cardinality | UV counting with small error |
| **Geo** | Geospatial index | nearby users, ride hailing |
| **Stream** | Append-only message stream | lightweight event stream, consumer groups |

### 1.3 String for Object Cache

```bash
# Cache user info as JSON
SET user:1001 '{"id":1001,"name":"Alice","age":25}'
GET user:1001

# Cache object fields as Hash
HSET user:1001 id 1001 name "Alice" age 25
HGET user:1001 name
HGETALL user:1001

# Set expiration
SETEX user:1001 3600 '{"id":1001,"name":"Alice"}'
```

String is simpler when the whole object is read and written together. Hash is better when fields are updated independently.

### 1.4 ZSet for Leaderboards

```bash
ZADD rank:score 1000 user:1001
ZADD rank:score 1500 user:1002
ZADD rank:score 1200 user:1003

# Top 10
ZREVRANGE rank:score 0 9 WITHSCORES

# User rank, 0-based
ZREVRANK rank:score user:1001

# User score
ZSCORE rank:score user:1001

# Increase score
ZINCRBY rank:score 100 user:1001

# Query score range
ZRANGEBYSCORE rank:score 1000 2000
```

### 1.5 Bitmap for Sign-In Records

```bash
# user 1001 sign-in status for January 2024
SETBIT sign:1001:202401 0 1
SETBIT sign:1001:202401 1 1
SETBIT sign:1001:202401 2 0

# Check a day
GETBIT sign:1001:202401 0

# Count sign-in days
BITCOUNT sign:1001:202401

# Find first zero bit
BITPOS sign:1001:202401 0
```

Bitmap is extremely memory efficient for boolean flags.

### 1.6 Internal Encodings ⭐⭐⭐

Redis chooses compact or general-purpose encodings based on data size and element count. Knowing this helps explain why Redis is fast and why large keys are dangerous.

### 1.6.1 String: SDS

SDS, Simple Dynamic String, improves on C strings.

| Feature | C String | SDS |
|---------|----------|-----|
| **Length lookup** | O(n), scan until `\0` | O(1), stored length |
| **Binary safe** | No, `\0` terminates string | Yes, length-aware |
| **Buffer overflow** | Easy to misuse | Automatic growth |
| **Allocation** | Frequent realloc | Pre-allocation and lazy free |

Simplified SDS:

```c
struct sdshdr {
    int len;
    int free;
    char buf[];
};
```

Pre-allocation:

```text
If new length < 1MB:
  allocate roughly 2 * len
If new length >= 1MB:
  allocate len + 1MB
```

Lazy free keeps unused space after shrinking so future appends may avoid realloc.

### 1.6.2 List: quicklist and listpack

Redis does not use a plain linked list for typical lists because pointer overhead is too high. It uses a linked structure whose nodes store compact contiguous memory blocks.

```text
quicklist:
[prev] <-> [listpack] <-> [listpack] <-> [listpack] <-> [next]
             |             |             |
          compact        compact       compact
          memory         memory        memory
```

Historically Redis used ziplist. Redis 7 uses listpack to avoid ziplist's cascading update problem.

**Ziplist cascading update**:

```text
Each entry stores previous entry length.
If a previous entry grows from <254 bytes to >=254 bytes,
the next entry's prevlen expands from 1 byte to 5 bytes.
This can cascade through later entries.
```

**Listpack improvement**:

```text
entry = [encoding][data][backlen]

It stores the current entry length for backward traversal,
so it avoids ziplist's cascading update problem.
```

### 1.6.3 Hash: listpack / hashtable

Small hashes use compact listpack encoding. Large hashes use hashtable.

Typical thresholds in older configs:

```bash
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
```

In Redis 7, these were renamed to listpack-related options.

Hashtable uses progressive rehashing:

```c
typedef struct dict {
    dictht ht[2];
    long rehashidx;
} dict;
```

Why progressive rehash?

```text
Rehashing a huge hash table all at once would block Redis.
Redis allocates ht[1] and gradually migrates buckets from ht[0] to ht[1].
Each normal operation helps migrate a small part.
During rehash, lookup checks both tables.
```

### 1.6.4 Set: intset / hashtable

If all elements are integers and the set is small, Redis can use intset.

```c
typedef struct intset {
    uint32_t encoding;  // INT16 / INT32 / INT64
    uint32_t length;
    int8_t contents[];
} intset;
```

When a larger integer is inserted, intset upgrades its encoding, such as INT16 -> INT32. It only upgrades and does not downgrade after large values are removed.

### 1.6.5 ZSet: listpack or skiplist + hashtable

Small sorted sets use compact encoding. Large sorted sets use both a skiplist and a hashtable.

```c
typedef struct zset {
    dict *dict;        // member -> score, O(1) score lookup
    zskiplist *zsl;    // ordered by score, O(log n) range query
} zset;
```

Why skiplist instead of red-black tree?

- Range queries are simple after locating the start.
- Implementation is simpler; no rotations.
- The bottom level is a linked list, which is convenient for ordered traversal.

Random level generation:

```text
Redis uses probability p=0.25.
About 75% of nodes have level 1.
About 18.75% have level 2.
About 4.69% have level 3.
```

Why both skiplist and hashtable?

| Operation | Skiplist | Hashtable | Combined |
|-----------|----------|-----------|----------|
| `ZADD` | O(log n) | O(1) | O(log n) |
| `ZSCORE` | O(log n) | O(1) | O(1) |
| `ZRANGE` | O(log n + k) | Not ordered | O(log n + k) |
| `ZRANK` | O(log n) | No rank | O(log n) |

### 1.7 Why Is Redis Fast Even with a Mostly Single-Threaded Model? ⭐⭐⭐

Common misconception: "Redis is fully single-threaded."

More accurate:

- Before Redis 6.0, core network IO and command execution were mostly single-threaded.
- Redis 6.0 introduced optional multi-threaded network IO.
- Command execution remains mostly single-threaded for simplicity and lock-free correctness.

Key reasons:

1. **In-memory operations**: memory access is orders of magnitude faster than disk.
2. **IO multiplexing**: `epoll` lets one thread manage many sockets.
3. **Efficient data structures**: SDS, listpack, intset, skiplist.
4. **No lock contention for command execution**.

`epoll` model:

```c
int epfd = epoll_create(1024);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = socket_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &ev);

struct epoll_event events[MAX_EVENTS];
int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);

for (int i = 0; i < nfds; i++) {
    if (events[i].events & EPOLLIN) {
        read(events[i].data.fd, buf, sizeof(buf));
        processCommand(buf);
    }
}
```

`epoll` vs `select/poll`:

| Feature | select | poll | epoll |
|---------|--------|------|-------|
| FD limit | Usually 1024 | No fixed small limit | No fixed small limit |
| Scan cost | O(n) | O(n) | Returns ready list |
| Copy overhead | Repeated fd_set copy | Repeated pollfd copy | Registered once |
| Trigger mode | LT | LT | LT and ET |

Redis 6.0 network IO threads:

```text
Clients -> IO threads read/write sockets -> main thread executes commands -> IO threads send responses
```

This helps when network IO or large values become the bottleneck.

---

## 2. Three Classic Cache Problems

### 2.1 Cache Penetration

Definition: requests query non-existing data. Both cache and database miss, so every request hits the database.

Typical scenario: malicious traffic repeatedly queries invalid IDs.

**Solution 1: cache null values**

```go
func GetUser(id int64) (*User, error) {
    user, err := redis.Get(fmt.Sprintf("user:%d", id))
    if err == nil {
        if user == "null" {
            return nil, ErrUserNotFound
        }
        return parseUser(user), nil
    }

    user, err = db.GetUser(id)
    if err != nil {
        redis.Set(fmt.Sprintf("user:%d", id), "null", 5*time.Minute)
        return nil, err
    }

    redis.Set(fmt.Sprintf("user:%d", id), user.ToJSON(), time.Hour)
    return user, nil
}
```

**Solution 2: Bloom filter**

```go
import "github.com/bits-and-blooms/bloom/v3"

var bf = bloom.NewWithEstimates(10000000, 0.01)

func InitBloomFilter() {
    users := db.GetAllUserIDs()
    for _, uid := range users {
        bf.AddString(fmt.Sprintf("%d", uid))
    }
}

func GetUser(id int64) (*User, error) {
    if !bf.TestString(fmt.Sprintf("%d", id)) {
        return nil, ErrUserNotFound
    }
    // cache -> database
}
```

| Solution | Pros | Cons |
|----------|------|------|
| Cache null | Simple | Many invalid keys consume cache |
| Bloom filter | Small memory, fast check | False positives, deletion is hard |

### 2.2 Cache Breakdown

Definition: a hot key expires and many concurrent requests hit the database at the same time.

**Solution 1: singleflight or mutex**

```go
import "golang.org/x/sync/singleflight"

var sf singleflight.Group

func GetProduct(id int64) (*Product, error) {
    product, err := redis.Get(fmt.Sprintf("product:%d", id))
    if err == nil {
        return parseProduct(product), nil
    }

    v, err, _ := sf.Do(fmt.Sprintf("product:%d", id), func() (interface{}, error) {
        product, err := redis.Get(fmt.Sprintf("product:%d", id))
        if err == nil {
            return parseProduct(product), nil
        }

        product, err := db.GetProduct(id)
        if err != nil {
            return nil, err
        }

        redis.Set(fmt.Sprintf("product:%d", id), product.ToJSON(), time.Hour)
        return product, nil
    })
    if err != nil {
        return nil, err
    }
    return v.(*Product), nil
}
```

**Solution 2: logical expiration**

```go
type CacheValue struct {
    Data     string
    ExpireAt int64
}

func GetProduct(id int64) (*Product, error) {
    val, err := redis.Get(fmt.Sprintf("product:%d", id))
    if err != nil {
        return queryDB(id)
    }

    cacheVal := parseCacheValue(val)
    if time.Now().Unix() > cacheVal.ExpireAt {
        go updateCache(id)
    }

    return parseProduct(cacheVal.Data), nil
}
```

Mutex ensures freshness better but makes requests wait. Logical expiration improves availability but may return stale data briefly.

### 2.3 Cache Avalanche

Definition: many keys expire around the same time, causing traffic to hit the database at once.

Solutions:

1. Add random jitter to expiration.
2. Use logical expiration for hot data.
3. Add multi-level cache.
4. Apply rate limiting and degradation.

```go
baseExpire := time.Hour
randomOffset := time.Duration(rand.Intn(600)) * time.Second
expire := baseExpire + randomOffset

redis.Set(key, value, expire)
```

```text
Local cache (1s) -> Redis (1min) -> DB
```

Rate limiter:

```go
import "golang.org/x/time/rate"

var limiter = rate.NewLimiter(1000, 2000)

func GetData(key string) (string, error) {
    if !limiter.Allow() {
        return "", ErrRateLimitExceeded
    }
    return queryData(key)
}
```

---

## 3. Distributed Locks

### 3.1 SETNX-Based Evolution

**Version 1: no expiration, unsafe**

```go
func Lock(key string) bool {
    return redis.SetNX(key, "1", 0).Val()
}

func Unlock(key string) {
    redis.Del(key)
}
```

If the process crashes, the lock is never released.

**Version 2: SETNX then EXPIRE, still unsafe**

```go
func Lock(key string) bool {
    if redis.SetNX(key, "1", 0).Val() {
        redis.Expire(key, 10*time.Second)
        return true
    }
    return false
}
```

`SETNX` and `EXPIRE` are not atomic.

**Version 3: SETNX with expiration, still incomplete**

```go
func Lock(key string) bool {
    return redis.SetNX(key, "1", 10*time.Second).Val()
}

func Unlock(key string) {
    redis.Del(key)
}
```

This can delete another process's lock if the old lock expires and a new owner acquires it.

**Version 4: unique value + Lua unlock**

```go
func Lock(key string, value string, expire time.Duration) bool {
    return redis.SetNX(key, value, expire).Val()
}

func Unlock(key string, value string) bool {
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    result := redis.Eval(script, []string{key}, value).Val()
    return result == int64(1)
}
```

**Version 5: auto-renewal**

```go
type RedisLock struct {
    key    string
    value  string
    expire time.Duration
    stopCh chan struct{}
}

func (l *RedisLock) Lock() bool {
    if !redis.SetNX(l.key, l.value, l.expire).Val() {
        return false
    }
    l.stopCh = make(chan struct{})
    go l.renewLoop()
    return true
}

func (l *RedisLock) renewLoop() {
    ticker := time.NewTicker(l.expire / 3)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            script := `
                if redis.call("GET", KEYS[1]) == ARGV[1] then
                    return redis.call("EXPIRE", KEYS[1], ARGV[2])
                else
                    return 0
                end
            `
            redis.Eval(script, []string{l.key}, l.value, int(l.expire.Seconds()))
        case <-l.stopCh:
            return
        }
    }
}
```

Production warning: Redis locks are coordination tools, not a substitute for database constraints, transactions, and idempotent state machines when money or strong correctness is involved.

### 3.2 Redlock

Redlock tries to reduce the single-node failure risk by acquiring locks on multiple independent Redis masters.

Basic steps:

1. Try to acquire the lock on N independent Redis nodes, usually N=5.
2. If a majority succeeds and total elapsed time is less than lock TTL, the lock is acquired.
3. Otherwise release all acquired locks.

```go
import (
    "github.com/go-redsync/redsync/v4"
    "github.com/go-redsync/redsync/v4/redis/goredis/v8"
)

func main() {
    client1 := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    client2 := redis.NewClient(&redis.Options{Addr: "localhost:6380"})
    client3 := redis.NewClient(&redis.Options{Addr: "localhost:6381"})

    pool := goredis.NewPool(client1, client2, client3)
    rs := redsync.New(pool)

    mutex := rs.NewMutex("lock:order:1001")
    if err := mutex.Lock(); err != nil {
        panic(err)
    }
    defer mutex.Unlock()
}
```

Redlock is controversial for strict correctness under partitions and pauses. Use it for best-effort mutual exclusion, not as the only correctness boundary for critical financial operations.

---

## 4. Persistence

### 4.1 RDB Snapshot

RDB periodically dumps memory data to disk as a compact binary file.

Trigger methods:

- Manual: `SAVE` blocks, `BGSAVE` forks a child process.
- Automatic: `save 900 1` means at least one write within 900 seconds.

```text
save 900 1
save 300 10
save 60 10000

dbfilename dump.rdb
dir /var/lib/redis
```

Pros:

- Compact file.
- Fast recovery.
- Lower runtime overhead because the child process writes the snapshot.

Cons:

- Data between snapshots may be lost.
- Fork and copy-on-write can consume memory.

### 4.2 AOF

AOF records every write command.

```text
appendonly yes
appendfilename "appendonly.aof"

appendfsync always
appendfsync everysec
appendfsync no
```

Sync policies:

- `always`: safest but slowest.
- `everysec`: common production trade-off, may lose about one second.
- `no`: OS decides when to flush, fastest but riskier.

AOF rewrite:

```text
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
BGREWRITEAOF
```

### 4.3 Hybrid Persistence

```text
aof-use-rdb-preamble yes
```

Hybrid AOF stores:

- RDB snapshot at the beginning for fast loading.
- Incremental AOF commands after that for better durability.

This is usually the recommended production choice when persistence is required.

---

## 5. High Availability Architecture

### 5.1 Master-Replica Replication

```text
[Master] -> [Replica 1]
         -> [Replica 2]
```

Replica configuration:

```bash
replicaof 103.230.239.204 6379
masterauth <master_password>
```

Replication:

1. Full sync: first connection, master sends RDB.
2. Incremental sync: subsequent write commands are replicated.

Read/write split:

- Master handles writes.
- Replicas handle reads, with possible replication lag.

### 5.2 Sentinel

Sentinel monitors master and replicas and performs automatic failover.

```text
[Sentinel 1] [Sentinel 2] [Sentinel 3]
      |            |            |
   [Master] -> [Replica 1] -> [Replica 2]
```

```text
sentinel monitor mymaster 106.75.144.36 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 15000
```

Failover flow:

1. Sentinels detect master failure.
2. Quorum confirms objective down.
3. One replica is promoted to master.
4. Other replicas follow the new master.
5. Clients are notified of the new master address.

### 5.3 Redis Cluster

Redis Cluster provides:

- Data sharding.
- Automatic failover.
- Horizontal scaling.

Redis Cluster uses 16,384 hash slots.

```bash
redis-cli --cluster create \
    172.200.156.167:7001 172.200.156.167:7002 172.200.156.167:7003 \
    172.200.156.167:7004 172.200.156.167:7005 172.200.156.167:7006 \
    --cluster-replicas 1
```

Slot calculation:

```go
func GetSlot(key string) int {
    return int(crc16.Checksum([]byte(key), crc16.IBMTable)) % 16384
}
```

Hash tags force multiple keys into the same slot:

```text
{user}:1001
{user}:profile:1001
```

### 5.4 Expiration Strategy ⭐⭐

Redis does not immediately delete every key at its expiration timestamp.

Two strategies work together:

```text
Lazy expiration:
- Check expiration when a key is accessed.
- If expired, delete and return nil.
- CPU friendly, but expired keys may remain if never accessed.

Active expiration:
- Periodically sample keys with TTL.
- Delete expired keys.
- Repeat if expired ratio is high.
- More memory friendly, but consumes CPU.
```

```bash
hz 10
```

`hz` controls how often Redis runs periodic tasks. Higher values clean more aggressively but use more CPU.

### 5.5 Eviction Policy ⭐⭐⭐

When memory reaches `maxmemory`, Redis chooses keys to evict based on policy.

| Policy | Scope | Algorithm | Meaning |
|--------|-------|-----------|---------|
| `noeviction` | none | none | Reject writes with OOM |
| `allkeys-lru` | all keys | approximate LRU | Evict least recently used |
| `allkeys-lfu` | all keys | LFU | Evict least frequently used |
| `allkeys-random` | all keys | random | Random eviction |
| `volatile-lru` | keys with TTL | approximate LRU | Evict old TTL keys |
| `volatile-lfu` | keys with TTL | LFU | Evict low-frequency TTL keys |
| `volatile-random` | keys with TTL | random | Random TTL key |
| `volatile-ttl` | keys with TTL | TTL | Evict keys expiring soon |

Approximate LRU:

```text
Exact LRU needs a global linked list and updates on every access.
Redis samples N keys and evicts the one with the oldest LRU clock.

maxmemory-samples 5
```

LFU in Redis 4.0+:

```text
Each key tracks:
- 8-bit logarithmic frequency counter.
- 16-bit last decrement time.

lfu-log-factor controls how slowly the counter grows.
lfu-decay-time controls time-based decay.
```

For cache workloads, `allkeys-lru` or `allkeys-lfu` is commonly used. `noeviction` is safer when Redis is used as a primary data store and failed writes should be explicit.

### 5.6 Transactions and Lua Scripts ⭐⭐

Redis transaction:

```text
MULTI
command 1
command 2
EXEC

DISCARD
```

Important: Redis transaction is not the same as an RDBMS transaction.

- No rollback if a command fails at execution time.
- Commands are executed sequentially without interleaving.
- `WATCH` provides optimistic locking.

```bash
WATCH balance:1001
MULTI
DECRBY balance:1001 100
INCRBY balance:1002 100
EXEC
```

Lua scripts provide stronger atomicity for read-modify-write logic:

```bash
EVAL "
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local current = tonumber(redis.call('GET', key) or '0')
    if current < limit then
        redis.call('INCR', key)
        if current == 0 then
            redis.call('EXPIRE', key, window)
        end
        return 1
    else
        return 0
    end
" 1 rate:api:/user 100 60
```

Lua precautions:

1. Keep scripts short; long scripts block Redis.
2. Avoid heavy loops.
3. Use `EVALSHA` to cache scripts.
4. In Cluster mode, all keys used by a script must be in the same slot.

---

## 6. Practical Cases

### Case 1: Cache Penetration Causes DB Pressure

Background: an API is attacked with non-existing user IDs.

Symptom: every request misses Redis and hits MySQL; DB QPS spikes.

Solution:

```go
var bf = bloom.NewWithEstimates(10000000, 0.01)

func GetUser(id int64) (*User, error) {
    if !bf.TestString(fmt.Sprintf("%d", id)) {
        return nil, ErrUserNotFound
    }
    // cache -> database
}
```

Result: invalid traffic is filtered before hitting the database.

### Case 2: Hot Product Cache Breakdown

Background: flash-sale product cache expires exactly when traffic spikes.

Symptom: database slow queries increase and API times out.

Solution:

```go
var sf singleflight.Group

func GetProduct(id int64) (*Product, error) {
    v, err, _ := sf.Do(fmt.Sprintf("product:%d", id), func() (interface{}, error) {
        return queryDB(id)
    })
    return v.(*Product), err
}
```

Result: one database query instead of thousands.

---

## Summary

Redis interview core topics:

1. **Data structures**: String, List, Hash, Set, ZSet, Bitmap, HyperLogLog, Geo, Stream.
2. **Cache problems**: penetration, breakdown, avalanche.
3. **Distributed locks**: `SET NX EX`, unique value, Lua unlock, renewal, Redlock limits.
4. **Persistence**: RDB, AOF, hybrid persistence.
5. **High availability**: replication, Sentinel, Cluster.
6. **Operations**: expiration, eviction, big keys, hot keys, Lua, Redis 7 features.

---

## 7. Redis 7.0+ Features

### 7.1 Redis Functions

Redis Functions improve on Lua scripts:

| Pain Point of Lua Script | Redis Functions Improvement |
|--------------------------|-----------------------------|
| Scripts are not persisted by name | Functions persist through RDB/AOF |
| SHA1 invocation is hard to manage | Named functions |
| No library concept | Library-based organization |
| Hard to version and deploy | `FUNCTION LOAD REPLACE` |
| Manual synchronization | Replicated to replicas |

Example:

```bash
FUNCTION LOAD "#!lua name=mylib
redis.register_function('rate_limit', function(keys, args)
    local key = keys[1]
    local limit = tonumber(args[1])
    local window = tonumber(args[2])
    local current = tonumber(redis.call('GET', key) or '0')
    if current < limit then
        redis.call('INCR', key)
        if current == 0 then
            redis.call('EXPIRE', key, window)
        end
        return 1
    end
    return 0
end)
"

FCALL rate_limit 1 rate:api:/order 100 60
FUNCTION LIST
FUNCTION LOAD REPLACE "#!lua name=mylib ..."
```

Organization suggestions:

- `orderlib`: order-related atomic operations.
- `cachelib`: cache governance logic.
- `locklib`: lock acquire, renew, release logic.
- Manage function code through CI/CD, not manual production editing.

### 7.2 Sharded Pub/Sub

Traditional Pub/Sub in Redis Cluster broadcasts messages across all nodes, wasting bandwidth as the cluster grows.

Sharded Pub/Sub routes messages only to the node responsible for the channel's hash slot.

```bash
SSUBSCRIBE channel:order:1001
SPUBLISH channel:order:1001 "paid"
SUNSUBSCRIBE channel:order:1001
```

| Feature | Sharded Pub/Sub | Stream |
|---------|-----------------|--------|
| **Persistence** | No | Yes |
| **Consumer group** | No | Yes |
| **Replay** | No | Yes |
| **Cluster friendly** | Yes, sharded routing | Yes |
| **Latency** | Very low | Low |
| **Use case** | real-time notification | reliable event stream |

Use Sharded Pub/Sub for fire-and-forget real-time notifications. Use Stream when you need persistence, replay, and consumer groups.

### 7.3 Redis Stack

Redis Stack integrates additional modules:

| Module | Capability | Typical Use |
|--------|------------|-------------|
| **RediSearch** | full-text search, secondary index, aggregation | product search, logs |
| **RedisJSON** | native JSON document storage | complex objects, partial update |
| **RedisTimeSeries** | time-series data | metrics, IoT |
| **RedisBloom** | probabilistic structures | deduplication, frequency |

RediSearch example:

```bash
FT.CREATE idx:product ON HASH PREFIX 1 product: SCHEMA
    name TEXT WEIGHT 5.0
    description TEXT
    price NUMERIC SORTABLE
    category TAG

FT.SEARCH idx:product "@name:phone @price:[1000 5000]" SORTBY price ASC LIMIT 0 10
```

RedisJSON example:

```bash
JSON.SET user:1001 $ '{"name":"Alice","age":25,"address":{"city":"Beijing"},"tags":["vip"]}'
JSON.GET user:1001 $.name
JSON.SET user:1001 $.age 26
JSON.ARRAPPEND user:1001 $.tags '"new"'
```

Selection:

| Scenario | Recommended |
|----------|-------------|
| Simple KV cache | Native Redis |
| Frequent partial object updates | RedisJSON |
| Lightweight search under tens of millions of records | RediSearch |
| Heavy search with large-scale analytics | Elasticsearch / OpenSearch |
| Time-series monitoring | RedisTimeSeries, Prometheus, InfluxDB depending on scale |

### 7.4 Other Redis 7 Improvements

**Multi-part AOF**:

```text
appendonlydir/
├── appendonly.aof.1.base.rdb
├── appendonly.aof.1.incr.aof
├── appendonly.aof.2.incr.aof
└── appendonly.aof.manifest
```

Benefits:

- Better disk space control during rewrite.
- Historical segments can be cleaned gradually.
- Rewrite failure does not replace existing valid data.

**Client eviction**:

```bash
maxmemory-clients 1gb
maxmemory-clients 5%
CLIENT NO-EVICT ON
```

This protects Redis from clients with huge output buffers, such as slow subscribers or excessive pipelines.

**ACL selectors**:

```bash
ACL SETUSER app1 ON >password \
    (+GET +SET ~cache:*) \
    (+SUBSCRIBE ~channel:order:*)
```

Selectors allow more flexible permission combinations.

**listpack replacing ziplist**:

Redis 7 fully replaces ziplist with listpack for compact encoding. When answering interviews, mention ziplist as historical context and listpack as the current direction.

---

## 8. Interview Self-Check

### Cache Governance Notes

- Redis is not "just add a cache". You must define write path consistency, invalidation strategy, hot-key handling, fallback, and observability.
- Hot keys and big keys require long-term governance. Do not wait until latency spikes or failover migration fails.
- When business correctness becomes more important than latency, the cache layer must support fallback to DB or read-through bypass.

### Quick Questions

### Q1: Why is Redis often used in front of MySQL instead of replacing MySQL?

**Answer:** Redis is excellent for low-latency access and hot data, but it is not a full replacement for relational storage, complex transactions, relational queries, and strong durability. It is usually an acceleration layer, not the primary system of record.

### Q2: When should you use Hash instead of String for object caching?

**Answer:** Use Hash when the object has many fields and partial updates are common. Use String when the object is read and written as a whole JSON blob. The decision is about access pattern, not just data type names.

### Q3: Why do caches need both TTL and max memory policy?

**Answer:** TTL alone does not prevent memory from being exhausted by sudden writes, hot keys, or dirty traffic. `maxmemory` and eviction policy are required to keep Redis stable under pressure.

### Q4: Why is cache hit ratio important?

**Answer:** Redis QPS alone does not prove that cache design works. Hit ratio shows how much traffic is actually blocked from reaching the database. Low hit ratio means Redis is busy but not effectively protecting the backend.

### Q5: Why is deleting cache often more common than updating cache?

**Answer:** Updating cache requires rebuilding exactly the same derived value as the database state, which can be error-prone. Deleting cache after writing the database is simpler and lets later reads rebuild it.

### Q6: What is the difference between Pipeline and Redis transaction?

**Answer:** Pipeline reduces network round trips by batching commands. It does not provide atomicity. Redis transaction groups commands for sequential execution without interleaving, but it does not provide rollback like relational database transactions.

### Q7: What are hot keys and big keys? Why are they dangerous?

**Answer:** A hot key receives extremely concentrated traffic and creates a single-node bottleneck. A big key stores too much data in one key and can block reads, deletes, replication, or migration. Hot key is mainly traffic distribution; big key is mainly data shape.

### Q8: Why can deleting a big key block Redis? How do you handle it?

**Answer:** Redis command execution is mostly single-threaded, and deleting a huge object may require large synchronous memory freeing. Use key splitting, `UNLINK` for asynchronous free, lazyfree options, and avoid generating big keys at design time.

### Q9: When should Redis no longer be used as a message queue?

**Answer:** If you need replay, complex retry, dead-letter handling, consumer group rebalancing, or long-term backlog, use Kafka, RabbitMQ, or a workflow engine. Redis queues are fine for simple lightweight scenarios, but message semantics become hard to govern at scale.

### Q10: How do you choose `appendfsync always/everysec/no`?

**Answer:** Choose based on how much data loss the business can tolerate. `always` is safest but slowest. `everysec` is the common trade-off, losing at most about one second. `no` is fastest but relies on OS flushing and risks more data loss.

### Deep-Dive Questions

### Q11: What Redis data structures exist and what are their internal implementations?

**Answer:** String uses SDS or integer encoding. List uses quicklist/listpack. Hash uses listpack for small hashes and hashtable for large ones. Set uses intset or hashtable. ZSet uses compact encoding for small sets and skiplist plus hashtable for large sets. Advanced structures include Bitmap, HyperLogLog, Geo, and Stream.

### Q12: What advantages does SDS have over C strings?

**Answer:** SDS stores length, so length lookup is O(1). It is binary safe because it does not rely on `\0` to determine the end. It automatically grows to avoid buffer overflow and uses pre-allocation plus lazy free to reduce reallocations.

### Q13: Why does ZSet use skiplist instead of red-black tree?

**Answer:** Skiplist supports efficient range queries by finding the start and then scanning the bottom level. It is simpler to implement than red-black tree and avoids complex rotations. Redis combines skiplist with hashtable so score lookup is O(1) while rank and range operations remain efficient.

### Q14: What is ziplist cascading update? How does Redis 7 address it?

**Answer:** Ziplist stores the previous entry length in each entry. If a previous entry grows from a one-byte length to a five-byte length, following entries may also need to expand, causing cascading updates. Redis 7 uses listpack, which stores the current entry length and avoids this issue.

### Q15: What are cache penetration, breakdown, and avalanche?

**Answer:** Penetration means querying non-existing data and missing both cache and DB; use Bloom filters or null caching. Breakdown means a hot key expires and concurrent requests hit DB; use singleflight/mutex or logical expiration. Avalanche means many keys expire together; add TTL jitter, multi-level cache, rate limiting, and degradation.

### Q16: How does a Bloom filter work?

**Answer:** A Bloom filter uses a bit array and multiple hash functions. Adding an item sets several bit positions. Querying checks whether all corresponding bits are set. If any bit is zero, the item definitely does not exist. If all are one, it may exist. It has false positives but no false negatives.

### Q17: How does a Redis distributed lock evolve from simple to production-grade?

**Answer:** Start with `SETNX`, then add expiration, then make set and expire atomic with `SET key value NX EX`, then add a unique value and Lua unlock to avoid deleting others' locks, and finally add renewal for long tasks. For multi-node scenarios, Redlock can be considered, but critical correctness should rely on database constraints and idempotent state machines.

### Q18: RDB vs AOF: what are their trade-offs?

**Answer:** RDB is compact and fast to restore, but may lose data between snapshots and fork consumes memory. AOF records write commands and can lose only about one second in `everysec` mode, but the file is larger and recovery is slower. Hybrid persistence combines RDB preamble with incremental AOF and is usually preferred.

### Q19: How does Redis master-replica synchronization work?

**Answer:** The first sync is usually full sync: the master generates an RDB and sends it to the replica while buffering new writes. Normal replication is incremental command propagation. After disconnection, the replica can use replication offset and backlog buffer for partial resync; otherwise it falls back to full sync.

### Q20: How does Sentinel perform failover?

**Answer:** Sentinels ping the master. After timeout, one Sentinel marks subjective down. If enough Sentinels agree, it becomes objective down. A replica is selected based on availability, priority, replication offset, and run ID. It is promoted to master, and other replicas are reconfigured to follow it.

### Q21: How does Redis Cluster route keys?

**Answer:** Redis Cluster divides keyspace into 16,384 slots. A key's slot is `CRC16(key) % 16384`. Clients cache the slot-to-node mapping. If the request goes to the wrong node, Redis returns MOVED or ASK redirection. Hash tags force related keys into the same slot.

### Q22: Why is Redis fast even though command execution is mostly single-threaded?

**Answer:** Redis is fast because it operates in memory, uses efficient data structures, relies on `epoll` for IO multiplexing, and avoids locks and context switching in command execution. Redis 6.0 multi-threading improves network IO, but command execution remains mostly single-threaded.

### Q23: How do you keep cache and database consistent?

**Answer:** The common Cache Aside pattern is update DB first, then delete cache. Delayed double deletion can reduce race windows, but it is not perfect. Stronger approaches include binlog/CDC-based invalidation, versioned cache values, short TTL for critical keys, and fallback reads from the source of truth.

### Q24: How are expired keys deleted?

**Answer:** Redis combines lazy expiration and active expiration. Lazy expiration checks TTL when a key is accessed. Active expiration periodically samples keys with TTL and deletes expired ones. This balances CPU usage and memory cleanup.

### Q25: Is Redis LRU exact?

**Answer:** No. Redis uses approximate LRU by sampling several keys and evicting the oldest among them. Exact LRU would need a global linked list and update on every access, which is too expensive. Redis also supports LFU with logarithmic counters and time decay.

### Q26: Redis transaction vs Lua script?

**Answer:** Redis transaction batches commands and prevents interleaving, but it does not roll back failed commands. Lua scripts execute atomically and can contain conditional logic, so they are better for read-modify-write operations like unlocking, stock deduction, and rate limiting.

### Q27: What is progressive rehash?

**Answer:** Redis uses two hash tables during rehash. It gradually moves buckets from the old table to the new table during normal operations. This avoids a large blocking O(n) rehash for huge dictionaries.

### Q28: What are Redis Cluster limitations and how do you handle cross-slot operations?

**Answer:** Multi-key operations, transactions, and Lua scripts require keys in the same slot. Redis Cluster also supports only DB 0. Use hash tags, such as `{order}:1` and `{order}:2`, to place related keys in the same slot.

### Q29: How do you detect and handle big keys?

**Answer:** Use `redis-cli --bigkeys`, `MEMORY USAGE key`, slow logs, and offline RDB analysis. Big keys can block read/write/delete, cause replication or migration imbalance, and increase memory fragmentation. Split big keys, use `UNLINK`, enable lazy free, and enforce data model limits.

### Q30: Why cannot hot keys be solved only by adding machines?

**Answer:** A hot key is a skewed traffic distribution problem. Adding machines increases total capacity but does not remove a single-key hotspot. Use local cache, request coalescing, logical sharding, multi-replica reads, and rate limiting.

### Q31: When should Redis distributed locks not be used?

**Answer:** Avoid relying only on Redis locks when the business requires strong transactional consistency, long lock duration, atomic updates across multiple resources, or when lock failure can cause financial loss. Use database constraints, transactions, and state machines as the correctness boundary.

### Q32: How do you switch from aggressive caching to correctness-first mode?

**Answer:** Shorten TTLs, force critical reads to read from the primary database, keep write paths as DB-first then cache invalidation, and use binlog/CDC correction. For critical keys, use version checks and make cache bypass a runtime switch.

### Open-Ended Design Questions

### D1: Design a distributed cache system supporting one million reads per second.

**Reference approach:**

- Use multi-level cache: local cache -> Redis Cluster -> database.
- Keep most hot reads in local cache with short TTL and jitter.
- Use Redis Cluster with at least 3 masters and replicas.
- Use request coalescing for hot keys.
- Split hot keys logically when needed.
- Use Cache Aside with DB update followed by cache deletion, plus CDC correction.
- Add fallback: if Redis is unavailable, extend local cache TTL for safe data and rate-limit DB access.
- Monitor hit ratio, latency, memory, big keys, hot keys, connection count, and eviction.

### D2: A Redis Cluster node suddenly has memory explosion and OOM. How do you locate and recover?

**Reference approach:**

- First recover availability through replica failover or manual failover if needed.
- Check memory usage, key count, eviction, client buffers, and blocked clients.
- Find big keys using `redis-cli --bigkeys`, `MEMORY USAGE`, and offline RDB analysis.
- Common causes: huge Hash/Set, unbounded Stream backlog, missing TTL, bad key prefix, slow subscriber output buffer.
- Fix by splitting big keys, setting TTL and maxmemory policy, trimming Streams, using `UNLINK`, and adding alerts at 70% and 85% memory thresholds.

### D3: How would you design cache consistency for a high-value order system?

**Reference approach:**

- Treat the database as the source of truth.
- Use cache only for acceleration, not correctness.
- Write path: update DB transaction first, then delete cache.
- Add CDC/binlog invalidation as a second correction path.
- Use versioned cache values to prevent stale writes.
- For sensitive reads, bypass cache or read primary DB after recent writes.
- Define a consistency SLA and monitoring for stale cache incidents.
