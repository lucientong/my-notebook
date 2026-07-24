# System Design Interview

Language: English | [中文](../算法与数据结构/02-系统设计面试专题.md)

> Purpose: an English playbook for system-design interviews, especially where design discussion meets algorithmic implementation: estimation, hashing, rate limiting, ID generation, pagination, idempotency, and reliability.

---

## Table of Contents

1. [Interview Framework](#1-interview-framework)
2. [Estimation](#2-estimation)
3. [URL Shortener](#3-url-shortener)
4. [Flash Sale System](#4-flash-sale-system)
5. [Feed System](#5-feed-system)
6. [Distributed Cache](#6-distributed-cache)
7. [Chat System](#7-chat-system)
8. [Search Engine](#8-search-engine)
9. [Rate Limiter](#9-rate-limiter)
10. [Notification System](#10-notification-system)
11. [Distributed ID Generation](#11-distributed-id-generation)
12. [Video Platform](#12-video-platform)
13. [Cloud Storage File System](#13-cloud-storage-file-system)
14. [RESTful API Gateway](#14-restful-api-gateway)
15. [Interview Self-Check](#15-interview-self-check)

---

## 1. Interview Framework

### Problem Pattern

A system-design interview is not about producing a perfect architecture. It is about showing a structured way to handle ambiguity, scale, reliability, and tradeoffs.

### Four-Step Flow

| Step | Time | Output |
|---|---:|---|
| Requirements clarification | 3-5 min | Functional scope, non-functional goals, constraints |
| High-level design | 10-15 min | APIs, data model, major components, data flow |
| Deep dive | 15-20 min | Bottlenecks, algorithms, storage, consistency, reliability |
| Scale and tradeoffs | 5-8 min | Failure modes, monitoring, cost, future extensions |

### Interview Invariant

At every point, keep the discussion anchored to requirements and scale. Do not introduce a component unless it solves a stated bottleneck, correctness issue, or operational concern.

### Useful English Phrases

| Situation | Phrase |
|---|---|
| Start | "I will first clarify the product scope and scale assumptions." |
| Estimate | "I will use back-of-the-envelope numbers to size the system." |
| Tradeoff | "This improves read latency, but it increases write amplification." |
| Uncertainty | "I am not fully sure about that product detail, so I will state an assumption and continue." |
| Deep dive | "The most interesting bottleneck here is likely ..." |

---

## 2. Estimation

### Problem Pattern

Use estimation to convert product assumptions into QPS, storage, bandwidth, and shard count.

### Invariant

Every number must come from an explicit assumption. If the interviewer changes the assumption, the formula should still work.

### QPS Estimate

Complexity: `O(1)` time, `O(1)` space.

```go
func estimatePeakQPS(dau int64, actionsPerUserPerDay int64, peakMultiplier int64) int64 {
	avgQPS := dau * actionsPerUserPerDay / 86400
	return avgQPS * peakMultiplier
}
```

```python
def estimate_peak_qps(dau: int, actions_per_user_per_day: int, peak_multiplier: int) -> int:
    avg_qps = dau * actions_per_user_per_day // 86_400
    return avg_qps * peak_multiplier
```

### Quick Reference

| Scenario | Mental Math |
|---|---|
| 10M DAU, 5 actions/day | average about 580 QPS, peak about 1.7K QPS with 3x |
| 100M items/day, 1 KB each | about 100 GB/day, 36 TB/year |
| 10 KB response, 10K QPS | about 100 MB/s outbound |

### Pitfalls

- Confusing average QPS with peak QPS.
- Ignoring replication, indexes, metadata, and retention.
- Overfitting exact numbers; interviews care more about order of magnitude.

---

## 3. URL Shortener

### Problem Pattern

A URL shortener is a read-heavy key-value mapping system with ID generation, redirect latency, cache strategy, and abuse prevention.

### Core Design

| Area | Choice |
|---|---|
| API | create short URL, redirect, optional custom alias |
| Storage | key-value table: short code -> long URL, owner, expiration |
| Cache | hot short code cache with TTL and jitter |
| ID generation | auto-increment segment, Snowflake, or random token with collision check |
| Redirect | 301 for permanent, 302 for tracking-friendly temporary redirect |

### Base62 Encoding

Invariant: repeatedly divide by 62; each remainder maps to a URL-safe character.

Complexity: `O(log_62 n)` time, `O(log_62 n)` space.

```go
func encodeBase62(num int64) string {
	const chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
	if num == 0 {
		return "0"
	}
	buf := []byte{}
	for num > 0 {
		buf = append(buf, chars[num%62])
		num /= 62
	}
	for i, j := 0, len(buf)-1; i < j; i, j = i+1, j-1 {
		buf[i], buf[j] = buf[j], buf[i]
	}
	return string(buf)
}
```

```python
def encode_base62(num: int) -> str:
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    if num == 0:
        return "0"
    out: list[str] = []
    while num:
        out.append(chars[num % 62])
        num //= 62
    return "".join(reversed(out))
```

### Pitfalls

- Random codes need collision detection.
- Sequential IDs are easy to guess; add randomization if abuse matters.
- Hot links can overload a single cache key; use local cache or cache-key replication.

---

## 4. Flash Sale System

### Problem Pattern

A flash sale system protects scarce inventory under extreme burst traffic. The design is mostly about filtering invalid traffic before it reaches the database.

### Core Design

| Layer | Responsibility |
|---|---|
| CDN/static page | serve static content and absorb reads |
| Gateway | rate limit, auth, anti-bot, request signing |
| Local cache | sold-out flag and product metadata |
| Redis | atomic stock decrement and duplicate purchase check |
| Queue | async order creation and traffic smoothing |
| Database | final order truth and reconciliation |

### Atomic Stock Check Shape

Use one atomic operation in Redis or an equivalent single-writer path. The code below shows the core invariant in local memory for interview explanation.

Invariant: stock cannot go below zero; a user can reserve at most once.

Complexity: `O(1)` average time, `O(u)` space for user IDs.

```go
type StockGuard struct {
	stock int
	seen  map[int64]bool
}

func NewStockGuard(stock int) *StockGuard {
	return &StockGuard{stock: stock, seen: map[int64]bool{}}
}

func (g *StockGuard) Reserve(userID int64) bool {
	if g.seen[userID] || g.stock <= 0 {
		return false
	}
	g.seen[userID] = true
	g.stock--
	return true
}
```

```python
class StockGuard:
    def __init__(self, stock: int):
        self.stock = stock
        self.seen: set[int] = set()

    def reserve(self, user_id: int) -> bool:
        if user_id in self.seen or self.stock <= 0:
            return False
        self.seen.add(user_id)
        self.stock -= 1
        return True
```

### Pitfalls

- Redis decrement success but queue publish failure requires compensation.
- Database remains the final source of truth.
- Rate limiting must be layered: per user, per IP, per product, and global.

---

## 5. Feed System

### Problem Pattern

Feed systems trade off read latency and write amplification.

### Design Choices

| Mode | Strength | Weakness | Best For |
|---|---|---|---|
| Push/fanout-on-write | fast reads | huge write amplification for celebrities | normal users |
| Pull/fanout-on-read | cheap writes | expensive reads | celebrities |
| Hybrid | balanced | more complex | production systems |

### Invariant

The user's feed timeline must be monotonic by ranking key, usually time or score. Pagination should not skip or duplicate items when new posts arrive.

### K-Way Merge Timeline

Complexity: `O(k log f)` time for `k` returned items and `f` followed authors, `O(f)` heap space.

```go
import "container/heap"

type FeedItem struct {
	Author int
	Index  int
	Time   int64
}

type FeedHeap []FeedItem

func (h FeedHeap) Len() int           { return len(h) }
func (h FeedHeap) Less(i, j int) bool { return h[i].Time > h[j].Time }
func (h FeedHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *FeedHeap) Push(x any)        { *h = append(*h, x.(FeedItem)) }
func (h *FeedHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}

func mergeFeeds(feeds [][]int64, limit int) []int64 {
	h := &FeedHeap{}
	heap.Init(h)
	for author, posts := range feeds {
		if len(posts) > 0 {
			heap.Push(h, FeedItem{Author: author, Index: 0, Time: posts[0]})
		}
	}
	ans := []int64{}
	for h.Len() > 0 && len(ans) < limit {
		item := heap.Pop(h).(FeedItem)
		ans = append(ans, item.Time)
		next := item.Index + 1
		if next < len(feeds[item.Author]) {
			heap.Push(h, FeedItem{Author: item.Author, Index: next, Time: feeds[item.Author][next]})
		}
	}
	return ans
}
```

```python
import heapq


def merge_feeds(feeds: list[list[int]], limit: int) -> list[int]:
    heap: list[tuple[int, int, int]] = []
    for author, posts in enumerate(feeds):
        if posts:
            heapq.heappush(heap, (-posts[0], author, 0))
    ans: list[int] = []
    while heap and len(ans) < limit:
        neg_time, author, idx = heapq.heappop(heap)
        ans.append(-neg_time)
        nxt = idx + 1
        if nxt < len(feeds[author]):
            heapq.heappush(heap, (-feeds[author][nxt], author, nxt))
    return ans
```

### Pitfalls

- Celebrity accounts cause write amplification in push mode.
- Cursor-based pagination is better than offset pagination for mutable timelines.

---

## 6. Distributed Cache

### Problem Pattern

Distributed cache design is about routing keys, handling node changes, preventing hot keys, and preserving acceptable consistency.

### Consistent Hashing

Invariant: a key is assigned to the first virtual node clockwise on the hash ring. Adding or removing a node affects only neighboring ranges.

Complexity: build `O(v log v)`, lookup `O(log v)`, where `v` is the number of virtual nodes.

```go
import (
	"hash/fnv"
	"sort"
	"strconv"
)

type ConsistentHash struct {
	ring map[uint32]string
	keys []uint32
}

func hashKey(s string) uint32 {
	h := fnv.New32a()
	_, _ = h.Write([]byte(s))
	return h.Sum32()
}

func NewConsistentHash(nodes []string, replicas int) *ConsistentHash {
	ch := &ConsistentHash{ring: map[uint32]string{}}
	for _, node := range nodes {
		for i := 0; i < replicas; i++ {
			key := hashKey(node + "#" + strconv.Itoa(i))
			ch.ring[key] = node
			ch.keys = append(ch.keys, key)
		}
	}
	sort.Slice(ch.keys, func(i, j int) bool { return ch.keys[i] < ch.keys[j] })
	return ch
}

func (c *ConsistentHash) Get(key string) string {
	if len(c.keys) == 0 {
		return ""
	}
	h := hashKey(key)
	i := sort.Search(len(c.keys), func(i int) bool { return c.keys[i] >= h })
	return c.ring[c.keys[i%len(c.keys)]]
}
```

```python
import bisect
import hashlib


class ConsistentHash:
    def __init__(self, nodes: list[str], replicas: int):
        self.ring: dict[int, str] = {}
        self.keys: list[int] = []
        for node in nodes:
            for i in range(replicas):
                key = self._hash(f"{node}#{i}")
                self.ring[key] = node
                self.keys.append(key)
        self.keys.sort()

    def _hash(self, value: str) -> int:
        return int(hashlib.md5(value.encode()).hexdigest()[:8], 16)

    def get(self, key: str) -> str:
        if not self.keys:
            return ""
        h = self._hash(key)
        i = bisect.bisect_left(self.keys, h)
        return self.ring[self.keys[i % len(self.keys)]]
```

### Pitfalls

- Use virtual nodes to reduce imbalance.
- Cache penetration, breakdown, and avalanche require different treatments: null caching/Bloom filter, mutex/singleflight, and TTL jitter.

---

## 7. Chat System

### Problem Pattern

Chat systems focus on connection management, message ordering, delivery guarantees, offline sync, and fanout.

### Core Design

| Concern | Common Choice |
|---|---|
| Connection | stateless WebSocket gateways with session registry |
| Message ID | per-conversation sequence number |
| Persistence | write message before ACK |
| Delivery | online push plus offline inbox |
| Ordering | sequence-based incremental sync |

### Missing Message Detection

Invariant: if the client has `lastSeq`, the server returns messages with sequence greater than `lastSeq`.

Complexity: `O(log n + k)` with indexed storage, represented below as `O(n)` scan for clarity.

```go
type Message struct {
	Seq  int64
	Body string
}

func syncMessages(messages []Message, lastSeq int64, limit int) []Message {
	ans := []Message{}
	for _, msg := range messages {
		if msg.Seq > lastSeq {
			ans = append(ans, msg)
			if len(ans) == limit {
				break
			}
		}
	}
	return ans
}
```

```python
from dataclasses import dataclass


@dataclass
class Message:
    seq: int
    body: str


def sync_messages(messages: list[Message], last_seq: int, limit: int) -> list[Message]:
    ans: list[Message] = []
    for msg in messages:
        if msg.seq > last_seq:
            ans.append(msg)
            if len(ans) == limit:
                break
    return ans
```

### Pitfalls

- ACK before persistence can lose messages.
- Group chat needs different strategies for small and large groups.
- Gateway restart should trigger reconnect and incremental sync.

---

## 8. Search Engine

### Problem Pattern

Search design combines ingestion, text analysis, inverted index, ranking, query execution, and near-real-time refresh.

### Core Design

| Component | Responsibility |
|---|---|
| Crawler/ingestion | collect and normalize documents |
| Analyzer | tokenize, lowercase, remove stop words, segment language-specific text |
| Inverted index | term -> posting list |
| Ranker | BM25, freshness, personalization, business features |
| Segment merge | keep search fast while supporting updates |

### Inverted Index Core

Invariant: every term points to the set of document IDs containing that term.

Complexity: build `O(total_terms)`, query intersection depends on posting-list lengths.

```go
import "strings"

func buildIndex(docs map[int]string) map[string]map[int]bool {
	index := map[string]map[int]bool{}
	for id, text := range docs {
		for _, term := range strings.Fields(strings.ToLower(text)) {
			if index[term] == nil {
				index[term] = map[int]bool{}
			}
			index[term][id] = true
		}
	}
	return index
}
```

```python
def build_index(docs: dict[int, str]) -> dict[str, set[int]]:
    index: dict[str, set[int]] = {}
    for doc_id, text in docs.items():
        for term in text.lower().split():
            index.setdefault(term, set()).add(doc_id)
    return index
```

### Pitfalls

- Chinese search needs segmentation; whitespace tokenization is not enough.
- Real search engines use immutable segments and background merge.
- Ranking quality matters as much as index correctness.

---

## 9. Rate Limiter

### Problem Pattern

Rate limiting protects services from overload and abuse. Common algorithms: fixed window, sliding window, token bucket, and leaky bucket.

### Token Bucket

Invariant: tokens refill over time up to capacity; each accepted request consumes one token.

Complexity: `O(1)` time, `O(1)` space.

```go
import "time"

type TokenBucket struct {
	capacity int
	tokens   float64
	rate     float64
	last     time.Time
}

func NewTokenBucket(capacity int, rate float64) *TokenBucket {
	return &TokenBucket{capacity: capacity, tokens: float64(capacity), rate: rate, last: time.Now()}
}

func (b *TokenBucket) Allow(now time.Time) bool {
	elapsed := now.Sub(b.last).Seconds()
	b.tokens += elapsed * b.rate
	if b.tokens > float64(b.capacity) {
		b.tokens = float64(b.capacity)
	}
	b.last = now
	if b.tokens < 1 {
		return false
	}
	b.tokens--
	return true
}
```

```python
import time


class TokenBucket:
    def __init__(self, capacity: int, rate: float):
        self.capacity = capacity
        self.tokens = float(capacity)
        self.rate = rate
        self.last = time.time()

    def allow(self, now: float | None = None) -> bool:
        now = time.time() if now is None else now
        elapsed = now - self.last
        self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
        self.last = now
        if self.tokens < 1:
            return False
        self.tokens -= 1
        return True
```

### Pitfalls

- Distributed rate limiting trades precision for latency.
- Redis cluster scripts must keep related keys in the same slot.
- If the limiter fails, decide fail-open or fail-closed based on business risk.

---

## 10. Notification System

### Problem Pattern

Notification systems route events to multiple channels while controlling deduplication, priority, user preferences, and provider failures.

### Core Design

| Stage | Responsibility |
|---|---|
| Event intake | validate and normalize events |
| Preference check | user channel opt-in, quiet hours, locale |
| Frequency control | per-user and per-type limits |
| Template rendering | localized content |
| Provider dispatch | push, email, SMS, webhook |
| Retry/DLQ | exponential backoff and manual recovery |

### Deduplication Window

Invariant: the same dedup key should be accepted at most once within the TTL window.

Complexity: `O(1)` average time, `O(n)` space for active keys.

```go
import "time"

type Deduper struct {
	ttl  time.Duration
	seen map[string]time.Time
}

func NewDeduper(ttl time.Duration) *Deduper {
	return &Deduper{ttl: ttl, seen: map[string]time.Time{}}
}

func (d *Deduper) Accept(key string, now time.Time) bool {
	if exp, ok := d.seen[key]; ok && now.Before(exp) {
		return false
	}
	d.seen[key] = now.Add(d.ttl)
	return true
}
```

```python
class Deduper:
    def __init__(self, ttl_seconds: float):
        self.ttl_seconds = ttl_seconds
        self.seen: dict[str, float] = {}

    def accept(self, key: str, now: float) -> bool:
        if key in self.seen and now < self.seen[key]:
            return False
        self.seen[key] = now + self.ttl_seconds
        return True
```

### Pitfalls

- Retry can create duplicate notifications without idempotency.
- Notification fatigue is a product and ranking problem, not only an infrastructure problem.

---

## 11. Distributed ID Generation

### Problem Pattern

Distributed IDs must balance uniqueness, ordering, availability, and operational simplicity.

### Options

| Approach | Strength | Weakness |
|---|---|---|
| Database auto-increment | simple | central bottleneck |
| Segment allocation | high throughput | depends on DB for segment refill |
| Snowflake | decentralized, time-sortable | clock rollback risk |
| UUID | decentralized | large and not naturally ordered |

### Snowflake Shape

Invariant: within the same millisecond and worker, the sequence number must be unique.

Complexity: `O(1)` time, `O(1)` space.

```go
type Snowflake struct {
	workerID int64
	lastMs   int64
	seq      int64
}

func (s *Snowflake) Next(nowMs int64) int64 {
	if nowMs == s.lastMs {
		s.seq = (s.seq + 1) & 4095
		if s.seq == 0 {
			nowMs++
		}
	} else {
		s.seq = 0
	}
	s.lastMs = nowMs
	return (nowMs << 22) | (s.workerID << 12) | s.seq
}
```

```python
class Snowflake:
    def __init__(self, worker_id: int):
        self.worker_id = worker_id
        self.last_ms = -1
        self.seq = 0

    def next_id(self, now_ms: int) -> int:
        if now_ms == self.last_ms:
            self.seq = (self.seq + 1) & 4095
            if self.seq == 0:
                now_ms += 1
        else:
            self.seq = 0
        self.last_ms = now_ms
        return (now_ms << 22) | (self.worker_id << 12) | self.seq
```

### Pitfalls

- Clock rollback must be handled: wait, use backup worker IDs, or reject.
- Worker ID assignment must be unique.
- Segment mode needs double buffering to avoid refill latency spikes.

---

## 12. Video Platform

### Problem Pattern

A video platform is dominated by large-file upload, transcoding pipeline, CDN distribution, metadata search, and playback quality.

### Core Design

| Area | Design Choice |
|---|---|
| Upload | multipart upload with resumability |
| Dedup | file fingerprint before upload |
| Processing | async transcoding queue |
| Storage | object storage plus metadata DB |
| Playback | HLS/DASH, adaptive bitrate, CDN |

### Chunk Planning

Invariant: chunks cover `[0, file_size)` without overlap or gaps.

Complexity: `O(number_of_chunks)` time and space.

```go
type Chunk struct {
	Index int
	Start int64
	End   int64
}

func planChunks(size int64, chunkSize int64) []Chunk {
	chunks := []Chunk{}
	for start, idx := int64(0), 0; start < size; start, idx = start+chunkSize, idx+1 {
		end := start + chunkSize
		if end > size {
			end = size
		}
		chunks = append(chunks, Chunk{Index: idx, Start: start, End: end})
	}
	return chunks
}
```

```python
from dataclasses import dataclass


@dataclass
class Chunk:
    index: int
    start: int
    end: int


def plan_chunks(size: int, chunk_size: int) -> list[Chunk]:
    chunks: list[Chunk] = []
    start = 0
    index = 0
    while start < size:
        end = min(size, start + chunk_size)
        chunks.append(Chunk(index, start, end))
        start = end
        index += 1
    return chunks
```

### Pitfalls

- Direct MP4 serving is less flexible than HLS/DASH for adaptive bitrate.
- Transcoding should be idempotent because workers can retry.
- CDN cache invalidation and signed URLs matter for paid or private content.

---

## 13. Cloud Storage File System

### Problem Pattern

Cloud storage combines file-tree metadata, object storage, permission checks, resumable upload, pagination, and conflict handling.

### Path Normalization

Invariant: normalized paths never contain empty segments, `.` segments, or unresolved `..` segments.

Complexity: `O(number_of_segments)` time, `O(number_of_segments)` space.

```go
import "strings"

func normalizePath(path string) string {
	stack := []string{}
	for _, part := range strings.Split(path, "/") {
		if part == "" || part == "." {
			continue
		}
		if part == ".." {
			if len(stack) > 0 {
				stack = stack[:len(stack)-1]
			}
			continue
		}
		stack = append(stack, part)
	}
	return "/" + strings.Join(stack, "/")
}
```

```python
def normalize_path(path: str) -> str:
    stack: list[str] = []
    for part in path.split("/"):
        if part in ("", "."):
            continue
        if part == "..":
            if stack:
                stack.pop()
            continue
        stack.append(part)
    return "/" + "/".join(stack)
```

### Pitfalls

- Metadata and object bytes may become inconsistent; use state machines and cleanup jobs.
- Pagination should be cursor-based for large folders.
- Upload completion must be idempotent.

---

## 14. RESTful API Gateway

### Problem Pattern

An API gateway centralizes authentication, authorization, rate limiting, routing, request IDs, logging, pagination, error format, and streaming support.

### Cursor Pagination

Invariant: a cursor encodes the last seen sort key, so the next page starts strictly after it.

Complexity: `O(limit)` after indexed seek in storage, represented below as a scan for clarity.

```go
type Row struct {
	ID        int64
	CreatedAt int64
}

func pageAfter(rows []Row, cursorCreatedAt int64, cursorID int64, limit int) []Row {
	ans := []Row{}
	for _, row := range rows {
		if row.CreatedAt < cursorCreatedAt || (row.CreatedAt == cursorCreatedAt && row.ID < cursorID) {
			ans = append(ans, row)
			if len(ans) == limit {
				break
			}
		}
	}
	return ans
}
```

```python
from dataclasses import dataclass


@dataclass
class Row:
    id: int
    created_at: int


def page_after(rows: list[Row], cursor_created_at: int, cursor_id: int, limit: int) -> list[Row]:
    ans: list[Row] = []
    for row in rows:
        if row.created_at < cursor_created_at or (row.created_at == cursor_created_at and row.id < cursor_id):
            ans.append(row)
            if len(ans) == limit:
                break
    return ans
```

### Pitfalls

- Offset pagination becomes slow and unstable on mutable datasets.
- Error models should be consistent: code, message, request ID, and details.
- Streaming endpoints need cancellation and heartbeat handling.

---

## 15. Interview Self-Check

### 15.1 Common Questions

- What are the functional and non-functional requirements?
- What are your DAU, QPS, storage, and bandwidth assumptions?
- Where is the read/write bottleneck?
- Which component is stateful, and how does it fail over?
- What is the cache invalidation strategy?
- What consistency level is required, and why?
- How do you make retries idempotent?
- What should be monitored and alerted?

### 15.2 System-Specific Prompts

| System | Key Follow-Up |
|---|---|
| URL shortener | How do you handle hot links and unguessable aliases? |
| Flash sale | How do you prevent overselling if queue publish fails? |
| Feed | How do you handle celebrities and unfollow cleanup? |
| Distributed cache | How do you scale out without moving most keys? |
| Chat | How do clients detect missing messages? |
| Search | How do you support near-real-time indexing? |
| Rate limiter | What happens if Redis is unavailable? |
| Notification | How do you avoid duplicate or excessive notifications? |
| ID generator | How do you handle clock rollback? |
| API gateway | How do you design pagination, idempotency, and errors? |

### 15.3 Closing Checklist

- Requirements clarified.
- Scale estimated.
- APIs and data model described.
- High-level architecture drawn verbally.
- One or two core components deep-dived.
- Algorithmic parts explained with invariants and complexity.
- Failure modes, monitoring, and tradeoffs discussed.
