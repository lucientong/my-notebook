# Java 深入

---

## 📑 目录

### 语言基础
- [Java核心特性](#java核心特性)
  - [面向对象](#面向对象)
  - [集合框架](#集合框架)
  - [异常处理](#异常处理)
  - [equals / hashCode / ==](#equals--hashcode--)
- [泛型与类型擦除](#泛型与类型擦除)
- [String与常量池](#string与常量池)
- [Stream 与 Optional](#stream-与-optional)
- [JPMS 与 SPI](#jpms-与-spi)
- [模式匹配与语言演进](#模式匹配与语言演进)

### 现代 Java（优先掌握）
- [Records 与 Sealed Classes](#records-与-sealed-classes)
- [Java 21+ 虚拟线程（Project Loom）](#java-21-虚拟线程project-loom)

### JVM 运行时
- [JVM深入](#jvm深入)
  - [JVM 内存结构](#jvm-内存结构)
  - [直接内存 / TLAB / 逃逸分析](#直接内存-tlab-逃逸分析)
  - [JIT 分层编译](#jit-分层编译)
  - [引用类型](#引用类型)
  - [SafePoint 与卡表](#safepoint-与卡表)
  - [垃圾回收基础](#垃圾回收基础)
  - [收集器演进与选型](#收集器演进与选型)
  - [类加载机制](#类加载机制)

### 并发编程
- [Java内存模型](#java内存模型)
- [并发编程基础](#并发编程基础)
- [synchronized 锁升级](#synchronized-锁升级)
- [并发核心](#并发核心)
  - [AQS / CLH / Condition](#aqs--clh--condition)
  - [CAS / ABA / LongAdder](#cas--aba--longadder)
  - [ConcurrentHashMap 深入](#concurrenthashmap-深入)
  - [同步工具与队列](#同步工具与队列)
  - [ThreadLocal 与 ForkJoin](#threadlocal-与-forkjoin)

### 框架与工程
- [Spring框架](#spring框架)
- [MyBatis实战](#mybatis实战)
- [性能调优](#性能调优)
- [实战案例](#实战案例)
- [面试题自查](#面试题自查)

---

## Java核心特性

### 面向对象

**1. 三大特性**

**封装（Encapsulation）**：
```java
public class BankAccount {
    private double balance; // 私有属性

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }
}
```

**继承（Inheritance）**：
```java
public class Animal {
    protected String name;

    public void eat() {
        System.out.println(name + " is eating");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println(name + " is barking");
    }
}
```

**多态（Polymorphism）**：
```java
// 编译时多态（方法重载）
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
}

// 运行时多态（方法重写）
Animal animal = new Dog();
animal.eat(); // 若 Dog 重写了 eat，则调用 Dog 版本
```

---

**2. 抽象类 vs 接口**

| 特性 | 抽象类 | 接口 |
|------|-------|------|
| 继承 | 单继承 | 多实现 |
| 方法 | 可有实现 | Java 8+ 可有 default/static；Java 9+ 可有 private |
| 字段 | 可有实例字段 | 只能有常量（`public static final`） |
| 构造器 | 可有 | 不能有 |
| 使用场景 | "is-a" 关系 | "can-do" 能力 |

```java
public abstract class Animal {
    protected String name;
    public Animal(String name) { this.name = name; }
    public abstract void makeSound();
    public void sleep() { System.out.println(name + " is sleeping"); }
}

public interface Flyable {
    void fly();
    default void land() { System.out.println("Landing..."); }
}

public class Bird extends Animal implements Flyable {
    public Bird(String name) { super(name); }
    @Override public void makeSound() { System.out.println("Chirp"); }
    @Override public void fly() { System.out.println(name + " is flying"); }
}
```

---

### 集合框架

**1. 集合体系**

```
Collection
├── List (有序、可重复)
│   ├── ArrayList (动态数组)
│   ├── LinkedList (双向链表)
│   └── Vector (线程安全，已少用)
├── Set (不重复)
│   ├── HashSet / LinkedHashSet
│   └── TreeSet (红黑树排序)
└── Queue / Deque
    ├── PriorityQueue
    └── ArrayDeque

Map
├── HashMap / LinkedHashMap
├── TreeMap
└── ConcurrentHashMap
```

---

**2. ArrayList 扩容**

底层是 `Object[]`（Java 8+ 实际为 `elementData`）。

| 阶段 | 行为 |
|------|------|
| 无参构造 | `elementData = {}`，容量 0（懒分配） |
| 第一次 add | 扩到默认容量 **10** |
| 容量不足 | `newCapacity = oldCapacity + (oldCapacity >> 1)`，即约 **1.5 倍** |
| 超大需求 | 至少扩到 `minCapacity`；超过 `MAX_ARRAY_SIZE` 走巨大数组逻辑 |

```java
List<String> list = new ArrayList<>(); // 容量 0
list.add("A");                         // 扩到 10
// 频繁 insert(0, x) 会整体搬移，O(n)；尾部 add 摊销 O(1)
```

**要点**：
- 随机访问 O(1)；中间插入/删除 O(n)
- 预知大小时用 `new ArrayList<>(expectedSize)`，减少扩容拷贝
- `ensureCapacity` 可手动预扩容；`trimToSize` 回收多余容量

---

**3. ArrayList vs LinkedList**

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头插/头删 | O(n) | O(1) |
| 尾插 | O(1) 摊销 | O(1) |
| 缓存局部性 | 好 | 差 |

工程上绝大多数场景选 **ArrayList**；仅当大量头尾增删且几乎不做随机访问时才考虑 LinkedList。

---

**4. HashMap 完整状态机（Java 8+）**

**结构**：数组 + 链表 + 红黑树（桶内）。

**关键常量**：
- 默认容量 16，负载因子 0.75 → `threshold = capacity * loadFactor`
- 树化：链表长度 ≥ **8** 且数组长度 ≥ **64**
- 退化：树节点 ≤ **6** 转回链表
- 扩容：容量变为 2 倍，节点要么留在原下标，要么落到 `index + oldCap`

**hash 扰动**：
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 下标：hash & (n - 1)  —— n 必须是 2 的幂
```

**put 状态机**：

```
put(key, value)
  │
  ├─ table == null / 长度 0 → resize() 初始化
  ├─ 计算 hash、i = hash & (n-1)
  ├─ 桶空 → newNode，直接挂上
  ├─ 桶头 key 相等 → 覆盖
  ├─ 桶是 TreeNode → putTreeVal
  └─ 链表遍历
        ├─ key 相等 → 覆盖
        ├─ 尾插新节点
        └─ binCount >= 7（第 8 个）→ treeifyBin
              ├─ table.length < 64 → 先 resize（不树化）
              └─ 否则 → 转红黑树
  最后：++size > threshold → resize()
```

**resize 迁移**：
- 旧链表按高位 hash 拆成 lo / hi 两条链，分别落到 `j` 与 `j + oldCap`
- 树节点走 `split`，可能拆成两棵树或退化链表

**自定义 key** 必须同时正确重写 `equals` 与 `hashCode`（见下节）。

**线程安全**：
- Java 7 并发扩容可能死循环；Java 8+ 主要是数据丢失/读到不一致
- 并发场景用 `ConcurrentHashMap`，不要 `Collections.synchronizedMap` 除非竞争极低

---

**5. LinkedHashMap 与 LRU**

`LinkedHashMap` = HashMap + 双向链表，可保持：
- **插入顺序**（默认）
- **访问顺序**（`accessOrder=true`）：get/put 会把节点移到链表尾

**LRU Cache**：
```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LruCache(int maxSize) {
        super(16, 0.75f, true); // accessOrder = true
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize; // 超出容量删最久未访问的
    }
}
```

注意：`LinkedHashMap` 本身非线程安全；高并发缓存更常用 Caffeine / Guava Cache。

---

**6. ConcurrentHashMap 概要**

- Java 7：Segment 分段锁
- Java 8+：**CAS + synchronized(桶头)**，取消 Segment
- 细节见 [ConcurrentHashMap 深入](#concurrenthashmap-深入)

---

### 异常处理

```
Throwable
├── Error（一般不捕获：OOM、StackOverflowError…）
└── Exception
    ├── RuntimeException（unchecked）
    └── 受检异常（checked，必须声明或捕获）
```

```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
} catch (IOException e) {
    throw new UncheckedIOException(e);
}
// AutoCloseable 资源自动关闭
```

原则：不要吞异常；边界转换异常；Error/严重资源问题让上层感知。

---

### equals / hashCode / ==

| 比较 | 含义 |
|------|------|
| `==` | 基本类型比值；引用类型比**是否同一对象** |
| `equals` | 默认同 `==`；业务相等需重写 |
| `hashCode` | 哈希容器分桶；**equals 相等 ⇒ hashCode 必须相等** |

```java
public final class User {
    private final long id;
    private final String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User other)) return false;
        return id == other.id && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}
```

**陷阱**：
- 可变字段参与 hashCode → 放入 HashMap 后修改会丢对象
- 只重写 equals 不重写 hashCode → HashMap/HashSet 行为错乱
- 包装类缓存：`Integer` 在 -128~127 可能 `==` 为 true，业务比较一律 `equals`

---

## 泛型与类型擦除

### 泛型基础

```java
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

public <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}
```

### 类型擦除（Type Erasure）

泛型是**编译期**特性，运行时擦除为原始类型（或上界）：

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass(); // true
```

限制：不能 `List<int>`、不能 `new T[]`、不能 `instanceof List<String>`、不能仅靠泛型参数重载。

### 通配符与 PECS

- `<? extends T>`：只读生产者（Producer Extends）
- `<? super T>`：只写消费者（Consumer Super）

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

## String与常量池

### String 的不可变性

```java
// Java 9+：byte[] + coder（Latin-1 / UTF-16）
public final class String {
    private final byte[] value;
    private final byte coder;
}
```

好处：线程安全共享、hashCode 可缓存、常量池复用、参数防篡改。

### 字符串常量池（堆上）

> **版本要点**：Java 7 起字符串常量池从永久代迁到**堆**；Java 8+ 方法区实现为 Metaspace（本地内存），但**字符串常量池仍在堆中**。

```java
String s1 = "hello";              // 字面量 → 常量池
String s2 = "hello";
System.out.println(s1 == s2);     // true

String s3 = new String("hello");  // 堆上新建 String 对象
System.out.println(s1 == s3);     // false
System.out.println(s1.equals(s3));// true

String s4 = s3.intern();          // 返回池中引用
System.out.println(s1 == s4);     // true
```

`new String("hello")` 对象数：
- 池中已有字面量 → 堆上 **1** 个新 String
- 池中没有 → 字面量入池 + 堆上新建，共 **2** 个（字面量本身也是池中对象）

| 类 | 可变 | 线程安全 | 场景 |
|----|------|---------|------|
| String | 否 | 是 | 少量拼接、常量 |
| StringBuilder | 是 | 否 | 单线程大量拼接 |
| StringBuffer | 是 | synchronized | 多线程拼接（更常用 Builder + 外层同步） |

---

## Stream 与 Optional

### Stream API（Java 8+）

```java
List<String> names = users.stream()
    .filter(u -> u.age() >= 18)
    .map(User::name)
    .distinct()
    .sorted()
    .toList(); // Java 16+；更早用 collect(Collectors.toList())

int sum = IntStream.rangeClosed(1, 100).parallel().sum();
```

原则：
- 中间操作惰性，终端操作触发计算
- 有状态中间操作（`sorted`/`distinct`）成本高
- `parallel()` 适合 CPU 密集 + 无共享可变状态；IO/小数据别盲目并行
- 不要在 stream 里改外部可变集合

### Optional

```java
Optional<User> opt = userRepository.findById(id);
String name = opt.map(User::name).orElse("anonymous");

// 反模式：Optional 作字段/参数；orElse(heavy()) 会无条件执行 heavy
opt.orElseGet(() -> loadDefault());
opt.orElseThrow(() -> new NotFoundException(id));
```

---

## JPMS 与 SPI

### JPMS（Java 9 模块系统）

```
module com.example.app {
    requires java.sql;
    requires com.example.lib;
    exports com.example.app.api;
    opens com.example.app.internal to spring.core; // 反射开放
}
```

要点：强封装、`requires`/`exports`/`opens`、`jlink` 裁剪运行时。许多遗留生态仍以 classpath 为主，但库作者需理解模块边界与反射 `opens`。

### SPI（Service Provider Interface）

```
META-INF/services/com.example.Codec
→ com.example.JsonCodec
```

```java
ServiceLoader.load(Codec.class).forEach(Codec::init);
```

SPI 常配合**线程上下文类加载器（TCCL）**打破双亲委派：Bootstrap/Platform 加载的 API 要加载应用实现类（JDBC、SLF4J 等）。

---

## 模式匹配与语言演进

```java
// instanceof 模式匹配（Java 16+）
if (obj instanceof String s) {
    System.out.println(s.toLowerCase());
}

// switch 模式匹配（Java 21 正式）
static String formatter(Object o) {
    return switch (o) {
        case Integer i -> "int " + i;
        case String s when s.length() > 5 -> "long string";
        case String s -> "string " + s;
        case null -> "null";
        default -> o.toString();
    };
}

// Record 解构
case Point(int x, int y) -> x + y;
```

与 [Sealed Classes](#sealed-classesjdk-17) 组合可做穷举 ADT。

---

## Records 与 Sealed Classes

### Record 类（JDK 16+）

不可变数据载体：自动生成规范构造器、访问器、`equals`/`hashCode`/`toString`。

```java
public record Point(int x, int y) {}

public record Range(int lo, int hi) {
    public Range {
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
    }
}

public record Circle(double radius) {
    public static final Circle UNIT = new Circle(1.0);
    public double area() { return Math.PI * radius * radius; }
}
```

| 维度 | Record | Lombok @Value/@Data |
|------|--------|---------------------|
| 版本 | 16+ 原生 | 注解处理器 |
| 可变性 | 始终不可变 | @Data 可变 |
| getter | `x()` | `getX()` |
| Builder | 需手写/第三方 | `@Builder` |
| 场景 | DTO、值对象 | 复杂可变实体 |

### Sealed Classes（JDK 17+）

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

## Java 21+ 虚拟线程（Project Loom）

### 原理

虚拟线程是 JVM 调度的轻量用户态线程，挂在少量 **Carrier（平台线程）** 上（M:N）。阻塞在 Java 管理的点（多数 socket/IO）时会 **unmount**，释放 Carrier。

```
Virtual Thread × N  ──mount/unmount──→  Carrier Pool（ForkJoinPool，约 = CPU 核数）
```

### 对比

| 维度 | 平台线程 | 虚拟线程 |
|------|---------|---------|
| 映射 | 1:1 OS 线程 | M:N |
| 栈 | 固定约 1MB | 堆上可伸缩 |
| 规模 | 数千 | 可达百万 |
| 场景 | CPU 密集 | IO 密集 |
| 池化 | 需要 | **不要池化**，按任务创建 |

### synchronized 与 Pinning（含 JDK 24）

历史上：虚拟线程在 `synchronized` 临界区内阻塞会 **pin** 住 Carrier，导致吞吐崩塌；建议改用 `ReentrantLock`。

> **JDK 24 / JEP 491（Synchronize Virtual Threads without Pinning）**：大幅消除虚拟线程在 `synchronized` 上的 pinning——在监视器上阻塞时可 unmount，不再长期霸占 Carrier。
> 仍需注意：少数 **native 帧 / JNI** 等场景仍可能 pin；持有 `synchronized` 做超长 CPU 计算仍不合理。升级到 JDK 24+ 后，旧文案「虚拟线程绝对不能用 synchronized」已过时，应改为「优先避免长时间占用；JDK 24 前严肃对待 pinning，JDK 24+ 关注剩余 pin 点与监控」。

监控：`jdk.VirtualThreadPinned` JFR 事件。

### 代码示例

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

### StructuredTaskScope（预览/演进中）

> **状态标注**：结构化并发 API（`StructuredTaskScope`）在 JDK 21–24 一带长期处于 **Preview / 孵化演进**（JEP 453 等），包名与用法可能随版本调整。生产采用前请核对目标 JDK 是否 `--enable-preview` 以及最终 API 形态。

```java
// Preview API 示意（以实际 JDK 文档为准）
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> user = scope.fork(() -> fetchUser(id));
    Subtask<Order> order = scope.fork(() -> fetchOrder(id));
    scope.join();
    scope.throwIfFailed();
    return new Response(user.get(), order.get());
}
```

### 适用与注意

- ✅ 高并发阻塞 IO、每请求一线程风格改写
- ❌ CPU 密集（用平台线程池 / ForkJoin）
- ⚠️ 百万虚拟线程下滥用 `ThreadLocal` 易爆内存 → 考虑 `ScopedValue`
- ⚠️ 不要用固定大小池去「复用」虚拟线程

---

## JVM深入

### JVM 内存结构

```
线程私有：程序计数器 | 虚拟机栈 | 本地方法栈
线程共享：堆（Eden / Survivor / Old）| 方法区（Java 8+ → Metaspace，本地内存）
堆外：直接内存（Direct Buffer）、JNI、Metaspace 等
```

| 区域 | 内容 | OOM 典型 |
|------|------|---------|
| 堆 | 对象、数组、字符串常量池（Java 7+） | `Java heap space` |
| Metaspace | 类元数据 | `Metaspace` |
| 直接内存 | NIO DirectByteBuffer 等 | `Direct buffer memory` |
| 栈 | 栈帧 | `StackOverflowError` / 视配置也可能 OOM |

### 直接内存 / TLAB / 逃逸分析

**直接内存**：
- `ByteBuffer.allocateDirect`，减少一次用户态拷贝，但分配/释放成本高，受 `-XX:MaxDirectMemorySize` 约束
- 泄漏时堆不一定满，需看 NMT / DirectBuffer 统计

**TLAB（Thread Local Allocation Buffer）**：
- 每个线程在 Eden 预留小块缓冲，对象先在 TLAB 无锁分配
- TLAB 满再慢路径（可能 CAS 或触发 GC）
- 大对象可能绕过 TLAB

**逃逸分析与标量替换**（JIT）：
- 方法逃逸 / 线程逃逸分析后：栈上分配（理想）、锁消除、**标量替换**（对象拆成寄存器/栈上标量）
- 调试：`-XX:+DoEscapeAnalysis`（默认开）、`-XX:-EliminateAllocations` 对比

### JIT 分层编译

```
解释执行 → C1（分层编译 Client）→ C2 / Graal（Server 深度优化）
         ↑________ 探测（MDO）与退化（逆优化）________↓
```

| 层级（示意） | 角色 |
|-------------|------|
| 0 | 解释 |
| 1–3 | C1 ± 性能计数 |
| 4 | C2 高优化 |

热点方法/循环被编译；激进假设失败则 **deoptimize** 回解释或低层。工具：`-XX:+PrintCompilation`、JFR `Compiler` 事件、`jitwatch`。

### 引用类型

| 类型 | GC 时机 | 典型用途 |
|------|--------|---------|
| 强引用 | 不回收 | 普通对象 |
| 软引用 SoftReference | 内存不足偏回收 | 内存敏感缓存 |
| 弱引用 WeakReference | 下次 GC 可回收 | WeakHashMap、ThreadLocal key |
| 虚引用 PhantomReference | 死亡后入队 | 堆外资源清理 |

### SafePoint 与卡表

**SafePoint**：线程走到可安全暂停的点（方法调用/循环回边等），GC、偏向撤销、代码逆优化等依赖它。进入 STW 前要 **到齐 SafePoint**——长时间不计次循环可能导致 Time To SafePoint 长尾。

**卡表（Card Table）**：
- 老年代 → 年轻代引用的粗粒度记录，配合写屏障更新
- Young GC 扫描脏卡而非整堆老年代
- G1 还有 Remembered Set；维护有写屏障成本

---

### 垃圾回收基础

**可达性**：从 GC Roots 做可达性分析（非引用计数）。Roots：栈引用、静态/常量、JNI、锁持有对象等。

**算法**：标记-清除（碎片）、标记-复制（年轻代）、标记-整理（老年代）。

**分代与 GC 类型（勿混淆）**：

| 名称 | 含义 |
|------|------|
| **Minor GC / Young GC** | 回收年轻代 |
| **Major GC** | 通常指回收**老年代**（不同资料口径不一） |
| **Full GC** | 回收**整个堆**（含年轻代+老年代，常含 Metaspace），停顿往往最长 |

> **硬伤修正**：`Major GC ≠ Full GC`。HotSpot 日志里更常见 Young / Mixed / Full；口语把「老年代 GC」叫 Major 时，也不等于 Full。分析以 **GC 日志实际标签**为准。

**对象晋升**：Eden → Survivor（年龄 +1）→ 年龄阈值（默认 15，动态年龄）进老年代；大对象可直接进老年代。

---

### 收集器演进与选型

本节合并原「GC 深度剖析」与「ZGC vs Shenandoah」重复叙事，统一口径。

#### 1. CMS（历史；JDK 14 移除）

> **JDK 9 废弃，JDK 14 删除**（JEP 363）。新项目不要再选 CMS；面试作历史对比即可。

目标：降低老年代 STW。四阶段：

```
初始标记 STW → 并发标记 → 重新标记 STW → 并发清除
```

三大问题：
1. **碎片**（标记-清除不整理）→ 连续分配失败触发 Full GC 整理
2. **Concurrent Mode Failure**：并发清理期间老年代装不下晋升 → 降级 Serial Old，停顿暴涨
3. **浮动垃圾**：并发阶段新产生的垃圾留到下次

#### 2. Parallel（吞吐量）

多线程 STW，吞吐优先，适合批处理 / ETL。

#### 3. G1（默认主流）

堆拆成等大 **Region**（Eden/Survivor/Old/Humongous 角色可动态变）。

流程要点：
- **Young GC（STW）**：回收年轻代 Region，复制存活对象
- **并发标记**：老年代占用到 IHOP（`-XX:InitiatingHeapOccupancyPercent`）后启动
- **Mixed GC**：回收年轻代 + 若干垃圾多的老年代 Region，用历史耗时预测凑近 `-XX:MaxGCPauseMillis`
- 算法以**标记-复制**为主 → 解决 CMS 碎片

代价：Remembered Set / 卡表写屏障；元数据占堆；小堆优势不明显（经验上更大堆更划算）。

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1ReservePercent=10
```

#### 4. ZGC（与下文 Shenandoah 一并对比，不重复开章）

**目标停顿（统一口径）**：以**亚毫秒级**为主（常见宣传 **< 1ms**），且不随堆增大线性恶化。早期材料写「< 10ms」是更保守的历史口径——正文与题库统一用「亚毫秒 / <1ms 量级」，不要一处 10ms、一处 1ms 互相打架。

**核心技术**：
1. **染色指针**：把标记/重映射元数据放进指针高位，少碰对象头
2. **读屏障**：加载引用时发现需重定位则查转发表并**自愈**写回

并发阶段覆盖标记与转移；STW 仅极短的根处理等步骤。

**版本与压缩指针（消矛盾）**：

| 版本 | 状态 |
|------|------|
| JDK 11–14 | 实验 |
| JDK 15+ | 生产可用 |
| JDK 21+ | **分代 ZGC**（JEP 439；后续版本逐步默认分代） |

统一表述：**早期染色方案与 CompressedOops 不兼容**；JDK 15+ 可生产使用；分代自 21；压缩指针/多映射能力随版本增强，**以目标 JDK 发行说明为准**，禁止同时写「永不支持」和「仅 JDK 21 前不支持」两套矛盾结论。

#### 5. Shenandoah

**屏障模型（纠正）**：现代 Shenandoah 以 **LRB（Load Reference Barriers，读屏障）** 做并发转移后的引用修正。早期教材「Brooks 指针 + 写屏障」叙事已过时——面试主答 **LRB**；写屏障若存在也不是与 ZGC 对照时的主标签。

**分代**：有**实验性分代 Shenandoah**，开关与成熟度看发行版，勿写「绝对无分代」。

| 维度 | ZGC | Shenandoah |
|------|-----|------------|
| 核心 | 染色指针 + 读屏障 | LRB（并发压缩） |
| 停顿 | 亚毫秒级 | 低毫秒级（同属低延迟） |
| 分代 | JDK 21+ 正式路线 | 实验性分代 |
| 生态 | Oracle/OpenJDK 主力选项之一 | 视发行版是否内置 |

#### 6. 选型决策

| 场景 | 建议 |
|------|------|
| 通用服务 / 容器 | G1 |
| 超低延迟、大堆 | ZGC（21+ 优先分代） |
| 批处理吞吐 | Parallel |
| 小工具 / 单核 | Serial |
| 遗留 CMS 参数 | 升级后迁 G1/ZGC |

```bash
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-XX:+UseZGC          # 21+ 关注是否默认分代 / -XX:+ZGenerational
-XX:+UseParallelGC
```

**Full GC 排查口令**（与 Major 区分）：看日志标签 `Pause Full` / `Full GC` 的原因字符串（Allocation Failure、Metadata、System.gc 等），再用 jstat/histo/JFR，而不是口头「又一次 Major」。

---

### 类加载机制

**过程**：加载 → 验证 → 准备 → 解析 → 初始化（`<clinit>`）。

**类加载器（Java 9+）**：

```
Bootstrap ClassLoader          → 加载 java.base 等（原生，曾对应 rt.jar）
        ↓
Platform ClassLoader           ← 取代旧 Extension ClassLoader（ext.dirs）
        ↓
Application / System ClassLoader
        ↓
自定义 ClassLoader
```

> Java 8 及以前：Bootstrap（`rt.jar`）→ Extension（`lib/ext`）→ Application。
> **Java 9+**：模块化后无 `rt.jar` / Extension，改为 **Bootstrap + Platform + Application**。

**双亲委派**：先父后子，保证核心类唯一性。打破场景：SPI（TCCL）、热部署、应用服务器隔离、部分 OSGi/模块系统。

---

## Java内存模型

### JMM（抽象模型）

JMM 规定多线程下共享变量的**可见性、有序性、原子性**语义，以及何时可视为同步到「主内存」。

> **硬伤修正**：所谓「工作内存」是 **JMM 抽象**，**不等于**把 CPU Cache/寄存器实体一一映射。真实硬件有多级缓存、写缓冲、乱序执行；JMM 用抽象规则（及 happens-before）约束编译器/处理器可见结果，答题勿说「工作内存就是 CPU 缓存」。

### happens-before

程序顺序、volatile 写→读、unlock→后续 lock、线程 start/终止、interrupt、传递性等。

```java
volatile boolean flag = false;
int x = 0;
// T1: x=42; flag=true;
// T2: if (flag) print(x);  // 必为 42
```

---

## 并发编程基础

### 线程基础

继承 `Thread` / 实现 `Runnable` / `Callable`+线程池。生产环境任务应提交到**有界队列的 `ThreadPoolExecutor`**，而不是无脑 `new Thread`。

### synchronized

实例方法锁 `this`，静态方法锁 `Class` 对象；也可锁私有 final 对象减小粒度。

### volatile

可见性 + 禁止重排；不保证复合操作原子性。DCL 单例需要 `volatile` 防指令重排。

### Lock

`ReentrantLock`：可中断、超时、公平、多 Condition。
`ReadWriteLock` / `StampedLock`：读多写少场景。

### 线程池

```java
new ThreadPoolExecutor(
    core, max, keepAlive, unit,
    new LinkedBlockingQueue<>(capacity), // 必须有界
    threadFactory,
    handler // Abort / CallerRuns / Discard / DiscardOldest
);
```

流程：`< core` 建线程 → 入队 → `< max` 建非核心线程 → 拒绝策略。

> **与题库一致的工程结论**：生产避免 `Executors.newFixedThreadPool`（无界队列）与 `newCachedThreadPool`（线程无上界）。可用 `Executors` 作**演示**，正文推荐 **`ThreadPoolExecutor` 显式参数**；虚拟线程用 `newVirtualThreadPerTaskExecutor()`。

### CompletableFuture

`supplyAsync` / `thenApply` / `thenCompose` / `allOf` / `exceptionally`，指定自定义 Executor，避免打满公共 `ForkJoinPool.commonPool()`。

---

## synchronized 锁升级

> 本节从原 JMM 章挪入并发体系，与锁实现放在一起（目录锚点亦挂在并发下）。

Java 6+ 根据竞争逐步膨胀：`无锁 → 偏向锁 → 轻量级锁 → 重量级锁`（Mark Word 标志位变化）。

```
无竞争单线程反复进入 → 偏向（Mark Word 写线程 ID，再入几乎只比较）
线程交替、短临界区   → 轻量级（栈上 Lock Record + CAS；失败自适应自旋）
持续碰撞            → 重量级（ObjectMonitor，park/unpark，入队等待）
```

| 锁 | 场景 | 机制 | 开销 |
|----|------|------|------|
| 偏向 | 单线程反复 | 线程 ID 在对象头 | 极低 |
| 轻量级 | 交替、短竞争 | CAS + 自旋 | 较低 |
| 重量级 | 高竞争 | OS 级等待 | 高（但避免空转） |

**偏向锁现状**：Java 15 起默认禁用（`-XX:-UseBiasedLocking`），撤销/epoch 成本在多线程服务里不划算。

> **「只能升级不能降级」要限定**：竞争路径上不会出现「重量级→轻量级→偏向」的阶梯降级。但 JVM 可对空闲 `ObjectMonitor` 做 **monitor deflation**，对象头回到无监视器形态，之后重新竞争再膨胀。答题金句：**升级路径单向 + 存在 monitor deflation，勿说死「锁永不降级」**。

---

## 并发核心

### AQS / CLH / Condition

**AQS（AbstractQueuedSynchronizer）**：`ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock` 等的底座。

```
state (volatile int)
   +
CLH 变体双向队列：head / tail，节点含 waitStatus、prev/next、thread
```

| 模式 | 模板方法 | 典型实现 |
|------|---------|---------|
| 独占 | `tryAcquire` / `tryRelease` | ReentrantLock（state=重入次数） |
| 共享 | `tryAcquireShared` / `tryReleaseShared` | Semaphore、CountDownLatch、读写锁读侧 |

**获取失败路径（示意）**：
1. `tryAcquire` 失败 → `addWaiter` 入队
2. `acquireQueued`：若前驱是 head 再试一次；否则 `shouldParkAfterFailedAcquire` 把前驱标为 SIGNAL，再 `LockSupport.park`
3. 被中断可记录/重抛（`lockInterruptibly`）

**释放**：`tryRelease` 成功后 `unparkSuccessor(head)` 唤醒后继。

**CLH**：经典 CLH 在本地自旋看前驱；AQS 改为**可阻塞**的变体，由前驱负责 SIGNAL + unpark，减少无效自旋。

**ConditionObject**：
- 独立条件队列；`await`：释放锁 → 入条件队列 → park
- `signal`：条件节点转移到同步队列，再竞争锁
- 可多条件（有界队列的 notFull/notEmpty）

```java
lock.lock();
try {
    while (!ready) condition.await(); // 必须 while 防虚假唤醒
    // ...
    condition.signal();
} finally {
    lock.unlock();
}
```

### CAS / ABA / LongAdder

**CAS**：`Unsafe` / `VarHandle` / `Atomic*` 的 compareAndSet；失败即重试（乐观）。

**ABA**：A→B→A 使 CAS 成功但语义已变（链表出队再入队同一节点）。缓解：
- `AtomicStampedReference`（版本戳）
- `AtomicMarkableReference`
- 业务不依赖「中间未变」或改用锁

**LongAdder / LongAccumulator**：
- `base` + 稀疏 `Cell[]`，写冲突时扩 Cell，热点计数分散
- `sum()` 非强一致快照，监控/统计足够；需要精确瞬时值仍用 `AtomicLong`

### ConcurrentHashMap 深入

**Java 7**：Segment 分段锁。
**Java 8+**：取消 Segment → **数组 + CAS + synchronized(桶头)**。

**`sizeCtl` 语义（面试高频）**：

| 值 | 含义 |
|----|------|
| 负数 | 初始化或扩容中（高位类似「签名」，低位与参与线程数相关） |
| 0 | 默认，尚未指定 |
| 正数 | 下次扩容阈值或初始容量相关控制 |

**put 路径**：
1. `table` 空 → `initTable`（CAS 抢 `sizeCtl`）
2. 桶空 → `casTabAt` 插入
3. 桶为 `ForwardingNode` → **helpTransfer** 协助扩容
4. 否则 `synchronized (首节点)` 链表/树插入；链表长且容量≥64 → 树化
5. `addCount`：累加计数，达阈值触发 `transfer`

**协助扩容**：扩容线程把旧桶置为 ForwardingNode，其他写线程撞见后帮忙迁移区间，而不是阻塞等到结束。

**计数**：`baseCount` + `CounterCell`（同 LongAdder 思想）；`size()` 为估算求和。
**迭代**：弱一致，不抛 CME；允许 null？**不允许** key/value 为 null（与 HashMap 不同，避免歧义）。

### 同步工具与队列

| 工具 | 语义 |
|------|------|
| CountDownLatch | 一次性倒数，等待方阻塞至 0 |
| Semaphore | 许可数，限流/资源池 |
| CyclicBarrier | 可循环栅栏，齐点后可跑 barrier action |
| Phaser | 更灵活的多阶段同步 |

**BlockingQueue**：
- `ArrayBlockingQueue` 有界数组
- `LinkedBlockingQueue` 可选有界（默认 `Integer.MAX_VALUE`，危险）
- `SynchronousQueue` 不存储，直接移交
- `PriorityBlockingQueue` / `DelayQueue`

线程池队列选型直接决定背压与 OOM 风险。

### ThreadLocal 与 ForkJoin

**ThreadLocal**：每线程一份值；实现是 `Thread.threadLocals` → `ThreadLocalMap`（key 弱引用）。线程池复用线程必须 `remove()`，否则泄漏。虚拟线程场景慎用，优先 `ScopedValue`（预览/正式视版本）。

**ForkJoinPool**：工作窃取；`RecursiveTask`/`RecursiveAction`；`CompletableFuture` 默认执行器、并行流共用 `commonPool`。CPU 密集可自建 ForkJoinPool；IO 阻塞任务不要塞满 commonPool。

---

## Spring框架

### IoC 与依赖注入

控制反转：对象创建与依赖交给容器。注入优先**构造器注入**（不可变、易测、环依赖早暴露）。

```java
@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
}
```

### Bean 生命周期与 BPP

```
扫描/解析 → BeanDefinition 注册
  → BeanFactoryPostProcessor（改定义，如 PropertySourcesPlaceholder）
  → 实例化（构造器 / 工厂方法）
  → 填充属性（@Autowired 等）
  → Aware：BeanNameAware / BeanFactoryAware / ApplicationContextAware…
  → BeanPostProcessor#postProcessBeforeInitialization
  → @PostConstruct → InitializingBean → init-method
  → BeanPostProcessor#postProcessAfterInitialization  ← AOP/事务代理常在此包装
  → 就绪使用
  → @PreDestroy → DisposableBean → destroy-method
```

`BeanPostProcessor`（BPP）是扩展核心：日志、校验、**生成代理**都挂这里。注意 BFPP 改的是**定义**，BPP 改的是**实例**。

### 三级缓存与循环依赖

| 级 | 结构 | 内容 |
|----|------|------|
| 一 | `singletonObjects` | 成品 |
| 二 | `earlySingletonObjects` | 早期暴露（可能已是代理） |
| 三 | `singletonFactories` | `ObjectFactory`，需要时才创建早期引用 |

**流程（A→B→A，Setter）**：
1. 建 A，把 A 的 ObjectFactory 放三级缓存
2. 填充 A 时要 B → 建 B，B 的工厂进三级
3. 填充 B 时要 A → 调三级工厂得到 A 的早期引用（若需增强则此处创建代理）→ 放入二级
4. B 完成进一级；A 拿完整 B，A 完成进一级

**构造器环**：实例化都完不成 → 直接失败。应用侧应拆环或 `@Lazy` 推迟。

为何三级：若只有二级、过早生成代理，可能在错误时机增强；三级延迟到「真有人需要早期引用」再经 `ObjectFactory` 决定是否代理。

### Boot 自动配置

`@SpringBootApplication` → `@EnableAutoConfiguration` → `AutoConfigurationImportSelector`：
- Boot 2.x：`META-INF/spring.factories` 的 `EnableAutoConfiguration`
- Boot 3.x：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

常用条件：`@ConditionalOnClass` / `OnMissingBean` / `OnProperty` / `OnWebApplication`。
用户手写 `@Bean` + `OnMissingBean` → **约定优于配置**的覆盖点。排除：`@SpringBootApplication(exclude=…)` 或 `spring.autoconfigure.exclude`。

### AOP 代理

| 方式 | 机制 | 条件 |
|------|------|------|
| JDK 动态代理 | `InvocationHandler` | 有接口 |
| CGLIB | 子类拦截 | 类非 final；Boot 2+ 常默认目标类代理 |

自调用 `this.x()` **不经代理** → 切面与 `@Transactional` 失效；拆 Bean、注入自身、`AopContext`（慎用）可解。

### 事务与失效场景

| 传播 | 含义 |
|------|------|
| REQUIRED | 有则加入，无则新建（默认） |
| REQUIRES_NEW | 挂起外层，新开事务 |
| NESTED | 嵌套保存点（非所有库/驱动都完美） |
| SUPPORTS / NOT_SUPPORTED / MANDATORY / NEVER | 见名知意 |

**`@Transactional` 失效**：
1. 同类自调用
2. 非 public（Spring 接口代理场景）
3. 吞异常 / 抛受检异常未 `rollbackFor`
4. 非 Spring 管理的实例（`new`）
5. 异步线程无事务上下文
6. 引擎不支持事务（如误用 MyISAM）

---

## MyBatis实战

### Mapper 与动态 SQL

注解或 XML；`<where>`/`<if>`/`<foreach>` 做动态 SQL；`resultMap` 处理关联。

### 一级 / 二级缓存

**一级缓存（本地会话缓存）**：
- 默认作用域 `SESSION`（同一 `SqlSession`）
- **可以关闭或缩小**：`localCacheScope=STATEMENT`（每语句清缓存），或在查询上 `flushCache="true"`；并非「绝对无法关闭」
- Spring 集成下通常每个事务/模板方法一个 SqlSession，一级缓存收益有限

**二级缓存**：namespace 级，跨会话；多表脏读、分布式一致性差，生产多用业务缓存（Redis/Caffeine）替代。

```yaml
mybatis:
  configuration:
    local-cache-scope: STATEMENT   # 削弱一级缓存
    cache-enabled: false           # 关二级缓存
```

---

## 性能调优

### JVM 参数与容器

```bash
-Xms4g -Xmx4g
-XX:+UseG1GC   # 或 ZGC
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# 容器感知（务必）
-XX:+UseContainerSupport          # JDK 10+ 默认倾向开启
-XX:MaxRAMPercentage=75.0         # 相对容器内存上限，优于盲写 -Xmx
-XX:InitialRAMPercentage=50.0
```

### 诊断工具

| 工具 | 用途 |
|------|------|
| jcmd / jstat / jstack | 运行诊断、GC、线程 |
| jmap | 堆史/直方图；谨慎线上 dump |
| **MAT / VisualVM** | 堆分析 |
| **JFR** | 低开销事件剖析（GC、锁、IO、VirtualThreadPinned） |
| **async-profiler** | CPU/alloc/lock 火焰图，生产利器 |

> **`jhat` 已废弃/移除**，勿再推荐。堆分析用 MAT、VisualVM、或 `jcmd GC.heap_dump` + MAT。

```bash
jcmd <pid> JFR.start name=app settings=profile duration=60s filename=app.jfr
# async-profiler 示例
./asprof -e cpu -d 30 -f cpu.html <pid>
```

### 数据库部分（收敛）

索引、慢 SQL、执行计划、连接池等详见：
- [`02-MySQL深入.md`](./02-MySQL深入.md)
- [`10-数据库综合.md`](./10-数据库综合.md)

此处仅保留与 Java 服务衔接的要点：HikariCP 池大小、超时、`maxLifetime`；避免在事务内远程调用放大连接占用。

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.max-lifetime=1800000
```

---

## 实战案例

### 金融支付：分布式锁

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
            throw new RuntimeException("并发冲突");
        }
        try {
            // 扣款业务...
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

### 游戏后台：有界线程池

```java
@Bean("taskExecutor")
public ThreadPoolExecutor taskExecutor() {
    return new ThreadPoolExecutor(
        4, 8, 60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy());
}
```

---

## 面试题自查

编号规则：`Q1` 起连续编号；初级补全答案；每题附「追问」。

### 初级

#### Q1：`==` 和 `equals` 的区别？

**答**：`==` 对基本类型比数值，对引用比是否同一对象；`equals` 默认同 `==`，可重写为值相等。实体作 HashMap key 时必须成对重写 `equals`/`hashCode`。

**追问**：`Integer a=128; Integer b=128; a==b` 为什么可能是 false？缓存范围如何影响？

---

#### Q2：`final` / `finally` / `finalize`？

**答**：`final` 修饰不可变/不可继承/不可覆盖；`finally` 保证退出 try 前执行（用于释放）；`finalize` 曾是 GC 前回调，**已弃用**，资源清理用 try-with-resources。

**追问**：`final` 引用不能变，引用指向的对象内容能变吗？

---

#### Q3：ArrayList 和 LinkedList 区别？

**答**：ArrayList 动态数组，随机访问 O(1)，中间改动 O(n)，扩容约 1.5 倍；LinkedList 双向链表，头尾 O(1)，随机访问 O(n)，缓存不友好。常用 ArrayList。

**追问**：为何 ArrayList 扩容不是 2 倍？预知 size 为何还要指定初始容量？

---

#### Q4：HashMap 和 ConcurrentHashMap 区别？

**答**：HashMap 非线程安全；CHM 用 CAS+桶头 synchronized（Java 8+），弱一致迭代，分段统计 size。不能用 HashMap 碰并发写。

**追问**：CHM 的 `sizeCtl` 在扩容时如何编码？工作线程如何协助扩容？

---

#### Q5：checked 与 unchecked 异常？

**答**：受检异常（Exception 非 Runtime）必须声明/捕获；RuntimeException/Error 不受检。API 设计上业务可恢复错误未必都做成受检。

**追问**：为什么现代框架更倾向未受检异常 + 统一错误处理？

---

### 中级 / 原理

#### Q6：泛型类型擦除是什么？限制有哪些？

**答**：编译期检查后擦除为原始类型，调用处插 cast。限制：无基本类型参数、无泛型数组、`instanceof` 不能带具体参数、不能靠泛型重载。

**追问**：桥接方法（bridge method）解决什么问题？

---

#### Q7：PECS 是什么？

**答**：Producer Extends，Consumer Super。`extends` 适合读，`super` 适合写。

**追问**：`Collections.copy` 的两个参数各用什么通配符？

---

#### Q8：String 为何不可变？`new String("hello")` 几个对象？常量池在哪？

**答**：线程安全、hash 缓存、池化、安全。对象数 1 或 2 取决于池中是否已有字面量。**Java 7+ 字符串常量池在堆上**。

**追问**：`intern` 在大量拼接场景的风险是什么？

---

#### Q9：JVM 内存区域？哪些线程私有？

**答**：私有：PC、VM 栈、本地方法栈；共享：堆、方法区/Metaspace；另有直接内存。

**追问**：Metaspace 和永久代差异？字符串常量池为何迁堆？

---

#### Q10：Minor / Major / Full GC 区别？对象如何晋升？

**答**：Minor/Young 收年轻代；Major 常指老年代回收（口径混乱）；**Full 收整堆（常含元空间）**，**Major ≠ Full**。晋升：Eden→Survivor 计年龄→阈值进老年代；大对象可直进老年代。

**追问**：动态年龄判定如何让对象提前晋升？

---

#### Q11：CMS 与 G1？CMS 现状？

**答**：CMS 并发标记清除，碎片与 CMF；**JDK 14 已移除**。G1 Region + 可预测停顿 + Mixed GC，复制减少碎片。

**追问**：G1 Mixed GC 触发与 IHOP 关系？

---

#### Q12：ZGC 核心技术？停顿为何能到亚毫秒？压缩指针与版本？

**答**：染色指针 + 读屏障自愈，转移并发做，STW 极短且与堆大小基本脱钩；口径统一为**亚毫秒级（~<1ms）**。JDK 15+ 生产；**21+ 分代 ZGC**。早期不兼容 CompressedOops，后随版本演进，以发行说明为准。

**追问**：分代 ZGC 解决了非分代 ZGC 的什么痛点？

---

#### Q13：Shenandoah 用什么屏障？有分代吗？

**答**：现代实现以 **LRB（Load Reference Barrier）** 为主做并发转移引用修正；存在**实验性分代**，勿答「只有写屏障、绝无分代」。

**追问**：ZGC 与 Shenandoah 选型时如何看发行版支持？

---

#### Q14：JMM 是什么？工作内存是 CPU 缓存吗？

**答**：JMM 是并发语义抽象；happens-before 约束可见性与顺序。**工作内存不是 CPU 缓存的实体同义词**。

**追问**：as-if-serial 与 happens-before 对单线程/多线程各意味着什么？

---

#### Q15：volatile 作用？DCL 为何需要？

**答**：可见性 + 禁重排；不保证 `i++` 原子。DCL 中 `new` 重排会导致发布未构造完对象。

**追问**：volatile 能否替换锁做复合操作？

---

#### Q16：synchronized 锁升级？能否降级？

**答**：无锁→偏向→轻量→重量。**升级路径单向**；但存在 **monitor deflation** 回收空闲监视器。Java 15+ 默认关闭偏向锁。

**追问**：轻量级锁自旋失败后一定膨胀吗？自适应自旋看什么？

---

#### Q17：AQS 原理？独占与共享差异？

**答**：`state` + CLH 变体队列 + park/unpark。独占每次仅一线程；共享允许累加许可（Semaphore/Latch）。Condition 挂条件队列。

**追问**：`tryAcquire` 失败到入队之间如何保证公平性？

---

#### Q18：ConcurrentHashMap put/扩容？树化？

**答**：CAS 建桶，冲突锁桶头；`sizeCtl` 管理 init/resize；遇到 ForwardingNode **协助扩容**；链表过长树化。

**追问**：为何 CHM 的 key/value 不能为 null（对比 HashMap）？

---

#### Q19：类加载器层次与双亲委派？Java 9 变化？

**答**：Java 9+ 为 Bootstrap → **Platform**（取代 Extension）→ Application；无 `rt.jar`。双亲委派保核心类唯一。SPI/热部署等可打破。

**追问**：Tomcat 如何用类加载器隔离 WebApp？

---

#### Q20：线程池参数与执行流程？为何慎用 Executors 工厂？

**答**：七大参数与「先线程后队列再扩再拒绝」。`newFixedThreadPool` 无界队列、`newCachedThreadPool` 无线程上限，易 OOM；生产显式 `ThreadPoolExecutor` + 有界队列。

**追问**：IO 密集如何估算 coreSize？CallerRuns 如何形成背压？

---

#### Q21：ReentrantLock vs synchronized？

**答**：Lock 可中断、超时、公平、多 Condition；synchronized 语法简洁、自动释放、JVM 优化成熟。虚拟线程在 JDK 24（JEP 491）前对 synchronized pinning 更敏感。

**追问**：公平锁吞吐为何通常更差？

---

#### Q22：CountDownLatch / Semaphore / CyclicBarrier？

**答**：Latch 一次性倒数；Semaphore 许可；Barrier 可重置齐点。

**追问**：能否用 CHM + CAS 手写简易 Latch？缺点？

---

#### Q23：ThreadLocal 泄漏？

**答**：key 弱、value 强；线程池线程不 `remove` 会漏。finally 清理；虚拟线程偏好 ScopedValue。

**追问**：InheritableThreadLocal 与虚拟线程/线程池的坑？

---

#### Q24：ForkJoin 与工作窃取？

**答**：任务拆分 + 窃取空闲线程队列尾部；适合 CPU 密集分治。阻塞 IO 勿打爆 commonPool。

**追问**：并行流与自定义 ForkJoinPool 如何隔离？

---

#### Q25：JIT 分层编译？逃逸分析收益？

**答**：解释→C1→C2；热点编译，可逆优化。逃逸分析支撑锁消除、标量替换、减少堆分配。

**追问**：如何用 JFR/日志确认方法已 C2 编译？

---

#### Q26：虚拟线程原理？何时 pin？JDK 24 变化？

**答**：M:N 挂 Carrier，阻塞可 unmount。旧版 `synchronized` 内阻塞会 pin；**JEP 491（JDK 24）大幅消除 synchronized pinning**。仍监控 `VirtualThreadPinned`；勿池化虚拟线程。

**追问**：为何 CPU 密集不宜百万虚拟线程？

---

#### Q27：Record / Sealed 解决什么？

**答**：Record 不可变数据载体；Sealed 封闭继承树，配合 switch 穷举。

**追问**：Record 能继承别的类吗？能作 JPA 实体吗？

---

#### Q28：Spring Bean 生命周期？BPP 作用？

**答**：定义→实例化→注入→Aware→BPP 前后→初始化回调→代理常在 after→销毁回调。BPP 可包装 Bean。

**追问**：`BeanFactoryPostProcessor` 与 `BeanPostProcessor` 谁更早？

---

#### Q29：三级缓存如何解循环依赖？

**答**：三级 ObjectFactory 暴露早期引用（可含代理）；构造器环不可解。

**追问**：为何需要三级而不是只有二级？

---

#### Q30：`@Transactional` 失效场景？

**答**：自调用、非 public、吞异常、异常类型不匹配、非代理对象、跨线程等。

**追问**：`REQUIRES_NEW` 内层成功外层回滚，数据如何？

---

#### Q31：Spring Boot 自动配置原理？

**答**：`@EnableAutoConfiguration` 导入选择器，读 factories/imports，条件注解过滤，`OnMissingBean` 让用户覆盖。

**追问**：如何排除某条自动配置？

---

#### Q32：MyBatis 一/二级缓存？一级能否关？

**答**：一级会话级，可用 `localCacheScope=STATEMENT` 等削弱/关闭行为；二级 namespace 级，生产常关。非「一级绝对无法关闭」。

**追问**：Spring 下为何感觉一级缓存「没有」？

---

#### Q33：如何排查 Full GC？工具栈？

**答**：GC 日志 → jstat → histo/dump → MAT；结合 JFR、async-profiler 看分配热点。不用 jhat。

**追问**：容器里只设 `-Xmx` 忽略 `MaxRAMPercentage` 会怎样？

---

#### Q34：直接内存 / TLAB 是什么？

**答**：直接内存堆外，NIO 常用；TLAB 线程本地 Eden 缓冲加速分配。

**追问**：DirectByteBuffer 谁负责回收？和虚引用关系？

---

#### Q35：SPI 与双亲委派如何协作？

**答**：接口可能由 Platform/Bootstrap 加载，实现由 App 加载，经 TCCL/`ServiceLoader` 加载实现，形成受控「打破」。

**追问**：JPMS 下 `provides ... with ...` 与经典 SPI 文件关系？

---

### 开放设计题

#### D1：设计 10 万 QPS Java 网关，要点？

Netty Reactor、堆外与池化、G1/ZGC、Filter 链、限流熔断、超时预算、Micrometer/追踪、背压。

**追问**：如何防止慢路由拖垮 EventLoop？

---

#### D2：Full GC 每小时一次、停顿 2s，如何压到极少且 <200ms？

日志定位根因→减分配/泄漏→换 ZGC/G1 并调分代→大对象外置→压测验证。

**追问**：若是 Metaspace Full 而非堆 Full，步骤有何不同？

---

**文档导航**：MySQL 与库表调优 → [`02-MySQL深入.md`](./02-MySQL深入.md)、[`10-数据库综合.md`](./10-数据库综合.md)；并发模型对照 → [`05-并发编程模型.md`](./05-并发编程模型.md)。
