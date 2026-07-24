# Java In Depth

Language: English | [中文](../后端知识库/04-Java深入.md)

---

## Table of Contents

### Language and JVM
1. [Java Core Features](#1-java-core-features)
2. [Generics and Type Erasure](#2-generics-and-type-erasure)
3. [String and Constant Pool](#3-string-and-constant-pool)
4. [JVM In Depth](#4-jvm-in-depth)
5. [Java Memory Model](#5-java-memory-model)

### Concurrency and Frameworks
6. [Concurrent Programming](#6-concurrent-programming)
7. [Spring Framework](#7-spring-framework)
8. [MyBatis Practice](#8-mybatis-practice)
9. [Performance Tuning](#9-performance-tuning)

### Modern Java
10. [Java 21 Virtual Threads](#10-java-21-virtual-threads)
11. [ZGC vs Shenandoah](#11-zgc-vs-shenandoah)
12. [Records and Sealed Classes](#12-records-and-sealed-classes)

### Practice and Interview Review
13. [Practical Cases](#13-practical-cases)
14. [Interview Self-Check](#14-interview-self-check)

---

## 1. Java Core Features

### 1.1 Object-Oriented Programming

Java's object model is based on encapsulation, inheritance, and polymorphism.

```java
interface Payment {
    void pay(Order order);
}

class CreditCardPayment implements Payment {
    @Override
    public void pay(Order order) {
        // pay by card
    }
}
```

Interview focus:

- Prefer composition over inheritance when behavior varies independently.
- Use interfaces to define stable contracts.
- Polymorphism is implemented through dynamic dispatch.

### 1.2 Collections Framework

Core collection types:

| Type | Common Implementations | Notes |
|------|------------------------|-------|
| `List` | `ArrayList`, `LinkedList` | ordered, allows duplicates |
| `Set` | `HashSet`, `TreeSet`, `LinkedHashSet` | uniqueness |
| `Map` | `HashMap`, `TreeMap`, `ConcurrentHashMap` | key-value |
| `Queue` | `ArrayDeque`, `PriorityQueue` | queue semantics |

`HashMap` basics:

- Uses array + linked list / red-black tree.
- Treeifies a bucket when collision chain grows large and table capacity is large enough.
- Not thread-safe.

`ConcurrentHashMap`:

- Supports concurrent access with finer-grained synchronization and CAS.
- Does not allow null keys or null values.
- Iterators are weakly consistent.

### 1.3 Exception Handling

Java has checked and unchecked exceptions.

```java
try {
    service.process();
} catch (BusinessException e) {
    // expected business error
} catch (Exception e) {
    // unexpected failure
}
```

Guidelines:

- Use exceptions for exceptional paths, not normal control flow.
- Preserve root cause when wrapping exceptions.
- Do not swallow exceptions silently.
- Define business exceptions with clear error codes if APIs need stable error semantics.

---

## 2. Generics and Type Erasure

### 2.1 Generic Basics

```java
class Box<T> {
    private T value;

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }
}
```

Generics provide compile-time type safety and reduce casts.

### 2.2 Type Erasure ⭐⭐⭐

Java generics are implemented mostly by type erasure. Generic type information is checked at compile time and erased to raw types or upper bounds at runtime.

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();

System.out.println(a.getClass() == b.getClass()); // true
```

Consequences:

- Cannot use `new T()`.
- Cannot create generic arrays directly.
- Cannot use `instanceof List<String>`.
- Overloads that differ only by generic parameter may conflict after erasure.

### 2.3 Wildcards and Bounds

PECS: Producer Extends, Consumer Super.

```java
void read(List<? extends Number> nums) {
    Number n = nums.get(0);
    // nums.add(1) is not allowed
}

void write(List<? super Integer> nums) {
    nums.add(1);
    Object x = nums.get(0);
}
```

Use `? extends T` when reading values as `T`. Use `? super T` when writing `T` values.

---

## 3. String and Constant Pool

### 3.1 String Immutability

`String` is immutable.

Reasons:

- Safe sharing in the string pool.
- Thread safety.
- Stable hash code for maps.
- Security for class loading, file paths, and URLs.

### 3.2 String Pool

```java
String a = "hello";
String b = "hello";
System.out.println(a == b); // true

String c = new String("hello");
System.out.println(a == c); // false
```

`new String("hello")` usually involves the literal in the pool and a new heap object.

### 3.3 StringBuilder vs StringBuffer

| Type | Mutability | Thread Safety | Use |
|------|------------|---------------|-----|
| `String` | immutable | safe | constants, keys |
| `StringBuilder` | mutable | not synchronized | local string building |
| `StringBuffer` | mutable | synchronized | legacy synchronized use |

Prefer `StringBuilder` for local concatenation in loops.

---

## 4. JVM In Depth

### 4.1 JVM Memory Areas

```text
Thread-private:
- Program Counter
- Java Virtual Machine Stack
- Native Method Stack

Shared:
- Heap
- Method Area / Metaspace
- Runtime Constant Pool
- Direct Memory outside heap
```

Heap is where most objects live. Stack stores stack frames, local variables, operand stack, and method invocation metadata.

### 4.2 Garbage Collection

Generational hypothesis:

```text
Most objects die young.
Objects that survive multiple young GCs are promoted to old generation.
```

Common collectors:

| Collector | Notes |
|-----------|-------|
| Serial | simple, single-threaded |
| Parallel | throughput-oriented |
| CMS | low-pause old collector, now removed in modern JDKs |
| G1 | region-based, predictable pause target |
| ZGC | ultra-low pause, colored pointers/load barriers |
| Shenandoah | concurrent compaction, low pause |

G1 key ideas:

- Heap is divided into regions.
- Young and old are logical sets of regions.
- Collects high-garbage regions first.
- Uses remembered sets to track cross-region references.

ZGC key ideas:

- Concurrent marking, relocation, and remapping.
- Load barriers.
- Pauses are usually very short and mostly independent of heap size.

### 4.3 Object Allocation and Promotion

Typical allocation path:

```text
Thread Local Allocation Buffer (TLAB) -> Eden -> Survivor -> Old
```

Minor GC collects young generation. Full GC or mixed GC involves old generation depending on collector and conditions.

Promotion can happen when:

- Object age reaches threshold.
- Survivor space cannot hold live objects.
- Large object is allocated directly into old generation, depending on collector/configuration.

### 4.4 Class Loading

Class loading phases:

```text
Loading -> Verification -> Preparation -> Resolution -> Initialization
```

Parent delegation model:

```text
Bootstrap ClassLoader
  -> Platform/Extension ClassLoader
    -> Application ClassLoader
      -> Custom ClassLoader
```

Benefits:

- Prevents core classes from being replaced accidentally.
- Ensures class identity consistency.
- Improves security.

Cases that break or customize delegation:

- SPI, such as JDBC.
- Application servers.
- Plugin systems.
- Hot deployment.

---

## 5. Java Memory Model

### 5.1 JMM

JMM defines visibility, ordering, and atomicity rules for Java threads.

Core problems:

- CPU cache visibility.
- Compiler and CPU reordering.
- Race conditions on shared data.

### 5.2 happens-before

Important happens-before rules:

- Program order within one thread.
- Monitor unlock happens-before subsequent lock on the same monitor.
- Volatile write happens-before subsequent volatile read.
- Thread start happens-before actions in started thread.
- Actions in a thread happen-before another thread successfully returns from `join`.
- Transitivity.

### 5.3 volatile

`volatile` provides:

- Visibility.
- Ordering constraints.
- No atomicity for compound operations like `i++`.

Double-checked locking needs volatile:

```java
class Singleton {
    private static volatile Singleton instance;

    static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

Without volatile, another thread may see a partially constructed object due to reordering.

### 5.4 synchronized Lock Upgrade ⭐⭐⭐

Modern JVMs optimize `synchronized`.

Historical lock states:

```text
No lock -> biased lock -> lightweight lock -> heavyweight lock
```

Notes:

- Biased locking has been disabled/removed in modern JDKs.
- Lightweight locking uses CAS and object headers.
- Heavyweight locks rely on OS mutex/monitor and are more expensive.

Interview answer should mention version differences instead of assuming biased locking still exists in all JDKs.

---

## 6. Concurrent Programming

### 6.1 Thread Basics

```java
Thread t = new Thread(() -> doWork());
t.start();
t.join();
```

In production, prefer thread pools over creating raw threads repeatedly.

### 6.2 synchronized

```java
synchronized (lock) {
    // critical section
}
```

It provides mutual exclusion and memory visibility through monitor enter/exit.

### 6.3 Lock and ReentrantLock

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

`ReentrantLock` supports:

- Interruptible lock acquisition.
- Timed try lock.
- Fair lock option.
- Multiple conditions.

Use it when you need features beyond `synchronized`.

### 6.4 Thread Pool

Core parameters:

```text
corePoolSize
maximumPoolSize
keepAliveTime
workQueue
threadFactory
rejectedExecutionHandler
```

Execution flow:

```text
1. If running threads < corePoolSize, create a core thread.
2. Else enqueue task.
3. If queue is full and threads < maximumPoolSize, create more threads.
4. If still cannot accept, reject.
```

Avoid unbounded queues for high-traffic services because they hide overload and increase latency.

### 6.5 CompletableFuture

```java
CompletableFuture<User> userFuture =
    CompletableFuture.supplyAsync(() -> userService.getUser(id), executor);

CompletableFuture<List<Order>> orderFuture =
    CompletableFuture.supplyAsync(() -> orderService.getOrders(id), executor);

CompletableFuture<Result> result =
    userFuture.thenCombine(orderFuture, Result::new);
```

Use for async composition. Always control executor, timeout, and exception handling.

---

## 7. Spring Framework

### 7.1 IoC Container

Spring IoC manages object creation, dependency injection, and lifecycle.

Bean lifecycle:

```text
Instantiate
-> populate properties
-> BeanPostProcessor before initialization
-> initialization callbacks
-> BeanPostProcessor after initialization
-> ready for use
-> destroy callbacks
```

Constructor injection is usually preferred for required dependencies.

### 7.2 AOP

Spring AOP is proxy-based.

Proxy types:

- JDK dynamic proxy for interfaces.
- CGLIB subclass proxy for concrete classes.

Common use cases:

- Transactions.
- Logging.
- Metrics.
- Security.
- Rate limiting.

Self-invocation pitfall: a method inside the same class calling another annotated method does not go through the proxy, so AOP may not apply.

### 7.3 Transaction Management

`@Transactional` works through AOP proxy.

Common failure cases:

- Method is not public in proxy-based mode.
- Self-invocation bypasses proxy.
- Exception is caught and not rethrown.
- Rollback rules do not include checked exception.
- Transaction manager is not correctly configured.
- Async execution moves work to another thread.

Propagation pitfalls:

- `REQUIRED` joins existing transaction.
- `REQUIRES_NEW` suspends existing transaction and starts a new one.
- `NESTED` uses savepoint if supported.

Use transaction boundaries intentionally. Do not put slow remote calls inside DB transactions.

---

## 8. MyBatis Practice

### 8.1 Basic Mapper

```java
public interface UserMapper {
    User selectById(long id);
    int insert(User user);
}
```

```xml
<select id="selectById" resultType="User">
    SELECT id, name, age FROM users WHERE id = #{id}
</select>
```

### 8.2 Result Mapping

```xml
<resultMap id="userMap" type="User">
    <id property="id" column="id"/>
    <result property="userName" column="user_name"/>
</resultMap>
```

Use result maps when database columns and Java fields differ or when nested mappings are needed.

### 8.3 Common Issues

- N+1 queries with nested select.
- Missing indexes for dynamic SQL filters.
- Overusing `SELECT *`.
- Huge result sets loaded into memory.
- SQL injection risk with `${}`; prefer `#{}` for parameters.

---

## 9. Performance Tuning

### 9.1 JVM Tuning

Start from evidence:

```bash
jps
jstat -gc <pid> 1000
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print
jcmd <pid> GC.class_histogram
```

Heap dump:

```bash
jmap -dump:format=b,file=heap.hprof <pid>
```

GC logs:

```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```

Tuning principles:

- Choose collector based on latency vs throughput goal.
- Set heap size according to live set and container limit.
- Watch allocation rate, promotion rate, old generation occupancy, and pause distribution.
- Do not tune blindly before reading GC logs.

### 9.2 Database Optimization

Common Java-side database issues:

- N+1 queries.
- Missing indexes.
- Large transactions.
- Connection pool exhaustion.
- Slow SQL hidden behind ORM.
- Too many round trips.

Use metrics:

- HikariCP active/idle/pending connections.
- SQL latency histogram.
- Transaction duration.
- Slow query logs.

---

## 10. Java 21 Virtual Threads

### 10.1 Principle

Virtual threads are lightweight threads managed by the JVM. They are scheduled onto carrier platform threads.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> blockingCall());
}
```

Virtual threads are designed to make blocking-style code scale better for IO-bound workloads.

### 10.2 Virtual Thread vs Platform Thread

| Dimension | Platform Thread | Virtual Thread |
|-----------|-----------------|----------------|
| Cost | expensive OS thread | lightweight JVM-managed |
| Quantity | thousands are costly | millions may be possible |
| Best for | CPU-bound or limited threads | high-concurrency blocking IO |
| Scheduling | OS | JVM scheduler over carrier threads |

Virtual threads do not make CPU-bound code faster. They improve concurrency for blocking IO by reducing thread cost.

### 10.3 Pitfalls

- Pinning can happen with some synchronized/native/blocking sections.
- ThreadLocal overuse becomes more expensive at massive thread counts.
- Database connection pool is still a bottleneck.
- Backpressure is still required.

---

## 11. ZGC vs Shenandoah

Both are low-pause collectors.

| Dimension | ZGC | Shenandoah |
|-----------|-----|------------|
| Goal | ultra-low pause | low pause |
| Key technique | colored pointers, load barriers | Brooks pointer / barriers |
| Compaction | concurrent | concurrent |
| Pause relation to heap size | mostly low and stable | low and stable |
| Best for | large heaps, strict latency | latency-sensitive workloads |

Use G1 for general server workloads. Consider ZGC or Shenandoah when tail latency and large heap pauses become the main problem.

---

## 12. Records and Sealed Classes

### 12.1 Records

Records are concise immutable data carriers.

```java
public record UserDTO(long id, String name) {}
```

The compiler generates:

- Constructor.
- Accessors.
- `equals`.
- `hashCode`.
- `toString`.

Records are suitable for DTOs and value-like data, not entities with complex mutable lifecycle.

### 12.2 Sealed Classes

Sealed classes restrict which classes can extend them.

```java
public sealed interface PaymentResult
    permits Success, Failed, Pending {}

public record Success(String txId) implements PaymentResult {}
public record Failed(String reason) implements PaymentResult {}
public record Pending() implements PaymentResult {}
```

They help model closed hierarchies and algebraic-data-type-like structures.

---

## 13. Practical Cases

### Case 1: Payment Distributed Lock

Use Redis lock only to reduce duplicate concurrent work. The correctness boundary should be database transaction, unique constraints, and state machine.

```java
@Transactional
public void pay(String orderId) {
    Order order = orderRepository.findForUpdate(orderId);
    if (order.isPaid()) {
        return;
    }
    order.markPaid();
    ledgerRepository.insert(orderId);
}
```

### Case 2: Game Backend Thread Pool

For game backend async processing:

- Separate CPU-bound and IO-bound pools.
- Use bounded queues.
- Set meaningful rejection policy.
- Propagate trace/request IDs.
- Monitor queue length, active threads, task latency, and rejection count.

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    8,
    16,
    60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(10000),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

---

## 14. Interview Self-Check

### Quick Questions

### Q1: What is type erasure in Java generics?

**Answer:** Generic type information is checked at compile time and erased at runtime. `List<String>` and `List<Integer>` have the same runtime class. This improves compatibility but prevents `new T()`, generic arrays, and `instanceof List<String>`.

### Q2: What is PECS?

**Answer:** PECS means Producer Extends, Consumer Super. Use `? extends T` when a collection produces values for reading. Use `? super T` when a collection consumes values for writing.

### Q3: Why is String immutable?

**Answer:** Immutability enables string pool sharing, thread safety, stable hash code, and security for sensitive runtime mechanisms such as class loading and file paths.

### Q4: What JVM memory areas are thread-private?

**Answer:** Program counter, JVM stack, and native method stack are thread-private. Heap and metaspace are shared.

### Q5: What is JMM?

**Answer:** The Java Memory Model defines visibility and ordering rules between threads. It explains how synchronized, volatile, final fields, thread start/join, and happens-before relationships work.

### Q6: What does volatile provide?

**Answer:** `volatile` provides visibility and ordering guarantees. It does not make compound operations like `i++` atomic.

### Q7: What are core thread pool parameters?

**Answer:** `corePoolSize`, `maximumPoolSize`, `keepAliveTime`, `workQueue`, `threadFactory`, and `rejectedExecutionHandler`.

### Q8: What is Spring IoC?

**Answer:** IoC means the container creates and wires objects instead of application code manually constructing dependencies. It improves decoupling and lifecycle management.

### Q9: How does Spring AOP work?

**Answer:** Spring AOP is proxy-based. It uses JDK dynamic proxies for interfaces and CGLIB subclass proxies for classes. Calls must go through the proxy for advice to apply.

### Q10: What are virtual threads good for?

**Answer:** Virtual threads are good for high-concurrency IO-bound workloads using blocking-style code. They do not speed up CPU-bound computation and do not remove downstream pool limits.

### Deep-Dive Questions

### Q11: Explain HashMap put flow.

**Answer:** HashMap calculates hash, locates bucket by index, inserts if empty, otherwise compares keys in list/tree, updates existing value or appends new node. If chain grows beyond threshold and capacity is large enough, it treeifies. If load factor threshold is exceeded, it resizes.

### Q12: CMS vs G1?

**Answer:** CMS is a low-pause old-generation collector using concurrent mark-sweep, but it suffers from fragmentation and concurrent mode failure. G1 divides heap into regions, collects high-garbage regions first, supports compaction, and targets predictable pauses.

### Q13: Why can ZGC achieve very low pauses?

**Answer:** ZGC performs most marking, relocation, and remapping concurrently. It uses colored pointers and load barriers so object movement does not require long stop-the-world pauses proportional to heap size.

### Q14: Why does double-checked locking require volatile?

**Answer:** Object creation can be reordered as allocate memory, assign reference, then initialize object. Without volatile, another thread may observe a non-null reference to a partially initialized object. Volatile prevents this unsafe reordering and ensures visibility.

### Q15: When does `@Transactional` fail?

**Answer:** It can fail on self-invocation, non-public methods in proxy mode, caught exceptions, checked exceptions not configured for rollback, wrong transaction manager, or work moved to another thread.

### Q16: How does Spring solve circular dependencies?

**Answer:** For singleton setter/field injection, Spring exposes early object references through a three-level cache. Constructor injection cycles cannot be solved because both objects require fully constructed dependencies.

### Q17: Why can ThreadLocal leak memory?

**Answer:** `ThreadLocalMap` keys are weak references, but values are strong references. In thread pools, stale values can remain as long as the thread lives. Always call `remove()` in finally.

### Q18: How do you troubleshoot frequent Full GC?

**Answer:** Collect GC logs, check heap usage and allocation rate, inspect old generation growth, dump heap if needed, find large objects or leaks, check metaspace and direct memory, and correlate with traffic and deployment changes.

### Q19: How do you diagnose Metaspace growth?

**Answer:** Look for classloader leaks, dynamic proxy generation, hot reload, script engines, or repeated deployment. Use class histogram, jcmd VM.classloader_stats, heap dump, and metaspace metrics.

### Q20: How do you design retry-friendly idempotent APIs?

**Answer:** Use idempotency keys, unique constraints, state machines, request deduplication, retry-safe error codes, and clear timeout semantics. The server should treat duplicate requests as successful reads of the existing result when possible.

### Open-Ended Design Questions

### D1: Design a high-throughput Java order service.

**Reference approach:**

- Use bounded thread pools and connection pools.
- Keep transaction boundaries short.
- Use idempotency keys and database constraints.
- Add async messaging for non-critical downstream work.
- Monitor JVM GC, thread pool queues, DB pool wait, P99 latency, and error rates.

### D2: A Java service has high P99 latency after traffic grows. How do you troubleshoot?

**Reference approach:**

- Check gateway and application latency distribution.
- Inspect thread pools, DB pool wait, slow SQL, downstream RPC, GC logs, lock contention, and CPU.
- Use jstack, jcmd, async-profiler, GC logs, and tracing.
- Fix the actual bottleneck instead of only increasing heap or thread count.
