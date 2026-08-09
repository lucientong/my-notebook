# Concurrency Programming Models

Language: English | [中文](../后端知识库/05-并发编程模型.md)

---

## Table of Contents

1. [Concurrency Basics](#1-concurrency-basics)
2. [Go Concurrency Model](#2-go-concurrency-model)
3. [Go Channel Internals](#3-go-channel-internals)
4. [Python Concurrency Model](#4-python-concurrency-model)
5. [Java Concurrency Model](#5-java-concurrency-model)
6. [Concurrency Primitives](#6-concurrency-primitives)
7. [Concurrency Design Patterns](#7-concurrency-design-patterns)
8. [Common Concurrency Problems](#8-common-concurrency-problems)
9. [Advanced Topics](#9-advanced-topics)
10. [Interview Self-Check](#10-interview-self-check)

---

## 1. Concurrency Basics

### 1.1 Concurrency vs Parallelism

Concurrency is about structuring multiple tasks that make progress over time. Parallelism is about executing multiple tasks at the same time.

```text
Concurrency: handle many things
Parallelism: do many things simultaneously
```

A single-core machine can be concurrent through scheduling. Parallelism requires multiple cores or execution units.

### 1.2 Process vs Thread vs Coroutine

| Unit | Isolation | Cost | Typical Use |
|------|-----------|------|-------------|
| Process | strong | high | isolation, multi-service |
| Thread | shared memory | medium | CPU/IO concurrency |
| Coroutine | user-space scheduling | low | massive IO concurrency |

Coroutines are lightweight but still need careful cancellation, backpressure, and resource control.

---

## 2. Go Concurrency Model

### 2.1 Goroutine

Goroutines are lightweight user-space execution units managed by the Go runtime.

```go
go func() {
    doWork()
}()
```

They are scheduled by the GMP model:

- G: goroutine.
- M: OS thread.
- P: processor, holds local run queue.

### 2.2 Channel

Channels are typed communication primitives.

```go
ch := make(chan int, 10)
ch <- 1
v := <-ch
```

Use channels to communicate ownership and events. Do not use them as a universal replacement for locks.

### 2.3 select

```go
select {
case v := <-ch:
    handle(v)
case <-ctx.Done():
    return
}
```

`select` waits on multiple channel operations and is central to cancellation and timeout handling.

### 2.4 WaitGroup and Context

`sync.WaitGroup` waits for goroutines. `context.Context` carries cancellation, deadline, and request-scoped values.

Always avoid goroutine leaks by ensuring every goroutine has a clear exit condition.

---

## 3. Go Channel Internals

### 3.1 hchan

Conceptually, a channel contains:

- circular buffer.
- send queue.
- receive queue.
- lock.
- element type and size.
- closed flag.

### 3.2 Send and Receive

Unbuffered channel:

- sender and receiver synchronize directly.
- send blocks until receive is ready.

Buffered channel:

- send blocks only when buffer is full.
- receive blocks only when buffer is empty.

### 3.3 Close Semantics

Closing a channel broadcasts completion to receivers.

Rules:

- Only sender side should close.
- Sending to closed channel panics.
- Receiving from closed channel returns zero value and `ok=false`.

---

## 4. Python Concurrency Model

### 4.1 GIL

CPython has a Global Interpreter Lock that allows only one thread to execute Python bytecode at a time.

Impact:

- Threads are useful for IO-bound work.
- CPU-bound Python code needs multiprocessing, native extensions, or other runtimes.

### 4.2 threading

Python threads are OS threads but constrained by GIL for bytecode execution.

### 4.3 multiprocessing

Multiprocessing bypasses GIL by using multiple processes.

Trade-offs:

- Higher memory cost.
- Serialization overhead.
- IPC complexity.

### 4.4 asyncio

`asyncio` uses event loop and cooperative scheduling.

```python
async def fetch():
    await client.get(url)
```

Best for high-concurrency IO when libraries are async-compatible.

---

## 5. Java Concurrency Model

### 5.1 Thread and ExecutorService

Java platform threads map to OS threads.

```java
ExecutorService pool = Executors.newFixedThreadPool(8);
pool.submit(() -> doWork());
```

Prefer explicitly configured `ThreadPoolExecutor` in production.

### 5.2 synchronized and Lock

`synchronized` provides mutual exclusion and visibility. `ReentrantLock` provides interruptible lock, timed lock, fairness option, and multiple conditions.

### 5.3 Concurrent Collections

Examples:

- `ConcurrentHashMap`.
- `BlockingQueue`.
- `CopyOnWriteArrayList`.
- `ConcurrentLinkedQueue`.

Choose by read/write pattern and consistency needs.

### 5.4 ForkJoinPool

ForkJoinPool uses work stealing and is good for recursive divide-and-conquer tasks.

Be careful with blocking operations in `parallelStream` because it uses the common pool by default.

---

## 6. Concurrency Primitives

### 6.1 Mutex

Mutex protects critical sections.

Guidelines:

- Keep critical sections short.
- Use consistent lock ordering.
- Avoid blocking IO while holding locks.

### 6.2 Read-Write Lock

RW locks help read-heavy workloads but may hurt performance if writes are frequent or critical sections are short.

### 6.3 Semaphore

A semaphore limits concurrent access to a finite resource, such as outbound requests or database connections.

### 6.4 Condition Variable

Condition variables coordinate state changes. Always wait in a loop or with a predicate to handle spurious wakeups.

---

## 7. Concurrency Design Patterns

### 7.1 Fan-Out / Fan-In

Fan-out distributes work to multiple workers. Fan-in aggregates results.

### 7.2 Pipeline

Pipeline splits processing into stages connected by queues or channels.

### 7.3 Worker Pool

Worker pool limits concurrency and protects downstream resources.

Production concerns:

- bounded queues.
- timeouts.
- cancellation.
- backpressure.
- metrics.

---

## 8. Common Concurrency Problems

### 8.1 Race Condition

A race happens when correctness depends on timing of unsynchronized access.

Use race detectors, locks, atomics, channels, or immutable data.

### 8.2 Deadlock

Deadlock requires mutual exclusion, hold-and-wait, no preemption, and circular wait.

Prevention:

- fixed lock order.
- timeout.
- smaller critical sections.
- avoid nested locks.

### 8.3 Livelock and Starvation

Livelock means threads keep reacting but make no progress. Starvation means a task cannot get required resources.

---

## 9. Advanced Topics

### 9.1 Go sync.Map

`sync.Map` uses a read map and dirty map to optimize read-mostly workloads.

Use it for:

- cache-like maps with many reads.
- keys written once and read many times.

For ordinary maps with balanced reads/writes, `map + RWMutex` is often clearer.

### 9.2 Lock-Free Queue

Lock-free queues often rely on CAS and linked nodes.

Hard problems:

- ABA.
- memory reclamation.
- fairness.
- correctness testing.

Use proven implementations unless you have a strong reason to build one.

---

## 10. Memory Model and IO Concurrency Updates

### Memory Model, CAS, and Barriers

happens-before is about visibility and ordering, not just wall-clock order. Mutex unlock/lock, channel send/receive, close observation, join, volatile, and atomics create synchronization edges. CAS is useful but can suffer from ABA; version tags, hazard pointers, epochs, or locks may be needed. Atomic operations suit single-variable state, while multi-field invariants usually need locks.

### IO Multiplexing, Reactor, and Proactor

select and poll linearly scan fd sets; epoll/kqueue keep interest sets in the kernel and return ready events. Reactor means readiness notification and application-managed read/write. Proactor means completion notification after the kernel completes IO. Edge-triggered mode requires draining until EAGAIN.

### Go GMP, Worker Pools, and Distributed Locks

Go netpoller parks goroutines waiting for network IO and wakes them when epoll/kqueue reports readiness, allowing goroutine-per-connection without one OS thread per connection. A production worker pool needs bounded queues, rejection/backpressure, timeouts, cancellation, and saturation metrics. Redis `SET NX PX` is only a lease; fencing tokens are needed to reject stale writers after pauses or network partitions.

## 11. Interview Self-Check

### Q1: What is the difference between concurrency and parallelism?

**Answer:** Concurrency is structuring multiple tasks to make progress. Parallelism is executing multiple tasks at the same time.

### Q2: Why can goroutines be cheaper than threads?

**Answer:** Goroutines start with small stacks and are scheduled by the Go runtime over OS threads, so creation and switching are cheaper.

### Q3: How do you prevent goroutine leaks?

**Answer:** Use context cancellation, close channels correctly, avoid blocked sends/receives, and make ownership of goroutine lifecycle explicit.

### Q4: What is the GIL?

**Answer:** CPython's GIL allows only one thread to execute Python bytecode at a time. It simplifies memory management but limits CPU-bound parallelism.

### Q5: When should you use a worker pool?

**Answer:** When you need to bound concurrency, protect resources, apply backpressure, and control task lifecycle.

### Q6: How do you diagnose a deadlock?

**Answer:** Capture stack dumps, inspect lock ownership and waiting relationships, identify cycles, and check recent code paths that changed lock order.

### Q7: What is CAS?

**Answer:** Compare-And-Swap atomically updates a value only if it still equals an expected value. It is a building block for lock-free algorithms.

### Q8: What is ABA?

**Answer:** A value changes from A to B and back to A, so CAS succeeds even though the state changed. Version tags or safe reclamation can help.

### Q9: Java `synchronized` vs `ReentrantLock`?

**Answer:** `synchronized` is simpler and JVM-optimized. `ReentrantLock` provides interruptible lock, timed try lock, fairness, and multiple conditions.

### Q10: What should you monitor in concurrent systems?

**Answer:** Queue length, active workers, rejection count, lock contention, latency distribution, goroutine/thread count, CPU, memory, and downstream saturation.

### Senior Interview Follow-Ups

### Q11: A service has increasing goroutine/thread count and rising P99 latency. How would you debug it?

**Answer:** First separate traffic growth from leaks by checking request rate, queue length, goroutine/thread count, memory, and blocked profile. Then capture runtime dumps: Go goroutine profiles, Java thread dumps, Python stack snapshots, and lock/blocking profiles. Look for blocked channel sends, missing context cancellation, unbounded executors, forgotten futures, slow downstream calls, and queues without backpressure. Mitigation is usually bounding concurrency, adding deadlines, making cancellation explicit, and exposing lifecycle metrics for every worker pool.

### Q12: How do you choose between locks, channels, atomics, and immutable data?

**Answer:** Start from ownership and invariants. Use immutable data when reads dominate and updates can be published safely. Use a mutex when multiple fields must change atomically under one invariant. Use channels for ownership transfer, event streams, and cancellation, not for every shared variable. Use atomics only for small independent state such as counters, flags, or carefully reviewed lock-free paths. The production trade-off is not just speed; debuggability and correctness usually matter more.

### Q13: What is a good failure mode for an overloaded worker pool?

**Answer:** A senior design uses bounded queues, admission control, deadlines, rejection metrics, and caller-visible fallback. Unbounded queues hide overload until memory grows and latency explodes. A better pool rejects early, sheds low-priority work, protects downstream dependencies, and preserves enough observability to answer whether bottleneck is CPU, lock contention, queueing, or external IO.


### Additional Senior Questions

### Q21: select/poll/epoll and Reactor/Proactor?
select/poll scan many fds; epoll/kqueue return ready events. Reactor handles readiness, Proactor handles completion.

### Q22: Why is an unbounded worker queue dangerous?
It hides overload as latency and memory growth until OOM. Use bounded queues and explicit backpressure or rejection.

### Q23: Why is Redis SET NX PX not a complete distributed lock?
The holder may pause past lease expiry and still write. Fencing tokens let downstream reject stale holders.
