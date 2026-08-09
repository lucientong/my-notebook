# Go In Depth

Language: English | [中文](../后端知识库/01-Go语言深入.md)

---

## Table of Contents

### Language Fundamentals and Core Types
1. [Core Type Internals](#1-core-type-internals)
2. [Language Semantics and Expressions](#2-language-semantics-and-expressions)
3. [container/heap Priority Queue](#3-containerheap-priority-queue)

### Concurrency and Runtime
4. [GMP Scheduler](#4-gmp-scheduler)
5. [Goroutine and Channel](#5-goroutine-and-channel)
6. [Context](#6-context)
7. [sync Primitives](#7-sync-primitives)

### Memory and Performance
8. [defer](#8-defer)
9. [Error Handling](#9-error-handling)
10. [Memory Management and GC](#10-memory-management-and-gc)
11. [Profiling and Performance Tuning](#11-profiling-and-performance-tuning)

### Web Practice and Interview Review
12. [Gin Practice](#12-gin-practice)
13. [Practical Cases](#13-practical-cases)
14. [Interview Self-Check](#14-interview-self-check)

---

## 1. Core Type Internals

### 1.1 Slice Internals and Growth ⭐⭐⭐

A slice is a descriptor that points to an underlying array.

```go
// runtime/slice.go, simplified
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

```text
slice variable                  underlying array
┌─────────────────┐             ┌───┬───┬───┬───┬───┐
│ array ──────────────────────> │ 1 │ 2 │ 3 │ 0 │ 0 │
│ len = 3         │             └───┴───┴───┴───┴───┘
│ cap = 5         │
└─────────────────┘
```

Important implication: a slice is often called a reference type, but the slice header itself is passed by value. Passing a slice copies the pointer, length, and capacity, not the underlying array.

#### Slice Parameter Semantics ⭐⭐

Go does **not** have C++-style pass-by-reference. When a slice is passed to a function, Go still passes the slice **by value**: it copies the header (pointer + len + cap), not the underlying array. The caller and callee headers usually point to the **same underlying array**.

```text
caller: a (header) ──→ underlying array
callee: s (header copy) ──→ same underlying array
```

```go
func modify(s []int) {
    s[0] = 99         // visible to caller: mutates shared array
    s = append(s, 4)  // by default only changes callee header copy
}

func main() {
    a := []int{1, 2, 3}
    modify(a)
    fmt.Println(a) // [99, 2, 3] — elements changed, but 4 was not appended
}
```

| Operation | Visible to caller? |
|-----------|-------------------|
| `s[i] = x` | Yes, mutates shared underlying array |
| `append` without reallocation | May write into shared array, but caller `len` unchanged |
| `append` with reallocation | No, callee points to a new array |
| `s = s[:0]`, `s = nil`, etc. | No, only callee header copy changes |

To change the caller's **len/cap** (for example after append), write the result back:

```go
// Option 1: return slice (most idiomatic)
func grow(s []int) []int {
    return append(s, 4)
}
a = grow(a)

// Option 2: *[]T (works, but less idiomatic)
func grow(s *[]int) {
    *s = append(*s, 4)
}
grow(&a)

// Option 3: pointer receiver (for example container/heap Push/Pop)
func (h *MinHeap) Push(x any) { /* mutates len of the slice variable h */ }
```

Rule of thumb: mutating elements needs only value passing; mutating the slice header itself needs `return`, `*[]T`, or a pointer receiver. `&slice` works, but `return append(...)` is preferred in Go.

Two common `make` patterns:

| Pattern | Meaning | Use Case |
|---------|---------|----------|
| `make([]T, n)` | length and capacity are both `n` | Fill by index: `arr[i] = v` |
| `make([]T, 0, n)` | empty slice with preallocated capacity | Build with `append` |

```go
dp := make([]int, n+1)
for i := 0; i <= n; i++ {
    dp[i] = compute(i)
}

results := make([][]string, 0, len(m))
for _, v := range m {
    results = append(results, v)
}
```

Growth policy in Go 1.18+:

```text
If old capacity < 256:
  roughly double.

If old capacity >= 256:
  grow by a smoother factor, roughly 1.25x with transition.

Final capacity is rounded by allocator size class.
```

Before Go 1.18, the commonly cited threshold was 1024. In interviews, mention version differences rather than giving a single absolute rule.

#### Common Slice Pitfalls

**Pitfall 1: slicing shares the underlying array**

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3] // b = [2, 3], shares underlying array with a

b[0] = 99
fmt.Println(a) // [1, 99, 3, 4, 5]

// safe approach: use copy
c := make([]int, len(b))
copy(c, b)
```

> **Note:** a small slice taken from a large array may still retain the entire underlying array, preventing GC until no references remain. Use `copy` when you need independent memory.

**Pitfall 2: append may stop sharing the underlying array**

```go
a := make([]int, 3, 5) // len=3, cap=5
b := a[0:3]

b = append(b, 4) // capacity enough, a and b still share
b = append(b, 5) // capacity enough
b = append(b, 6) // capacity exceeded, b reallocates, no longer shares with a
```

**Pitfall 3: nil slice vs empty slice**

```go
var s1 []int        // nil slice: s1 == nil, len=0, cap=0
s2 := []int{}       // empty slice: s2 != nil, len=0, cap=0
s3 := make([]int, 0) // empty slice: s3 != nil, len=0, cap=0

// all three support append and range, but JSON differs:
// nil slice -> null
// empty slice -> []
```

**Pitfall 4: mixing `make` with len and then `append`**

```go
m := map[string][]int{ /* ... */ }

// anti-pattern: pre-filled len(m) nil slots, append goes to the tail
results := make([][]int, len(m))
for _, v := range m {
    results = append(results, v)
}
// result: [[],[],[],[1,2,3],...] leading slots stay zero-value

// correct: match pattern B
results := make([][]int, 0, len(m))
for _, v := range m {
    results = append(results, v)
}
```

**Pitfall 5: in-place writes while ranging over a slice**

In `for i, v := range s`, `v` is a copy, but `s[i] = x` or maintaining a separate write index `s[write] = v` is safe. For in-place two-pointer algorithms such as deduplication, the invariant is **read index >= write index**, so writes do not overwrite unread data. This differs from deleting while iterating with `append(s[:i], s[i+1:]...)`, which shifts elements and can skip entries.

### 1.2 Arrays and Map Key Restrictions

Arrays are values. `[3]int` and `[4]int` are different types. Arrays can be map keys if their element type is comparable.

Map keys must be comparable:

```go
map[string]int{}      // ok
map[int64]string{}    // ok
map[[2]int]string{}   // ok

// invalid:
// map[[]int]string{}
// map[map[string]int]string{}
// map[func()]string{}
```

Slices, maps, and functions are not comparable, so they cannot be map keys.

Typical pattern: use fixed-size arrays such as `[26]int` or `[128]int` as keys for frequency vectors or bitmap-like signatures when you need comparable keys without string sorting overhead.

### 1.3 Map Internals ⭐⭐⭐

Go map is implemented as a **hash table with buckets**. Each map has an `hmap` header that manages `2^B` buckets. Each bucket stores up to 8 key-value pairs, with overflow buckets linked when collisions exceed bucket capacity.

#### Hash and Bucket Selection

Runtime computes one 64-bit hash per key and splits it:

```text
hash = 0x1A 2B 3C 4D 5E 6F 70 81
       └high 8┘ └────low B bits pick bucket────┘

high 8 bits -> tophash (compare 1 byte first, then full key)
low B bits  -> bucket_index = hash & (2^B - 1)
```

Example with `B = 2` (4 buckets, mask = 3):

```text
hash("alice") = ...70 81
  bucket_index = 81 & 3 = 1
  tophash      = 0x81

hash("bob") = ...70 01
  bucket_index = 1 & 3 = 1   // different keys, same bucket — normal collision
  tophash      = 0x01
```

Collisions are handled with 8 slots per bucket plus overflow bucket chains. Eight slots is a fixed runtime choice for cache-friendly sequential scans; beyond that, overflow buckets are linked.

#### Underlying Structures

```go
// runtime/map.go (simplified)
type hmap struct {
    count     int
    flags     uint8
    B         uint8          // number of buckets = 2^B
    noverflow uint16
    hash0     uint32         // hash seed, randomized per map
    buckets    unsafe.Pointer
    oldbuckets unsafe.Pointer // old bucket array during growth
    nevacuate  uintptr        // evacuation progress
}

// each bucket stores up to 8 key-value pairs
type bmap struct {
    tophash [8]uint8
    // followed by 8 keys and 8 values (layout fixed at compile time)
    // keys   [8]keytype
    // values [8]valuetype
    // overflow *bmap
}
```

```text
hmap
┌──────────────┐
│ count = 5    │
│ B = 2        │  → 2^2 = 4 buckets
│ buckets ─────────→ ┌────────────┐
│              │     │ bucket 0   │
│              │     │ tophash[8] │
│              │     │ keys[8]    │
│              │     │ values[8]  │
│              │     │ overflow ──────→ overflow bucket
│              │     ├────────────┤
│              │     │ bucket 1   │
│              │     ├────────────┤
│              │     │ bucket 2   │
│              │     ├────────────┤
│              │     │ bucket 3   │
│              │     └────────────┘
└──────────────┘
```

#### Lookup, Insert, and Delete

Lookup:

```text
key = "hello"
    │
    ▼
1. hash = hash("hello", hash0)
2. bucket_index = hash & (2^B - 1)
3. tophash = hash >> (64 - 8)
4. scan tophash[0..7] in the bucket:
   ├── match -> compare full key -> return value if equal
   ├── no match -> next slot
   └── all 8 empty/mismatch -> follow overflow chain -> zero value if not found
```

Insert: same as lookup until the key is located; overwrite if key exists, use empty slot if available, otherwise write to an overflow bucket.

Delete: locate the key and mark the slot deleted; runtime may shrink when load drops (internal detail).

#### Growth Mechanism

Trigger conditions:

```text
Condition 1: load factor > 6.5 (count / 2^B > 6.5)
→ double bucket count (2^B -> 2^(B+1))

Condition 2: too many overflow buckets
(noverflow > 2^B and B < 15, or noverflow > 2^15)
→ same-size reorganization to reduce sparse overflow buckets
```

Incremental evacuation:

```text
During growth, oldbuckets is retained. Each later read/write migrates 1-2 old buckets
into the new bucket array. oldbuckets is freed when done. Cost is spread across access,
not paid in one stop-the-world rehash.
```

#### Comparison with Java HashMap

Java 8+ uses linked lists per bucket; when chain length reaches 8 and table capacity is at least 64, the bucket is converted to a **red-black tree**, giving O(log n) worst case per bucket.

Go **does not treeify**. It controls collision chain length with:

| | Java HashMap | Go map |
|---|---|---|
| Collision structure | linked list -> red-black tree | fixed 8 slots + overflow bucket chain |
| Per-bucket scan | O(n) list or O(log n) tree | O(1), at most 8 slots |
| Anti-degradation | treeify | load-factor growth + overflow reorganization + random `hash0` |

**Three controls in Go:**

1. **Fixed 8 slots per bucket**: not a long list of n nodes, but `bucket -> overflow bucket -> ...`, at most 8 comparisons per hop.
2. **Load factor > 6.5 triggers doubling**: more buckets and rehashing shorten overflow chains. 6.5 is average elements per bucket (each bucket holds up to 8), not Java's 0.75 slot load factor.
3. **Too many overflow buckets triggers same-size reorganization**: re-flattens chains even when total element count is not over the load threshold.

**Can Go guarantee non-degradation to O(n)?** Not in the strict sense. If every key lands in the same bucket (collision attack or bad hash), the overflow chain can reach O(n/8) buckets, so total cost is still O(n), with a small constant (at most 8 slots plus tophash filtering per hop). Go trades a theoretical O(log n) worst-case guarantee for simpler implementation and tighter memory layout, relying on hash spread and early growth for O(1) average case.

#### Iteration Is Unordered

Map iteration order is not logically sorted because hash table layout is not ordered. Go also intentionally randomizes iteration order so code cannot depend on accidental ordering.

Do not write logic that assumes stable `for range m` order across runs.

#### Concurrency Rules

```go
// map is not safe for concurrent read + write
m := map[string]int{}
go func() { m["a"] = 1 }()
go func() { _ = m["a"] }()
// fatal error: concurrent map read and map write
```

Interview trap: Go detects concurrent map writes at runtime via `flags`. It throws `concurrent map writes`, which is a **fatal error** and cannot be recovered with `recover`.

Common options:

- `sync.Mutex` for general protection.
- `sync.RWMutex` for read-heavy workloads.
- `sync.Map` for specific cache-like patterns.
- Sharded maps when you need high write concurrency.

Concurrent reads are safe only when there is no concurrent writer.

Common use cases:

**1. `sync.Mutex`: simple mixed read/write state**

```go
type Counter struct {
    mu sync.Mutex
    m  map[string]int
}

func (c *Counter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key]++
}

func (c *Counter) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.m[key]
}
```

**2. `sync.RWMutex`: read-heavy state**

```go
type Cache struct {
    mu sync.RWMutex
    m  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.m[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key] = value
}
```

**3. `sync.Map`: mostly read, stable keys, write once/read many**

```go
var users sync.Map // key: userID, value: *User

func GetUser(id string) (*User, bool) {
    v, ok := users.Load(id)
    if !ok {
        return nil, false
    }
    return v.(*User), true
}

func SetUser(id string, u *User) {
    users.Store(id, u)
}
```

`sync.Map` fits caches, registries, and configuration snapshots that are read often and deleted rarely. If you need multi-key invariants, `Mutex + map` is usually clearer. See [7.6 sync.Map](#76-syncmap) for the read/dirty internals and comparison with `Mutex + map`.

**4. Sharded map: high write concurrency**

```go
const shardCount = 32

type shard struct {
    mu sync.Mutex
    m  map[string]int
}

type ShardedMap struct {
    shards [shardCount]shard
}

func fnv32(key string) uint32 {
    h := fnv.New32a() // import "hash/fnv"
    _, _ = h.Write([]byte(key))
    return h.Sum32()
}

func (sm *ShardedMap) Inc(key string) {
    s := &sm.shards[fnv32(key)%shardCount]
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key]++
}
```

The idea is to hash keys into different shards, each with its own lock. It reduces write contention but complicates cross-shard atomic operations.

### 1.4 Interface Internals ⭐⭐⭐

A non-empty interface is represented by type information and data pointer.

```go
// simplified
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```

An empty interface stores a runtime type pointer and data pointer.

```go
type eface struct {
    typ  *_type
    data unsafe.Pointer
}
```

The classic nil pitfall:

```go
var p *MyError = nil
var err error = p

fmt.Println(err == nil) // false
```

The interface value is non-nil because it contains type information, even though the underlying pointer is nil.

Correct guideline:

- Return `nil` directly when there is no error.
- Avoid returning typed nil pointers as interface values.

### 1.5 string and []byte

String is immutable. `[]byte` is mutable.

#### Copy conversion vs zero-copy

**Standard conversion (use by default)**

```go
s := "hello"
b := []byte(s)  // allocates a new slice, copies bytes
s2 := string(b) // allocates a new string, copies bytes
```

Standard conversion always copies because string must stay immutable:

```go
b := []byte(s)
b[0] = 'H'      // only b changes, s stays "hello"
s2 := string(b) // s2 is "Hello", not sharing memory with b
```

Zero-copy sharing would let slice mutation break string immutability.

**Unsafe zero-copy (Go 1.22+: `unsafe.String` / `unsafe.StringData`)**

```go
func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

| | `[]byte(s)` / `string(b)` | unsafe zero-copy |
|---|---|---|
| Copy | yes, new allocation each time | no, shares backing bytes |
| Safety | language guarantees | you must guarantee lifetime and no mutation |
| Mutating []byte affects string | no | **yes**, same memory |
| After append | N/A (already copied) | string from `bytesToString` may become invalid |
| Typical use | normal application code | parser hot paths, read-only scans |

**Why "recommended API, use with caution"?**

- **Recommended**: since Go 1.22, `unsafe.String` and `unsafe.StringData` are the official zero-copy APIs, cleaner than old `reflect.StringHeader` tricks.
- **Use with caution**: you bypass the safety of standard conversion and must ensure:
  1. never mutate the slice returned from `stringToBytes`
  2. never mutate or append-reallocate `b` after `bytesToString`
  3. backing memory stays valid for the whole lifetime of the string/slice

```go
// unsafe misuse
s := "hello"
b := stringToBytes(s)
b[0] = 'H' // breaks string immutability

b = []byte("world")
s2 := bytesToString(b)
b[0] = 'W' // s2 content changes too
```

**Rule of thumb**: use standard conversion unless profiling proves conversion is a hotspot and you can guarantee read-only access or controlled lifetime.

Compiler may avoid copies in some cases:

---

## 2. Language Semantics and Expressions

Short variable declaration:

```go
x := 1
```

Inside a function, `:=` declares at least one new variable. It can also shadow outer variables:

```go
err := doA()
if err != nil {
    return err
}

if v, err := doB(); err != nil {
    return err
} else {
    use(v)
}
```

Closure captures variables, not values. This matters in loops:

```go
for i := 0; i < 3; i++ {
    i := i
    go func() {
        fmt.Println(i)
    }()
}
```

In Go 1.22, range loop variables are per-iteration for many common cases, but you should still understand the older closure capture pitfall because legacy code and interviews often discuss it.

### Integer Remainder and Negative Numbers

Go's `%` is a **remainder** operator, not mathematical modulo that always returns a non-negative result. The result has the same sign as the dividend:

```go
 5 % 4  //  1
-5 % 4  // -1
 5 % -4 //  1
-5 % -4 // -1
```

This is the same broad rule as C, C++, Java, and JavaScript. Python is different: Python's `%` result follows the divisor's sign, so `-1 % 4 == 3`.

| Language | Expression | Result | Rule |
|----------|------------|--------|------|
| Go | `-1 % 4` | `-1` | remainder sign follows dividend |
| Java | `-1 % 4` | `-1` | remainder sign follows dividend |
| C/C++ | `-1 % 4` | `-1` | remainder sign follows dividend |
| JavaScript | `-1 % 4` | `-1` | remainder, not mathematical modulo |
| Python | `-1 % 4` | `3` | result sign follows divisor |

For circular arrays, queues, and ring indexes, add one full cycle before taking `%` when moving left:

```go
// move right
next := (i + 1) % n

// move left
prev := (i - 1 + n) % n // example: (0-1+4)%4 = 3
```

If the offset may exceed one cycle, normalize with:

```go
func mod(i, n int) int {
    return (i%n + n) % n
}

mod(-1, 4) // 3
mod(-5, 4) // 3
```

A common interview/algorithm bug is carrying Python intuition into Go and writing `(i - 1) % n`; when `i == 0`, the result is `-1`, and using it as an index will panic.

---

## 3. container/heap Priority Queue

Go's `container/heap` is interface-based. You define ordering by implementing `heap.Interface`.

```go
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x any) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() any {
    old := *h
    x := old[len(old)-1]
    *h = old[:len(old)-1]
    return x
}
```

Min-heap vs max-heap:

```go
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // min-heap
func (h IntHeap) Less(i, j int) bool { return h[i] > h[j] } // max-heap
```

Top K pattern:

- Keep a min-heap of size K for top K largest elements.
- If the new element is larger than heap top, pop top and push new element.
- Complexity: `O(n log k)`.

Common pitfalls:

- `Push` and `Pop` must have pointer receivers because they modify slice length.
- Use `heap.Push` and `heap.Pop`, not direct method calls, so heap invariants are maintained.
- `Pop` removes the last element because the heap package first swaps root with the last element.

---

## 4. GMP Scheduler

### 4.1 Why GMP Exists ⭐⭐⭐

Go needs to schedule a large number of goroutines onto a smaller number of OS threads efficiently. GMP is Go runtime's user-level scheduler:

```text
G: Goroutine, the user-level task.
M: Machine, an OS thread.
P: Processor, scheduler context that owns local run queues and resources.
```

Without `P`, every `M` would compete for global scheduler state. `P` reduces contention by keeping most scheduling local.

Before GMP details, understand the core problem Go is solving:

**OS threads are too expensive**:

```text
Create one OS thread:
- stack: usually 1MB~8MB, fixed upfront
- creation cost: about 10~100 us, involves kernel transitions
- context switch: about 1~10 us, save/restore registers, TLB effects
- practical limit: thousands to tens of thousands, bounded by memory

Create one goroutine:
- stack: starts around 2KB, grows on demand, max about 1GB
- creation cost: about 0.3 us, user-space only
- context switch: about 0.1~0.2 us, no kernel switch for goroutine scheduling itself
- practical limit: millions are feasible
```

**Core tension**: Go wants massive concurrency, but the OS can only manage thousands of threads efficiently.

**Solution**: implement a user-space scheduler that multiplexes many goroutines onto a small number of OS threads. That is GMP.

> **Analogy**: G is the task backlog, M is the worker (OS thread), P is the workstation plus toolbox. A company cannot hire one worker per task, so a small worker pool processes a large backlog.

### 4.2 G, M, P

| Component | Meaning | Responsibility |
|-----------|---------|----------------|
| **G** | Goroutine | stack, state, function, scheduling metadata |
| **M** | OS thread | executes goroutines |
| **P** | Processor token | local run queue, resource ownership, scheduling context |

`GOMAXPROCS` controls how many `P`s can execute Go code simultaneously.

```go
runtime.GOMAXPROCS(runtime.NumCPU())
```

#### Counts and binding rules

```text
P count: GOMAXPROCS (default = CPU cores), fixed
M count: created on demand, default max 10000 (extra Ms appear during blocking syscalls)
G count: as many as the program creates, can be far larger than P

Accurate relationships:
- Goroutines actively executing Go code on CPU <= P (= GOMAXPROCS)
- At one instant, one P binds at most one M, and one M running Go code holds at most one P (1:1)
- M may exceed P: extra Ms are often blocked in syscalls without a P, so they do not steal runq work
- G >> P: most goroutines wait in local queues, global queue, or netpoller
```

#### Common misconception: M > P means multiple Ms fight over one P

No. This is the most common GMP misunderstanding.

```text
Normal execution:
  P0 --1:1-- M0 --runs-- G1
  P1 --1:1-- M1 --runs-- G2

G enters a blocking syscall such as read/write:
  1. G becomes _Gsyscall
  2. M2 hands off its P
  3. M2 stays blocked in the kernel with no P
  4. another M3 can take over that P and run other goroutines
```

So **M > P** means some threads are stuck in syscalls without a P, while the runtime still needs enough M+P pairs to keep up to `GOMAXPROCS` goroutines executing Go code. It does not mean multiple Ms bind the same P and compete for its run queue.

#### Goroutine memory is cheap, but not free

Each goroutine is much lighter than an OS thread, but every G still has its own stack and metadata. Total memory is still bounded.

| | M (OS thread) | G (goroutine) |
|---|---|---|
| Stack | usually MB-scale reserved (1~8MB) | starts around 2KB, grows on demand, max about 1GB per G |
| Scheduling cost | kernel thread + g0 | small struct + heap-allocated stack |
| While waiting | thread still consumes resources | run queue stores G pointers; stacks live on heap |

A P run queue stores **pointers to G**, not full execution contexts for millions of goroutines. Memory pressure comes from:

```text
sum of all goroutine stacks + G structs + objects retained by goroutines
```

Rough estimate: 1 million goroutines at 2KB average stack is about 2GB for stacks alone. Stack growth, closures, channel buffers, and business objects increase this further. "Millions are feasible" means feasible compared with millions of OS threads, not zero memory cost and no OOM risk.

In production, control goroutine count with worker pools, context cancellation, and bounded concurrency. Monitor `runtime.NumGoroutine()` and heap usage.

### 4.3 Scheduling Flow ⭐⭐⭐

```text
1. A goroutine is created.
2. It is often placed in the current P's runnext slot first; if runnext is occupied, the old waiting G moves to the local run queue.
3. M bound to P executes runnable G; runqget prefers runnext, then the local queue.
4. If local queue is empty, P steals work from other P's queues.
5. If still empty, it checks the global run queue and network poller.
```

#### runnext: what it is and what it is not

`runnext` is a **single slot for the next runnable G**, not a parallel execution lane.

1. **Only one G runs on a P at a time.** `runnext` holds a `_Grunnable` goroutine that has **not started user code yet**.
2. **"Bump to local queue" changes queue position only.** The displaced G has not entered its function body, so there is no partial execution, no corrupted locals, and no "run twice" semantics.
3. **Each G still executes once from entry.** It either runs directly from `runnext`, or runs later after sitting in `runq`.
4. **This affects scheduling latency, not program correctness.** A burst of `go func()` calls makes the latest goroutine take `runnext`; earlier ones wait slightly longer in `runq`.

```text
M is currently running G_cur

runnext = G_a            // G_a has not run yet
go func() creates G_b
  -> move G_a to runq tail
  -> runnext = G_b

After G_cur yields:
  runqget() takes G_b from runnext first
  G_a runs later from runq
```

This is different from **preemption**: preemption stops a running G and resumes it later with saved registers/stack. `runnext` replacement happens before user code starts.

#### Why runnext? Start from "parent waits for child"

The caller of `go func()` (parent G) and the new G (child G) are often a **paired task**: the parent parks within a few lines—channel handshake, lock, IO wait—and needs the child to run soon.

> After the parent yields, should the next G be the **freshly spawned child**, or **unrelated G already waiting in runq**?

Without runnext, the child joins the **tail** of `runq` behind other HTTP handlers, log goroutines, stolen work, etc. The parent is already waiting, but the child must run unrelated work first. **Correctness unchanged; handshake latency grows.** `runnext` reserves one **"run this next"** slot for: **parent spawns child → parent parks → child should take the CPU first**.

**Walkthrough: unbuffered channel + backlog in runq**

Assume one P (P0), one M; no stealing/global queue for clarity.

```go
func handleRequestA() {
    ch := make(chan Result)
    go func() { r := <-ch; saveToDB(r) }()  // G_recv
    ch <- computeResult()                    // parent parks if no receiver yet
}
```

Initial state before `go`:

```text
Running on P0: G_handler (request A)
P0.runnext: (empty)
P0.runq:    [ G_handlerB, G_handlerC ]   // unrelated backlog on same P
```

**Without runnext**

```text
T1  go creates G_recv → runq = [ B, C, recv ]
T2  ch <- → G_handler parks
T3  schedule picks B from runq head (not recv!)
T4  B runs… then C runs…
T5  finally recv runs → parent wakes
```

Parent waited from T2 until T5 while B and C ran—**delay = sum of unrelated G runtime**.

**With runnext (Go today)**

```text
T1  go creates G_recv → runnext = recv; runq still [ B, C ]
T2  ch <- → G_handler parks
T3  runqget takes recv from runnext first → handshake completes
T4  B and C still in runq, run later in FIFO order
```

```text
No runnext:  parent park → B → C → child recv
With runnext: parent park → child recv → then B, C
```

This does not change channel semantics—only **how long the parent waits**.

**Two spawns then wait (why LIFO bumps old runnext)**

```go
ctx := buildContext()
go logAccess(ctx)      // G_log
go processOrder(ctx)   // G_order — main path, uses ctx
wg.Wait()
```

```text
go G_log   → runnext = G_log
go G_order → G_log to runq tail, runnext = G_order
parent park → G_order runs before G_log
```

Parent usually spawns in code order: helper first, main path last. LIFO lets the **last spawned** child take the CPU when the parent parks.

| Single `go` then park | Two `go` then park |
|-----------------------|---------------------|
| Child skips runq tail vs unrelated backlog | Last child runs before first child |

| | Current (new G takes runnext) | Alternative (enqueue new G if full) |
|--|--|--|
| Writes when runnext occupied | 1 runqput (old G) | 1 runqput (new G) |
| Who runs first after yield | **Most recent** child | **First** child in runnext |

Go never guarantees global FIFO. Use channels/WaitGroup for strict ordering.

Work stealing:

- Balances load across Ps.
- Reduces global lock contention.
- Allows local queues to stay hot in CPU cache.

Network poller:

- Goroutines blocked on network IO are parked.
- The runtime uses platform mechanisms such as epoll/kqueue/iocp.
- When IO is ready, goroutines become runnable again.

System calls:

- If a G enters a blocking syscall, its M may block.
- The P can detach and run other goroutines on another M.
- This prevents one blocking syscall from stopping all Go execution.

### 4.4 Important Scheduler Mechanisms

**Goroutine stack**:

- Starts small, typically a few KB.
- Grows and shrinks dynamically.
- Much cheaper than fixed-size OS thread stacks.

**Preemption**:

- Older Go versions relied more on cooperative preemption.
- Modern Go supports asynchronous preemption, improving fairness for CPU-bound loops.

**Go goroutine vs Python coroutine**:

| Dimension | Go Goroutine | Python Coroutine |
|-----------|--------------|------------------|
| Scheduling | Runtime scheduler, preemptive improvements | Event loop, cooperative |
| Parallelism | Can run in parallel across OS threads | `asyncio` is single-threaded unless combined with threads/processes |
| Blocking style | Blocking-looking code is common | `await` points are explicit |
| Best for | concurrent servers, pipelines | high-concurrency IO with explicit async APIs |

---

## 5. Goroutine and Channel

### 5.1 Goroutine

Create a goroutine:

```go
go func() {
    doWork()
}()
```

Goroutines are cheap but not free. Production code must manage:

- Lifecycle.
- Cancellation.
- Backpressure.
- Panic recovery if goroutine failure must not crash the process.

Common leak pattern:

```go
func worker(ch <-chan int) {
    for v := range ch {
        process(v)
    }
}
```

If `ch` is never closed and no cancellation exists, the goroutine may live forever.

### 5.2 Channel

Unbuffered channel:

```go
ch := make(chan int)
```

Send and receive synchronize directly.

Buffered channel:

```go
ch := make(chan int, 100)
```

Buffer decouples producer and consumer up to capacity.

Channel states:

| Operation | nil channel | open channel | closed channel |
|-----------|-------------|--------------|----------------|
| send | block forever | send or block | panic |
| receive | block forever | receive or block | zero value, `ok=false` |
| close | panic | close | panic |

Receive with `ok`:

```go
v, ok := <-ch
if !ok {
    // channel closed
}
```

`select`:

```go
select {
case v := <-ch:
    handle(v)
case <-ctx.Done():
    return ctx.Err()
default:
    // non-blocking fallback
}
```

Channel ownership rule:

- The sender usually closes the channel.
- Receivers should not close a channel they do not own.

### 5.3 Channel vs Mutex

Use channel when:

- You are transferring ownership of data.
- You are modeling a pipeline.
- You need cancellation or fan-in/fan-out coordination.

Use mutex when:

- You are protecting shared state.
- The critical section is small.
- A shared map or counter is simpler than message passing.

Good Go is not "channels everywhere"; it is choosing the simplest synchronization model for the invariant.

---

## 6. Context

`context.Context` carries cancellation, deadlines, and request-scoped values across API boundaries.

### 6.1 Core Methods

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

Common constructors:

```go
ctx := context.Background()
ctx, cancel := context.WithCancel(ctx)
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
ctx, cancel := context.WithDeadline(ctx, deadline)
ctx = context.WithValue(ctx, key, value)
```

Always call `cancel` when you create a cancellable context:

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()
```

### 6.2 Implementation Model

Context forms a tree:

```text
Background
└─ request context
   ├─ DB query context
   └─ downstream RPC context
```

Cancelling a parent cancels all children.

### 6.3 Do Not Store Context ⭐⭐⭐

Do not store `context.Context` in a struct for later reuse.

Bad:

```go
type Service struct {
    ctx context.Context
}
```

Good:

```go
func (s *Service) GetUser(ctx context.Context, id int64) (*User, error) {
    return s.repo.GetUser(ctx, id)
}
```

Reasons:

- Context is request-scoped.
- Storing it can leak deadlines, cancellation, and values across requests.
- It makes lifecycle unclear.

### 6.4 Best Practices

- Pass `ctx` as the first parameter.
- Do not pass `nil`; use `context.TODO()` if unsure.
- Use `Value` only for request-scoped metadata, not optional parameters.
- Use typed, unexported keys for values.
- Propagate cancellation into DB, RPC, and worker goroutines.

HTTP timeout example:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    user, err := service.GetUser(ctx, 1001)
    if err != nil {
        http.Error(w, err.Error(), http.StatusGatewayTimeout)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

---

## 7. sync Primitives

### 7.1 Mutex

```go
var mu sync.Mutex

mu.Lock()
defer mu.Unlock()
// critical section
```

Keep critical sections small. Do not hold locks while doing slow IO unless it is intentional.

### 7.2 RWMutex

```go
var mu sync.RWMutex

mu.RLock()
read()
mu.RUnlock()

mu.Lock()
write()
mu.Unlock()
```

Use `RWMutex` when reads heavily outnumber writes and read critical sections are not too small. Otherwise `Mutex` may be simpler and faster.

### 7.3 WaitGroup

```go
var wg sync.WaitGroup

for _, job := range jobs {
    wg.Add(1)
    go func(job Job) {
        defer wg.Done()
        process(job)
    }(job)
}

wg.Wait()
```

Call `Add` before starting the goroutine to avoid races.

### 7.4 Once

```go
var once sync.Once

once.Do(func() {
    initConfig()
})
```

Used for one-time initialization. If the function panics, `Once` considers it done in current Go behavior, so be careful.

### 7.5 Pool

```go
var bufPool = sync.Pool{
    New: func() any {
        return make([]byte, 0, 4096)
    },
}
```

`sync.Pool` is for temporary objects and may be cleared by GC. Do not use it as a cache with correctness requirements.

### 7.6 sync.Map

Use `sync.Map` for read-mostly workloads: stable keys, write once and read many times.

```go
var cache sync.Map

cache.Store("key", "value")

if val, ok := cache.Load("key"); ok {
    fmt.Println(val)
}

actual, loaded := cache.LoadOrStore("key", "value")
```

#### Internals: read + dirty

`sync.Map` is not a normal map wrapped in one lock. It uses **read + dirty dual maps**:

```go
// sync/map.go (simplified)
type Map struct {
    mu     Mutex
    read   atomic.Pointer[readOnly] // lock-free read path
    dirty  map[any]any              // write buffer
    misses int                      // number of Load misses on read
}

type readOnly struct {
    m       map[any]any
    amended bool // whether dirty contains keys not in read
}
```

**Load flow**:

```text
1. Lock-free read from read.m
   ├── hit -> return immediately (hot path, no lock)
   └── miss
       ├── amended=false -> key does not exist, return
       └── amended=true  -> lock mu, check dirty, misses++
2. When misses is large enough -> promote dirty into a new read snapshot
```

**Store / Delete flow**:

```text
key already in read and dirty is nil -> update read (shorter locked path)
otherwise -> lock mu, write dirty, amended=true
```

Think of it as: **reads use a snapshot, writes go to a buffer, promote merges them after enough misses**.

#### sync.Map vs RWMutex + map

Both are used for read-heavy workloads, but they are not interchangeable. `sync.Map` is not simply an upgraded `RWMutex + map`.

| | `RWMutex + map` | `sync.Map` |
|---|---|---|
| Read path | `RLock` -> lookup -> `RUnlock` every time | lock-free when `read` hit |
| Write path | `Lock` -> mutate -> `Unlock` | write to `dirty`, promote when needed |
| Does write block read? | yes, readers and writers exclude each other | readers are not fully blocked by writers |
| Visibility after write | visible on the next read after `Unlock` | keys only in `dirty` may stay invisible until promote |
| Types | `map[K]V` | `any`, assertions required |
| Range / multi-key updates | straightforward under one lock | awkward, not for complex invariants |
| Complexity | low, predictable | higher, read/dirty behavior matters |
| Typical pattern | general read-heavy cache | write once, read many; `LoadOrStore` lazy init |

**RWMutex + map**

```go
type Cache struct {
    mu sync.RWMutex
    m  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    v, ok := c.m[key]
    c.mu.RUnlock()
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    c.m[key] = value
    c.mu.Unlock()
}
```

Every read pays `RLock` cost. Writers wait for active readers, and new readers wait for writers. The upside is clear semantics: **after `Unlock`, every reader sees the latest write**. Good when reads dominate but fresh writes still need to become visible quickly.

**sync.Map**

Its main advantage over `RWMutex + map` is lock-free reads for hot keys already in `read`. The trade-offs are:

1. **Delayed visibility**: new keys go to `dirty` first; lock-free readers only scan `read`, so other goroutines may miss recent writes until promote.
2. **Advantage fades with churn**: when `amended=true`, Loads miss more often and take locks, and internal bookkeeping can cost more than `RWMutex + map`.
3. **Less ergonomic API**: `any` plus assertions; custom locked wrappers are often clearer for complex logic.

**When to choose which**

```text
Prefer RWMutex + map:
- read-heavy, but writes should become visible quickly to other goroutines
- need map[K]V, Range, or multi-key updates under one lock
- default choice when behavior predictability matters more than micro-optimization

Consider sync.Map:
- stable key set, write once and read heavily afterward
- extreme concurrent reads of the same hot keys, and profiling shows RLock contention
- classic pattern: LoadOrStore lazy init per key, then massive read traffic

Avoid sync.Map:
- counters, LRU, frequent updates to the same key
- new keys must be globally visible immediately
- writes are infrequent but continuous enough to keep invalidating the read snapshot
```

Typical fit: `LoadOrStore(key, factory())` for per-key lazy initialization, then many concurrent reads of the same keys.

**Good fit**:
- keys written once and then read heavily
- registries, config snapshots, object caches
- per-key lazy init with `LoadOrStore`

**Poor fit**:
- counters or LRU-style frequent updates/deletes
- atomic updates across multiple keys
- strongly typed maps without assertions
- writes must be immediately visible to all goroutines

Use `RWMutex + map` when reads dominate but write visibility still matters. Use plain `Mutex + map` when contention is low and simplicity matters most.

---

## 8. defer

### 8.1 Execution Order ⭐⭐⭐

`defer` uses LIFO order.

```go
defer fmt.Println("1")
defer fmt.Println("2")
defer fmt.Println("3")
// output: 3, 2, 1
```

### 8.2 defer and return ⭐⭐⭐

Return flow:

```text
1. Assign return values.
2. Execute deferred functions.
3. Return to caller.
```

Named return values can be modified by deferred functions:

```go
func f() (x int) {
    defer func() {
        x++
    }()
    return 1 // final x = 2
}
```

### 8.3 Arguments Are Evaluated Immediately

```go
x := 1
defer fmt.Println(x)
x = 2
// prints 1
```

The arguments to the deferred call are evaluated when `defer` is executed, not when the deferred function runs.

### 8.4 panic and recover

`recover` only works inside a deferred function in the same goroutine.

```go
defer func() {
    if r := recover(); r != nil {
        log.Printf("panic: %v", r)
    }
}()
```

Use `panic/recover` for truly exceptional cases or framework boundaries. Do not use it for normal control flow.

---

## 9. Error Handling

Go treats errors as explicit values.

```go
if err != nil {
    return err
}
```

### 9.1 `error`, `panic`, and `fatal error`

| Mechanism | Source | Recoverable | Typical use |
|-----------|--------|-------------|-------------|
| `error` | ordinary return value | caller decides | IO failure, invalid input, business rule failure |
| `panic` | explicit panic or runtime panic | recoverable by `recover` in the same goroutine | index out of range, nil pointer, broken internal invariant |
| `fatal error` | runtime `throw` | **not recoverable** | concurrent map writes, stack overflow, runtime consistency failure |

Use `error` for normal failures. Use `panic` only when continuing the current execution path is meaningless. A `fatal error` is lower-level than panic: the runtime considers the process state unsafe and exits directly; `recover` cannot catch it.

### 9.2 Error Wrapping

```go
if err != nil {
    return fmt.Errorf("query user %d: %w", id, err)
}
```

Inspect wrapped errors:

```go
errors.Is(err, sql.ErrNoRows)

var target *MyError
errors.As(err, &target)
```

### 9.3 Custom Error Types

```go
type CodeError struct {
    Code int
    Msg  string
}

func (e *CodeError) Error() string {
    return e.Msg
}
```

Guidelines:

- Add context when returning errors upward.
- Keep sentinel errors stable if used across packages.
- Use typed errors when callers need structured handling.
- Avoid logging and returning the same error at every layer; choose clear ownership.

### 9.4 panic and recover

When `panic` happens, the current goroutine stops normal execution and runs deferred calls in LIFO order. If a deferred function calls `recover()` and receives a non-nil value, the panic is stopped and the function returns through the deferred path. Without recover, panic keeps unwinding and eventually crashes the process.

```go
func parseConfig(raw string) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("parse config panic: %v", r)
        }
    }()

    mustParse(raw)
    return nil
}
```

`recover` rules:

1. It only works when called directly inside a deferred function.
2. It only catches panic in the same goroutine.
3. It cannot catch runtime fatal errors such as `concurrent map writes`.

```go
func main() {
    defer func() {
        recover() // cannot catch panic from the goroutine below
    }()

    go func() {
        panic("worker failed")
    }()

    time.Sleep(time.Second)
}
```

Protect goroutines at their entry point:

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("worker panic: %v", r)
        }
    }()

    runWorker()
}()
```

### 9.5 fatal error and runtime throw

`fatal error` usually comes from runtime `throw`, not ordinary panic. It means the runtime found a state that cannot be safely recovered.

```go
m := map[string]int{}

go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()

// fatal error: concurrent map writes
// recover cannot catch it
```

Common fatal cases:

- concurrent map writes
- concurrent map read and map write
- stack overflow
- runtime internal consistency errors

Do not rely on `recover` for fatal errors. Prevent them with the correct concurrency primitive, such as `Mutex`, `RWMutex`, `sync.Map`, or sharded maps.

---

## 10. Memory Management and GC

### 10.1 Allocation

Go uses stack allocation and heap allocation. Escape analysis decides whether a value can stay on the stack.

```bash
go build -gcflags="-m" ./...
```

Common causes of heap escape:

- Returning pointer to a local value.
- Storing values in interfaces.
- Capturing variables in closures.
- Values that outlive the stack frame.

### 10.2 GC Mechanism

Go uses a concurrent mark-sweep garbage collector with a write barrier.

High-level flow:

```text
1. Mark setup: enable write barrier.
2. Concurrent mark: trace reachable objects.
3. Mark termination: brief stop-the-world.
4. Sweep: reclaim unreachable objects.
```

Important tuning knob:

```bash
GOGC=100
```

`GOGC` controls the target heap growth ratio. Lower values reduce memory but increase GC frequency. Higher values reduce GC CPU but increase memory.

Go also supports memory limit:

```bash
GOMEMLIMIT=2GiB
```

GC interview focus:

- GC cost is closely tied to live heap size and pointer density.
- Reducing allocations often improves both CPU and tail latency.
- Object reuse can help, but overusing pools can complicate code.

---

## 11. Profiling and Performance Tuning

### 11.1 pprof

Enable pprof in an HTTP service:

```go
import _ "net/http/pprof"

go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

Common commands:

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

Inside pprof:

```text
top
list FunctionName
web
```

### 11.2 trace

`go tool trace` helps analyze:

- Goroutine blocking.
- Scheduler latency.
- Network blocking.
- Syscalls.
- GC pauses.

```bash
go test -trace trace.out ./...
go tool trace trace.out
```

### 11.3 Common Performance Issues

- Too many allocations in hot paths.
- Excessive string/[]byte conversions.
- Lock contention.
- Goroutine leaks.
- Unbounded channels.
- JSON reflection overhead.
- N+1 database or RPC calls.

General approach:

1. Measure before optimizing.
2. Identify CPU, memory, lock, IO, or scheduler bottleneck.
3. Change one thing at a time.
4. Verify with benchmark and production metrics.

---

## 12. Gin Practice

### 12.1 Basic Usage

```go
r := gin.Default()

r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(http.StatusOK, gin.H{"id": id})
})

r.Run(":8080")
```

### 12.2 Middleware

```go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := uuid.New().String()
        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)
        c.Next()
    }
}
```

Middleware order matters:

```text
middleware before c.Next()
  -> downstream handlers
middleware after c.Next()
```

### 12.3 Error Handling

```go
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) > 0 {
            err := c.Errors.Last().Err
            c.JSON(http.StatusInternalServerError, gin.H{
                "error": err.Error(),
            })
        }
    }
}
```

Production tips:

- Use request-scoped context and timeout.
- Add structured logs and request IDs.
- Validate input explicitly.
- Do not expose internal error details to clients.
- Put authentication, rate limiting, and panic recovery in middleware.

---

## 13. Practical Cases

### Case 1: Hot Reload Activity Configuration in a Game Backend

Architecture:

```text
Admin updates config -> DB/Config Center -> Go service watches update -> atomic swap
```

Use `atomic.Value` for read-mostly config:

```go
var currentConfig atomic.Value

func GetConfig() *Config {
    return currentConfig.Load().(*Config)
}

func UpdateConfig(cfg *Config) {
    currentConfig.Store(cfg)
}
```

Why this works:

- Reads are lock-free.
- Updates replace the whole immutable config.
- Avoids partial updates and inconsistent reads.

### Case 2: Payment Flow with Distributed Lock

Use a lock only to reduce concurrent duplicate work. The real correctness boundary should be database unique constraints and idempotent state transitions.

```go
func Pay(ctx context.Context, orderID string) error {
    lockValue := uuid.New().String()
    if !Lock("pay:"+orderID, lockValue, 10*time.Second) {
        return ErrBusy
    }
    defer Unlock("pay:"+orderID, lockValue)

    return db.Transaction(ctx, func(tx *Tx) error {
        order := tx.GetOrderForUpdate(orderID)
        if order.Status == "paid" {
            return nil
        }
        tx.MarkPaid(orderID)
        tx.InsertLedger(orderID)
        return nil
    })
}
```

Interview point: Redis lock is an optimization. The database transaction and idempotent state machine are the correctness mechanism.

---

## 14. Advanced Runtime, Correctness, and Engineering Updates

### Method Sets and Generics

`T` and `*T` do not have the same method set. A value of type `T` only has methods with value receivers, while `*T` has both value-receiver and pointer-receiver methods. If an interface requires a pointer-receiver method, only `*T` implements it. Generics are best used for containers, algorithms, and type-safe helpers; interfaces describe behavior, while type parameters describe a family of concrete types.

### Concurrent Correctness: Memory Model, atomic, and race

A write in one goroutine is only guaranteed to be visible to another goroutine through a synchronization edge: `Mutex.Unlock -> Lock`, channel send -> receive, `close(ch)` -> observing close, `sync.Once`, or atomic operations. Use `go test -race ./...`, `go run -race`, or `go build -race`; there is no `go vet -race`. Atomic operations are useful for single-variable state, counters, and configuration pointers, but multi-field invariants are usually clearer with a mutex.

### Allocator and GC

Go allocation flows through P-local `mcache`, shared `mcentral`, and global `mheap`. Small objects use size classes and spans; tiny no-pointer objects may use the tiny allocator. Modern GC is concurrent mark-sweep with short STW phases, hybrid write barriers, pacer control, `GOGC`, and `GOMEMLIMIT`. `GOGC` controls heap growth ratio; `GOMEMLIMIT` gives the runtime a soft memory target, especially useful in containers.

### Important Correctness Fixes

- `delete` does not shrink a Go map automatically. After deleting many keys, rebuild the map if memory compaction matters.
- `sync.Map.Store` has synchronization semantics; new keys may force a locked slow path before promotion, but they are not “invisible until promote.”
- Do not call `gin.Context.Next()` from a new goroutine in a timeout middleware. Propagate `Request.Context()` and let downstream calls honor cancellation.

### Engineering Practice

A reliable Go engineering answer should include table-driven tests, benchmarks with `b.ResetTimer()` and `b.ReportAllocs()`, fuzzing, module hygiene, build tags, private module settings, and version-aware answers such as Go 1.14 async preemption, Go 1.18 generics, Go 1.19 `GOMEMLIMIT`, and Go 1.22 range-variable changes.


## 15. Interview Self-Check

### Quick Questions

### Q1: What is the difference between array and slice in Go?

**Answer:** Array is a fixed-length value type, and length is part of its type. Slice is a descriptor containing pointer, length, and capacity. Passing a slice copies the descriptor, but multiple slices can still share the same underlying array.

### Q2: When should you use `make([]T, n)` vs `make([]T, 0, n)`?

**Answer:** Use `make([]T, n)` when you will fill by index. Use `make([]T, 0, n)` when you will build the slice with `append`. Mixing them often creates zero-value elements or duplicated results.

### Q3: Why is Go map not safe for concurrent read/write?

**Answer:** Map mutation may change buckets, overflow buckets, and incremental growth state. Concurrent read/write can observe inconsistent internal state. Runtime detects concurrent writes and throws `concurrent map writes`, which is a fatal error, not ordinary panic, and cannot be recovered. Concurrent reads are safe only when there is no concurrent writer. Use `Mutex` for simple shared state, `RWMutex` for read-heavy state, `sync.Map` for stable-key read-mostly caches/registries, or sharded maps for high write concurrency.

### Q4: Why can an interface value be non-nil while the underlying pointer is nil?

**Answer:** An interface value contains type information and a data pointer. If the type information is present but the data pointer is nil, the interface itself is still non-nil.

### Q5: What is the difference between goroutine and OS thread?

**Answer:** Goroutine is a lightweight user-level execution unit managed by the Go runtime. OS thread is managed by the kernel. Goroutines start with small growable stacks and are multiplexed onto OS threads by the GMP scheduler.

### Q6: What are G, M, and P?

**Answer:** G is goroutine, M is OS thread, and P is processor context. P owns run queues and scheduler resources. M must hold a P to execute Go code. `GOMAXPROCS` controls the number of Ps. At one instant, P and M bind 1:1 when running Go code. M may exceed P because some Ms block in syscalls without a P; they do not compete for the same P's run queue. Goroutines are lighter than threads, but each G still has its own stack, so millions of goroutines still consume memory and can OOM.

### Q7: What happens when a goroutine blocks on network IO?

**Answer:** The runtime parks the goroutine and registers the fd with the network poller. The M/P can run other goroutines. When IO is ready, the goroutine becomes runnable again.

### Q8: What is the difference between channel and mutex?

**Answer:** Channel is better for communication, ownership transfer, and pipelines. Mutex is better for protecting shared state. The right choice depends on the invariant, not on style preference.

### Q9: Why should context not be stored in a struct?

**Answer:** Context is request-scoped. Storing it can leak cancellation, deadlines, and values across requests. Pass it explicitly as the first parameter.

### Q10: How does defer interact with return values?

**Answer:** Return values are assigned first, then deferred functions run, then the function returns. Deferred functions can modify named return values.

### Deep-Dive Questions

### Q11: Explain Go slice growth and common pitfalls.

**Answer:** A slice has pointer, length, and capacity. Passing a slice copies the header by value, not the underlying array, so index writes are visible to the caller but header changes such as `s = append(s, x)` are not unless you return the slice, use `*[]T`, or use a pointer receiver. If append stays within capacity, it writes to the same underlying array. If it exceeds capacity, a new array is allocated. Go 1.18+ uses a smaller threshold around 256 before transitioning from doubling to smoother growth. Pitfalls include shared arrays, retaining large arrays through small slices, and confusing length with capacity.

### Q12: How does Go map grow?

**Answer:** Growth triggers when load factor exceeds about 6.5 (`count / 2^B > 6.5`), causing bucket count to double, or when overflow buckets become too many, causing same-size reorganization. Go keeps `oldbuckets` and gradually evacuates 1-2 buckets during later map operations, spreading rehash cost instead of doing one large stop-the-world rehash. Unlike Java HashMap, Go does not treeify buckets; it relies on fixed 8-slot buckets, overflow chains, and these growth triggers to keep average lookup O(1). Theoretical worst case remains O(n) under pathological hash collisions.

### Q13: How does the GMP scheduler reduce contention?

**Answer:** Each P owns a local run queue, so goroutine scheduling is mostly local. Global run queue is used only when needed. Work stealing balances load between Ps. This avoids every thread contending on one global queue.

### Q14: What causes goroutine leaks and how do you prevent them?

**Answer:** Common causes include goroutines waiting forever on channels, missing cancellation, unbounded producers, and blocked sends to channels nobody reads. Prevent them with context cancellation, closing owned channels, bounded queues, timeouts, and clear ownership.

### Q15: How does channel close behave?

**Answer:** Sending to a closed channel panics. Receiving from a closed channel returns the zero value and `ok=false`. Closing a nil or already closed channel panics. The sender that owns the channel should close it.

### Q16: What should and should not be stored in context values?

**Answer:** Store request-scoped metadata like request ID, trace ID, auth principal. Do not store optional function parameters, large objects, or business dependencies. Use typed keys to avoid collisions.

### Q17: How does Go GC work at a high level?

**Answer:** Go uses concurrent mark-sweep GC. It briefly stops the world to set up marking, concurrently marks reachable objects with write barriers, briefly stops again for mark termination, and then sweeps unreachable objects. GC cost depends heavily on live heap size and pointer density.

### Q18: How do you investigate high memory usage in a Go service?

**Answer:** Use heap pprof, allocation profile, goroutine profile, and runtime metrics. Check whether memory is live heap, allocation churn, goroutine stacks, buffers, or retained references. Use escape analysis and benchmarks to verify fixes.

### Q19: How do you diagnose a Go service with high P99 latency?

**Answer:** First separate CPU, GC, lock, IO, scheduler, and downstream dependency latency. Use pprof for CPU/heap/block/mutex profiles, trace for scheduler and blocking, and application metrics for dependency latency. Optimize based on evidence.

### Q20: What are the principles for error handling in Go?

**Answer:** Return errors explicitly, wrap with context using `%w`, use `errors.Is/As` for inspection, and avoid panic for normal control flow. Use panic only for broken internal invariants or startup failures where continuing is meaningless. `recover` only works in a deferred function in the same goroutine and is useful at framework/goroutine boundaries. Runtime fatal errors such as concurrent map writes are not recoverable. Log errors at ownership boundaries, not repeatedly at every layer.

### Open-Ended Design Questions

### D1: Design a high-concurrency Go HTTP service. What runtime and engineering issues do you consider?

**Reference approach:**

- Use bounded worker pools or backpressure for expensive tasks.
- Propagate `context.Context` deadlines to DB and RPC.
- Tune connection pools and HTTP timeouts.
- Protect shared state with simple synchronization.
- Monitor goroutine count, GC, heap, CPU, file descriptors, pprof, and P99 latency.
- Add graceful shutdown and request draining.

### D2: A Go service has goroutine count growing continuously. How do you troubleshoot?

**Reference approach:**

- Capture goroutine profile with pprof.
- Group stacks by waiting location.
- Check channels with blocked sends/receives, missing context cancellation, unclosed response bodies, and worker loops without exit condition.
- Add cancellation, close owned channels, bound queues, and ensure all goroutines have a lifecycle.

### D3: How would you make a payment handler idempotent in Go?

**Reference approach:**

- Use idempotency key or order ID as business key.
- Enforce unique constraints in the database.
- Use transaction and state machine: created -> paying -> paid.
- Redis lock can reduce duplicate concurrent work, but cannot be the correctness boundary.
- Use context deadlines and retry-safe error classification.

### Additional Senior Questions

### Q25: Why can `T` fail to implement an interface that `*T` implements?
Because pointer-receiver methods are only in the method set of `*T`. This matters for interfaces, large structs, mutable receivers, and structs containing locks.

### Q26: When should you use atomic instead of a mutex?
Use atomic for a single independent state variable. Use a mutex when multiple fields must change together or when the invariant is more important than avoiding lock overhead.

### Q27: How do `GOGC` and `GOMEMLIMIT` differ?
`GOGC` is a heap-growth ratio knob. `GOMEMLIMIT` is a soft memory target that makes the pacer more aggressive as the process approaches the limit.
