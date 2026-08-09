# Java In Depth

Language: English | [中文](../后端知识库/04-Java深入.md)

---

## Table of Contents

### Language Fundamentals
- [Java Core Features](#java-core-features)
  - [Object-Oriented Programming](#object-oriented-programming)
  - [Collections Framework](#collections-framework)
  - [Exception Handling](#exception-handling)
  - [equals / hashCode / ==](#equals--hashcode--)
- [Generics and Type Erasure](#generics-and-type-erasure)
- [String and Constant Pool](#string-and-constant-pool)
- [Stream and Optional](#stream-and-optional)
- [JPMS and SPI](#jpms-and-spi)
- [Pattern Matching and Language Evolution](#pattern-matching-and-language-evolution)

### Modern Java (Learn First)
- [Records and Sealed Classes](#records-and-sealed-classes)
- [Java 21+ Virtual Threads (Project Loom)](#java-21-virtual-threads-project-loom)

### JVM Runtime
- [JVM In Depth](#jvm-in-depth)
  - [JVM Memory Structure](#jvm-memory-structure)
  - [Direct Memory / TLAB / Escape Analysis](#direct-memory--tlab--escape-analysis)
  - [JIT Tiered Compilation](#jit-tiered-compilation)
  - [Reference Types](#reference-types)
  - [SafePoint and Card Table](#safepoint-and-card-table)
  - [Garbage Collection Basics](#garbage-collection-basics)
  - [Collector Evolution and Selection](#collector-evolution-and-selection)
  - [Class Loading](#class-loading)

### Concurrency
- [Java Memory Model](#java-memory-model)
- [Concurrency Basics](#concurrency-basics)
- [synchronized Lock Upgrade](#synchronized-lock-upgrade)
- [Concurrency Core](#concurrency-core)
  - [AQS / CLH / Condition](#aqs--clh--condition)
  - [CAS / ABA / LongAdder](#cas--aba--longadder)
  - [ConcurrentHashMap In Depth](#concurrenthashmap-in-depth)
  - [Synchronizers and Queues](#synchronizers-and-queues)
  - [ThreadLocal and ForkJoin](#threadlocal-and-forkjoin)

### Frameworks and Engineering
- [Spring Framework](#spring-framework)
- [MyBatis Practice](#mybatis-practice)
- [Performance Tuning](#performance-tuning)
- [Practical Cases](#practical-cases)
- [Interview Self-Check](#interview-self-check)

---

## Java Core Features

### Object-Oriented Programming

**Interview Answer — three pillars**

**Encapsulation**: hide state behind methods; validate mutations at the boundary.

```java
public class BankAccount {
    private double balance;

    public double getBalance() { return balance; }

    public void deposit(double amount) {
        if (amount > 0) balance += amount;
    }
}
```

**Inheritance**: reuse and specialize "is-a" relationships; prefer composition when behavior varies independently.

**Polymorphism**: compile-time (overloading) vs runtime (overriding / dynamic dispatch).

**Abstract class vs interface**

| Dimension | Abstract class | Interface |
|-----------|----------------|-----------|
| Inheritance | single | multiple |
| Methods | may have body | Java 8+ `default`/`static`; Java 9+ `private` |
| Fields | instance fields OK | constants only (`public static final`) |
| Constructor | yes | no |
| Use | "is-a" base | "can-do" capability |

---

### Collections Framework

**Hierarchy (sketch)**

```text
Collection
├── List → ArrayList, LinkedList, Vector (legacy)
├── Set  → HashSet / LinkedHashSet, TreeSet
└── Queue/Deque → PriorityQueue, ArrayDeque

Map → HashMap / LinkedHashMap, TreeMap, ConcurrentHashMap
```

#### ArrayList growth

Backing store is `Object[]` (`elementData` on Java 8+).

| Stage | Behavior |
|-------|----------|
| No-arg ctor | `elementData = {}`, capacity 0 (lazy) |
| First `add` | grow to default capacity **10** |
| Need more | `newCapacity ≈ old + (old >> 1)` (~**1.5×**) |
| Huge request | at least `minCapacity`; giant-array path past `MAX_ARRAY_SIZE` |

**Production Practice**: random access O(1); mid insert/delete O(n). Prefer `new ArrayList<>(expectedSize)` when size is known. Use `ensureCapacity` / `trimToSize` deliberately. Prefer **ArrayList** unless heavy head/tail mutations with almost no random access.

#### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|--|-----------|------------|
| Structure | dynamic array | doubly linked list |
| Random access | O(1) | O(n) |
| Head insert/delete | O(n) | O(1) |
| Tail insert | amortized O(1) | O(1) |
| Cache locality | good | poor |

#### HashMap state machine (Java 8+)

**Structure**: array + linked list + red-black tree (per bucket).

**Constants**:
- Default capacity 16, load factor 0.75 → `threshold = capacity * loadFactor`
- Treeify when list length ≥ **8** and table length ≥ **64**
- Untreeify when tree nodes ≤ **6**
- Resize doubles capacity; nodes stay at `i` or move to `i + oldCap`

**Hash mixing**:
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// index: hash & (n - 1)  — n must be power of two
```

**`put` state machine**:

```text
put(key, value)
  │
  ├─ table null / empty → resize() init
  ├─ hash, i = hash & (n-1)
  ├─ empty bucket → newNode
  ├─ head key equals → overwrite
  ├─ TreeNode → putTreeVal
  └─ list walk
        ├─ key equals → overwrite
        ├─ append at tail
        └─ binCount >= 7 (8th node) → treeifyBin
              ├─ table.length < 64 → resize only (no tree)
              └─ else → red-black tree
  finally: ++size > threshold → resize()
```

**Resize**: split each chain into lo/hi by high bit; trees `split` into two trees or degrade to lists.

**Thread safety**: Java 7 concurrent resize could infinite-loop; Java 8+ risks lost updates / torn reads. Use `ConcurrentHashMap` under concurrency—not `Collections.synchronizedMap` unless contention is tiny.

#### LinkedHashMap and LRU

`LinkedHashMap` = HashMap + doubly linked list for **insertion order** (default) or **access order** (`accessOrder=true`).

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LruCache(int maxSize) {
        super(16, 0.75f, true); // accessOrder
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

Not thread-safe. High-concurrency caches usually use Caffeine / Guava Cache.

#### ConcurrentHashMap (summary)

- Java 7: Segment locks
- Java 8+: **CAS + `synchronized` on bin head**; Segments gone
- Details in [ConcurrentHashMap In Depth](#concurrenthashmap-in-depth)

---

### Exception Handling

```text
Throwable
├── Error (usually do not catch: OOM, StackOverflowError…)
└── Exception
    ├── RuntimeException (unchecked)
    └── checked exceptions (must declare or catch)
```

```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
} catch (IOException e) {
    throw new UncheckedIOException(e);
}
```

**Guidelines**: do not swallow; translate at boundaries; let Error / severe resource failures surface.

---

### equals / hashCode / ==

| Compare | Meaning |
|---------|---------|
| `==` | primitives by value; references by **identity** |
| `equals` | default = `==`; override for value equality |
| `hashCode` | bucket placement; **equal ⇒ same hashCode** |

**Traps**: mutable fields in `hashCode` after map insert → lost entries; override `equals` without `hashCode` → broken HashMap/HashSet; `Integer` cache `-128..127` may make `==` true—always `equals` for business compares.

---

## Generics and Type Erasure

### Basics

```java
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
```

Compile-time type safety; fewer casts.

### Type Erasure

Generics are mostly a **compile-time** feature; erased to raw types / upper bounds at runtime.

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass(); // true
```

Limits: no `List<int>`, no `new T[]`, no `instanceof List<String>`, overloads that differ only by type args collide after erasure.

### Wildcards and PECS

- `<? extends T>`: read producer (Producer Extends)
- `<? super T>`: write consumer (Consumer Super)

```java
public double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();
    return total;
}

public void addIntegers(List<? super Integer> list) {
    list.add(1);
}
```

---

## String and Constant Pool

### Immutability

Java 9+: `byte[]` + `coder` (Latin-1 / UTF-16). Benefits: safe sharing, cached `hashCode`, pool reuse, tamper-resistant APIs.

### String pool is on the heap (Java 7+)

> **Version fact**: From Java 7 the string pool moved from PermGen to the **heap**. Java 8+ method area is Metaspace (native), but the **string pool remains on the heap**.

```java
String s1 = "hello";
String s2 = "hello";
System.out.println(s1 == s2);     // true

String s3 = new String("hello");  // new heap String
System.out.println(s1 == s3);     // false

String s4 = s3.intern();
System.out.println(s1 == s4);     // true
```

`new String("hello")` object count: **1** if literal already pooled; **2** if literal must enter the pool plus the new heap String.

| Type | Mutable | Thread-safe | Use |
|------|---------|-------------|-----|
| String | no | yes | constants, light concat |
| StringBuilder | yes | no | single-thread heavy concat |
| StringBuffer | yes | synchronized | rare; prefer Builder + outer sync |

---

## Stream and Optional

### Stream API (Java 8+)

```java
List<String> names = users.stream()
    .filter(u -> u.age() >= 18)
    .map(User::name)
    .distinct()
    .sorted()
    .toList(); // Java 16+; earlier: collect(Collectors.toList())
```

**Practice**: intermediate ops are lazy; stateful ops (`sorted`/`distinct`) cost more; `parallel()` only for CPU-bound work without shared mutable state; never mutate external collections inside a stream.

### Optional

```java
Optional<User> opt = userRepository.findById(id);
String name = opt.map(User::name).orElse("anonymous");
opt.orElseGet(() -> loadDefault()); // not orElse(heavy())
opt.orElseThrow(() -> new NotFoundException(id));
```

Anti-pattern: Optional as field/parameter; `orElse(heavy())` always evaluates `heavy`.

---

## JPMS and SPI

### JPMS (Java 9 modules)

```text
module com.example.app {
    requires java.sql;
    exports com.example.app.api;
    opens com.example.app.internal to spring.core;
}
```

Strong encapsulation, `requires`/`exports`/`opens`, `jlink` custom runtimes. Much legacy code stays on classpath; library authors must understand module boundaries and reflective `opens`.

### SPI

```text
META-INF/services/com.example.Codec
→ com.example.JsonCodec
```

```java
ServiceLoader.load(Codec.class).forEach(Codec::init);
```

SPI often uses the **thread context class loader (TCCL)** to break parent delegation so APIs loaded by Bootstrap/Platform can load application implementations (JDBC, SLF4J, …).

---

## Pattern Matching and Language Evolution

```java
// instanceof pattern matching (Java 16+)
if (obj instanceof String s) {
    System.out.println(s.toLowerCase());
}

// switch pattern matching (final in Java 21)
static String formatter(Object o) {
    return switch (o) {
        case Integer i -> "int " + i;
        case String s when s.length() > 5 -> "long string";
        case String s -> "string " + s;
        case null -> "null";
        default -> o.toString();
    };
}
```

Combine with Sealed Classes for exhaustive ADTs.

---

## Records and Sealed Classes

### Record (JDK 16+)

Immutable data carriers with canonical ctor, accessors, `equals`/`hashCode`/`toString`.

```java
public record Point(int x, int y) {}

public record Range(int lo, int hi) {
    public Range {
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
    }
}
```

| | Record | Lombok `@Value`/`@Data` |
|--|--------|-------------------------|
| Version | 16+ language | annotation processor |
| Mutability | always immutable | `@Data` mutable |
| Accessors | `x()` | `getX()` |
| Use | DTO / value object | complex mutable entities |

### Sealed Classes (JDK 17+)

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public final class Rectangle implements Shape { /* ... */ }
public non-sealed class Triangle implements Shape { /* ... */ }

public double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> t.calculateArea();
    };
}
```

---

## Java 21+ Virtual Threads (Project Loom)

### Principle

Virtual threads are lightweight JVM-scheduled threads mounted on a small pool of **carriers** (platform threads)—M:N. Blocking at JVM-managed points (most socket/IO) **unmounts** and frees the carrier.

```text
Virtual Thread × N  ──mount/unmount──→  Carrier Pool (ForkJoinPool ≈ CPU cores)
```

### Comparison

| | Platform thread | Virtual thread |
|--|-----------------|----------------|
| Mapping | 1:1 OS | M:N |
| Stack | ~1MB fixed | heap-backed, growable |
| Scale | thousands | up to millions |
| Best for | CPU-bound | IO-bound |
| Pooling | yes | **do not pool**—create per task |

### synchronized and Pinning (incl. JDK 24)

Historically: blocking inside `synchronized` **pinned** the carrier and crushed throughput → prefer `ReentrantLock`.

> **JDK 24 / JEP 491 (Synchronize Virtual Threads without Pinning)**: largely removes pinning on `synchronized`—blocking on a monitor can unmount instead of monopolizing a carrier.
>
> Remaining caveats: some **native / JNI** frames may still pin; holding `synchronized` for long CPU work is still wrong. After JDK 24+, the old slogan “never use synchronized with virtual threads” is outdated—say “avoid long holds; treat pinning seriously before JDK 24; on JDK 24+ watch remaining pin sites and monitor.”

Monitor: JFR event `jdk.VirtualThreadPinned`.

### Code

```java
Thread.startVirtualThread(() -> doIo());

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofMillis(100));
            return fetch();
        });
    }
}
```

### StructuredTaskScope — Preview / evolving

> **Status**: Structured concurrency (`StructuredTaskScope`) stayed **Preview / incubating** across JDK 21–24 (JEP 453 etc.). Package and APIs may change. Before production, check whether the target JDK needs `--enable-preview` and what the final API looks like.

```java
// Preview sketch — verify against your JDK docs
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> user = scope.fork(() -> fetchUser(id));
    Subtask<Order> order = scope.fork(() -> fetchOrder(id));
    scope.join();
    scope.throwIfFailed();
    return new Response(user.get(), order.get());
}
```

### Fit and caveats

- ✅ High-concurrency blocking IO; thread-per-request style rewrites
- ❌ CPU-bound (use platform pools / ForkJoin)
- ⚠️ Massive `ThreadLocal` use under millions of VTs → prefer `ScopedValue`
- ⚠️ Do not reuse virtual threads via a fixed-size pool

---

## JVM In Depth

### JVM Memory Structure

```text
Thread-private: PC | JVM Stack | Native Method Stack
Thread-shared:  Heap (Eden / Survivor / Old) | Method Area (Java 8+ → Metaspace, native)
Off-heap:       Direct memory, JNI, Metaspace, …
```

| Area | Content | Typical OOM |
|------|---------|-------------|
| Heap | objects, arrays, string pool (Java 7+) | `Java heap space` |
| Metaspace | class metadata | `Metaspace` |
| Direct | NIO DirectByteBuffer, … | `Direct buffer memory` |
| Stack | frames | `StackOverflowError` / OOM by config |

### Direct Memory / TLAB / Escape Analysis

**Direct memory**: `ByteBuffer.allocateDirect` avoids one user→kernel copy; alloc/free is costly; capped by `-XX:MaxDirectMemorySize`. Leaks may not fill the heap—use NMT / DirectBuffer stats.

**TLAB**: each thread reserves a small Eden chunk; most allocations are lock-free inside TLAB; overflow takes slow path (CAS / GC). Large objects may skip TLAB.

**Escape analysis / scalar replacement** (JIT): after method/thread escape analysis → lock elision, **scalar replacement** (fields → registers/stack scalars), reduced heap traffic. Flags: `-XX:+DoEscapeAnalysis` (on by default), compare with `-XX:-EliminateAllocations`.

### JIT Tiered Compilation

```text
Interpreter → C1 (client tiers) → C2 / Graal (deep opts)
            ↑____ profiling (MDO) & deoptimization ____↓
```

| Tier (sketch) | Role |
|---------------|------|
| 0 | interpret |
| 1–3 | C1 ± counters |
| 4 | C2 heavy opts |

Hot methods/loops compile; failed speculative assumptions **deoptimize**. Tools: `-XX:+PrintCompilation`, JFR `Compiler`, jitwatch.

### Reference Types

| Type | GC behavior | Typical use |
|------|-------------|-------------|
| Strong | not collected | normal refs |
| SoftReference | prefer reclaim under pressure | memory-sensitive caches |
| WeakReference | reclaimable next GC | WeakHashMap, ThreadLocal keys |
| PhantomReference | enqueued after death | off-heap cleanup |

### SafePoint and Card Table

**SafePoint**: points where a thread can pause safely (calls, loop back-edges). GC, biased revoke, deopt depend on threads reaching safepoints—long uncounted loops → Time-To-SafePoint tails.

**Card Table**: coarse map of old→young refs updated by write barriers; Young GC scans dirty cards, not the whole old gen. G1 also uses Remembered Sets (write-barrier cost).

---

### Garbage Collection Basics

**Reachability**: GC Roots (stack, statics/constants, JNI, locked objects, …)—not reference counting.

**Algorithms**: mark-sweep (fragmentation), mark-copy (young), mark-compact (old).

**Generational GC names — do not confuse**

| Name | Meaning |
|------|---------|
| **Minor GC / Young GC** | collect young generation |
| **Major GC** | usually means collect **old** generation (terminology varies) |
| **Full GC** | collect the **entire heap** (young + old, often Metaspace)—typically longest pause |

> **Hard fix**: `Major GC ≠ Full GC`. HotSpot logs more often show Young / Mixed / Full. Calling old-gen collection “Major” still does not mean Full. Trust **log labels**, not slang.

**Promotion**: Eden → Survivor (age++) → old at age threshold (default 15, dynamic age); large objects may go old directly.

---

### Collector Evolution and Selection

This section merges former “GC deep dive” and “ZGC vs Shenandoah” into one consistent narrative.

#### 1. CMS (history; removed in JDK 14)

> **Deprecated in JDK 9, removed in JDK 14** (JEP 363). Do not choose CMS for new work; interview as history only.

Goal: lower old-gen STW. Phases: initial mark STW → concurrent mark → remark STW → concurrent sweep.

Pain: **fragmentation**, **Concurrent Mode Failure** (falls back to Serial Old), **floating garbage**.

#### 2. Parallel (throughput)

Multi-threaded STW; batch / ETL.

#### 3. G1 (default mainstream)

Heap = equal **Regions** (roles dynamic). Young GC (STW copy); concurrent mark after IHOP; **Mixed GC** young + selected old regions targeting `-XX:MaxGCPauseMillis`. Mark-copy reduces CMS fragmentation. Cost: RSet / card barriers and metadata.

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
```

#### 4. ZGC (same section as Shenandoah — no duplicate chapter)

**Pause target (unified)**: **sub-millisecond** (commonly advertised **< 1ms**), not scaling linearly with heap. Older “< 10ms” wording is a conservative historical bar—body and Q&A both use sub-ms / ~<1ms; do not mix 10ms and 1ms claims.

**Core tech**:
1. **Colored pointers** — mark/remap metadata in pointer high bits
2. **Load barrier** — self-heal via forwarding table on load

Most mark/relocate is concurrent; STW is short root processing.

**Versions and compressed oops (no contradictions)**

| Version | Status |
|---------|--------|
| JDK 11–14 | experimental |
| JDK 15+ | production-ready |
| JDK 21+ | **Generational ZGC** (JEP 439; later builds trend toward gen by default) |

Unified statement: **early coloring was incompatible with CompressedOops**; JDK 15+ is production-ready; generational from 21; compressed-oops / multi-mapping evolved by release—**follow your target JDK release notes**. Never claim both “never supports” and “unsupported only before 21”.

#### 5. Shenandoah

**Barrier model (corrected)**: modern Shenandoah uses **LRB (Load Reference Barriers)** for concurrent relocation. The textbook “Brooks pointer + write barrier” story is outdated—interview primary answer is **LRB**.

**Generational**: **experimental generational Shenandoah** exists; maturity/flags depend on distro—do not say “absolutely no generations”.

| | ZGC | Shenandoah |
|--|-----|------------|
| Core | colored pointers + load barrier | LRB (concurrent compact) |
| Pause | sub-ms | low-ms (still low-latency) |
| Generational | JDK 21+ product path | experimental |
| Ecosystem | Oracle/OpenJDK mainline option | distro-dependent |

#### 6. Selection

| Scenario | Choice |
|----------|--------|
| General / containers | G1 |
| Ultra-low latency, large heaps | ZGC (prefer gen on 21+) |
| Throughput batch | Parallel |
| Tiny / single-core | Serial |
| Legacy CMS flags | migrate to G1/ZGC after upgrade |

**Full GC triage**: look for `Pause Full` / `Full GC` reason strings (Allocation Failure, Metadata, System.gc, …), then jstat / histo / JFR—not “another Major”.

---

### Class Loading

Phases: Loading → Verification → Preparation → Resolution → Initialization (`<clinit>`).

**Loaders (Java 9+)**:

```text
Bootstrap ClassLoader          → java.base etc. (native; historically rt.jar)
        ↓
Platform ClassLoader           ← replaces old Extension ClassLoader (ext.dirs)
        ↓
Application / System ClassLoader
        ↓
Custom ClassLoader
```

> Pre–Java 9: Bootstrap (`rt.jar`) → Extension (`lib/ext`) → Application.
> **Java 9+**: no `rt.jar` / Extension; hierarchy is **Bootstrap + Platform + Application**.

**Parent delegation**: ask parent first → core class uniqueness. Breaks: SPI (TCCL), hot reload, app-server isolation, some OSGi/module systems.

---

## Java Memory Model

### JMM (abstract model)

JMM defines **visibility, ordering, atomicity** semantics for shared variables and when writes may be seen as synchronized to “main memory”.

> **Hard fix**: “working memory” is a **JMM abstraction**, **not** a 1:1 map of CPU caches/registers. Real hardware has multi-level caches, store buffers, and out-of-order execution; JMM constrains observable results via abstract rules (and happens-before). Do not answer “working memory is the CPU cache”.

### happens-before

Program order, volatile write→read, unlock→later lock, thread start/termination, interrupt, transitivity, …

```java
volatile boolean flag = false;
int x = 0;
// T1: x=42; flag=true;
// T2: if (flag) print(x);  // must be 42
```

---

## Concurrency Basics

### Threads

Prefer bounded-queue `ThreadPoolExecutor` in production over raw `new Thread`.

### synchronized

Instance methods lock `this`; static methods lock the `Class`; prefer a private final lock object for finer granularity.

### volatile

Visibility + ordering; no atomicity for compound ops. DCL singletons need `volatile` against publication reordering.

### Lock

`ReentrantLock`: interruptible, timed, fair, multiple Conditions. `ReadWriteLock` / `StampedLock` for read-heavy workloads.

### Thread pools

```java
new ThreadPoolExecutor(
    core, max, keepAlive, unit,
    new LinkedBlockingQueue<>(capacity), // must be bounded
    threadFactory,
    handler // Abort / CallerRuns / Discard / DiscardOldest
);
```

Flow: `< core` create → enqueue → `< max` create non-core → reject.

> **Aligned with Q&A**: avoid `Executors.newFixedThreadPool` (unbounded queue) and `newCachedThreadPool` (unbounded threads) in production. `Executors` is fine for demos; prefer explicit `ThreadPoolExecutor`. For virtual threads use `newVirtualThreadPerTaskExecutor()`.

### CompletableFuture

`supplyAsync` / `thenApply` / `thenCompose` / `allOf` / `exceptionally`—always pass a custom Executor; do not saturate `ForkJoinPool.commonPool()`.

---

## synchronized Lock Upgrade

Java 6+ expands with contention: `unlocked → biased → lightweight → heavyweight` (Mark Word bits).

```text
Single-thread reentry     → biased (thread id in header)
Alternating short critical → lightweight (stack Lock Record + CAS; adaptive spin)
Sustained collision       → heavyweight (ObjectMonitor, park/unpark)
```

| Lock | Scenario | Mechanism | Cost |
|------|----------|-----------|------|
| Biased | single-thread reentry | thread id in header | tiny |
| Lightweight | alternating short races | CAS + spin | low |
| Heavyweight | high contention | OS wait | high (but no busy-spin) |

**Biased locking status**: disabled by default from Java 15 (`-XX:-UseBiasedLocking`)—revoke/epoch costs hurt multi-threaded servers.

> **“Only upgrade, never downgrade” needs a qualifier**: the contention path does not step heavyweight→lightweight→biased. But the JVM can **deflate idle ObjectMonitors**, returning the header to a monitor-less form; later contention inflates again. Interview line: **upgrade path is one-way + monitor deflation exists—do not say locks never downgrade**.

---

## Concurrency Core

### AQS / CLH / Condition

**AQS** backs `ReentrantLock`, `Semaphore`, `CountDownLatch`, `ReentrantReadWriteLock`, …

```text
state (volatile int)
  +
CLH-variant queue: head/tail, nodes with waitStatus, prev/next, thread
```

| Mode | Template | Typical |
|------|----------|---------|
| Exclusive | `tryAcquire` / `tryRelease` | ReentrantLock (`state` = reentry count) |
| Shared | `tryAcquireShared` / `tryReleaseShared` | Semaphore, CountDownLatch, RW read side |

**Failed acquire path**: `tryAcquire` fail → `addWaiter` → if predecessor is head retry; else mark predecessor SIGNAL and `LockSupport.park`.

**Release**: successful `tryRelease` → `unparkSuccessor(head)`.

**CLH**: classic CLH spins on predecessor; AQS uses a **blockable** variant—predecessor SIGNAL + unpark.

**ConditionObject**: `await` releases lock → condition queue → park; `signal` moves node to sync queue. Always `while` against spurious wakeups.

```java
lock.lock();
try {
    while (!ready) condition.await();
    condition.signal();
} finally {
    lock.unlock();
}
```

### CAS / ABA / LongAdder

**CAS**: `Unsafe` / `VarHandle` / `Atomic*` compareAndSet; retry on failure (optimistic).

**ABA**: A→B→A makes CAS succeed though meaning changed. Mitigations: `AtomicStampedReference`, `AtomicMarkableReference`, or locks / redesign.

**LongAdder / LongAccumulator**: `base` + sparse `Cell[]`; collisions grow cells. `sum()` is a weakly consistent snapshot—fine for metrics; need exact instantaneous value → `AtomicLong`.

### ConcurrentHashMap In Depth

**Java 7**: Segments. **Java 8+**: array + **CAS + `synchronized` on bin head**.

**`sizeCtl` (high-frequency interview topic)**

| Value | Meaning |
|-------|---------|
| Negative | initializing or resizing (high bits signature-like; low bits relate to helper count) |
| 0 | default unset |
| Positive | next resize threshold / initial capacity control |

**`put` path**:
1. Empty table → `initTable` (CAS on `sizeCtl`)
2. Empty bin → `casTabAt`
3. `ForwardingNode` → **helpTransfer**
4. Else `synchronized (first)` list/tree insert; long list + capacity ≥ 64 → treeify
5. `addCount` may trigger `transfer`

**Helping resize**: movers install ForwardingNodes; writers that hit them migrate ranges instead of waiting for the whole resize.

**Counting**: `baseCount` + `CounterCell` (LongAdder-like); `size()` sums estimate. Iterators are weakly consistent. **null key/value forbidden** (unlike HashMap).

### Synchronizers and Queues

| Tool | Semantics |
|------|-----------|
| CountDownLatch | one-shot countdown |
| Semaphore | permits / rate limit |
| CyclicBarrier | reusable barrier + optional action |
| Phaser | multi-phase sync |

**BlockingQueue**: `ArrayBlockingQueue` (bounded array); `LinkedBlockingQueue` (optional bound—default `Integer.MAX_VALUE` is dangerous); `SynchronousQueue` (handoff); `PriorityBlockingQueue` / `DelayQueue`. Queue choice = backpressure vs OOM risk for thread pools.

### ThreadLocal and ForkJoin

**ThreadLocal**: per-thread value via `Thread.threadLocals` → `ThreadLocalMap` (weak keys). Always `remove()` in pools. Prefer `ScopedValue` with virtual threads (preview/final per JDK).

**ForkJoinPool**: work-stealing; `RecursiveTask`/`RecursiveAction`; default executor for many `CompletableFuture` / parallel streams is `commonPool`. Build your own pool for CPU work; do not flood commonPool with blocking IO.

---

## Spring Framework

### IoC and DI

Container owns creation and wiring. Prefer **constructor injection** (immutable, testable, cycles fail early).

```java
@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
}
```

### Bean lifecycle and BPP

```text
Scan/parse → BeanDefinition registry
  → BeanFactoryPostProcessor (mutate definitions)
  → Instantiate
  → Populate (@Autowired …)
  → Aware callbacks
  → BeanPostProcessor#postProcessBeforeInitialization
  → @PostConstruct → InitializingBean → init-method
  → BeanPostProcessor#postProcessAfterInitialization  ← AOP/tx proxies often wrap here
  → Ready
  → @PreDestroy → DisposableBean → destroy-method
```

BPP is the extension core (logging, validation, **proxy creation**). BFPP edits **definitions**; BPP edits **instances**.

### Three-level cache and circular dependencies

| Level | Structure | Content |
|-------|-----------|---------|
| 1 | `singletonObjects` | finished beans |
| 2 | `earlySingletonObjects` | early refs (may already be proxies) |
| 3 | `singletonFactories` | `ObjectFactory` — create early ref on demand |

**Setter cycle A→B→A**: create A, put factory in L3 → need B → create B → B needs A → invoke A's factory (proxy if needed) → L2 → finish B → finish A.

**Constructor cycles** fail during instantiation. Break the cycle or use `@Lazy`.

Why three levels: delaying proxy creation until someone actually needs an early reference avoids enhancing at the wrong time.

### Boot autoconfiguration

`@SpringBootApplication` → `@EnableAutoConfiguration` → `AutoConfigurationImportSelector`:
- Boot 2.x: `META-INF/spring.factories`
- Boot 3.x: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

Conditions: `@ConditionalOnClass` / `OnMissingBean` / `OnProperty` / `OnWebApplication`. User `@Bean` + `OnMissingBean` = override point. Exclude via `@SpringBootApplication(exclude=…)` or `spring.autoconfigure.exclude`.

### AOP proxies

| Style | Mechanism | When |
|-------|-----------|------|
| JDK dynamic proxy | `InvocationHandler` | interface present |
| CGLIB | subclass | class not final; Boot 2+ often defaults to target-class proxy |

Self-invocation `this.x()` **bypasses the proxy** → aspects / `@Transactional` skip. Split beans, inject self, or (carefully) `AopContext`.

### Transactions and failure modes

| Propagation | Meaning |
|-------------|---------|
| REQUIRED | join or create (default) |
| REQUIRES_NEW | suspend outer, new tx |
| NESTED | savepoint (DB/driver dependent) |

**`@Transactional` fails when**:
1. Same-class self-invocation
2. Non-public method (interface-proxy mode)
3. Swallowed exceptions / checked exceptions without `rollbackFor`
4. Non-Spring-managed instance (`new`)
5. Async thread without tx context
6. Non-transactional storage (e.g. MyISAM)

Keep remote calls out of DB transactions.

---

## MyBatis Practice

### Mapper and dynamic SQL

Annotations or XML; `<where>`/`<if>`/`<foreach>`; `resultMap` for associations. Prefer `#{}` over `${}` to avoid injection.

### Level-1 / Level-2 cache

**L1 (local session cache)**:
- Default scope `SESSION` (same `SqlSession`)
- **Can be disabled or narrowed**: `localCacheScope=STATEMENT` (clear per statement), or `flushCache="true"` on queries—not “impossible to turn off”
- Under Spring, one SqlSession per tx/template often makes L1 benefits small

**L2**: namespace-scoped across sessions; dirty-read / multi-table / distributed consistency issues—prefer Redis/Caffeine in production.

```yaml
mybatis:
  configuration:
    local-cache-scope: STATEMENT
    cache-enabled: false
```

---

## Performance Tuning

### JVM flags and containers

```bash
-Xms4g -Xmx4g
-XX:+UseG1GC   # or ZGC
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# Container awareness (required)
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=50.0
```

### Diagnostic tools

| Tool | Use |
|------|-----|
| jcmd / jstat / jstack | runtime, GC, threads |
| jmap | histo / careful live dumps |
| **MAT / VisualVM** | heap analysis |
| **JFR** | low-overhead events (GC, locks, IO, VirtualThreadPinned) |
| **async-profiler** | CPU/alloc/lock flame graphs |

> **`jhat` is gone**—do not recommend it. Use MAT, VisualVM, or `jcmd GC.heap_dump` + MAT.

```bash
jcmd <pid> JFR.start name=app settings=profile duration=60s filename=app.jfr
./asprof -e cpu -d 30 -f cpu.html <pid>
```

### Database (converged)

Indexes, slow SQL, plans, pools → see [`02-MySQL-In-Depth.md`](./02-MySQL-In-Depth.md) and [`10-Database-Comprehensive.md`](./10-Database-Comprehensive.md).

Java-side keepers: HikariCP size / timeout / `maxLifetime`; avoid remote calls inside transactions that pin connections.

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.max-lifetime=1800000
```

---

## Practical Cases

### Finance payment: distributed lock

Redis lock reduces duplicate concurrent work; correctness still needs DB transactions, unique constraints, and a state machine.

```java
@Service
public class PaymentService {
    @Autowired private StringRedisTemplate redisTemplate;

    public boolean deduct(Long userId, BigDecimal amount) {
        String lockKey = "lock:user:" + userId;
        String lockValue = UUID.randomUUID().toString();
        Boolean ok = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, 5, TimeUnit.SECONDS);
        if (Boolean.FALSE.equals(ok)) {
            throw new RuntimeException("concurrent conflict");
        }
        try {
            // deduct …
            return true;
        } finally {
            String script =
                "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) else return 0 end";
            redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                List.of(lockKey), lockValue);
        }
    }
}
```

### Game backend: bounded thread pool

```java
@Bean("taskExecutor")
public ThreadPoolExecutor taskExecutor() {
    return new ThreadPoolExecutor(
        4, 8, 60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy());
}
```

Separate CPU vs IO pools; monitor queue depth, active threads, latency, rejections.

---

## Interview Self-Check

Numbered continuously from Q1. Each answer includes a **Follow-up**. High-frequency selection below (full Chinese set is Q1–Q35 + design).

### Junior

#### Q1: `==` vs `equals`?

**Answer**: `==` compares values for primitives and identity for references; `equals` defaults to identity and can be overridden for value equality. Map keys need paired `equals`/`hashCode`.

**Follow-up**: Why may `Integer a=128; Integer b=128; a==b` be false? How does the cache range matter?

---

#### Q2: `final` / `finally` / `finalize`?

**Answer**: `final` restricts mutation/inheritance/overrides; `finally` runs on try exit; `finalize` is **deprecated**—use try-with-resources.

**Follow-up**: Can a `final` reference's object contents still mutate?

---

#### Q3: ArrayList vs LinkedList?

**Answer**: ArrayList = dynamic array, O(1) random access, ~1.5× growth; LinkedList = doubly linked, O(1) ends, poor locality. Prefer ArrayList.

**Follow-up**: Why not 2× growth? Why set initial capacity when size is known?

---

#### Q4: HashMap vs ConcurrentHashMap?

**Answer**: HashMap is not thread-safe; CHM uses CAS + bin-head `synchronized` (Java 8+), weakly consistent iteration, segmented counting. Never concurrent-write a HashMap.

**Follow-up**: How is `sizeCtl` encoded during resize? How do workers help transfer?

---

#### Q5: Checked vs unchecked exceptions?

**Answer**: Checked (`Exception` minus `RuntimeException`) must declare/catch; RuntimeException/Error are unchecked. Modern APIs often prefer unchecked + centralized error handling.

**Follow-up**: Why do frameworks lean unchecked?

---

### Intermediate / Internals

#### Q6: What is type erasure? Limits?

**Answer**: Compile-time checks then erase to raw types with inserted casts. Limits: no primitive type args, no generic arrays, no parameterized `instanceof`, no overload-by-type-arg-only.

**Follow-up**: What problem do bridge methods solve?

---

#### Q7: What is PECS?

**Answer**: Producer Extends, Consumer Super.

**Follow-up**: Which wildcards do `Collections.copy`'s two parameters use?

---

#### Q8: Why is String immutable? Objects from `new String("hello")`? Where is the pool?

**Answer**: Safety, hash caching, pooling. Object count 1 or 2 depending on whether the literal was already pooled. **Java 7+ string pool is on the heap**.

**Follow-up**: Risk of heavy `intern` with many unique strings?

---

#### Q9: JVM memory areas? Which are thread-private?

**Answer**: Private: PC, VM stack, native method stack. Shared: heap, method area/Metaspace. Also direct memory.

**Follow-up**: Metaspace vs PermGen? Why did the string pool move to the heap?

---

#### Q10: Minor / Major / Full GC? Object promotion?

**Answer**: Minor/Young = young gen; Major often means old gen (messy term); **Full = whole heap (often Metaspace)**; **Major ≠ Full**. Promotion: Eden→Survivor ages→old; large objects may go old directly.

**Follow-up**: How does dynamic age force early promotion?

---

#### Q11: CMS vs G1? CMS status?

**Answer**: CMS concurrent mark-sweep → fragmentation & CMF; **removed in JDK 14**. G1 regions + pause targets + Mixed GC with copying.

**Follow-up**: How do Mixed GC and IHOP relate?

---

#### Q12: ZGC core tech? Why sub-ms pauses? Compressed oops & versions?

**Answer**: Colored pointers + self-healing load barriers; relocation concurrent; STW tiny and largely heap-size independent—**sub-ms (~<1ms)**. JDK 15+ production; **21+ Generational ZGC**. Early coloring incompatible with CompressedOops; later releases evolved—follow release notes.

**Follow-up**: What pain does Generational ZGC fix vs non-gen ZGC?

---

#### Q13: Shenandoah barriers? Generational?

**Answer**: Modern answer is **LRB (Load Reference Barrier)** for concurrent relocation; **experimental generational** exists—do not say “write barrier only / never generational”.

**Follow-up**: How do distro support matrices affect ZGC vs Shenandoah choice?

---

#### Q14: What is JMM? Is working memory the CPU cache?

**Answer**: JMM is a concurrency-semantics abstraction; happens-before constrains visibility/order. **Working memory is not synonymous with CPU cache**.

**Follow-up**: What do as-if-serial vs happens-before mean for single- vs multi-thread?

---

#### Q15: volatile? Why DCL needs it?

**Answer**: Visibility + forbid reorder; not atomic for `i++`. DCL without volatile can publish a partially constructed object.

**Follow-up**: Can volatile replace locks for compound ops?

---

#### Q16: synchronized lock upgrade? Can it downgrade?

**Answer**: unlocked→biased→lightweight→heavyweight. **Upgrade path is one-way**; **monitor deflation** can reclaim idle monitors. Biased locking off by default from Java 15+.

**Follow-up**: Does failed lightweight spin always inflate? What does adaptive spinning watch?

---

#### Q17: AQS? Exclusive vs shared?

**Answer**: `state` + CLH-variant queue + park/unpark. Exclusive = one owner; shared = cumulative permits (Semaphore/Latch). Condition uses a condition queue.

**Follow-up**: How is fairness preserved between failed `tryAcquire` and enqueue?

---

#### Q18: ConcurrentHashMap put/resize/treeify?

**Answer**: CAS empty bins; lock bin head on conflict; `sizeCtl` governs init/resize; hit ForwardingNode → **help resize**; long lists treeify.

**Follow-up**: Why are CHM key/value nulls forbidden vs HashMap?

---

#### Q19: ClassLoader hierarchy and parent delegation? Java 9 change?

**Answer**: Java 9+: Bootstrap → **Platform** (replaces Extension) → Application; no `rt.jar`. Parent delegation keeps core classes unique. SPI/hot-deploy may break it.

**Follow-up**: How does Tomcat isolate webapps with loaders?

---

#### Q20: Thread-pool params/flow? Why avoid Executors factories?

**Answer**: Seven params; create → queue → expand → reject. `newFixedThreadPool` unbounded queue and `newCachedThreadPool` unbounded threads risk OOM—use explicit `ThreadPoolExecutor` + bounded queue.

**Follow-up**: How to size cores for IO-bound? How does CallerRuns create backpressure?

---

#### Q21: ReentrantLock vs synchronized?

**Answer**: Lock = interruptible/timed/fair/multi-Condition; synchronized = concise, auto-unlock, mature JVM opts. Pre–JDK 24 virtual threads were more sensitive to synchronized pinning (**JEP 491**).

**Follow-up**: Why is fair locking usually lower throughput?

---

#### Q22: CountDownLatch / Semaphore / CyclicBarrier?

**Answer**: Latch one-shot; Semaphore permits; Barrier reusable rendezvous.

**Follow-up**: Could you fake a Latch with CHM+CAS? Downsides?

---

#### Q23: ThreadLocal leaks?

**Answer**: Weak keys, strong values; pool threads without `remove()` leak. Prefer ScopedValue with virtual threads.

**Follow-up**: Pitfalls of InheritableThreadLocal with pools / VTs?

---

#### Q24: ForkJoin and work-stealing?

**Answer**: Split tasks; idle workers steal from others' queue tails; good for CPU divide-and-conquer. Do not flood commonPool with blocking IO.

**Follow-up**: How to isolate parallel streams onto a custom ForkJoinPool?

---

#### Q25: JIT tiered compilation? Escape-analysis wins?

**Answer**: Interpreter→C1→C2; hot compile; deopt possible. Escape analysis enables lock elision, scalar replacement, fewer heap allocs.

**Follow-up**: How to confirm a method reached C2 via JFR/logs?

---

#### Q26: Virtual thread principle? When pin? JDK 24 change?

**Answer**: M:N on carriers; block can unmount. Older JDKs pin on blocking inside `synchronized`; **JEP 491 (JDK 24) largely removes synchronized pinning**. Still watch `VirtualThreadPinned`; do not pool VTs.

**Follow-up**: Why are millions of VTs bad for CPU-bound work?

---

#### Q27: What do Record / Sealed solve?

**Answer**: Record = immutable data carrier; Sealed = closed hierarchy for exhaustive switch.

**Follow-up**: Can a Record extend another class? Be a JPA entity?

---

#### Q28: Spring Bean lifecycle? Role of BPP?

**Answer**: Definition→instantiate→inject→Aware→BPP before/after→init callbacks→proxy often in after→destroy. BPP can wrap beans.

**Follow-up**: Which runs earlier, BFPP or BPP?

---

#### Q29: How does the three-level cache break cycles?

**Answer**: L3 ObjectFactory exposes early refs (possibly proxied); constructor cycles cannot be solved.

**Follow-up**: Why three levels instead of only two?

---

#### Q30: When does `@Transactional` fail?

**Answer**: Self-invocation, non-public, swallowed/wrong exceptions, non-proxy object, cross-thread, …

**Follow-up**: Inner `REQUIRES_NEW` commits then outer rolls back—what happens to data?

---

#### Q31: Spring Boot autoconfiguration?

**Answer**: `@EnableAutoConfiguration` import selector reads factories/imports; conditions filter; `OnMissingBean` lets users override.

**Follow-up**: How do you exclude one autoconfig?

---

#### Q32: MyBatis L1/L2 caches? Can L1 be turned off?

**Answer**: L1 is session-scoped; can weaken/disable via `localCacheScope=STATEMENT` etc. L2 is namespace-scoped; often off in prod. Not “L1 cannot be closed”.

**Follow-up**: Why does L1 feel “absent” under Spring?

---

#### Q33: How to triage Full GC? Tool stack?

**Answer**: GC logs → jstat → histo/dump → MAT; JFR + async-profiler for allocation hotspots. Do not use jhat.

**Follow-up**: What goes wrong setting only `-Xmx` in containers and ignoring `MaxRAMPercentage`?

---

#### Q34: Direct memory / TLAB?

**Answer**: Direct memory is off-heap (NIO); TLAB is a per-thread Eden buffer for fast allocation.

**Follow-up**: Who reclaims DirectByteBuffer? Relation to phantom refs?

---

#### Q35: How do SPI and parent delegation cooperate?

**Answer**: Interfaces may load via Platform/Bootstrap while implementations load via App through TCCL/`ServiceLoader`—controlled “break”.

**Follow-up**: JPMS `provides … with …` vs classic SPI files?

---

### Open Design

#### D1: Design a 100k QPS Java gateway — key points?

Netty Reactor, off-heap + pooling, G1/ZGC, filter chain, rate-limit/circuit-break, timeout budgets, Micrometer/tracing, backpressure.

**Follow-up**: How do you stop a slow route from starving the EventLoop?

---

#### D2: Full GC hourly with 2s pauses — drive to rare and <200ms?

Log root cause → cut allocation/leaks → G1/ZGC + generational tuning → offload huge objects → load-test verify.

**Follow-up**: If it is Metaspace Full rather than heap Full, what changes?

---

**Doc navigation**: MySQL / schema tuning → [`02-MySQL-In-Depth.md`](./02-MySQL-In-Depth.md), [`10-Database-Comprehensive.md`](./10-Database-Comprehensive.md); concurrency models → [`05-Concurrency-Programming-Models.md`](./05-Concurrency-Programming-Models.md).
