# Java 深入

---

## 📑 目录

- [Java核心特性](#java核心特性)
- [泛型与类型擦除](#泛型与类型擦除)
- [String与常量池](#string与常量池)
- [JVM深入](#jvm深入)
- [Java内存模型](#java内存模型)
- [并发编程](#并发编程)
- [Spring框架](#spring框架)
- [MyBatis实战](#mybatis实战)
- [性能调优](#性能调优)
- [常见面试题](#常见面试题)
- [实战案例](#实战案例)
- [📝 面试题自查](#-面试题自查)

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
    public int add(int a, int b) {
        return a + b;
    }
    
    public double add(double a, double b) {
        return a + b;
    }
}

// 运行时多态（方法重写）
Animal animal = new Dog();
animal.eat(); // 调用 Dog 的 eat 方法（如果重写了）
```

---

**2. 抽象类 vs 接口**

| 特性 | 抽象类 | 接口 |
|------|-------|------|
| 继承 | 单继承 | 多实现 |
| 方法 | 可有实现 | 默认抽象（Java 8+ 可有默认实现） |
| 字段 | 可有实例字段 | 只能有常量（`public static final`） |
| 构造器 | 可有 | 不能有 |
| 使用场景 | "is-a" 关系 | "can-do" 能力 |

```java
// 抽象类
public abstract class Animal {
    protected String name;
    
    public Animal(String name) {
        this.name = name;
    }
    
    public abstract void makeSound(); // 抽象方法
    
    public void sleep() { // 具体方法
        System.out.println(name + " is sleeping");
    }
}

// 接口
public interface Flyable {
    void fly();
    
    // Java 8+ 默认方法
    default void land() {
        System.out.println("Landing...");
    }
}

// 实现
public class Bird extends Animal implements Flyable {
    public Bird(String name) {
        super(name);
    }
    
    @Override
    public void makeSound() {
        System.out.println("Chirp");
    }
    
    @Override
    public void fly() {
        System.out.println(name + " is flying");
    }
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
│   └── Vector (线程安全的 ArrayList)
├── Set (无序、不重复)
│   ├── HashSet (哈希表)
│   ├── LinkedHashSet (哈希表 + 链表，保持插入顺序)
│   └── TreeSet (红黑树，自动排序)
└── Queue (队列)
    ├── PriorityQueue (优先队列)
    └── Deque (双端队列)
        └── ArrayDeque

Map (键值对)
├── HashMap (哈希表)
├── LinkedHashMap (哈希表 + 链表，保持插入顺序)
├── TreeMap (红黑树，按键排序)
└── ConcurrentHashMap (线程安全)
```

---

**2. ArrayList vs LinkedList**

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 插入/删除（头部） | O(n) | O(1) |
| 插入/删除（尾部） | O(1) 摊销 | O(1) |
| 内存占用 | 连续内存 | 每个节点额外开销 |

```java
// ArrayList
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.get(0); // O(1)

// LinkedList
List<String> linkedList = new LinkedList<>();
linkedList.add("A");
linkedList.add("B");
linkedList.get(0); // O(n)
```

---

**3. HashMap 原理**

**底层结构**（Java 8+）：
- **数组 + 链表 + 红黑树**
- 链表长度 >= 8 且数组长度 >= 64 时，转为红黑树
- 红黑树节点 <= 6 时，转回链表

**put 流程**：
1. 计算 key 的 hash 值
2. 根据 hash 定位数组索引：`index = hash & (length - 1)`
3. 如果位置为空，直接插入
4. 如果位置有元素：
   - key 相同，覆盖 value
   - key 不同，尾插法插入链表/红黑树
5. 如果 `size > threshold`，扩容（2倍）

```java
Map<String, Integer> map = new HashMap<>();
map.put("Alice", 30); // hash("Alice") -> index -> 存储

// 自定义对象作为 key，必须重写 hashCode 和 equals
public class User {
    private int id;
    private String name;
    
    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof User)) return false;
        User other = (User) obj;
        return id == other.id && Objects.equals(name, other.name);
    }
}
```

**线程安全问题**：
- `HashMap` 非线程安全，并发修改可能死循环（Java 7）或数据丢失（Java 8）
- 解决方案：
  - `ConcurrentHashMap`（推荐）
  - `Collections.synchronizedMap(new HashMap<>())`
  - `Hashtable`（已过时）

---

**4. ConcurrentHashMap 原理**

**Java 7**：
- **分段锁（Segment）**：数组分成多个 Segment，每个 Segment 有独立的锁
- 并发度 = Segment 数量（默认 16）

**Java 8+**：
- 取消 Segment，使用 **CAS + synchronized**
- 粒度更细：锁单个数组节点
- 并发度更高

```java
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("key", 1); // 线程安全
```

---

### 异常处理

**异常体系**：
```
Throwable
├── Error (严重错误，不应捕获)
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── RuntimeException (运行时异常，不强制捕获)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   └── IllegalArgumentException
    └── Checked Exception (受检异常，必须捕获或声明)
        ├── IOException
        └── SQLException
```

**try-catch-finally**：
```java
try {
    // 可能抛出异常的代码
    int result = 10 / 0;
} catch (ArithmeticException e) {
    // 捕获特定异常
    System.err.println("Division by zero: " + e.getMessage());
} catch (Exception e) {
    // 捕获其他异常
    System.err.println("Error: " + e);
} finally {
    // 无论是否异常，都会执行（用于资源清理）
    System.out.println("Finally block");
}
```

**try-with-resources**（自动资源管理）：
```java
// 实现 AutoCloseable 接口的资源会自动关闭
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
// br 自动关闭，无需 finally
```

---

## 泛型与类型擦除

### 泛型基础

泛型（Generics）是 Java 5 引入的类型参数化机制，让代码在编译时进行类型检查，避免运行时的 `ClassCastException`。

```java
// 泛型类
public class Box<T> {
    private T value;
    
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

Box<String> box = new Box<>();
box.set("hello");
String s = box.get();    // 无需强制转换
// box.set(123);          // 编译错误！类型安全

// 泛型方法
public <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}

// 泛型接口
public interface Repository<T, ID> {
    T findById(ID id);
    void save(T entity);
}
```

---

### 类型擦除（Type Erasure）

Java 的泛型是**编译期特性**，在运行时会被擦除：

```java
// 编译前
List<String> list1 = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();

// 运行时（类型被擦除）
list1.getClass() == list2.getClass()  // true！都是 ArrayList.class

// 编译器做了什么？
// Box<String> 编译后变成：
public class Box {
    private Object value;           // T → Object
    public void set(Object value) { this.value = value; }
    public Object get() { return value; }
}

// 编译器在调用处自动插入强制转换：
String s = (String) box.get();     // 编译器自动加的
```

**类型擦除带来的限制**：

```java
// ❌ 不能用基本类型作为泛型参数
List<int> list;                // 编译错误
List<Integer> list;            // ✅ 使用包装类

// ❌ 不能创建泛型数组
T[] arr = new T[10];           // 编译错误

// ❌ 不能用 instanceof 检查泛型类型
if (obj instanceof List<String>)  // 编译错误
if (obj instanceof List<?>)       // ✅ 可以

// ❌ 不能重载泛型方法（擦除后签名相同）
void process(List<String> list) {}
void process(List<Integer> list) {}  // 编译错误！擦除后都是 List
```

---

### 通配符与上下界

```java
// 上界通配符 <? extends T>（只读，生产者）
// "某个 Number 的子类型"——可以安全读取为 Number，但不能写入
public double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) {     // ✅ 安全读取
        total += n.doubleValue();
    }
    // list.add(1);              // ❌ 编译错误！不知道具体类型
    return total;
}

sum(List.of(1, 2, 3));          // List<Integer> ✅
sum(List.of(1.0, 2.0));         // List<Double> ✅

// 下界通配符 <? super T>（只写，消费者）
// "某个 Integer 的父类型"——可以安全写入 Integer，但读取只能是 Object
public void addIntegers(List<? super Integer> list) {
    list.add(1);                // ✅ 安全写入
    list.add(2);
    // Integer n = list.get(0); // ❌ 编译错误！只能读为 Object
    Object obj = list.get(0);   // ✅
}

addIntegers(new ArrayList<Number>());   // ✅
addIntegers(new ArrayList<Object>());   // ✅
```

> **PECS 原则**（Producer Extends, Consumer Super）：如果只需要从集合中读取，用 `extends`；如果只需要向集合中写入，用 `super`。这是 Collections 工具类广泛使用的设计模式。

---

## String与常量池

### String 的不可变性

`String` 在 Java 中是**不可变对象**（Immutable），其底层实现：

```java
// Java 8：char[]
public final class String {
    private final char[] value;    // final + private，不可修改
    private int hash;              // 缓存 hashCode
}

// Java 9+：byte[]（节省内存，Latin-1 字符用 1 字节）
public final class String {
    private final byte[] value;
    private final byte coder;      // LATIN1(0) 或 UTF16(1)
}
```

**不可变的好处**：
1. **线程安全**：多线程共享无需加锁
2. **哈希值缓存**：`hashCode()` 只计算一次，HashMap 的 key 首选
3. **字符串常量池**：相同内容的字符串可以共享同一个对象
4. **安全性**：类加载器、网络连接等使用 String 作为参数，不可变保证不被篡改

---

### 字符串常量池（String Pool）

```java
// 字面量字符串自动进入常量池
String s1 = "hello";           // 常量池中创建 "hello"
String s2 = "hello";           // 复用常量池中的同一个对象
System.out.println(s1 == s2);  // true（同一个对象）

// new 创建的字符串在堆上，不在常量池
String s3 = new String("hello");  // 堆上创建新对象
System.out.println(s1 == s3);     // false（不同对象）
System.out.println(s1.equals(s3)); // true（内容相同）

// intern()：将字符串放入常量池
String s4 = s3.intern();
System.out.println(s1 == s4);     // true（指向常量池同一对象）
```

**`new String("hello")` 创建了几个对象？**

- 如果常量池中已有 `"hello"` → 创建 **1 个对象**（堆上的 String 对象）
- 如果常量池中没有 `"hello"` → 创建 **2 个对象**（常量池中的 + 堆上的）

---

### String vs StringBuilder vs StringBuffer

| 类 | 可变性 | 线程安全 | 性能 | 使用场景 |
|----|--------|---------|------|---------|
| **String** | 不可变 | 安全 | 拼接慢（创建新对象） | 少量字符串操作 |
| **StringBuilder** | 可变 | 不安全 | 快 | 单线程大量拼接 |
| **StringBuffer** | 可变 | 安全（synchronized） | 较慢 | 多线程大量拼接 |

```java
// ❌ 循环中用 String 拼接（每次创建新对象）
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;  // 创建 10000 个中间对象，O(n²)
}

// ✅ 使用 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);  // 原地修改，O(n)
}
String s = sb.toString();
```

---

## JVM深入

### JVM 内存结构

```
┌─────────────────────────────────────────────┐
│              JVM 内存                        │
├─────────────────────────────────────────────┤
│  程序计数器 (Program Counter Register)       │  ← 线程私有
│  - 当前线程执行的字节码行号                   │
├─────────────────────────────────────────────┤
│  虚拟机栈 (VM Stack)                         │  ← 线程私有
│  - 方法调用栈帧（局部变量表、操作数栈等）      │
├─────────────────────────────────────────────┤
│  本地方法栈 (Native Method Stack)            │  ← 线程私有
│  - 调用 native 方法时使用                    │
├─────────────────────────────────────────────┤
│  堆 (Heap)                                  │  ← 线程共享
│  - 对象实例、数组                            │
│  - GC 主要区域                               │
│  ├── 新生代 (Young Generation)              │
│  │   ├── Eden 区                            │
│  │   ├── Survivor 0 (From)                 │
│  │   └── Survivor 1 (To)                   │
│  └── 老年代 (Old Generation)                │
├─────────────────────────────────────────────┤
│  方法区 (Method Area / Metaspace)            │  ← 线程共享
│  - 类信息、常量池、静态变量                   │
│  - Java 8+ 使用 Metaspace（在本地内存）     │
└─────────────────────────────────────────────┘
```

---

### 垃圾回收（GC）

**1. 判断对象是否存活**

**引用计数法**（Python 使用）：
- 每个对象维护引用计数
- 问题：无法解决循环引用

**可达性分析**（Java 使用）：
- 从 GC Roots 开始，标记所有可达对象
- GC Roots 包括：
  - 虚拟机栈中的局部变量
  - 方法区中的静态变量
  - 方法区中的常量
  - 本地方法栈中的引用

---

**2. 垃圾回收算法**

**标记-清除（Mark-Sweep）**：
- 标记存活对象，清除未标记对象
- 问题：内存碎片

**标记-复制（Mark-Copy）**：
- 将内存分为两块，每次只使用一块
- GC 时，将存活对象复制到另一块，清空当前块
- 优点：无碎片，速度快
- 缺点：浪费 50% 内存
- **用于新生代**

**标记-整理（Mark-Compact）**：
- 标记存活对象，将其移动到内存一端，清除边界外的对象
- 优点：无碎片
- 缺点：移动对象成本高
- **用于老年代**

---

**3. 分代回收**

**新生代（Young Generation）**：
- **Eden 区**：新对象分配
- **Survivor 区**：From 和 To 两块，存放经过一次 GC 后存活的对象
- **Minor GC**：
  1. Eden 区满时触发
  2. 存活对象复制到 Survivor To
  3. From 和 To 交换

**老年代（Old Generation）**：
- 长期存活的对象（经过多次 Minor GC）
- **Major GC / Full GC**：老年代满时触发，回收整个堆

**对象晋升**：
- 新对象在 Eden 区分配
- 经过一次 Minor GC，移动到 Survivor
- 经过多次 GC（默认 15 次），晋升到老年代
- 大对象直接进入老年代

---

**4. 垃圾回收器**

| 回收器 | 特点 | 适用场景 |
|--------|------|---------|
| **Serial** | 单线程，Stop-The-World | 单核 CPU、小堆 |
| **Parallel** | 多线程，吞吐量优先 | 后台计算任务 |
| **CMS** | 并发标记清除，低延迟 | Web 应用（已废弃） |
| **G1** | 分区回收，可预测停顿 | 大堆（> 4GB）|
| **ZGC** | 超低延迟（< 10ms） | 对延迟极其敏感的应用 |

**G1 GC（默认）**：
- 将堆分为多个 Region（1-32MB）
- 优先回收垃圾最多的 Region
- 可设置停顿时间目标：`-XX:MaxGCPauseMillis=200`

---

#### 垃圾回收器底层原理深度剖析⭐⭐⭐

**为什么需要了解GC底层？**
- 面试高频：CMS为什么会有碎片？G1如何解决？
- 性能调优：如何选择合适的GC器？
- 故障排查：为什么Full GC频繁？

---

##### 1. CMS（Concurrent Mark Sweep）详解⭐⭐⭐

**设计目标**：低延迟（减少STW时间）

**CMS的4个阶段**：

```
阶段1：初始标记（Initial Mark）⏸ STW
- 标记GC Roots直接引用的对象
- 时间：很短（只标记一层）

阶段2：并发标记（Concurrent Mark）✅ 并发
- 从GC Roots遍历整个对象图
- 时间：最长（但不STW，用户线程可继续运行）

阶段3：重新标记（Remark）⏸ STW
- 修正并发标记期间，用户线程导致的标记变动
- 时间：较短（比初始标记长）

阶段4：并发清除（Concurrent Sweep）✅ 并发
- 清除未标记的对象
- 时间：较长（但不STW）

总STW时间 = 初始标记 + 重新标记（很短）
```

**CMS的优势**：
```
传统GC（Serial Old）：
STW: ████████████████████████ (500ms)

CMS：
初始标记: ██ (50ms)
并发标记: ████████████ (400ms, 不STW)
重新标记: ████ (100ms)
并发清除: ████████ (300ms, 不STW)

用户感知停顿：50ms + 100ms = 150ms（减少70%）
```

**CMS的3大问题**：

**问题1：内存碎片**

```
为什么会有碎片？
- CMS使用"标记-清除"算法（不整理）
- 清除后留下很多小块空闲空间

影响：
Old Gen: [obj1][free][obj2][free][obj3][free]
         ↑ 10KB ↑ 5KB  ↑ 8KB  ↑ 3KB  ↑ 7KB

当需要分配20KB对象时：
- 虽然总空闲空间 = 5+8+3+7 = 23KB > 20KB
- 但没有连续的20KB空间
- 触发Full GC（STW整理内存）
```

**解决方案**：
```java
// 配置参数
-XX:+UseCMSCompactAtFullCollection  // Full GC时整理内存
-XX:CMSFullGCsBeforeCompaction=5   // 5次Full GC后整理一次

// 但这会导致Full GC变慢（整理需要时间）
```

**问题2：并发模式失败（Concurrent Mode Failure）**

```
原因：
- CMS并发清除期间，老年代空间不足
- 新对象晋升到老年代，但老年代已满
- CMS还没完成清理工作

后果：
- CMS中断，降级为Serial Old（单线程、STW、整理内存）
- 停顿时间暴增（几秒）

示例：
Old Gen: 80%已使用 ← CMS触发（-XX:CMSInitiatingOccupancyFraction=70）
并发清理中...（需要1秒）
0.5秒后：晋升大量对象，老年代满 → Concurrent Mode Failure
→ Serial Old接管（STW 3秒）
```

**解决方案**：
```java
// 提前触发CMS（给CMS更多时间清理）
-XX:CMSInitiatingOccupancyFraction=70  // 70%时触发（默认92%）
-XX:+UseCMSInitiatingOccupancyOnly     // 只使用该阈值

// 增大老年代
-Xmx4g -XX:NewRatio=2  // 老年代2GB
```

**问题3：浮动垃圾（Floating Garbage）**

```
原因：
- 并发标记阶段，用户线程产生新垃圾
- 这些垃圾在本次GC不会被回收（已标记为"存活"）

示例：
初始标记：obj1引用obj2 → 标记obj2为存活
并发标记中：用户线程断开obj1→obj2的引用
→ obj2成为垃圾，但本次GC不会回收
→ 等待下次GC

影响：
- 老年代空间利用率降低
- 可能触发Concurrent Mode Failure
```

---

##### 2. G1（Garbage First）详解⭐⭐⭐

**设计目标**：可预测的停顿时间（例如200ms）

**G1的核心设计**：

```
传统分代（堆划分为固定区域）：
[Young Gen: Eden | S0 | S1][Old Gen: 大块连续空间]

G1分代（堆划分为多个Region）：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│Eden│Eden│ S  │ S  │Old │Old │Old │ H  │ ← 每个Region 1-32MB
└────┴────┴────┴────┴────┴────┴────┴────┘
E = Eden, S = Survivor, O = Old, H = Humongous（大对象）

灵活性：
- Region动态分配（Eden、Survivor、Old可变）
- 大对象（> Region 50%）占用连续Region
```

**G1的回收过程**：

**1. Young GC（STW）**：
```
触发：Eden Region满
步骤：
1. 扫描GC Roots
2. 复制存活对象到Survivor/Old Region
3. 回收整个Young Gen（Eden + Survivor）

为什么快？
- 只回收Young Gen（Region少）
- 使用多线程并行复制
- 停顿时间：10-50ms
```

**2. Mixed GC（混合回收，STW）**：
```
触发：老年代占比达到45%（-XX:InitiatingHeapOccupancyPercent=45）

步骤：
1. 并发标记（找出垃圾最多的Old Region）
2. Mixed GC：回收Young Gen + 部分Old Region

关键：只回收垃圾最多的Old Region（不是全部）
→ 控制停顿时间

示例：
Old Regions: [R1:90%垃圾][R2:80%][R3:70%][R4:30%][R5:20%]
Mixed GC: 回收R1、R2、R3（达到停顿时间目标200ms）
R4、R5等待下次GC
```

**G1如何预测停顿时间？**

```java
// 设置停顿时间目标
-XX:MaxGCPauseMillis=200  // 期望停顿≤200ms

G1的预测模型：
1. 历史统计：记录每个Region的回收耗时
   R1: 平均20ms
   R2: 平均15ms
   R3: 平均18ms
   ...

2. 动态选择：根据目标停顿时间，选择Region
   目标200ms → 可回收R1+R2+R3+...+R10（总计195ms）
   → 优先选择垃圾最多的Region

3. 自适应调整：
   - 如果实际停顿>200ms → 减少回收Region数量
   - 如果实际停顿<200ms → 增加回收Region数量
```

**G1如何解决CMS的问题？**

**问题1：内存碎片 → 已解决**
```
G1使用"标记-复制"算法（整理内存）：
回收过程：
1. 标记存活对象
2. 复制到新Region
3. 回收整个旧Region

结果：新Region内存连续，无碎片

Old Regions（回收前）：
[obj1][free][obj2][free][obj3] ← 碎片

Old Regions（回收后）：
[obj1][obj2][obj3]... ← 连续，无碎片
```

**问题2：Concurrent Mode Failure → 改进**
```
G1的Full GC触发条件更宽松：
- CMS：老年代满 → 立即Full GC
- G1：Mixed GC来不及 → Full GC

G1优势：
- Mixed GC可以回收部分老年代（不是全部或不回）
- 更灵活，减少Full GC频率

但仍可能Full GC：
- 对象分配过快，Mixed GC来不及
- 大对象（Humongous）分配失败
```

**问题3：浮动垃圾 → 依然存在**
```
G1也有并发标记阶段，同样存在浮动垃圾
但影响较小（Mixed GC可以快速回收）
```

**G1的缺点**：

```
1. 内存占用更高
   - 每个Region需要额外的元数据（Remembered Set）
   - 记录Region间的引用关系
   - 占用约10%堆内存

2. 卡表维护开销
   - 更新对象引用时，需要更新卡表（写屏障）
   - 吞吐量下降5-10%

3. 不适合小堆
   - Region数量少，灵活性差
   - 推荐堆大小 > 4GB
```

---

##### 3. ZGC（Z Garbage Collector）详解⭐⭐⭐

**设计目标**：超低延迟（停顿 < 10ms，不随堆大小增加）

**ZGC的革命性技术**：

**1. 染色指针（Colored Pointers）**

```
传统GC：对象头存储标记位
ZGC：指针本身存储标记位

64位指针布局（Linux x86-64）：
┌─────────────────────────────────────────────┐
│ 未使用 │ Finalizable │ Remapped │ Marked1 │ Marked0 │ 对象地址 (42位) │
│ 16位   │ 1位         │ 1位      │ 1位     │ 1位     │ 42位            │
└─────────────────────────────────────────────┘

为什么用指针存储？
- 不需要访问对象，就能知道标记状态（快）
- 支持并发标记（不需要STW）
```

**2. 读屏障（Load Barrier）**

```
传统GC：需要STW移动对象
ZGC：使用读屏障并发移动对象

示例：
Object obj = field.object;  // 读取对象引用

ZGC编译后：
Object obj = field.object;
if (obj需要重定位) {  // 读屏障
    obj = 重定位后的地址;
    field.object = obj;  // 更新指针
}

关键：
- 对象移动后，旧地址→新地址的映射存储在转发表
- 首次访问时，通过读屏障修正指针
- 后续访问直接使用新地址（无开销）
```

**ZGC的回收过程**：

```
阶段1：初始标记（STW）⏸ < 1ms
- 标记GC Roots

阶段2：并发标记（Concurrent Mark）✅ 并发
- 遍历对象图，标记存活对象
- 使用染色指针，无需访问对象

阶段3：再标记（STW）⏸ < 1ms
- 处理并发标记期间的变化

阶段4：初始转移（STW）⏸ < 1ms
- 选择要回收的Region

阶段5：并发转移（Concurrent Relocate）✅ 并发
- 移动存活对象到新Region
- 使用读屏障，并发修正指针

总STW时间：< 3ms（不随堆大小增加）
```

**ZGC vs G1 性能对比**：

| 指标 | G1 (4GB堆) | G1 (64GB堆) | ZGC (4GB堆) | ZGC (64GB堆) |
|------|-----------|------------|------------|-------------|
| **Young GC** | 20ms | 50ms | < 1ms | < 1ms |
| **Mixed GC** | 50ms | 200ms | < 3ms | < 3ms |
| **Full GC** | 3s | 10s | < 10ms | < 10ms |
| **吞吐量** | 95% | 93% | 90% | 90% |

**ZGC的适用场景**：

```
✅ 适合：
- 对延迟极其敏感（金融交易、游戏）
- 大堆（> 16GB）
- 可接受10%吞吐量损失

❌ 不适合：
- 小堆（< 4GB，ZGC开销大）
- 吞吐量优先的场景（用Parallel GC）
- 堆内存紧张（ZGC需要额外内存存储元数据）
```

---

##### 4. GC选择指南⭐⭐⭐

**根据应用场景选择**：

| 场景 | 推荐GC | 原因 |
|------|--------|------|
| **Web应用（< 4GB堆）** | G1 | 平衡延迟和吞吐量 |
| **Web应用（> 4GB堆）** | ZGC | 低延迟，堆大小不影响停顿 |
| **批处理任务** | Parallel GC | 吞吐量优先 |
| **金融交易系统** | ZGC | 超低延迟（< 10ms） |
| **游戏服务器** | ZGC / Shenandoah | 低延迟，避免卡顿 |
| **微服务（小堆）** | G1 | ZGC开销大，不适合小堆 |

**调优建议**：

```java
// G1调优（推荐）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          // 目标停顿200ms
-XX:G1HeapRegionSize=16m          // Region大小（1-32MB）
-XX:InitiatingHeapOccupancyPercent=45  // 45%时触发Mixed GC
-XX:G1ReservePercent=10           // 保留10%空间防止Full GC

// ZGC调优（低延迟）
-XX:+UseZGC
-XX:ConcGCThreads=4               // 并发GC线程数（推荐CPU核数/4）
-XX:ZAllocationSpikeTolerance=5   // 允许5倍的内存分配峰值

// Parallel GC调优（吞吐量）
-XX:+UseParallelGC
-XX:MaxGCPauseMillis=100          // 尽力达到100ms
-XX:GCTimeRatio=99                // GC时间/总时间 = 1/100（吞吐量99%）
```

**Full GC排查**：

```java
// 开启GC日志
-Xlog:gc*:file=gc.log:time,uptime,level,tags

// 分析Full GC原因
[Full GC (Allocation Failure)] ← 内存不足
[Full GC (Metadata GC Threshold)] ← 元空间不足
[Full GC (System.gc())] ← 代码显式调用（避免）

// 解决方案
1. Allocation Failure → 增大堆、调整GC器
2. Metadata GC → 增大元空间（-XX:MetaspaceSize=256m）
3. System.gc() → 禁用（-XX:+DisableExplicitGC）
```

---

### 类加载机制

**1. 类加载过程**

```
加载 (Loading)
  ↓
验证 (Verification)
  ↓
准备 (Preparation)
  ↓
解析 (Resolution)
  ↓
初始化 (Initialization)
```

**加载**：读取 .class 文件，生成 Class 对象

**验证**：确保字节码合法（文件格式、元数据、字节码、符号引用）

**准备**：为静态变量分配内存，设置默认值
```java
public static int value = 123; // 准备阶段：value = 0，初始化阶段：value = 123
```

**解析**：将符号引用转为直接引用

**初始化**：执行类构造器 `<clinit>()`（静态变量赋值、静态代码块）

---

**2. 类加载器**

```
Bootstrap ClassLoader (启动类加载器)
  ↓ 父加载器
Extension ClassLoader (扩展类加载器)
  ↓ 父加载器
Application ClassLoader (应用类加载器)
  ↓ 父加载器
Custom ClassLoader (自定义类加载器)
```

**双亲委派模型**：
1. 类加载器收到加载请求
2. 委派给父加载器加载
3. 父加载器无法加载，子加载器才尝试加载

**优点**：
- 避免类重复加载
- 保护核心 API（例如无法自定义 `java.lang.String`）

```java
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        return defineClass(name, classData, 0, classData.length);
    }
    
    private byte[] loadClassData(String name) {
        // 从文件/网络加载字节码
        return ...;
    }
}
```

---

## Java内存模型

### JMM（Java Memory Model）

JMM 定义了多线程环境下**共享变量的可见性、有序性和原子性**规则。它是 Java 并发编程的理论基础。

**核心问题**：每个线程有自己的**工作内存**（CPU 缓存），共享变量存储在**主内存**中。线程对变量的操作都在工作内存中进行，何时同步到主内存由 JMM 规定。

```
线程1                         线程2
┌──────────────┐             ┌──────────────┐
│  工作内存     │             │  工作内存     │
│  x = 1 (副本) │             │  x = 0 (副本) │  ← 可能看到旧值！
└──────┬───────┘             └──────┬───────┘
       │                            │
       ▼          同步               ▼
  ┌────────────────────────────────────┐
  │           主内存                    │
  │           x = 1                    │
  └────────────────────────────────────┘
```

---

### happens-before 规则

JMM 通过 **happens-before** 关系保证操作的可见性和有序性。如果操作 A happens-before 操作 B，那么 A 的结果对 B 可见。

**8 条 happens-before 规则**：

| 规则 | 说明 |
|------|------|
| **程序顺序规则** | 同一线程内，前面的操作 happens-before 后面的操作 |
| **volatile 规则** | volatile 写 happens-before 后续的 volatile 读 |
| **锁规则** | unlock 操作 happens-before 后续对同一个锁的 lock |
| **线程启动规则** | `Thread.start()` happens-before 被启动线程的任何操作 |
| **线程终止规则** | 线程的所有操作 happens-before 其他线程检测到该线程终止 |
| **中断规则** | `thread.interrupt()` happens-before 被中断线程检测到中断 |
| **终结器规则** | 对象构造函数 happens-before `finalize()` |
| **传递性** | A happens-before B，B happens-before C → A happens-before C |

```java
// volatile 规则示例
volatile boolean flag = false;
int x = 0;

// 线程1
x = 42;           // ① 普通写
flag = true;       // ② volatile 写

// 线程2
if (flag) {        // ③ volatile 读
    print(x);      // ④ 一定看到 42（由传递性保证）
}

// 推导：
// ① happens-before ② （程序顺序规则）
// ② happens-before ③ （volatile 规则）
// 由传递性：① happens-before ③
// 所以 ④ 一定能看到 x = 42
```

---

### synchronized 锁升级⭐⭐⭐

Java 6 引入了**锁优化**机制，`synchronized` 不再直接使用重量级锁（OS 互斥量），而是根据竞争程度逐步升级：

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

**对象头（Mark Word）结构**（64 位 JVM）：

```
┌─────────────────────────────────────────────────────┬──────┬───────┐
│                 Mark Word (64 bits)                  │ 状态  │ 说明   │
├─────────────────────────────────────────────────────┼──────┼───────┤
│ unused:25 | hashCode:31 | unused:1 | age:4 | 0 | 01│ 无锁  │        │
│ thread:54 | epoch:2 | unused:1 | age:4 | 1 | 01    │ 偏向锁 │        │
│ ptr_to_lock_record:62                         | 00  │ 轻量级锁│       │
│ ptr_to_heavyweight_monitor:62                 | 10  │ 重量级锁│       │
│                                               | 11  │ GC标记 │        │
└─────────────────────────────────────────────────────┴──────┴───────┘
```

**偏向锁（Biased Locking）**：

```
适用场景：只有一个线程反复获取同一把锁（无竞争）

原理：
1. 第一次获取锁：CAS 将线程 ID 写入 Mark Word
2. 后续获取锁：只检查线程 ID 是否是自己 → 无需 CAS，几乎零开销
3. 另一个线程尝试获取 → 撤销偏向锁，升级为轻量级锁

性能：无竞争时，获取锁的开销接近于无锁操作
```

**轻量级锁（Lightweight Lock）**：

```
适用场景：多个线程交替获取锁（低竞争，无并发）

原理：
1. 在当前线程栈帧中创建 Lock Record
2. CAS 将 Mark Word 复制到 Lock Record，并将 Mark Word 替换为指向 Lock Record 的指针
3. CAS 成功 → 获取锁
4. CAS 失败 → 自旋等待（空转 CPU）
5. 自旋超过阈值 → 升级为重量级锁

性能：短暂竞争时，自旋避免了线程阻塞/唤醒的开销
```

**重量级锁（Heavyweight Lock）**：

```
适用场景：激烈竞争（多个线程同时争抢锁）

原理：
1. 使用操作系统的 Mutex（互斥量）
2. 获取锁失败 → 线程阻塞（内核态切换）
3. 锁释放 → 唤醒等待线程（内核态切换）

性能：最慢，但在高竞争下最合理（避免 CPU 空转浪费）
```

**锁升级对比**：

| 锁类型 | 竞争程度 | 获取方式 | 开销 | 适用场景 |
|--------|---------|---------|------|---------|
| **偏向锁** | 无竞争 | 检查线程 ID | 极低 | 单线程反复同步 |
| **轻量级锁** | 低竞争 | CAS + 自旋 | 较低 | 线程交替执行 |
| **重量级锁** | 高竞争 | OS Mutex | 高 | 多线程并发竞争 |

> **注意**：锁只能升级，不能降级。Java 15 默认禁用了偏向锁（`-XX:-UseBiasedLocking`），因为现代应用大多是多线程环境，偏向锁的撤销反而是开销。

---

## 并发编程

### 线程基础

**创建线程**：

**方式1：继承 Thread**
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running");
    }
}

MyThread thread = new MyThread();
thread.start();
```

**方式2：实现 Runnable**（推荐）
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable running");
    }
}

Thread thread = new Thread(new MyRunnable());
thread.start();

// Lambda 写法
Thread thread = new Thread(() -> {
    System.out.println("Lambda running");
});
thread.start();
```

**方式3：实现 Callable**（可返回结果）
```java
import java.util.concurrent.*;

public class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        Thread.sleep(1000);
        return "Result";
    }
}

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<String> future = executor.submit(new MyCallable());
String result = future.get(); // 阻塞等待结果
System.out.println(result);
executor.shutdown();
```

---

### synchronized 同步

**1. 对象锁**
```java
public class Counter {
    private int count = 0;
    
    // 同步方法（锁当前对象 this）
    public synchronized void increment() {
        count++;
    }
    
    // 同步代码块（锁指定对象）
    public void decrement() {
        synchronized (this) {
            count--;
        }
    }
}
```

**2. 类锁**
```java
public class Counter {
    private static int count = 0;
    
    // 同步静态方法（锁 Class 对象）
    public static synchronized void increment() {
        count++;
    }
    
    // 同步代码块（锁 Class 对象）
    public static void decrement() {
        synchronized (Counter.class) {
            count--;
        }
    }
}
```

---

### volatile 关键字

**作用**：
1. **可见性**：一个线程修改后，其他线程立即可见
2. **禁止指令重排序**

```java
public class Singleton {
    private static volatile Singleton instance; // volatile 保证可见性
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) { // 第二次检查（有锁）
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**为什么需要 volatile？**
- `instance = new Singleton()` 分为三步：
  1. 分配内存
  2. 初始化对象
  3. 将 instance 指向内存
- 可能发生指令重排序（2 和 3 交换），导致其他线程看到未初始化的对象
- `volatile` 禁止重排序

---

### Lock 锁

**ReentrantLock**（可重入锁）：

```java
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // 必须在 finally 中释放
        }
    }
    
    // 尝试获取锁（非阻塞）
    public void tryIncrement() {
        if (lock.tryLock()) {
            try {
                count++;
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("Failed to acquire lock");
        }
    }
}
```

**ReentrantLock vs synchronized**：

| 特性 | synchronized | ReentrantLock |
|------|--------------|---------------|
| 使用 | 简单，自动释放 | 复杂，手动释放 |
| 可中断 | 不可中断 | `lockInterruptibly()` |
| 公平锁 | 非公平 | 可配置公平锁 |
| 条件变量 | 1 个（wait/notify） | 多个（Condition） |
| 性能 | Java 6+ 优化后相当 | 略低 |

---

**ReadWriteLock**（读写锁）：

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    private final Map<String, String> data = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public String get(String key) {
        lock.readLock().lock(); // 读锁（多个线程可同时持有）
        try {
            return data.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(String key, String value) {
        lock.writeLock().lock(); // 写锁（排他）
        try {
            data.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

### 线程池

**为什么使用线程池？**
- 降低资源消耗（复用线程）
- 提高响应速度（无需创建线程）
- 便于管理（统一分配、调优、监控）

**线程池参数**：
```java
public ThreadPoolExecutor(
    int corePoolSize,       // 核心线程数
    int maximumPoolSize,    // 最大线程数
    long keepAliveTime,     // 空闲线程存活时间
    TimeUnit unit,          // 时间单位
    BlockingQueue<Runnable> workQueue, // 任务队列
    ThreadFactory threadFactory,       // 线程工厂
    RejectedExecutionHandler handler   // 拒绝策略
)
```

**执行流程**：
1. 如果线程数 < corePoolSize，创建新线程执行任务
2. 如果线程数 >= corePoolSize，任务放入队列
3. 如果队列满了且线程数 < maximumPoolSize，创建新线程
4. 如果线程数 >= maximumPoolSize，执行拒绝策略

**拒绝策略**：
- `AbortPolicy`：抛出异常（默认）
- `CallerRunsPolicy`：由调用线程执行任务
- `DiscardPolicy`：丢弃任务
- `DiscardOldestPolicy`：丢弃队列最旧的任务

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                           // 核心线程数
    5,                           // 最大线程数
    60, TimeUnit.SECONDS,        // 空闲线程存活时间
    new LinkedBlockingQueue<>(10), // 队列容量 10
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.AbortPolicy()
);

executor.execute(() -> {
    System.out.println("Task running");
});

executor.shutdown(); // 优雅关闭
```

---

**常用线程池**：

```java
// 固定大小线程池
ExecutorService executor = Executors.newFixedThreadPool(5);

// 单线程线程池
ExecutorService executor = Executors.newSingleThreadExecutor();

// 缓存线程池（线程数不固定，适合短时任务）
ExecutorService executor = Executors.newCachedThreadPool();

// 定时任务线程池
ScheduledExecutorService executor = Executors.newScheduledThreadPool(5);
executor.schedule(() -> {
    System.out.println("Task");
}, 5, TimeUnit.SECONDS); // 5 秒后执行

executor.scheduleAtFixedRate(() -> {
    System.out.println("Task");
}, 0, 1, TimeUnit.SECONDS); // 每秒执行一次
```

---

### CompletableFuture

**异步编程**：

```java
import java.util.concurrent.CompletableFuture;

// 异步执行任务
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // 耗时操作
    return "Result";
});

// 回调处理
future.thenAccept(result -> {
    System.out.println("Result: " + result);
});

// 阻塞等待结果
String result = future.get();
```

**链式调用**：

```java
CompletableFuture.supplyAsync(() -> {
    return "Hello";
}).thenApply(s -> {
    return s + " World";
}).thenAccept(s -> {
    System.out.println(s); // Hello World
});
```

**组合多个任务**：

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "B");

// 等待所有完成
CompletableFuture.allOf(future1, future2).join();

// 等待任意一个完成
CompletableFuture.anyOf(future1, future2).join();
```

---

## Spring框架

### IoC 容器

**控制反转（IoC）**：将对象创建和依赖关系的管理交给容器。

**依赖注入（DI）**：容器自动注入依赖对象。

**方式1：构造器注入**（推荐）
```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    // 单个构造器可省略 @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**方式2：Setter 注入**
```java
@Service
public class UserService {
    private UserRepository userRepository;
    
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**方式3：字段注入**（不推荐，测试困难）
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

---

### AOP 面向切面编程

**核心概念**：
- **切面（Aspect）**：横切关注点的模块化（日志、事务）
- **连接点（Join Point）**：程序执行的某个点（方法调用）
- **切点（Pointcut）**：匹配连接点的表达式
- **通知（Advice）**：在切点执行的代码（Before、After、Around）

```java
@Aspect
@Component
public class LogAspect {
    
    // 切点：匹配 com.example.service 包下的所有方法
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}
    
    // 前置通知
    @Before("serviceMethods()")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Before: " + joinPoint.getSignature().getName());
    }
    
    // 后置通知
    @After("serviceMethods()")
    public void logAfter(JoinPoint joinPoint) {
        System.out.println("After: " + joinPoint.getSignature().getName());
    }
    
    // 环绕通知
    @Around("serviceMethods()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed(); // 执行方法
        long end = System.currentTimeMillis();
        System.out.println("Time: " + (end - start) + "ms");
        return result;
    }
    
    // 返回通知
    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("Returned: " + result);
    }
    
    // 异常通知
    @AfterThrowing(pointcut = "serviceMethods()", throwing = "error")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable error) {
        System.out.println("Exception: " + error.getMessage());
    }
}
```

---

### 事务管理

**声明式事务**：

```java
@Service
public class TransferService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();
        
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        
        accountRepository.save(from);
        accountRepository.save(to);
        
        // 抛出异常会自动回滚
    }
}
```

**事务传播行为**：

| 传播行为 | 说明 |
|---------|------|
| `REQUIRED` | 如果有事务，加入；没有，创建新事务（默认） |
| `REQUIRES_NEW` | 总是创建新事务，挂起当前事务 |
| `SUPPORTS` | 如果有事务，加入；没有，以非事务执行 |
| `NOT_SUPPORTED` | 以非事务执行，挂起当前事务 |
| `MANDATORY` | 必须在事务中执行，否则抛出异常 |
| `NEVER` | 必须以非事务执行，否则抛出异常 |
| `NESTED` | 嵌套事务，外层回滚影响内层，内层不影响外层 |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void innerMethod() {
    // 独立事务
}
```

---

## MyBatis实战

### 基础使用

**Mapper 接口**：
```java
@Mapper
public interface UserMapper {
    
    @Select("SELECT * FROM users WHERE id = #{id}")
    User findById(Long id);
    
    @Insert("INSERT INTO users (name, email) VALUES (#{name}, #{email})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    void insert(User user);
    
    @Update("UPDATE users SET name = #{name} WHERE id = #{id}")
    void update(User user);
    
    @Delete("DELETE FROM users WHERE id = #{id}")
    void delete(Long id);
}
```

**XML 映射**：
```xml
<mapper namespace="com.example.mapper.UserMapper">
    
    <select id="findById" resultType="com.example.entity.User">
        SELECT * FROM users WHERE id = #{id}
    </select>
    
    <insert id="insert" parameterType="com.example.entity.User" 
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (name, email) VALUES (#{name}, #{email})
    </insert>
    
    <!-- 复杂查询 -->
    <select id="findByCondition" resultType="com.example.entity.User">
        SELECT * FROM users
        <where>
            <if test="name != null">
                AND name LIKE CONCAT('%', #{name}, '%')
            </if>
            <if test="email != null">
                AND email = #{email}
            </if>
        </where>
    </select>
    
    <!-- 批量插入 -->
    <insert id="batchInsert">
        INSERT INTO users (name, email) VALUES
        <foreach collection="list" item="user" separator=",">
            (#{user.name}, #{user.email})
        </foreach>
    </insert>
    
</mapper>
```

---

### 结果映射

```xml
<resultMap id="UserResultMap" type="com.example.entity.User">
    <id property="id" column="id"/>
    <result property="name" column="user_name"/>
    <result property="email" column="user_email"/>
    
    <!-- 一对一 -->
    <association property="profile" javaType="com.example.entity.Profile">
        <id property="id" column="profile_id"/>
        <result property="bio" column="bio"/>
    </association>
    
    <!-- 一对多 -->
    <collection property="orders" ofType="com.example.entity.Order">
        <id property="id" column="order_id"/>
        <result property="amount" column="amount"/>
    </collection>
</resultMap>

<select id="findUserWithOrders" resultMap="UserResultMap">
    SELECT
        u.id, u.name AS user_name, u.email AS user_email,
        p.id AS profile_id, p.bio,
        o.id AS order_id, o.amount
    FROM users u
    LEFT JOIN profiles p ON u.id = p.user_id
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = #{id}
</select>
```

---

## 性能调优

### JVM 调优

**1. 常用参数**

```bash
# 堆大小
-Xms2g               # 初始堆大小
-Xmx4g               # 最大堆大小
-XX:NewRatio=2       # 老年代/新生代比例（默认 2）
-XX:SurvivorRatio=8  # Eden/Survivor 比例（默认 8）

# 垃圾回收器
-XX:+UseG1GC                    # 使用 G1 GC
-XX:MaxGCPauseMillis=200        # 最大停顿时间
-XX:+UseZGC                     # 使用 ZGC

# GC 日志
-Xlog:gc*:file=gc.log:time,level,tags

# 线程栈大小
-Xss512k             # 每个线程栈大小（默认 1MB）

# Metaspace
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

---

**2. 内存溢出排查**

**OutOfMemoryError: Java heap space**（堆溢出）

```bash
# 生成堆转储文件
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# 分析堆转储
jhat heapdump.hprof
# 或使用 MAT（Eclipse Memory Analyzer）
```

**OutOfMemoryError: Metaspace**（元空间溢出）

```bash
# 增大 Metaspace
-XX:MaxMetaspaceSize=512m

# 检查是否类加载过多
jcmd <pid> GC.class_stats
```

---

**3. 性能监控工具**

**jps**：查看 Java 进程
```bash
jps -l
```

**jstat**：查看 GC 统计
```bash
jstat -gc <pid> 1000   # 每秒输出 GC 统计
jstat -gcutil <pid>    # 输出 GC 使用率
```

**jstack**：查看线程堆栈
```bash
jstack <pid> > thread.txt
# 查找死锁
jstack <pid> | grep -A 10 "deadlock"
```

**jmap**：查看堆内存
```bash
jmap -heap <pid>              # 查看堆配置
jmap -histo <pid>             # 查看对象统计
jmap -dump:format=b,file=heap.hprof <pid>  # 导出堆转储
```

---

### 数据库优化

**1. 索引优化**

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM users WHERE name = 'Alice';

-- 创建索引
CREATE INDEX idx_name ON users(name);

-- 联合索引（最左前缀原则）
CREATE INDEX idx_name_email ON users(name, email);
-- 可以使用 (name) 或 (name, email)，不能只用 (email)
```

---

**2. 慢查询优化**

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1; -- 超过 1 秒记录

-- 分析慢查询
mysqldumpslow -s t -t 10 slow.log  # 按时间排序，显示前 10
```

---

**3. 连接池配置**

```properties
# HikariCP（Spring Boot 默认）
spring.datasource.hikari.maximum-pool-size=10     # 最大连接数
spring.datasource.hikari.minimum-idle=5           # 最小空闲连接数
spring.datasource.hikari.connection-timeout=30000 # 连接超时（30秒）
spring.datasource.hikari.idle-timeout=600000      # 空闲超时（10分钟）
spring.datasource.hikari.max-lifetime=1800000     # 连接最大存活时间（30分钟）
```

---

## 常见面试题

**1. HashMap 和 ConcurrentHashMap 的区别？**

- `HashMap` 非线程安全，`ConcurrentHashMap` 线程安全
- `ConcurrentHashMap` 使用 CAS + synchronized，并发度更高

---

**2. synchronized 和 ReentrantLock 的区别？**

- `synchronized` 简单，自动释放，`ReentrantLock` 需手动释放
- `ReentrantLock` 支持可中断、公平锁、多个条件变量

---

**3. Spring Bean 的作用域？**

- `singleton`：单例（默认）
- `prototype`：每次请求创建新实例
- `request`：每个 HTTP 请求一个实例
- `session`：每个 Session 一个实例

---

**4. Spring 如何解决循环依赖？**

- **三级缓存**：
  - 一级缓存：完整的 Bean
  - 二级缓存：早期的 Bean（未初始化完成）
  - 三级缓存：Bean 工厂
- 构造器循环依赖无法解决（抛出异常）

---

**5. MyBatis 的一级缓存和二级缓存？**

- **一级缓存**：SqlSession 级别，默认开启
- **二级缓存**：Mapper 级别，需手动开启

---

## 实战案例

### 金融支付：分布式锁

```java
@Service
public class PaymentService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean deduct(Long userId, BigDecimal amount) {
        String lockKey = "lock:user:" + userId;
        String lockValue = UUID.randomUUID().toString();
        
        // 尝试获取锁（NX：不存在才设置，EX：过期时间）
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, 5, TimeUnit.SECONDS);
        
        if (Boolean.FALSE.equals(success)) {
            throw new RuntimeException("并发冲突，请稍后重试");
        }
        
        try {
            // 业务逻辑：扣款
            Account account = accountRepository.findById(userId).orElseThrow();
            if (account.getBalance().compareTo(amount) < 0) {
                throw new RuntimeException("余额不足");
            }
            
            account.setBalance(account.getBalance().subtract(amount));
            accountRepository.save(account);
            
            return true;
        } finally {
            // 释放锁（Lua 脚本保证原子性）
            String script = 
                "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "    return redis.call('del', KEYS[1]) " +
                "else " +
                "    return 0 " +
                "end";
            
            redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                Collections.singletonList(lockKey),
                lockValue
            );
        }
    }
}
```

---

### 游戏后台：线程池异步处理

```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean("taskExecutor")
    public ThreadPoolExecutor taskExecutor() {
        return new ThreadPoolExecutor(
            4,                           // 核心线程数
            8,                           // 最大线程数
            60, TimeUnit.SECONDS,        // 空闲线程存活时间
            new LinkedBlockingQueue<>(100), // 队列容量
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
        );
    }
}

@Service
public class RewardService {
    
    @Autowired
    @Qualifier("taskExecutor")
    private ThreadPoolExecutor taskExecutor;
    
    public void sendReward(Long userId, int rewardId) {
        taskExecutor.execute(() -> {
            try {
                // 发放奖励（耗时操作）
                Reward reward = rewardRepository.findById(rewardId).orElseThrow();
                userRewardRepository.save(new UserReward(userId, reward));
                
                // 发送通知
                notificationService.send(userId, "您获得了奖励：" + reward.getName());
            } catch (Exception e) {
                log.error("发放奖励失败: userId={}, rewardId={}", userId, rewardId, e);
            }
        });
    }
}
```

---

**总结**

Java 面试重点：
- **核心特性**：面向对象、集合框架（HashMap、ConcurrentHashMap）
- **JVM**：内存结构、GC 机制、类加载、性能调优
- **并发编程**：synchronized、Lock、线程池、CompletableFuture
- **Spring**：IoC、AOP、事务管理
- **MyBatis**：映射、动态 SQL、性能优化
- **实战经验**：结合金融/游戏场景，展示你的工程能力

面试官最看重：**你是否理解底层原理，能否解决实际问题**。💪

---

## 📝 面试题自查

### Q1：Java 泛型的类型擦除是什么？它带来了哪些限制？

**答**：Java 泛型是**编译期特性**，编译后泛型信息被擦除，`List<String>` 和 `List<Integer>` 在运行时都是 `List`（类型参数替换为 Object 或上界）。编译器在调用处自动插入强制转换。限制包括：不能使用基本类型作为泛型参数（只能用包装类）、不能创建泛型数组（`new T[]`）、不能用 `instanceof` 检查泛型类型、不能重载仅泛型参数不同的方法（擦除后签名相同）。

---

### Q2：什么是 PECS 原则？`<? extends T>` 和 `<? super T>` 的区别？

**答**：PECS = Producer Extends, Consumer Super。`<? extends T>` 表示 T 或其子类型，只能安全**读取**为 T（生产者），不能写入；`<? super T>` 表示 T 或其父类型，可以安全**写入** T 类型（消费者），但读取只能是 Object。例如 `Collections.copy(List<? super T> dest, List<? extends T> src)` 中 src 用 extends 读取，dest 用 super 写入。

---

### Q3：String 为什么设计成不可变的？`new String("hello")` 创建了几个对象？

**答**：不可变的好处：① **线程安全**，多线程共享无需加锁；② **hashCode 缓存**，只计算一次，适合做 HashMap 的 key；③ **字符串常量池复用**，节省内存；④ **安全性**，类加载器等使用 String 作为参数不会被篡改。`new String("hello")` 创建 1 或 2 个对象：如果常量池中已有 `"hello"` 则只在堆上创建 1 个；如果没有则先在常量池创建，再在堆上创建。

---

### Q4：JVM 内存结构有哪些区域？哪些是线程私有的？

**答**：① **程序计数器**（线程私有）——当前线程执行的字节码行号；② **虚拟机栈**（线程私有）——方法调用栈帧（局部变量表、操作数栈、返回地址）；③ **本地方法栈**（线程私有）——native 方法调用；④ **堆**（线程共享）——对象实例和数组，GC 主要区域，分为新生代（Eden+Survivor）和老年代；⑤ **方法区/Metaspace**（线程共享）——类信息、常量池、静态变量，Java 8+ 使用本地内存实现。

---

### Q5：对象在堆中是如何分配和晋升的？什么情况会触发 Minor GC 和 Full GC？

**答**：新对象在 **Eden 区**分配 → Eden 满时触发 **Minor GC** → 存活对象复制到 Survivor（From/To 交替）→ 对象年龄达到阈值（默认15）**晋升**到老年代；大对象直接进入老年代。**Full GC** 触发条件：老年代空间不足、Metaspace 不足、`System.gc()` 显式调用、CMS Concurrent Mode Failure 等。

---

### Q6：CMS 和 G1 收集器的区别？CMS 有哪些问题？G1 如何解决？

**答**：**CMS** 目标是低延迟，使用"标记-清除"算法，4 个阶段（初始标记 STW→并发标记→重新标记 STW→并发清除）。三大问题：① 内存碎片（标记-清除不整理）；② Concurrent Mode Failure（并发清除时老年代满，降级为 Serial Old）；③ 浮动垃圾。**G1** 将堆分为多个 Region，使用"标记-复制"算法解决碎片问题；优先回收垃圾最多的 Region（Garbage First），可预测停顿时间（`-XX:MaxGCPauseMillis`）；Mixed GC 增量回收老年代减少 Full GC。

---

### Q7：ZGC 的核心技术是什么？为什么能做到 10ms 以内的停顿？

**答**：ZGC 两大核心技术：① **染色指针**（Colored Pointers）——在 64 位指针中嵌入标记位（Marked0/Marked1/Remapped/Finalizable），无需访问对象就能知道状态；② **读屏障**（Load Barrier）——对象移动后通过读屏障在首次访问时修正指针，实现并发转移。ZGC 几乎所有阶段（并发标记、并发转移）都是并发的，STW 仅限于初始标记和再标记（各 < 1ms），与堆大小无关。

---

### Q8：什么是 JMM（Java 内存模型）？happens-before 规则是什么？

**答**：JMM 定义了多线程环境下共享变量的可见性、有序性和原子性规则。线程有自己的工作内存（CPU 缓存），共享变量在主内存中。**happens-before** 保证：如果 A happens-before B，则 A 的结果对 B 可见。核心规则包括：程序顺序规则（同一线程内前后有序）、volatile 规则（volatile 写 hb 后续读）、锁规则（unlock hb 后续 lock）、线程启动/终止规则、传递性等。

---

### Q9：volatile 的作用？为什么双重检查锁的单例模式需要 volatile？

**答**：volatile 保证 **可见性**（一个线程修改后其他线程立即可见）和**禁止指令重排序**。双重检查锁中 `instance = new Singleton()` 分三步：① 分配内存 ② 初始化对象 ③ 将 instance 指向内存。指令重排可能导致 ② 和 ③ 交换，其他线程在第一次 null 检查时看到未初始化的对象。volatile 通过内存屏障禁止重排序，保证 ② 在 ③ 之前。

---

### Q10：synchronized 的锁升级过程？偏向锁、轻量级锁、重量级锁有什么区别？

**答**：Java 6 引入锁升级：**无锁→偏向锁→轻量级锁→重量级锁**，只能升级不能降级。**偏向锁**：单线程重复获取锁，CAS 写入线程 ID 到 Mark Word，后续只检查 ID，几乎零开销。**轻量级锁**：线程交替获取锁（低竞争），CAS 将 Mark Word 复制到栈帧 Lock Record，失败则自旋等待。**重量级锁**：激烈竞争，使用 OS 互斥量，线程阻塞/唤醒需要内核态切换，开销最大但避免 CPU 空转。

---

### Q11：线程池的核心参数有哪些？任务提交后的执行流程是什么？

**答**：7 个核心参数：corePoolSize（核心线程数）、maximumPoolSize（最大线程数）、keepAliveTime（空闲线程存活时间）、unit（时间单位）、workQueue（任务队列）、threadFactory（线程工厂）、handler（拒绝策略）。执行流程：① 线程数 < core → 创建新线程；② 线程数 ≥ core → 入队列；③ 队列满且线程数 < max → 创建新线程；④ 线程数 ≥ max → 执行拒绝策略（AbortPolicy/CallerRunsPolicy/DiscardPolicy/DiscardOldestPolicy）。

---

### Q12：ReentrantLock 和 synchronized 的区别？什么场景用 ReentrantLock？

**答**：**synchronized**：自动获取释放、不可中断、非公平锁、只有一个条件变量（wait/notify）。**ReentrantLock**：手动释放（必须 finally）、可中断（`lockInterruptibly()`）、可配置公平锁、支持多个 Condition、支持 `tryLock()` 非阻塞。使用场景：需要**可中断锁**（避免死锁）、**公平锁**（按等待顺序获取）、**多条件变量**（生产者-消费者不同条件）、**尝试获取锁**（超时机制）时用 ReentrantLock。

---

### Q13：CompletableFuture 和 Future 的区别？如何实现异步编排？

**答**：**Future** 只能通过 `get()` 阻塞获取结果，不支持回调和链式调用。**CompletableFuture** 支持：① 回调（`thenAccept`/`thenApply`/`thenRun`）；② 链式编排（`thenCompose`/`thenCombine`）；③ 异常处理（`exceptionally`/`handle`）；④ 组合多个任务（`allOf`/`anyOf`）。异步编排：`supplyAsync(() -> fetchUser()).thenCompose(user -> supplyAsync(() -> fetchOrders(user))).thenAccept(orders -> process(orders))`。

---

### Q14：Spring IoC 容器的初始化流程？Bean 的生命周期是什么？

**答**：初始化流程：读取配置/注解 → BeanDefinition 注册 → BeanFactoryPostProcessor 处理 → Bean 实例化 → 属性注入 → 初始化。**Bean 生命周期**：① 实例化（构造器）→ ② 属性注入（`@Autowired`）→ ③ BeanNameAware/BeanFactoryAware → ④ `@PostConstruct` / `InitializingBean.afterPropertiesSet()` → ⑤ 使用 → ⑥ `@PreDestroy` / `DisposableBean.destroy()` → ⑦ 销毁。

---

### Q15：Spring 如何解决循环依赖？构造器注入的循环依赖为什么无法解决？

**答**：Spring 使用**三级缓存**解决 Setter/字段注入的循环依赖：一级缓存（完整 Bean）、二级缓存（早期引用，半成品 Bean）、三级缓存（ObjectFactory，用于生成代理对象）。流程：A 实例化 → 将 A 的 ObjectFactory 放入三级缓存 → 注入 B → B 实例化 → B 注入 A → 从三级缓存获取 A 的早期引用 → B 初始化完成 → A 注入 B → A 初始化完成。**构造器注入无法解决**因为实例化本身就依赖对方，形成死锁。

---

### Q16：Spring AOP 的实现原理？JDK 动态代理和 CGLIB 的区别？

**答**：Spring AOP 基于**代理模式**实现。两种代理方式：① **JDK 动态代理**——基于接口，通过 `Proxy.newProxyInstance()` 创建代理对象，被代理类必须实现接口；② **CGLIB**——基于继承，通过生成目标类的子类来实现代理，不需要接口。Spring 默认策略：有接口用 JDK 动态代理，无接口用 CGLIB。Spring Boot 2.x 默认使用 CGLIB（`spring.aop.proxy-target-class=true`）。

---

### Q17：`@Transactional` 在什么情况下会失效？

**答**：常见失效场景：① **自调用**——同一类中方法 A 调用方法 B（B 有 @Transactional），因为不走代理；② **非 public 方法**——Spring AOP 只拦截 public 方法；③ **异常类型不匹配**——默认只回滚 RuntimeException，checked exception 不回滚（需指定 `rollbackFor`）；④ **try-catch 吞掉异常**——异常被捕获不抛出，事务不会回滚；⑤ **多线程**——新线程不在事务上下文中；⑥ **数据库不支持事务**（如 MyISAM）。

---

### Q18：HashMap 的 put 流程？为什么链表长度达到 8 时转红黑树？

**答**：put 流程：① 计算 key 的 hash（`(h = key.hashCode()) ^ (h >>> 16)` 高低位混合）→ ② `hash & (length-1)` 定位桶 → ③ 桶为空直接插入 → ④ 桶不空：key 相同则覆盖，不同则尾插链表/红黑树 → ⑤ `size > threshold` 则 2 倍扩容。**链表转红黑树**的阈值 8 基于泊松分布计算：在良好 hash 分布下，链表长度达到 8 的概率仅为千万分之六。红黑树查找 O(log n) 优于链表 O(n)，但节点数少时链表更轻量。节点 ≤ 6 时退化回链表（保留缓冲区避免频繁转换）。

---

### Q19：类加载器和双亲委派模型是什么？什么时候需要打破双亲委派？

**答**：类加载器层次：Bootstrap（加载 rt.jar）→ Extension → Application → Custom。**双亲委派**：收到加载请求先委派给父加载器，父无法加载才自己加载。好处：避免类重复加载、保护核心 API。**打破场景**：① **SPI 机制**（`ServiceLoader`）——Bootstrap 加载的接口需要加载 classpath 上的实现类，使用线程上下文类加载器；② **热部署**——如 Tomcat 每个 WebApp 用独立类加载器，实现类隔离和热替换；③ **OSGi** 模块化加载。

---

### Q20：如何排查线上 Java 应用的 Full GC 频繁问题？给出排查步骤。

**答**：排查步骤：① **看 GC 日志**（`-Xlog:gc*`）确认 Full GC 频率和原因（Allocation Failure / Metadata / System.gc）；② **`jstat -gcutil <pid>`** 查看各代使用率，确认是老年代满还是 Metaspace 满；③ **`jmap -histo <pid>`** 查看对象统计，找出占用内存最多的类；④ **dump 堆**（`jmap -dump:format=b,file=heap.hprof`），用 MAT/VisualVM 分析大对象和泄漏；⑤ **代码排查**：检查大集合/缓存未淘汰、大对象频繁创建、类加载器泄漏等；⑥ **调优**：增大堆、调整 NewRatio、更换 GC（G1/ZGC）、优化代码。
