# C++ System Programming

Language: English | [中文](../后端知识库/15-C++系统编程.md)

---

## Table of Contents

1. [Modern C++ Evolution](#1-modern-c-evolution)
2. [Memory Model and Object Lifetime](#2-memory-model-and-object-lifetime)
3. [Smart Pointers and Resource Management](#3-smart-pointers-and-resource-management)
4. [Move Semantics and Perfect Forwarding](#4-move-semantics-and-perfect-forwarding)
5. [Templates and Generic Programming](#5-templates-and-generic-programming)
6. [Multithreading and Synchronization](#6-multithreading-and-synchronization)
7. [Atomics and Memory Ordering](#7-atomics-and-memory-ordering)
8. [STL Containers and Algorithms](#8-stl-containers-and-algorithms)
9. [Performance Optimization](#9-performance-optimization)
10. [Compilation, Linking, and Build Systems](#10-compilation-linking-and-build-systems)
11. [Common Pitfalls and Best Practices](#11-common-pitfalls-and-best-practices)
12. [Practical Cases](#12-practical-cases)
13. [Interview Self-Check](#13-interview-self-check)

---

## 1. Modern C++ Evolution

### 1.1 C++11: The Start of Modern C++

Important features:

- `auto`.
- Range-based `for`.
- Lambda expressions.
- Move semantics.
- Rvalue references.
- Smart pointers.
- `nullptr`.
- `constexpr`.
- Thread library.

```cpp
auto values = std::vector<int>{1, 2, 3};
for (auto v : values) {
    std::cout << v << '\n';
}
```

C++11 changed the style of C++ from manual resource management to RAII-centered modern C++.

### 1.2 C++14: Convenience and Refinement

Important features:

- Generic lambdas.
- Return type deduction.
- `std::make_unique`.
- Relaxed `constexpr`.

```cpp
auto add = [](auto a, auto b) {
    return a + b;
};
```

### 1.3 C++17: Practical Utility Explosion

Important features:

- Structured bindings.
- `if constexpr`.
- `std::optional`.
- `std::variant`.
- `std::string_view`.
- Filesystem.
- Parallel algorithms.

```cpp
std::map<std::string, int> scores;
for (const auto& [name, score] : scores) {
    std::cout << name << ": " << score << '\n';
}
```

### 1.4 C++20: Four Flagship Features

Important features:

- Concepts.
- Ranges.
- Coroutines.
- Modules.

```cpp
template <typename T>
concept Addable = requires(T a, T b) {
    a + b;
};

template <Addable T>
T add(T a, T b) {
    return a + b;
}
```

For backend/system interviews, focus more on RAII, move semantics, memory model, concurrency, STL internals, and performance than on syntax lists.

---

## 2. Memory Model and Object Lifetime

### 2.1 Process Memory Layout

Typical process memory layout:

```text
High address
----------------
Stack
----------------
Heap
----------------
BSS
----------------
Data segment
----------------
Text segment
Low address
```

Key points:

- Stack stores function frames and local automatic variables.
- Heap stores dynamically allocated objects.
- Data segment stores initialized globals/statics.
- BSS stores zero-initialized globals/statics.
- Text segment stores executable code.

### 2.2 Object Construction and Destruction

C++ gives precise control over object lifetime.

```cpp
struct File {
    explicit File(const char* path) {
        fp = std::fopen(path, "r");
        if (!fp) throw std::runtime_error("open failed");
    }

    ~File() {
        if (fp) std::fclose(fp);
    }

    std::FILE* fp{};
};
```

Construction order:

```text
Base classes -> member fields -> constructor body
```

Destruction order is the reverse.

### 2.3 vtable and Polymorphism

For classes with virtual functions, compilers typically use a vptr pointing to a vtable.

```cpp
struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};
```

Virtual dispatch cost:

- One extra indirection.
- Harder inlining.
- Potential branch prediction cost.

Always use a virtual destructor in polymorphic base classes.

### 2.4 Rule of Three / Five / Zero

Rule of Three:

- Destructor.
- Copy constructor.
- Copy assignment.

Rule of Five adds:

- Move constructor.
- Move assignment.

Rule of Zero:

- Prefer composing RAII types so the compiler-generated special member functions are correct.

```cpp
class User {
    std::string name_;
    std::vector<int> scores_;
};
```

Rule of Zero is usually the best production style.

### 2.5 Alignment and `sizeof`

Objects may contain padding to satisfy alignment.

```cpp
struct A {
    char c;
    int i;
};
```

`sizeof(A)` is often 8, not 5, due to padding.

For performance-sensitive structs:

- Put larger-alignment fields first.
- Avoid false sharing.
- Consider cache line boundaries for hot counters.

---

## 3. Smart Pointers and Resource Management

### 3.1 RAII ⭐⭐⭐

RAII means Resource Acquisition Is Initialization. Bind resource lifetime to object lifetime.

```cpp
void write_file(const std::string& path) {
    std::ofstream out(path);
    out << "hello";
} // file closes automatically
```

Benefits:

- Exception safety.
- Clear ownership.
- Fewer leaks.
- Deterministic release.

RAII is the central idiom of modern C++.

### 3.2 `unique_ptr`

Exclusive ownership.

```cpp
auto p = std::make_unique<User>("lucien");
auto q = std::move(p);
```

Use it when exactly one owner exists. It has minimal overhead.

### 3.3 `shared_ptr`

Shared ownership through reference counting.

```cpp
auto p = std::make_shared<User>();
auto q = p;
```

Notes:

- Reference count operations are thread-safe.
- The pointed object is not automatically thread-safe.
- Cycles cause leaks unless broken by `weak_ptr`.

Prefer `make_shared` because it usually allocates object and control block together.

### 3.4 `weak_ptr`

Non-owning weak reference used to break cycles.

```cpp
std::weak_ptr<User> weak = shared;
if (auto locked = weak.lock()) {
    locked->do_work();
}
```

### 3.5 Smart Pointer Best Practices

- Prefer values and stack objects when possible.
- Use `unique_ptr` by default for ownership.
- Use `shared_ptr` only when ownership is truly shared.
- Use raw pointers or references for non-owning views.
- Avoid passing `shared_ptr` everywhere only to access an object.

---

## 4. Move Semantics and Perfect Forwarding

### 4.1 Value Categories

Common mental model:

```text
lvalue: has identity, can appear on left side
rvalue: temporary or movable value
```

```cpp
std::string s = "abc";      // s is lvalue
std::string t = std::move(s); // std::move(s) is rvalue expression
```

### 4.2 Move Constructor and Move Assignment

```cpp
class Buffer {
public:
    Buffer(Buffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

private:
    char* data_{};
    std::size_t size_{};
};
```

Move operations transfer resources instead of deep-copying them.

### 4.3 `std::move` and `std::forward`

`std::move` does not move anything by itself. It casts to an rvalue reference.

`std::forward` preserves value category in forwarding references.

```cpp
template <typename T>
void wrapper(T&& arg) {
    process(std::forward<T>(arg));
}
```

### 4.4 RVO and NRVO

Return Value Optimization allows construction directly in the caller's storage.

```cpp
std::string build() {
    return std::string("hello");
}
```

Do not write `return std::move(local)` for local variables in most cases, because it can inhibit NRVO.

---

## 5. Templates and Generic Programming

### 5.1 Function and Class Templates

```cpp
template <typename T>
T max_value(T a, T b) {
    return a < b ? b : a;
}
```

Templates are compile-time code generation. They enable zero-cost abstractions but can increase compile time and binary size.

### 5.2 Specialization and Partial Specialization

```cpp
template <typename T>
struct Traits;

template <>
struct Traits<int> {
    static constexpr const char* name = "int";
};
```

Full specialization handles exact types. Partial specialization handles a subset of type patterns.

### 5.3 Variadic Templates

```cpp
template <typename... Args>
void log(Args&&... args) {
    (std::cout << ... << args) << '\n';
}
```

Useful for forwarding, formatting, tuple-like utilities, and generic factories.

### 5.4 SFINAE and `enable_if`

SFINAE means Substitution Failure Is Not An Error.

```cpp
template <typename T>
std::enable_if_t<std::is_integral_v<T>, bool>
is_even(T x) {
    return x % 2 == 0;
}
```

SFINAE enables conditional overloads but error messages can be hard to read.

### 5.5 C++20 Concepts

Concepts express constraints directly.

```cpp
template <typename T>
concept Integral = std::is_integral_v<T>;

template <Integral T>
bool is_even(T x) {
    return x % 2 == 0;
}
```

Benefits:

- Clearer constraints.
- Better compiler errors.
- More readable generic APIs.

### 5.6 CRTP Static Polymorphism

```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};
```

CRTP avoids virtual dispatch when polymorphism can be resolved at compile time.

---

## 6. Multithreading and Synchronization

### 6.1 `std::thread`

```cpp
std::thread t([] {
    do_work();
});
t.join();
```

Always join or detach a thread before its destructor runs. Otherwise `std::terminate` is called.

### 6.2 Mutex Family

```cpp
std::mutex mu;
std::lock_guard<std::mutex> lock(mu);
```

Common mutex types:

- `std::mutex`.
- `std::recursive_mutex`.
- `std::shared_mutex`.
- `std::timed_mutex`.

Prefer RAII wrappers:

- `std::lock_guard`.
- `std::unique_lock`.
- `std::scoped_lock`.

### 6.3 Condition Variable

```cpp
std::mutex mu;
std::condition_variable cv;
bool ready = false;

std::unique_lock<std::mutex> lock(mu);
cv.wait(lock, [&] { return ready; });
```

Always wait with a predicate to handle spurious wakeups.

### 6.4 `std::async`, `future`, and `promise`

```cpp
auto fut = std::async(std::launch::async, [] {
    return query();
});

auto result = fut.get();
```

Use `std::async` for simple asynchronous work. For production servers, explicit thread pools usually provide better control.

### 6.5 Thread Pool

Thread pool design concerns:

- Bounded queue.
- Worker lifecycle.
- Shutdown semantics.
- Exception handling.
- Backpressure.
- Metrics.

Unbounded task queues can turn overload into latency and memory growth.

---

## 7. Atomics and Memory Ordering

### 7.1 `std::atomic`

```cpp
std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);
```

Atomics provide operations that are free from data races.

### 7.2 Six Memory Orders ⭐⭐⭐

| Memory Order | Meaning |
|--------------|---------|
| `relaxed` | atomicity only, no ordering |
| `consume` | dependency ordering, rarely used |
| `acquire` | prevents later operations from moving before load |
| `release` | prevents earlier operations from moving after store |
| `acq_rel` | acquire + release for read-modify-write |
| `seq_cst` | strongest global sequential consistency |

Acquire-release example:

```cpp
std::atomic<bool> ready{false};
int data = 0;

void producer() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {}
    std::cout << data << '\n';
}
```

Release publishes data. Acquire observes the publication.

### 7.3 Lock-Free Structures

Lock-free programming usually relies on CAS.

```cpp
std::atomic<int> x{0};
int expected = 0;
x.compare_exchange_strong(expected, 1);
```

Risks:

- ABA problem.
- Memory reclamation.
- Starvation.
- Hard testing and debugging.

Use proven libraries or simpler locks unless lock-free behavior is clearly required.

### 7.4 Compared with Go and Java

| Language | Model |
|----------|-------|
| C++ | explicit atomics and memory order |
| Java | JMM with volatile, synchronized, final field rules |
| Go | happens-before through goroutines, channels, mutexes, atomics |

C++ gives the most control but also the most responsibility.

---

## 8. STL Containers and Algorithms

### 8.1 Sequence Containers

| Container | Characteristics |
|-----------|-----------------|
| `vector` | contiguous memory, fast random access |
| `deque` | segmented storage, efficient push/pop at both ends |
| `list` | doubly linked list, poor cache locality |
| `array` | fixed-size contiguous array |

`vector` growth:

- Usually grows by a multiplicative factor.
- Reallocation invalidates pointers/references/iterators.
- Use `reserve` when size is predictable.

### 8.2 Associative Containers

| Container | Implementation | Complexity |
|-----------|----------------|------------|
| `map` / `set` | balanced tree | O(log n) |
| `unordered_map` / `unordered_set` | hash table | average O(1) |

`unordered_map` is not safe for concurrent read-write access without external synchronization.

### 8.3 Iterators

Iterator categories:

- Input.
- Output.
- Forward.
- Bidirectional.
- Random access.
- Contiguous.

Invalidation rules matter. For example, `vector` reallocation invalidates all iterators.

### 8.4 Algorithms

Prefer STL algorithms over hand-written loops when they express intent clearly.

```cpp
std::sort(values.begin(), values.end());
auto it = std::lower_bound(values.begin(), values.end(), target);
```

### 8.5 Custom Allocator

Allocators are useful for:

- High-frequency small allocations.
- Memory pools.
- Shared memory.
- Specialized performance requirements.

Do not introduce custom allocators unless profiling shows allocation overhead is important.

---

## 9. Performance Optimization

### 9.1 Compiler Options

Common options:

```bash
-O2
-O3
-g
-march=native
-flto
-fsanitize=address
-fsanitize=thread
-fsanitize=undefined
```

Use sanitizers in testing, not in latency-critical production builds.

### 9.2 Common Performance Pitfalls

- Excessive heap allocation.
- Accidental copies.
- Poor cache locality.
- Lock contention.
- False sharing.
- Virtual dispatch in hot loops.
- Branch misprediction.
- `shared_ptr` overuse.
- Large `unordered_map` rehashing.

### 9.3 Profiling Tools

Useful tools:

- `perf`.
- `gprof`.
- `valgrind`.
- `heaptrack`.
- `AddressSanitizer`.
- `ThreadSanitizer`.
- `UndefinedBehaviorSanitizer`.
- `gdb` / `lldb`.

Optimization workflow:

```text
Measure -> identify bottleneck -> form hypothesis -> change -> verify
```

### 9.4 Cache-Friendly Design

Cache-friendly code:

- Uses contiguous memory.
- Keeps hot data compact.
- Separates hot and cold fields.
- Avoids pointer chasing.
- Avoids false sharing.

```cpp
struct alignas(64) Counter {
    std::atomic<long> value{0};
};
```

---

## 10. Compilation, Linking, and Build Systems

### 10.1 Four Compilation Stages

```text
Preprocess -> Compile -> Assemble -> Link
```

Stages:

- Preprocess expands macros and includes headers.
- Compile converts source to assembly.
- Assemble creates object files.
- Link resolves symbols and creates executable/library.

### 10.2 Static and Dynamic Linking

Static linking:

- Includes library code into binary.
- Easier deployment.
- Larger binaries.

Dynamic linking:

- Loads shared libraries at runtime.
- Smaller binaries and shared memory.
- Requires compatible runtime libraries.

### 10.3 ODR

ODR means One Definition Rule.

Violations may cause:

- Link errors.
- Undefined behavior.
- Different behavior across builds.
- Rare production crashes.

Common causes:

- Non-inline function definitions in headers.
- Inconsistent macros across translation units.
- ABI mismatch between libraries.

### 10.4 CMake

Modern CMake prefers target-based configuration.

```cmake
add_library(core core.cpp)
target_include_directories(core PUBLIC include)
target_compile_features(core PUBLIC cxx_std_20)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE core)
```

Avoid global include paths and global compiler flags when possible.

---

## 11. Common Pitfalls and Best Practices

### 11.1 Undefined Behavior Checklist

Common UB:

- Use-after-free.
- Out-of-bounds access.
- Data races.
- Signed integer overflow.
- Dereferencing null pointer.
- Returning reference to local variable.
- Using uninitialized memory.
- Strict aliasing violations.

### 11.2 Modern C++ Best Practices

- Prefer RAII.
- Prefer Rule of Zero.
- Prefer `unique_ptr` over raw owning pointers.
- Prefer `std::vector` over manual arrays.
- Use `string_view` only when lifetime is clear.
- Use `span` for non-owning array views.
- Mark move operations `noexcept` when appropriate.
- Keep ownership explicit.
- Use sanitizers and static analysis in CI.

### 11.3 C++ vs Go vs Rust Resource Management

| Language | Resource Management |
|----------|---------------------|
| C++ | deterministic RAII, manual control |
| Go | garbage collection, defer for cleanup |
| Rust | ownership and borrow checker |

C++ is powerful for systems programming but demands careful lifetime and concurrency design.

---

## 12. Practical Cases

### 12.1 Feature Join Engine for Recommendation

Goal: join user/item/context features with low latency.

Design ideas:

- Store hot features in contiguous arrays where possible.
- Avoid per-request heap allocation.
- Pre-size vectors and maps.
- Use thread-local buffers carefully.
- Batch remote calls.
- Use profiling to locate hot paths.

```cpp
struct Feature {
    uint32_t id;
    float value;
};

using FeatureVector = std::vector<Feature>;
```

### 12.2 High-Performance Object Pool

Object pools reduce allocation overhead for high-frequency objects.

Risks:

- Lifetime confusion.
- Memory retention.
- Thread safety complexity.
- Worse performance if allocation is not the bottleneck.

Use only after profiling.

### 12.3 Lock-Free Ring Buffer

Single-producer single-consumer ring buffer:

```cpp
template <typename T, std::size_t N>
class RingBuffer {
public:
    bool push(const T& v) {
        auto next = (head_ + 1) % N;
        if (next == tail_.load(std::memory_order_acquire)) {
            return false;
        }
        data_[head_] = v;
        head_ = next;
        return true;
    }

private:
    std::array<T, N> data_{};
    std::size_t head_{0};
    std::atomic<std::size_t> tail_{0};
};
```

Real production lock-free structures need rigorous memory-order and reclamation design.

---

## 13. Interview Self-Check

### Quick Questions

### Q1: What is RAII and why is it important?

**Answer:** RAII binds resource lifetime to object lifetime. Resources are acquired in constructors and released in destructors, which gives deterministic cleanup and exception safety.

### Q2: `unique_ptr`, `shared_ptr`, and `weak_ptr`: when do you use each?

**Answer:** Use `unique_ptr` for exclusive ownership, `shared_ptr` for true shared ownership, and `weak_ptr` for non-owning references that can observe a `shared_ptr` without creating cycles.

### Q3: What does `std::move` do?

**Answer:** `std::move` casts an expression to an rvalue reference. It does not move by itself. The actual move happens in a move constructor or move assignment.

### Q4: What is acquire-release ordering?

**Answer:** Release publishes prior writes before a store. Acquire ensures later reads observe data published before the corresponding release. Together they build a synchronization relationship.

### Q5: How are virtual functions implemented?

**Answer:** Compilers typically add a vptr to objects of polymorphic classes. The vptr points to a vtable containing function addresses. A virtual call loads the function pointer and calls through it.

### Q6: How does `vector` grow?

**Answer:** `vector` stores elements contiguously and usually grows by a multiplicative factor. Reallocation moves/copies elements and invalidates iterators, references, and pointers.

### Q7: What is SFINAE?

**Answer:** Substitution Failure Is Not An Error means template substitution failure removes an overload from consideration instead of causing a hard compile error.

### Q8: What is perfect forwarding?

**Answer:** Perfect forwarding preserves the original value category of an argument through a forwarding reference and `std::forward`.

### Q9: Why are Concepts better than SFINAE?

**Answer:** Concepts express constraints directly, improve readability, and produce clearer compiler diagnostics.

### Q10: How is a lambda implemented?

**Answer:** A lambda is compiled into an unnamed function object. Captured values become data members, and `operator()` contains the body.

### Deep-Dive Questions

### Q11: What is the ABA problem?

**Answer:** In CAS-based algorithms, a value may change from A to B and back to A. CAS sees A and succeeds, but the underlying state changed. Solutions include version tags, hazard pointers, epoch reclamation, or different algorithms.

### Q12: Is `shared_ptr` thread-safe?

**Answer:** The reference count operations are thread-safe across different `shared_ptr` instances. The pointed object is not automatically thread-safe, and modifying the same `shared_ptr` object concurrently still needs synchronization.

### Q13: What is the difference between `new/delete` and `malloc/free`?

**Answer:** `new` allocates memory and calls constructors. `delete` calls destructors and frees memory. `malloc/free` only manage raw memory. Do not mix them.

### Q14: Explain Rule of Five and Rule of Zero.

**Answer:** Rule of Five says if a class owns resources and defines one special member, it probably needs destructor, copy/move constructors, and copy/move assignment. Rule of Zero says prefer RAII members so the compiler-generated special members are correct.

### Q15: Why does `noexcept` affect standard containers?

**Answer:** Containers like `vector` may prefer move during reallocation only if move constructors are `noexcept`; otherwise they may copy to preserve strong exception safety.

### Q16: Why is `string_view` risky?

**Answer:** `string_view` does not own data. If it references a temporary or a destroyed string, it becomes dangling.

### Q17: What are exception safety guarantees?

**Answer:** Basic guarantee keeps invariants and avoids leaks. Strong guarantee makes the operation transactional. No-throw guarantee promises the operation will not throw.

### Q18: Why is direct concurrent access to `unordered_map` dangerous?

**Answer:** Concurrent reads and writes can race, and rehashing can invalidate internal structure. Use locks, sharding, copy-on-write, or a concurrent map implementation.

### Q19: What is ODR and how can it fail in production?

**Answer:** ODR requires entities to have exactly one consistent definition across the program. Violations can come from header definitions, macro differences, or ABI mismatches, causing link errors or undefined runtime behavior.

### Q20: How do you analyze a C++ core dump?

**Answer:** Load core with `gdb` or `lldb`, inspect backtrace, threads, registers, variables, and memory. Check whether the final frame is only a symptom. Correlate with logs, recent deploys, sanitizers, and ownership/concurrency risks.

### Open-Ended Design Questions

### D1: How would you optimize an online recommendation service in C++?

**Reference approach:**

- Profile first to identify CPU, memory, lock, or IO bottlenecks.
- Reduce allocations in hot paths.
- Use contiguous layouts for hot features.
- Batch remote calls.
- Avoid unnecessary copies.
- Control tail latency with timeouts, bounded queues, and backpressure.

### D2: How would you review modern C++ production code?

**Reference approach:**

- Check ownership and lifetimes.
- Look for raw owning pointers, dangling references, and unsafe `string_view`.
- Inspect copy/move behavior and `noexcept`.
- Review concurrency for data races and lock ordering.
- Check iterator invalidation and container complexity.
- Ensure sanitizers and tests cover critical paths.
