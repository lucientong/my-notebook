# Python Development

Language: English | [中文](../后端知识库/03-Python开发.md)

---

## Table of Contents

### Language Mechanisms
1. [Python Core Features](#1-python-core-features)
2. [Memory Management and Garbage Collection](#2-memory-management-and-garbage-collection)
3. [Closures and Scope](#3-closures-and-scope)
4. [Iterators and Generators](#4-iterators-and-generators)
5. [GIL and Concurrency Model](#5-gil-and-concurrency-model)
6. [Decorators and Metaprogramming](#6-decorators-and-metaprogramming)

### Async and Frameworks
7. [asyncio Coroutines](#7-asyncio-coroutines)
8. [Django ORM Optimization](#8-django-orm-optimization)
9. [FastAPI Practice](#9-fastapi-practice)

### Engineering Practice and Interview Review
10. [Performance Optimization and Debugging](#10-performance-optimization-and-debugging)
11. [Practical Cases](#11-practical-cases)
12. [Interview Self-Check](#12-interview-self-check)

---

## 1. Python Core Features

### 1.1 Data Model and Magic Methods

Python's data model is built around protocols. Special methods, often called magic methods, let objects participate in built-in language behavior.

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __len__(self):
        return 2
```

Common protocols:

| Protocol | Methods | Use |
|----------|---------|-----|
| String representation | `__repr__`, `__str__` | debugging and display |
| Container | `__len__`, `__getitem__`, `__contains__` | list-like behavior |
| Iterator | `__iter__`, `__next__` | `for` loops |
| Context manager | `__enter__`, `__exit__` | `with` |
| Callable | `__call__` | object as function |
| Comparison | `__eq__`, `__lt__` | sorting and equality |

Python is "protocol-oriented" in the sense that behavior is often determined by whether an object implements the right methods, not by explicit inheritance.

### 1.2 Mutable vs Immutable Objects

Immutable examples:

- `int`
- `float`
- `str`
- `tuple`
- `frozenset`

Mutable examples:

- `list`
- `dict`
- `set`
- most custom objects

Classic pitfall: mutable default arguments.

```python
def append_item(x, items=[]):  # bad
    items.append(x)
    return items
```

Correct version:

```python
def append_item(x, items=None):
    if items is None:
        items = []
    items.append(x)
    return items
```

Default arguments are evaluated once when the function is defined, not each time it is called.

### 1.3 Argument Passing

Python uses object reference passing, often described as "call by sharing".

```python
def mutate(xs):
    xs.append(1)

def rebind(xs):
    xs = [1, 2, 3]

a = []
mutate(a)  # a becomes [1]
rebind(a)  # a is still [1]
```

Mutation affects the object. Rebinding the local name does not change the caller's variable binding.

---

## 2. Memory Management and Garbage Collection

### 2.1 Reference Counting

CPython primarily uses reference counting. Each object tracks how many references point to it.

```python
import sys

x = []
print(sys.getrefcount(x))
```

When reference count drops to zero, the object can be deallocated immediately.

Pros:

- Simple.
- Most objects are reclaimed promptly.
- Predictable cleanup in many cases.

Cons:

- Reference cycles cannot be handled by reference counting alone.
- Increment/decrement operations add overhead.

### 2.2 Cycles and Mark-and-Sweep

Reference cycle:

```python
a = []
b = []
a.append(b)
b.append(a)
```

Even if external references are gone, `a` and `b` still reference each other. CPython's cyclic GC detects such cycles.

High-level process:

```text
1. Track container objects.
2. Find groups of objects reachable only from each other.
3. Determine whether they are unreachable from roots.
4. Collect them if safe.
```

### 2.3 Generational GC

Python's cyclic GC is generational:

```text
Generation 0: young objects, collected frequently.
Generation 1: survivors, collected less often.
Generation 2: long-lived objects, collected least often.
```

Rationale: most objects die young.

```python
import gc

print(gc.get_threshold())
print(gc.get_count())
gc.collect()
```

### 2.4 PyMalloc

CPython uses PyMalloc for small object allocation.

```text
Arena -> Pool -> Block
```

This reduces system allocator overhead for many small Python objects.

Important interview nuance:

- Freeing Python objects does not always immediately return memory to the OS.
- Memory may be kept in Python's allocator for reuse.
- RSS staying high does not necessarily mean a leak, but it needs investigation.

---

## 3. Closures and Scope

### 3.1 LEGB Rule

Name resolution order:

```text
L: Local
E: Enclosing
G: Global
B: Builtins
```

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        return x

    return inner()
```

### 3.2 global and nonlocal

`global` modifies a module-level variable.

```python
count = 0

def inc():
    global count
    count += 1
```

`nonlocal` modifies a variable in an enclosing function scope.

```python
def counter():
    count = 0

    def inc():
        nonlocal count
        count += 1
        return count

    return inc
```

### 3.3 Closure

A closure captures variables from an enclosing scope.

```python
def make_multiplier(n):
    def mul(x):
        return x * n
    return mul

double = make_multiplier(2)
print(double(10))
```

### 3.4 Loop Variable Capture Pitfall

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])  # [2, 2, 2]
```

Fix by binding the current value as a default argument:

```python
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)
```

---

## 4. Iterators and Generators

### 4.1 Iterator Protocol

An iterator implements `__iter__` and `__next__`.

```python
class CountDown:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return self

    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n
```

`for` loop internally calls `iter()` and `next()`.

### 4.2 Generator

A generator function uses `yield`.

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1
```

Benefits:

- Lazy evaluation.
- Low memory usage.
- Natural representation of streams and pipelines.

### 4.3 send, throw, and close

```python
def echo():
    while True:
        value = yield
        print(value)

g = echo()
next(g)
g.send("hello")
g.close()
```

Generators can receive values, handle injected exceptions, and be closed.

### 4.4 yield from

`yield from` delegates to another iterator or generator.

```python
def chain():
    yield from [1, 2, 3]
    yield from range(4, 6)
```

It also forwards `send`, `throw`, and `close` in coroutine-style generator code.

---

## 5. GIL and Concurrency Model

### 5.1 What Is the GIL? ⭐⭐⭐

GIL, Global Interpreter Lock, is a CPython mechanism that allows only one thread to execute Python bytecode at a time in one process.

Why it exists:

- Simplifies memory management and reference counting.
- Protects CPython object internals.
- Makes C extension integration easier historically.

What it does not mean:

- It does not mean Python cannot do concurrent IO.
- It does not mean Python cannot use multiple CPU cores through multiprocessing or native extensions.

### 5.2 Impact

CPU-bound multi-threading:

```text
Multiple Python threads compete for the GIL.
Only one executes Python bytecode at a time.
CPU-bound threads often do not scale.
```

IO-bound multi-threading:

```text
When a thread blocks on IO, CPython releases the GIL.
Other threads can run.
Threading can improve IO concurrency.
```

### 5.3 Concurrency Options

| Model | Best For | Notes |
|-------|----------|-------|
| `threading` | IO-bound tasks | Simple, shares memory, GIL limits CPU parallelism |
| `multiprocessing` | CPU-bound tasks | Real parallelism, higher memory and serialization cost |
| `asyncio` | High-concurrency IO | Cooperative scheduling, requires async libraries |
| Native extension | CPU-heavy libraries | NumPy and similar libraries may release the GIL |

### 5.4 threading

```python
from concurrent.futures import ThreadPoolExecutor

def fetch(url):
    return requests.get(url, timeout=3).text

with ThreadPoolExecutor(max_workers=20) as pool:
    results = list(pool.map(fetch, urls))
```

Use for blocking IO, not heavy Python CPU loops.

### 5.5 multiprocessing

```python
from multiprocessing import Pool

def compute(x):
    return x * x

with Pool() as pool:
    results = pool.map(compute, range(1000))
```

Trade-offs:

- True CPU parallelism.
- Separate memory spaces.
- Serialization and IPC overhead.
- More operational cost.

### 5.6 Free-Threaded Python Note

Python 3.13 introduces an experimental free-threaded build from PEP 703. It is important for the future, but most production deployments still use the traditional GIL build today. In interviews, distinguish CPython's mainstream behavior from emerging versions.

---

## 6. Decorators and Metaprogramming

### 6.1 Decorators

A decorator is a function that takes a function and returns a new function.

```python
import functools
import time

def timing(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        try:
            return func(*args, **kwargs)
        finally:
            print(func.__name__, time.time() - start)
    return wrapper

@timing
def work():
    pass
```

Use `functools.wraps` to preserve metadata such as name and docstring.

Decorator with parameters:

```python
def retry(times):
    def deco(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last = None
            for _ in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last = e
            raise last
        return wrapper
    return deco
```

### 6.2 Context Manager

```python
class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc, tb):
        self.cost = time.time() - self.start
        print(self.cost)
```

With `contextlib`:

```python
from contextlib import contextmanager

@contextmanager
def transaction():
    begin()
    try:
        yield
        commit()
    except Exception:
        rollback()
        raise
```

Context managers are ideal for resource lifecycle: locks, files, DB transactions, tracing spans.

### 6.3 Metaclass

A metaclass controls class creation.

```python
class RegistryMeta(type):
    registry = {}

    def __new__(mcls, name, bases, namespace):
        cls = super().__new__(mcls, name, bases, namespace)
        if name != "Base":
            mcls.registry[name] = cls
        return cls

class Base(metaclass=RegistryMeta):
    pass
```

Use cases:

- ORM model construction.
- Plugin registration.
- Validation of class definitions.
- Framework-level declarative APIs.

Guideline: prefer decorators or class decorators unless metaclass is truly needed.

---

## 7. asyncio Coroutines

### 7.1 Basic Concepts

`asyncio` is cooperative concurrency built around an event loop.

```python
import asyncio

async def fetch():
    await asyncio.sleep(1)
    return "ok"

async def main():
    result = await fetch()
    print(result)

asyncio.run(main())
```

`async def` creates a coroutine function. `await` yields control to the event loop.

### 7.2 Concurrent Execution

```python
async def main():
    results = await asyncio.gather(
        fetch_user(),
        fetch_orders(),
        fetch_recommendations(),
    )
```

`gather` runs awaitables concurrently, but only if they actually await non-blocking operations.

### 7.3 Timeout and Cancellation

```python
try:
    result = await asyncio.wait_for(fetch(), timeout=2)
except asyncio.TimeoutError:
    handle_timeout()
```

Cancellation is cooperative. A coroutine receives `CancelledError` at an await point.

```python
task = asyncio.create_task(fetch())
task.cancel()
```

### 7.4 Async Context Manager

```python
class AsyncResource:
    async def __aenter__(self):
        await self.open()
        return self

    async def __aexit__(self, exc_type, exc, tb):
        await self.close()
```

### 7.5 Async Iterator

```python
class AsyncCounter:
    def __init__(self, n):
        self.n = n

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.n <= 0:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        self.n -= 1
        return self.n
```

### 7.6 aiohttp Concurrent Crawler

```python
import aiohttp
import asyncio

async def fetch(session, url):
    async with session.get(url, timeout=3) as resp:
        return await resp.text()

async def main(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

asyncio.run(main(urls))
```

Do not call blocking libraries such as `requests` inside async code. Use async-compatible libraries or run blocking work in an executor.

---

## 8. Django ORM Optimization

### 8.1 N+1 Query Problem

Bad:

```python
orders = Order.objects.all()
for order in orders:
    print(order.user.name)  # extra query per order
```

Fix with `select_related` for foreign keys:

```python
orders = Order.objects.select_related("user").all()
```

Use `prefetch_related` for many-to-many or reverse foreign keys:

```python
authors = Author.objects.prefetch_related("books").all()
```

### 8.2 Query Optimization

Fetch only needed fields:

```python
User.objects.only("id", "name")
User.objects.values("id", "name")
```

Avoid loading huge result sets:

```python
for user in User.objects.iterator(chunk_size=1000):
    process(user)
```

Use database indexes for filter, join, and sort paths.

### 8.3 Transactions

```python
from django.db import transaction

with transaction.atomic():
    order = Order.objects.select_for_update().get(id=order_id)
    if order.status == "paid":
        return
    order.status = "paid"
    order.save(update_fields=["status"])
```

Use `select_for_update` to lock rows when implementing state transitions such as payment, inventory deduction, or account balance updates.

### 8.4 Bulk Operations

```python
User.objects.bulk_create(users, batch_size=1000)
User.objects.bulk_update(users, ["name"], batch_size=1000)
```

Bulk operations reduce round trips but may skip per-object `save()` logic and signals depending on method. Be explicit about side effects.

---

## 9. FastAPI Practice

### 9.1 Basic Usage

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class UserOut(BaseModel):
    id: int
    name: str

@app.get("/users/{user_id}", response_model=UserOut)
async def get_user(user_id: int):
    return {"id": user_id, "name": "Alice"}
```

FastAPI strengths:

- Type-hint based validation.
- Pydantic models.
- Automatic OpenAPI docs.
- Natural async support.

### 9.2 Dependency Injection

```python
from fastapi import Depends

def get_db():
    db = Session()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{user_id}")
def get_user(user_id: int, db=Depends(get_db)):
    return db.get(User, user_id)
```

Use dependencies for DB sessions, authentication, authorization, configuration, and reusable request-scoped resources.

### 9.3 Middleware

```python
@app.middleware("http")
async def add_request_id(request, call_next):
    request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

Middleware is useful for logging, tracing, CORS, auth boundaries, and error handling.

### 9.4 Background Tasks

```python
from fastapi import BackgroundTasks

def send_email(email: str):
    ...

@app.post("/signup")
async def signup(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_email, email)
    return {"ok": True}
```

Use background tasks for short non-critical work. For reliable asynchronous workflows, use a real queue such as Celery, Kafka, RabbitMQ, or Redis Streams.

---

## 10. Performance Optimization and Debugging

### 10.1 cProfile

```python
import cProfile
import pstats

cProfile.run("main()", "profile.out")

stats = pstats.Stats("profile.out")
stats.sort_stats("cumtime").print_stats(20)
```

Use it to locate CPU hotspots in synchronous Python code.

### 10.2 memory_profiler

```python
from memory_profiler import profile

@profile
def work():
    data = [i for i in range(1_000_000)]
    return sum(data)
```

Also use:

- `tracemalloc`
- object counts
- heap snapshots
- RSS and container memory metrics

### 10.3 Common Optimization Techniques

- Avoid N+1 queries.
- Use generators for large streams.
- Use `__slots__` for many small objects when appropriate.
- Prefer built-in operations and comprehensions for simple loops.
- Move CPU-heavy work to native libraries, processes, or separate services.
- Use async only when the whole IO path is async-compatible.
- Add timeouts to all network and database calls.

---

## 11. Practical Cases

### Case 1: Async Task Queue in a Game Backend

Scenario: player activity events need to be processed asynchronously without blocking request latency.

Simple in-process async queue:

```python
queue = asyncio.Queue(maxsize=10000)

async def producer(event):
    await queue.put(event)

async def worker():
    while True:
        event = await queue.get()
        try:
            await process_event(event)
        finally:
            queue.task_done()
```

Production notes:

- In-process queues are not durable.
- Use Redis Streams, RabbitMQ, Kafka, or Celery for reliability.
- Add backpressure and dead-letter handling.

### Case 2: Database Transaction Retry in Payment

```python
from django.db import transaction, OperationalError

def pay(order_id):
    for _ in range(3):
        try:
            with transaction.atomic():
                order = Order.objects.select_for_update().get(id=order_id)
                if order.status == "paid":
                    return
                order.status = "paid"
                order.save(update_fields=["status"])
                Ledger.objects.create(order=order)
            return
        except OperationalError:
            continue
    raise PaymentRetryExceeded()
```

Correctness should rely on database constraints, transactions, idempotency keys, and state machines, not only on application locks.

---

## 12. Python Engineering Updates

### Descriptor, Typing, and Attribute Lookup

Descriptors implement `__get__`, `__set__`, or `__delete__`. Attribute lookup prioritizes data descriptors, then instance `__dict__`, then non-data descriptors/class attributes, then `__getattr__`. `property`, method binding, ORM fields, `staticmethod`, and `classmethod` all rely on this model. `Protocol` expresses structural typing; type hints mainly support static checking and framework contracts.

### asyncio, Frameworks, Tests, and Dependencies

`TaskGroup` gives structured concurrency and cancellation propagation. CPU-heavy work should go to a process pool or executor, not the event loop. WSGI is synchronous; ASGI supports async, WebSocket, and lifespan. FastAPI `async def` is not automatically faster if it calls blocking DB or HTTP clients. pytest fixtures manage resources, parametrize covers input matrices, and lockfiles make dependency resolution reproducible.

### Correctness Notes

`await` has historical roots in generator delegation, but modern `await` uses the `__await__` protocol; do not say it is simply `yield from`. CPython `list.append` is usually atomic as a single C operation under the GIL, but compound invariants like check-then-append still require locks or single-thread ownership. `SIGALRM` timeout decorators are Unix/main-thread specific and are not a general web-service timeout strategy.

## 13. Interview Self-Check

### Quick Questions

### Q1: What is Python's data model?

**Answer:** Python's data model defines how objects interact with language syntax and built-ins through special methods such as `__iter__`, `__len__`, `__enter__`, and `__call__`. It is protocol-based rather than only inheritance-based.

### Q2: Why are mutable default arguments dangerous?

**Answer:** Default arguments are evaluated once at function definition time. A mutable default such as `[]` is shared across calls, so later calls may see data from earlier calls. Use `None` and create a new object inside the function.

### Q3: How does Python pass function arguments?

**Answer:** Python passes object references by assignment, often called call by sharing. Mutating a passed object affects the caller's object, but rebinding the local variable does not change the caller's binding.

### Q4: How does CPython manage memory?

**Answer:** CPython primarily uses reference counting, with cyclic GC for reference cycles and PyMalloc for small object allocation. Reference count zero usually releases objects immediately, while cycles require the garbage collector.

### Q5: What is the LEGB rule?

**Answer:** Name lookup follows Local, Enclosing, Global, Builtins. `global` targets module-level names, while `nonlocal` targets variables in enclosing function scopes.

### Q6: What is the loop closure pitfall?

**Answer:** Closures capture variables, not their immediate values. Lambdas created in a loop may all reference the final loop variable. Bind the current value with a default argument such as `lambda i=i: i`.

### Q7: What is the difference between iterator and generator?

**Answer:** An iterator implements `__iter__` and `__next__`. A generator is a convenient way to create an iterator using `yield`, with lazy execution and lower memory usage.

### Q8: What is the GIL?

**Answer:** The GIL is CPython's Global Interpreter Lock. It allows only one thread to execute Python bytecode at a time in one process. It simplifies memory management but limits CPU-bound multi-threaded parallelism.

### Q9: When should you use threading, multiprocessing, and asyncio?

**Answer:** Use threading for blocking IO, multiprocessing for CPU-bound parallel work, and asyncio for high-concurrency IO with async-compatible libraries.

### Q10: Why should decorators use `functools.wraps`?

**Answer:** `wraps` preserves function metadata such as `__name__`, `__doc__`, and annotations. Without it, introspection, debugging, documentation, and frameworks may see the wrapper instead of the original function.

### Deep-Dive Questions

### Q11: Why does CPython need cyclic GC if it already has reference counting?

**Answer:** Reference counting cannot reclaim objects in cycles because their reference counts never drop to zero. Cyclic GC identifies groups of objects that are reachable only from each other and not from program roots, then collects them.

### Q12: Why does process RSS not always drop after deleting Python objects?

**Answer:** CPython and the underlying allocator may keep memory arenas for reuse instead of returning them to the OS immediately. RSS staying high may be allocator behavior, fragmentation, or true retention; it requires profiling.

### Q13: How does `yield from` work?

**Answer:** `yield from` delegates iteration to a sub-iterator. It forwards yielded values and also propagates `send`, `throw`, and `close` for generator-based coroutine patterns. It simplifies nested generator composition.

### Q14: Why does GIL not hurt IO-bound workloads as much?

**Answer:** When a thread blocks on IO, CPython can release the GIL, allowing other threads to run. For IO-bound workloads, waiting time dominates CPU bytecode execution, so threading can still improve concurrency.

### Q15: How do you make CPU-heavy Python code faster?

**Answer:** Use better algorithms first. Then move hot loops to built-ins, NumPy, C extensions, Numba, Cython, multiprocessing, or a separate service. Threads usually do not speed up CPU-bound Python bytecode because of the GIL.

### Q16: What is a metaclass and when should you use it?

**Answer:** A metaclass controls class creation. It can register classes, validate class definitions, or build declarative APIs such as ORMs. Use it sparingly; decorators or class decorators are often simpler.

### Q17: How does asyncio achieve concurrency?

**Answer:** `asyncio` uses an event loop and cooperative scheduling. Coroutines yield control at `await` points. While one coroutine waits for non-blocking IO, the event loop runs other ready coroutines.

### Q18: What happens if you call blocking code inside an async function?

**Answer:** It blocks the event loop, preventing other coroutines from running. Use async-compatible libraries or run blocking work in an executor.

### Q19: How do you solve Django ORM N+1 queries?

**Answer:** Use `select_related` for foreign keys and one-to-one relations, and `prefetch_related` for many-to-many or reverse relations. Also inspect SQL queries and add indexes for common filters and joins.

### Q20: How do you handle database consistency in Django payment flows?

**Answer:** Use `transaction.atomic`, row-level locks such as `select_for_update`, unique constraints, idempotency keys, and state machines. Retry only safe transient errors and avoid double-charging through database constraints.

### Q21: FastAPI sync vs async endpoint: how do you choose?

**Answer:** Use async endpoints when the call path uses async-compatible IO libraries. Use sync endpoints for blocking libraries or CPU-light synchronous work. Async syntax alone does not improve performance if the underlying calls block.

### Q22: How do you investigate high memory usage in a Python service?

**Answer:** Check RSS, Python heap, object counts, and allocation traces with `tracemalloc` or memory profilers. Look for unbounded caches, retained references, large response bodies, queues, cycles with finalizers, and allocator fragmentation.

### Q23: How do you investigate high P99 latency in a Python web service?

**Answer:** Break it down into application CPU, DB queries, downstream RPC, event loop blocking, thread pool saturation, GC, connection pool wait, and serialization. Use tracing, slow query logs, profiling, and per-dependency latency metrics.

### Open-Ended Design Questions

### D1: Design a high-concurrency Python API service.

**Reference approach:**

- Use FastAPI or similar framework.
- Use async DB/HTTP clients only if the full path is async-compatible.
- Add timeouts, connection pools, request IDs, structured logging, and tracing.
- Avoid blocking event loop; offload CPU-heavy work.
- Use queues for reliable async tasks.
- Monitor event loop lag, P99 latency, DB pool usage, memory, and error rate.

### D2: A Python service CPU is 100% but throughput does not improve with more threads. Why?

**Reference approach:**

- CPU-bound Python bytecode is limited by the GIL.
- More threads increase context switching but do not provide real parallel execution.
- Profile to find hot code.
- Optimize algorithm or move CPU work to multiprocessing, native extensions, NumPy, Cython, Numba, or a separate compute service.

### D3: How would you build reliable background processing for a Python backend?

**Reference approach:**

- Do not rely only on in-process background tasks for critical work.
- Use a durable queue such as Celery/RabbitMQ, Redis Streams, Kafka, or a workflow engine.
- Add idempotency keys, retry budget, exponential backoff, DLQ, observability, and replay tooling.
- Keep DB state transitions transactional and make consumers idempotent.


### Additional Senior Questions

### Q23: What is descriptor lookup order?
Data descriptor, instance dictionary, non-data descriptor/class attribute, then `__getattr__`.

### Q24: WSGI vs ASGI?
WSGI is synchronous request/response. ASGI supports async, WebSocket, and application lifespan.

### Q25: Is `list.append` thread-safe?
Single append is usually atomic in CPython, but that is not a language guarantee and does not make compound operations thread-safe.
