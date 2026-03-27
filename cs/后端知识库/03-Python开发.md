# Python 开发深入

---

## 📑 目录

### 语言机制
- [Python核心特性](#python核心特性)
- [内存管理与垃圾回收](#内存管理与垃圾回收)
- [闭包与作用域](#闭包与作用域)
- [迭代器与生成器](#迭代器与生成器)
- [GIL与并发模型](#gil与并发模型)
- [装饰器与元编程](#装饰器与元编程)

### 异步与框架
- [asyncio协程](#asyncio协程)
- [Django ORM优化](#django-orm优化)
- [FastAPI实战](#fastapi实战)

### 工程实践与自查
- [性能优化与调试](#性能优化与调试)
- [实战案例](#实战案例)
- [面试题自查](#面试题自查)

---

## Python核心特性

### 数据模型与魔法方法

**1. 对象模型**

Python 中一切皆对象，对象由三部分组成：
- **id**：对象的内存地址（`id(obj)`）
- **type**：对象的类型（`type(obj)`）
- **value**：对象的值

```python
a = 42
print(id(a))      # 内存地址
print(type(a))    # <class 'int'>
print(a)          # 42

# Python 中小整数对象池（-5 到 256）
b = 42
print(a is b)     # True（同一个对象）

c = 1000
d = 1000
print(c is d)     # False（不同对象）
```

---

**2. 常用魔法方法**

**对象创建与销毁**：
```python
class User:
    def __new__(cls, *args, **kwargs):
        """创建对象（在 __init__ 之前调用）"""
        print("__new__ called")
        instance = super().__new__(cls)
        return instance
    
    def __init__(self, name):
        """初始化对象"""
        print("__init__ called")
        self.name = name
    
    def __del__(self):
        """对象销毁时调用（不保证一定执行）"""
        print(f"{self.name} deleted")

user = User("Alice")
# 输出：
# __new__ called
# __init__ called
```

---

**字符串表示**：
```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __str__(self):
        """str(obj) 或 print(obj) 时调用（用户友好）"""
        return f"Point({self.x}, {self.y})"
    
    def __repr__(self):
        """repr(obj) 时调用（开发者友好，应该是合法的 Python 表达式）"""
        return f"Point(x={self.x}, y={self.y})"

p = Point(1, 2)
print(str(p))   # Point(1, 2)
print(repr(p))  # Point(x=1, y=2)
```

---

**容器类型**：
```python
class MyList:
    def __init__(self):
        self.data = []
    
    def __len__(self):
        """len(obj) 时调用"""
        return len(self.data)
    
    def __getitem__(self, index):
        """obj[index] 时调用"""
        return self.data[index]
    
    def __setitem__(self, index, value):
        """obj[index] = value 时调用"""
        self.data[index] = value
    
    def __delitem__(self, index):
        """del obj[index] 时调用"""
        del self.data[index]
    
    def __contains__(self, item):
        """item in obj 时调用"""
        return item in self.data
    
    def __iter__(self):
        """for item in obj 时调用"""
        return iter(self.data)

mylist = MyList()
mylist.data = [1, 2, 3]
print(len(mylist))        # 3
print(mylist[0])          # 1
print(2 in mylist)        # True
for item in mylist:
    print(item)
```

---

**上下文管理器**：
```python
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """with 语句进入时调用"""
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """with 语句退出时调用（即使发生异常）"""
        if self.file:
            self.file.close()
        # 返回 True 表示抑制异常，False 表示继续传播异常
        return False

with FileManager('test.txt', 'w') as f:
    f.write('Hello')
# 自动关闭文件
```

---

### 可变与不可变对象

**不可变对象**（Immutable）：
- `int`, `float`, `str`, `tuple`, `frozenset`
- 修改会创建新对象

```python
a = "hello"
b = a
a = a + " world"  # 创建新对象
print(b)  # "hello"（b 不变）
```

**可变对象**（Mutable）：
- `list`, `dict`, `set`, 自定义类
- 修改会影响原对象

```python
a = [1, 2, 3]
b = a
a.append(4)  # 修改原对象
print(b)  # [1, 2, 3, 4]（b 也变了）

# 避免问题：使用拷贝
import copy
b = a.copy()         # 浅拷贝
c = copy.deepcopy(a) # 深拷贝
```

---

### 函数参数传递

Python 使用**对象引用传递**（类似传指针）：
- 不可变对象：传递后修改不影响原对象（看起来像值传递）
- 可变对象：传递后修改会影响原对象（看起来像引用传递）

```python
def modify_immutable(x):
    x = x + 1  # 创建新对象，不影响原对象

def modify_mutable(lst):
    lst.append(4)  # 修改原对象

a = 10
modify_immutable(a)
print(a)  # 10（不变）

b = [1, 2, 3]
modify_mutable(b)
print(b)  # [1, 2, 3, 4]（改变）
```

**默认参数陷阱**：

```python
# ❌ 错误：默认参数是可变对象
def append_to(element, target=[]):
    target.append(element)
    return target

print(append_to(1))  # [1]
print(append_to(2))  # [1, 2]（共享同一个列表）

# ✅ 正确：使用 None 作为默认值
def append_to(element, target=None):
    if target is None:
        target = []
    target.append(element)
    return target

print(append_to(1))  # [1]
print(append_to(2))  # [2]（独立的列表）
```

---

## 内存管理与垃圾回收

### 引用计数（Reference Counting）

CPython 使用**引用计数**作为主要的内存管理机制。每个 Python 对象内部都维护一个 `ob_refcnt` 计数器，记录有多少个引用指向它。

```python
import sys

a = []           # 创建列表对象，refcount = 1
b = a            # refcount = 2
c = [a, a]       # refcount = 4（列表c本身持有2个引用）

print(sys.getrefcount(a))  # 5（+1 是因为 getrefcount 的参数也持有引用）

del b            # refcount - 1
c.pop()          # refcount - 1
```

**引用计数变化规则**：

| 操作 | 引用计数变化 |
|------|------------|
| `x = obj` 赋值 | +1 |
| `del x` 删除引用 | -1 |
| `func(obj)` 作为参数传入 | 函数执行期间 +1 |
| 容器（list/dict/set）添加 | +1 |
| 容器移除或容器被销毁 | -1 |
| 变量离开作用域 | -1 |

**优点**：

- **实时性**：引用计数归零时立即回收，不需要等待 GC 周期
- **确定性**：对象的销毁时机可预测（`__del__` 立即调用）
- **分摊开销**：回收工作分散在每次引用变化时，不会造成长时间停顿

**缺点**：

- **循环引用无法回收**：两个对象互相引用时，引用计数永远不会归零
- **线程安全开销**：多线程下需要保护 `ob_refcnt`（这正是 GIL 存在的根本原因）
- **空间开销**：每个对象额外占用 8 字节（64 位系统）

---

### 循环引用与标记-清除（Mark and Sweep）

引用计数无法处理**循环引用**问题：

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

# 创建循环引用
a = Node(1)
b = Node(2)
a.next = b    # a → b
b.next = a    # b → a（循环引用！）

del a         # a 的 refcount = 1（b.next 仍指向它）
del b         # b 的 refcount = 1（a.next 仍指向它）

# 此时 a 和 b 都不可达，但引用计数不为 0，无法回收 → 内存泄漏
```

**标记-清除算法**解决循环引用：

1. **标记阶段**：从根对象（全局变量、栈变量、寄存器）出发，遍历所有可达对象，标记为"存活"
2. **清除阶段**：遍历所有容器对象，未标记的对象即为垃圾，予以回收

```
根对象集合 → 遍历引用关系 → 标记所有可达对象
                                ↓
                    未标记的对象 = 不可达 = 垃圾 → 回收
```

> **重要细节**：标记-清除只会扫描**容器对象**（list、dict、set、自定义类实例等），因为只有容器才可能产生循环引用。`int`、`str`、`float` 等不可变原子类型不会被扫描。

---

### 分代回收（Generational GC）

CPython 基于"弱代假说"（大多数对象生命周期很短）实现了**三代**垃圾回收器，以减少标记-清除的扫描范围：

| 代 | 触发条件 | 说明 |
|---|---------|------|
| **第 0 代（young）** | 新分配对象数 - 释放对象数 > 700 | 新创建的对象进入第 0 代，扫描频率最高 |
| **第 1 代（middle）** | 第 0 代 GC 执行 10 次后触发 | 在第 0 代中存活下来的对象晋升到第 1 代 |
| **第 2 代（old）** | 第 1 代 GC 执行 10 次后触发 | 长期存活的对象，扫描频率最低 |

```python
import gc

# 查看分代回收阈值
print(gc.get_threshold())  # (700, 10, 10)

# 手动设置阈值
gc.set_threshold(800, 15, 15)

# 查看每代中的对象数量
print(gc.get_count())  # (288, 3, 0)

# 手动触发 GC
gc.collect()           # 全代回收
gc.collect(0)          # 只回收第 0 代
```

**分代回收的工作流程**：

```
新对象 → 第 0 代
            │
            ├── 被回收（不可达）
            │
            └── 存活 → 晋升到第 1 代
                            │
                            ├── 被回收
                            │
                            └── 存活 → 晋升到第 2 代
                                            │
                                            ├── 被回收
                                            │
                                            └── 长期存活
```

---

### PyMalloc 内存池

CPython 使用三层内存分配架构，对小对象进行高效管理：

```
层级3：应用层 —— Python 对象分配（list、dict、int...）
层级2：PyMalloc —— 小对象内存池（≤ 512 字节）
层级1：系统层 —— malloc/free / mmap
层级0：操作系统 —— 虚拟内存管理
```

**PyMalloc 内存池层次结构**：

| 概念 | 大小 | 说明 |
|------|------|------|
| **Arena** | 256 KB | 从操作系统申请的大块内存 |
| **Pool** | 4 KB（一个内存页） | Arena 中划分出来的，每个 Pool 管理固定大小的 Block |
| **Block** | 8~512 字节 | 实际分配给 Python 对象的最小单元，按 8 字节对齐 |

```python
# 小对象（≤ 512 字节）→ PyMalloc 内存池分配（快）
a = 42           # int 对象约 28 字节
b = "hello"      # str 对象约 54 字节

# 大对象（> 512 字节）→ 直接调用系统 malloc（慢）
c = bytearray(1024)
```

> **面试要点**：PyMalloc 释放的内存不会立即归还给操作系统，而是留在内存池中复用。这就是为什么 Python 进程的内存占用只增不减（RSS 不降），需要多进程模型来解决长期运行的内存膨胀问题。

---

### gc 模块常用操作

```python
import gc

# 1. 禁用/启用自动 GC
gc.disable()    # 高性能场景下可禁用，手动控制回收时机
gc.enable()

# 2. 查找循环引用（调试利器）
gc.set_debug(gc.DEBUG_SAVEALL)  # 将不可达对象保存到 gc.garbage
gc.collect()
print(gc.garbage)  # 查看无法回收的循环引用对象

# 3. 弱引用（避免循环引用）
import weakref

class Cache:
    pass

obj = Cache()
ref = weakref.ref(obj)    # 弱引用不增加引用计数
print(ref())              # <Cache object>
del obj
print(ref())              # None（对象已被回收）

# 4. WeakValueDictionary（缓存场景常用）
cache = weakref.WeakValueDictionary()
obj = Cache()
cache['key'] = obj        # 弱引用
del obj                   # 对象被回收，cache['key'] 自动消失
```

---

## 闭包与作用域

### LEGB 作用域规则

Python 变量查找遵循 **LEGB** 规则，按以下顺序逐层查找：

| 层级 | 名称 | 说明 | 示例 |
|------|------|------|------|
| **L** | Local | 函数内部局部变量 | 函数体内定义的变量 |
| **E** | Enclosing | 外层嵌套函数的局部变量 | 闭包场景 |
| **G** | Global | 模块级全局变量 | 模块顶层定义的变量 |
| **B** | Built-in | Python 内置名称 | `print`, `len`, `range` 等 |

```python
x = "global"                    # G：全局

def outer():
    x = "enclosing"             # E：外层函数
    
    def inner():
        x = "local"             # L：内层函数
        print(x)                # → "local"（L 层找到）
    
    inner()
    print(x)                    # → "enclosing"（E 层）

outer()
print(x)                       # → "global"（G 层）
print(len)                     # → <built-in function len>（B 层）
```

**变量查找的常见陷阱**：

```python
x = 10

def foo():
    print(x)     # ❌ UnboundLocalError!
    x = 20       # 因为函数内有赋值，Python 在编译时就认为 x 是局部变量

# Python 的变量作用域在编译时确定，不是运行时！
# 函数内只要有对 x 的赋值，整个函数中 x 都被视为局部变量
```

---

### global 与 nonlocal 关键字

```python
count = 0

def increment():
    global count       # 声明使用全局变量
    count += 1

increment()
print(count)           # 1

# -------

def outer():
    count = 0
    
    def inner():
        nonlocal count  # 声明使用外层函数的变量（E 层）
        count += 1
    
    inner()
    inner()
    print(count)        # 2

outer()
```

**`global` vs `nonlocal` 区别**：

| 关键字 | 作用 | 使用场景 |
|--------|------|---------|
| `global` | 声明变量来自模块全局作用域（G 层） | 函数内修改全局变量 |
| `nonlocal` | 声明变量来自最近的外层函数作用域（E 层） | 闭包内修改外层函数变量 |

---

### 闭包（Closure）

**闭包**是一个函数对象，它记住了定义时所在作用域中的变量，即使外层函数已经执行完毕。

**闭包的形成条件**：
1. 存在嵌套函数
2. 内层函数引用了外层函数的变量
3. 外层函数返回内层函数

```python
def make_multiplier(factor):
    # factor 是外层函数的局部变量
    def multiplier(x):
        return x * factor   # 引用外层变量 factor
    return multiplier        # 返回内层函数

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))   # 10（factor=2 被"记住"了）
print(triple(5))   # 15（factor=3 被"记住"了）

# 查看闭包捕获的变量
print(double.__closure__)         # (<cell object>,)
print(double.__closure__[0].cell_contents)  # 2
```

**闭包的底层实现**：

CPython 通过 **cell 对象**实现闭包。当编译器检测到内层函数引用了外层变量时：
- 外层函数将该变量存储在 cell 对象中（而非栈帧上）
- 内层函数通过 `__closure__` 元组持有对 cell 对象的引用
- 即使外层函数的栈帧已销毁，cell 对象仍然存活

```python
import dis

def outer():
    x = 10
    def inner():
        return x
    return inner

# 查看字节码
dis.dis(outer)
# LOAD_CLOSURE   x      ← 创建 cell 对象
# MAKE_FUNCTION          ← 将 cell 传给内层函数

dis.dis(outer())
# LOAD_DEREF     x      ← 从 cell 对象中读取值
```

---

### 闭包经典陷阱：循环变量捕获

```python
# ❌ 经典错误：闭包捕获的是变量本身，不是值
funcs = []
for i in range(5):
    funcs.append(lambda: i)

print([f() for f in funcs])  # [4, 4, 4, 4, 4]（全是4！）
# 因为 lambda 捕获的是变量 i，循环结束后 i=4

# ✅ 修复方法1：使用默认参数（在定义时绑定值）
funcs = []
for i in range(5):
    funcs.append(lambda i=i: i)

print([f() for f in funcs])  # [0, 1, 2, 3, 4]

# ✅ 修复方法2：使用 functools.partial
from functools import partial

funcs = [partial(lambda x: x, i) for i in range(5)]
print([f() for f in funcs])  # [0, 1, 2, 3, 4]

# ✅ 修复方法3：使用闭包工厂
def make_func(i):
    return lambda: i

funcs = [make_func(i) for i in range(5)]
print([f() for f in funcs])  # [0, 1, 2, 3, 4]
```

---

### 闭包实战应用

**1. 计数器**：
```python
def make_counter(start=0):
    count = start
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

c = make_counter()
print(c(), c(), c())  # 1, 2, 3
```

**2. 带状态的装饰器**：
```python
def rate_limiter(max_calls, period):
    """基于闭包的限流器"""
    calls = []
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            # 清除过期记录
            calls[:] = [t for t in calls if now - t < period]
            if len(calls) >= max_calls:
                raise Exception("Rate limit exceeded")
            calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limiter(max_calls=5, period=60)
def api_call():
    pass
```

---

## 迭代器与生成器

### 迭代器协议

Python 的迭代器协议由两个魔法方法定义：

- `__iter__()`：返回迭代器对象本身
- `__next__()`：返回下一个值，没有更多值时抛出 `StopIteration`

```python
class Countdown:
    """自定义迭代器：倒计时"""
    def __init__(self, start):
        self.current = start
    
    def __iter__(self):
        return self  # 返回自身（迭代器本身也是可迭代对象）
    
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

# 使用
for n in Countdown(5):
    print(n)  # 5, 4, 3, 2, 1

# 手动迭代
it = Countdown(3)
print(next(it))  # 3
print(next(it))  # 2
print(next(it))  # 1
# next(it)       # StopIteration
```

**可迭代对象 vs 迭代器**：

| 概念 | 需要实现 | 说明 |
|------|---------|------|
| **可迭代对象（Iterable）** | `__iter__()` | 可以被 `for` 循环遍历，如 list、dict、str |
| **迭代器（Iterator）** | `__iter__()` + `__next__()` | 有状态的对象，记住遍历位置 |

```python
# list 是可迭代对象，不是迭代器
lst = [1, 2, 3]
hasattr(lst, '__iter__')    # True
hasattr(lst, '__next__')    # False

# iter() 将可迭代对象转换为迭代器
it = iter(lst)
hasattr(it, '__next__')     # True
```

> **关键区别**：可迭代对象可以多次遍历（每次调用 `__iter__()` 返回新的迭代器），迭代器只能遍历一次（遍历完后就耗尽了）。

---

### 生成器（Generator）

生成器是一种特殊的迭代器，使用 `yield` 关键字定义，**自动实现**迭代器协议。

```python
def countdown(n):
    """生成器函数"""
    while n > 0:
        yield n      # 暂停执行，返回值
        n -= 1       # 下次调用 next() 时从这里继续

gen = countdown(5)
print(type(gen))       # <class 'generator'>
print(next(gen))       # 5
print(next(gen))       # 4

# 生成器也可以用 for 遍历
for n in countdown(3):
    print(n)           # 3, 2, 1
```

**生成器的执行流程**：

```
调用 countdown(5)  → 不执行函数体，返回生成器对象
第1次 next()       → 执行到第1个 yield，返回 5，暂停
第2次 next()       → 从上次暂停处继续，执行到 yield，返回 4，暂停
...
第5次 next()       → 执行到 yield，返回 1，暂停
第6次 next()       → while 条件不满足，函数返回 → StopIteration
```

**生成器表达式**（类似列表推导式）：

```python
# 列表推导式：立即创建完整列表（占用内存）
squares_list = [x**2 for x in range(1000000)]   # ~8MB 内存

# 生成器表达式：惰性求值（几乎不占内存）
squares_gen = (x**2 for x in range(1000000))     # 仅占约 120 字节

# 内存对比
import sys
print(sys.getsizeof(squares_list))  # 8448728
print(sys.getsizeof(squares_gen))   # 112
```

---

### send()、throw() 与 close()

生成器不仅可以产出值，还可以**接收**外部传入的值和异常。

**send()：向生成器发送值**

```python
def accumulator():
    """累加器：通过 send() 接收值"""
    total = 0
    while True:
        value = yield total    # yield 既产出值，也接收 send() 的值
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)             # 启动生成器（必须先 next 或 send(None)），返回 0
print(gen.send(10))   # 发送 10，返回 10
print(gen.send(20))   # 发送 20，返回 30
print(gen.send(30))   # 发送 30，返回 60
```

**throw()：向生成器抛入异常**

```python
def safe_generator():
    try:
        while True:
            yield "working"
    except ValueError:
        yield "error handled"
    finally:
        print("cleanup")

gen = safe_generator()
print(next(gen))                          # "working"
print(gen.throw(ValueError, "oops"))      # "error handled"
```

**close()：关闭生成器**

```python
def infinite():
    try:
        n = 0
        while True:
            yield n
            n += 1
    except GeneratorExit:
        print("generator closed")  # close() 触发 GeneratorExit

gen = infinite()
print(next(gen))  # 0
print(next(gen))  # 1
gen.close()       # "generator closed"
```

---

### yield from（委托生成器）

`yield from` 用于在一个生成器中委托另一个可迭代对象或子生成器，简化嵌套生成器的编写。

```python
# 不使用 yield from
def chain_manual(*iterables):
    for it in iterables:
        for item in it:
            yield item

# 使用 yield from（等价但更简洁）
def chain(*iterables):
    for it in iterables:
        yield from it

list(chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]
```

**`yield from` 与子生成器的双向通道**：

```python
def sub_generator():
    """子生成器"""
    total = 0
    while True:
        value = yield total
        if value is None:
            return total      # return 值会成为 yield from 的返回值
        total += value

def delegator():
    """委托生成器"""
    result = yield from sub_generator()  # 委托给子生成器
    print(f"Sub-generator returned: {result}")

gen = delegator()
next(gen)            # 启动，进入子生成器
gen.send(10)         # 发送给子生成器，返回 10
gen.send(20)         # 发送给子生成器，返回 30
try:
    gen.send(None)   # 子生成器 return，触发 StopIteration
except StopIteration:
    pass
# 输出：Sub-generator returned: 30
```

> **`yield from` 的核心价值**：它自动处理了 `send()`、`throw()`、`close()` 的转发，建立了调用方与子生成器之间的**透明双向通道**。这是 `asyncio` 协程的基础（`await` 本质上就是 `yield from`）。

---

### 生成器实战应用

**1. 大文件逐行处理（内存友好）**：

```python
def read_large_file(filepath, chunk_size=8192):
    """逐块读取大文件"""
    with open(filepath, 'r') as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk

# 处理 10GB 文件也不会内存溢出
for chunk in read_large_file('huge_log.txt'):
    process(chunk)
```

**2. 生成器管道（Pipeline）**：

```python
def read_lines(filepath):
    with open(filepath) as f:
        for line in f:
            yield line.strip()

def filter_comments(lines):
    for line in lines:
        if not line.startswith('#'):
            yield line

def parse_json(lines):
    import json
    for line in lines:
        yield json.loads(line)

def filter_by_status(records, status):
    for record in records:
        if record.get('status') == status:
            yield record

# 组装管道：每一步都是惰性的，内存占用极低
pipeline = filter_by_status(
    parse_json(
        filter_comments(
            read_lines('data.jsonl')
        )
    ),
    status='error'
)

for record in pipeline:
    print(record)
```

**3. 协程式状态机**：

```python
def tcp_state_machine():
    """TCP 连接状态机"""
    state = "CLOSED"
    while True:
        event = yield state
        if state == "CLOSED" and event == "connect":
            state = "SYN_SENT"
        elif state == "SYN_SENT" and event == "syn_ack":
            state = "ESTABLISHED"
        elif state == "ESTABLISHED" and event == "close":
            state = "FIN_WAIT"
        elif state == "FIN_WAIT" and event == "ack":
            state = "CLOSED"

sm = tcp_state_machine()
next(sm)                    # "CLOSED"
print(sm.send("connect"))   # "SYN_SENT"
print(sm.send("syn_ack"))   # "ESTABLISHED"
print(sm.send("close"))     # "FIN_WAIT"
print(sm.send("ack"))       # "CLOSED"
```

---

## GIL与并发模型

### 全局解释器锁（GIL）

**定义**：CPython 中的全局互斥锁，确保同一时间只有一个线程执行 Python 字节码。

---

#### CPython 与 Python 的关系⭐⭐

> **核心概念**：Python 是一门语言规范，CPython 是它的默认（官方）实现。GIL 是 CPython 的实现细节，**不是 Python 语言本身的特性**。

**Python（语言规范）**：
- 由 [Python 语言参考](https://docs.python.org/3/reference/) 定义的语法、语义规则
- 规定了 `if`、`for`、`class` 等语法该如何表现
- **不要求也不禁止 GIL**——语言规范层面没有提到 GIL

**CPython（默认实现）**：
- 用 **C 语言**编写的 Python 解释器，由 Python 核心团队维护
- 当你执行 `python3 xxx.py` 时，绝大多数情况下用的就是 CPython
- **GIL 存在于 CPython 中**，是因为 CPython 选择用引用计数做内存管理

**其他 Python 实现**（它们没有 GIL 或有不同的 GIL 策略）：

| 实现 | 运行平台 | GIL | 说明 |
|------|----------|-----|------|
| **CPython** | C / 原生 | ✅ 有 | 默认实现，pip 安装的包基本都基于它 |
| **Jython** | JVM | ❌ 无 | 用 Java 实现，利用 JVM 的垃圾回收，不需要 GIL |
| **IronPython** | .NET CLR | ❌ 无 | 用 C# 实现，利用 CLR 的垃圾回收 |
| **PyPy** | RPython | ✅ 有 | 带 JIT 编译的高性能实现，也有 GIL，但有去除计划 |
| **GraalPython** | GraalVM | ❌ 无 | Oracle 的实验性实现 |

```python
# 查看当前 Python 实现
import platform
print(platform.python_implementation())  # 'CPython'

# 查看详细版本信息
import sys
print(sys.version)
# '3.11.5 (main, Sep 11 2023) [Clang 14.0.3]'
# 注意：这里没有特别标注"CPython"，因为它就是默认的
```

> **面试要点**：当面试官问"为什么Python多线程不能利用多核"时，准确说法是"**CPython** 因为 GIL 导致多线程无法利用多核"，而不是"Python 这门语言不支持多核"。

---

**存在原因**：
- 简化内存管理（引用计数不需要加锁）
- C 扩展兼容性

**影响**：
- **CPU 密集型任务**：多线程无法利用多核（受 GIL 限制）
- **IO 密集型任务**：多线程有效（IO 操作会释放 GIL）

---

#### 多进程绕过 GIL⭐⭐⭐

> **核心结论**：GIL 是**进程级别**的锁，每个进程有自己独立的 GIL。多进程可以真正利用多核 CPU。

**为什么多进程不受 GIL 限制？**

```
多线程（共享 GIL）：
┌──────────────────────────────────┐
│ 进程                              │
│ ┌─────────┐ ┌─────────┐         │
│ │ 线程1   │ │ 线程2   │ ← 竞争同一个 GIL │
│ └─────────┘ └─────────┘         │
│         🔒 GIL（唯一）            │
│    共享内存（堆、全局变量）         │
└──────────────────────────────────┘

多进程（各自独立 GIL）：
┌───────────────────┐  ┌───────────────────┐
│ 进程1              │  │ 进程2              │
│ 🔒 GIL-1          │  │ 🔒 GIL-2          │
│ 独立内存空间        │  │ 独立内存空间        │
│ 独立 Python 解释器  │  │ 独立 Python 解释器  │
│ ← 核心1            │  │ ← 核心2            │
└───────────────────┘  └───────────────────┘
```

**CPU 密集型任务：多线程 vs 多进程对比**

```python
import time
import threading
import multiprocessing

def cpu_heavy(n):
    """CPU密集型任务：计算斐波那契"""
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a

N = 500000

# ========== 1. 单线程 ==========
start = time.time()
cpu_heavy(N)
cpu_heavy(N)
print(f"单线程: {time.time() - start:.2f}s")
# 单线程: 3.20s

# ========== 2. 多线程（受 GIL 限制，更慢） ==========
start = time.time()
t1 = threading.Thread(target=cpu_heavy, args=(N,))
t2 = threading.Thread(target=cpu_heavy, args=(N,))
t1.start(); t2.start()
t1.join(); t2.join()
print(f"多线程: {time.time() - start:.2f}s")
# 多线程: 3.50s  ← 比单线程还慢！（GIL竞争开销）

# ========== 3. 多进程（真正并行，快一倍） ==========
start = time.time()
p1 = multiprocessing.Process(target=cpu_heavy, args=(N,))
p2 = multiprocessing.Process(target=cpu_heavy, args=(N,))
p1.start(); p2.start()
p1.join(); p2.join()
print(f"多进程: {time.time() - start:.2f}s")
# 多进程: 1.70s  ← 接近2倍加速！
```

**多线程 vs 多进程 适用场景对比**

| 维度 | 多线程 `threading` | 多进程 `multiprocessing` |
|------|-------------------|------------------------|
| **GIL 影响** | ✅ 受 GIL 限制 | ❌ 不受限制（各自独立 GIL） |
| **CPU 密集型** | ❌ 无法加速，甚至更慢 | ✅ 真正并行，利用多核 |
| **IO 密集型** | ✅ 有效（IO 时释放 GIL） | ✅ 有效（但开销更大） |
| **内存开销** | 低（共享内存） | 高（每个进程独立内存空间） |
| **启动速度** | 快（微秒级） | 慢（毫秒级，需 fork/spawn） |
| **数据共享** | 简单（共享变量，需加锁） | 复杂（需 Queue/Pipe/共享内存） |
| **调试难度** | 中等 | 较高（多进程调试工具支持差） |

**实际选型建议**

```python
# ========== CPU 密集型 → 多进程 ==========
# 数值计算、图像处理、数据分析
from multiprocessing import Pool

def process_image(path):
    # 图像处理（CPU密集）
    ...

with Pool(processes=4) as pool:  # 4个进程并行
    results = pool.map(process_image, image_paths)

# ========== IO 密集型 → 多线程 / 协程 ==========
# 网络请求、文件读写、数据库查询
import asyncio

async def fetch_url(url):
    # 网络请求（IO密集）
    ...

# 推荐：协程（比多线程更轻量）
asyncio.run(asyncio.gather(*[fetch_url(url) for url in urls]))

# ========== 混合型 → 多进程 + 协程 ==========
# 每个进程内跑协程：进程处理CPU任务，协程处理IO任务
```

> **面试要点**：被问到"如何绕过 GIL"时，答案包括：
> 1. **多进程**（`multiprocessing`）——最常用
> 2. **C 扩展**（用 C/C++ 编写计算密集代码，在 C 层释放 GIL）
> 3. **使用无 GIL 的解释器**（Jython、GraalPython 等）
> 4. **Python 3.13+ free-threaded 模式**（实验性，`--disable-gil` 编译选项）

---

#### GIL底层实现原理深度剖析⭐⭐⭐

**为什么需要了解GIL底层？**
- 面试高频：为什么Python多线程不能利用多核？
- 性能调优：如何绕过GIL限制？
- 避免误区：为什么有GIL还会有线程安全问题？

---

##### 1. 为什么需要GIL？（引用计数的线程安全问题）⭐⭐⭐

**Python的内存管理：引用计数**

```python
# 每个对象都有一个引用计数器
a = []  # 创建对象，引用计数 = 1
b = a   # 引用计数 = 2
c = a   # 引用计数 = 3
del b   # 引用计数 = 2
del c   # 引用计数 = 1
del a   # 引用计数 = 0 → 对象销毁

# 查看引用计数
import sys
x = []
print(sys.getrefcount(x))  # 2（x本身 + getrefcount参数）
```

**引用计数的C实现**（CPython源码）：

```c
// Include/object.h
typedef struct _object {
    Py_ssize_t ob_refcnt;  // 引用计数
    PyTypeObject *ob_type;  // 对象类型
} PyObject;

// 增加引用计数
#define Py_INCREF(op) \
    do { \
        ((PyObject *)(op))->ob_refcnt++; \
    } while (0)

// 减少引用计数
#define Py_DECREF(op) \
    do { \
        PyObject *_py_decref_tmp = (PyObject *)(op); \
        if (--(_py_decref_tmp)->ob_refcnt == 0) \
            _Py_Dealloc(_py_decref_tmp); \
    } while (0)
```

**多线程下的引用计数问题**：

```c
// 假设没有GIL，两个线程同时执行
// 初始：obj->ob_refcnt = 1

// 线程1                    // 线程2
Py_INCREF(obj)             Py_INCREF(obj)
// 读取：refcnt = 1        // 读取：refcnt = 1
// 计算：refcnt = 2        // 计算：refcnt = 2
// 写入：refcnt = 2        // 写入：refcnt = 2

// 预期：refcnt = 3
// 实际：refcnt = 2（错误！）
// 导致：对象过早释放，野指针，崩溃
```

**解决方案对比**：

| 方案 | 优点 | 缺点 | 采用 |
|------|------|------|------|
| **每个对象加锁** | 细粒度，并发高 | 性能差（百万级对象，百万级锁） | ❌ |
| **全局锁（GIL）** | 简单，性能好 | 多线程无法利用多核 | ✅ CPython |
| **无锁引用计数** | 并发高 | 实现复杂（Jython、IronPython） | 部分Python实现 |

---

##### 2. GIL的底层实现（互斥锁 + 条件变量）⭐⭐⭐

**GIL的数据结构**（Python 3.2+）：

```c
// Python/ceval_gil.h（简化）
struct _gil_runtime_state {
    unsigned long interval;  // 切换间隔（默认5ms）
    _Py_atomic_int locked;   // GIL状态（0=空闲，1=锁定）
    unsigned long switch_number;  // 切换次数
    PyCOND_T cond;  // 条件变量（用于等待GIL）
    PyMUTEX_T mutex;  // 互斥锁
    PyThread_type_lock switch_cond;  // 切换条件
};
```

**GIL的获取过程**（简化流程）：

```python
# 线程1持有GIL，线程2尝试获取

def acquire_gil(thread):
    while True:
        # 1. 尝试获取GIL（CAS操作）
        if atomic_compare_exchange(gil.locked, 0, 1):
            return  # 成功获取
        
        # 2. GIL被占用，等待
        wait_for_gil(thread)
        
def wait_for_gil(thread):
    # 3. 设置"有线程在等待"标志
    gil.waiting_threads += 1
    
    # 4. 等待条件变量（休眠）
    cond_wait(gil.cond, gil.mutex)
    
    # 5. 被唤醒后，重新尝试获取
    gil.waiting_threads -= 1
```

**GIL的释放过程**：

```python
def release_gil(thread):
    # 1. 释放GIL
    atomic_store(gil.locked, 0)
    
    # 2. 如果有线程在等待，唤醒一个
    if gil.waiting_threads > 0:
        cond_signal(gil.cond)
```

---

##### 3. GIL的切换机制（时间片 vs 字节码计数）⭐⭐⭐

**Python 2.x：字节码计数切换**

```python
# Python 2.x的GIL切换
_Py_Ticker = 100  # 每执行100条字节码切换一次

for bytecode in bytecodes:
    execute(bytecode)
    _Py_Ticker -= 1
    
    if _Py_Ticker <= 0:
        _Py_Ticker = 100
        release_gil()  # 释放GIL
        # 其他线程可能获取GIL
        acquire_gil()  # 重新获取GIL
```

**问题**：CPU密集型线程可能饿死IO线程

```
场景：线程1（CPU密集），线程2（IO密集）

线程1：执行100条字节码 → 释放GIL → 立即重新获取 → 继续执行
线程2：尝试获取GIL → 失败（线程1太快） → 继续等待

结果：线程2可能等待很久（饥饿）
```

**Python 3.2+：时间片切换**

```python
# Python 3.2+的GIL切换
switch_interval = 5  # 默认5ms切换一次

def gil_switch():
    while True:
        # 持有GIL的线程每5ms主动释放
        sleep(switch_interval / 1000)
        if has_waiting_thread():
            release_gil()
            yield_to_other_thread()
            acquire_gil()
```

**改进**：更公平，IO线程不再饥饿

---

##### 4. 为什么IO操作会释放GIL？⭐⭐⭐

**IO操作会主动释放GIL**：

```c
// Python C扩展代码示例
static PyObject* read_file(PyObject* self, PyObject* args) {
    const char* filename;
    PyArg_ParseTuple(args, "s", &filename);
    
    // 释放GIL（允许其他线程运行）
    Py_BEGIN_ALLOW_THREADS
    
    // 执行IO操作（阻塞）
    int fd = open(filename, O_RDONLY);
    char buffer[1024];
    read(fd, buffer, 1024);
    close(fd);
    
    // 重新获取GIL
    Py_END_ALLOW_THREADS
    
    return PyUnicode_FromString(buffer);
}
```

**为什么IO操作要释放GIL？**

```
场景：线程1执行IO（5秒），线程2执行计算

如果不释放GIL：
时刻0：线程1持有GIL，执行IO（阻塞5秒）
      线程2等待GIL（什么都不做）
时刻5：线程1完成，释放GIL
      线程2获取GIL，执行计算
→ 总耗时：5秒（IO）+ 1秒（计算）= 6秒

如果释放GIL：
时刻0：线程1持有GIL，释放GIL，执行IO（阻塞5秒）
      线程2获取GIL，执行计算（1秒）
时刻1：线程2完成
时刻5：线程1完成
→ 总耗时：max(5秒, 1秒) = 5秒（并行！）
```

**哪些操作会释放GIL？**

```python
import time
import requests

# 1. time.sleep（释放GIL）
time.sleep(1)

# 2. 文件IO（释放GIL）
with open('file.txt', 'r') as f:
    data = f.read()

# 3. 网络IO（释放GIL）
response = requests.get('https://example.com')

# 4. 数据库操作（释放GIL）
cursor.execute('SELECT * FROM users')

# 5. NumPy计算（释放GIL，C实现）
import numpy as np
arr = np.random.rand(10000, 10000)
result = arr @ arr.T
```

---

##### 5. 为什么有GIL还会有线程安全问题？⭐⭐⭐

**误区**：GIL保证线程安全

**真相**：GIL只保证**字节码级别**的原子性，不保证**Python语句级别**的原子性

**示例1：+=操作不是原子的**

```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1  # 不是原子操作！

# 字节码分解
import dis
dis.dis(increment)
# 输出：
# LOAD_GLOBAL counter   ← 字节码1
# LOAD_CONST 1          ← 字节码2
# BINARY_ADD            ← 字节码3
# STORE_GLOBAL counter  ← 字节码4

# GIL可能在任何字节码之间切换！
# 线程1：LOAD_GLOBAL(counter=0) → 切换
# 线程2：LOAD_GLOBAL(counter=0) → BINARY_ADD(1) → STORE_GLOBAL(counter=1)
# 线程1：BINARY_ADD(1) → STORE_GLOBAL(counter=1)
# 结果：counter=1（预期=2，错误！）

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # 可能不是200000！
```

**示例2：列表操作不是原子的**

```python
import threading

lst = []

def append_many():
    for i in range(1000):
        lst.append(i)

threads = [threading.Thread(target=append_many) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(len(lst))  # 可能不是10000！（有些元素丢失）

# 原因：list.append的内部实现涉及多个步骤
# 1. 检查容量是否足够
# 2. 扩容（如果需要）
# 3. 写入新元素
# 4. 更新长度
# GIL可能在步骤之间切换！
```

**哪些操作是原子的？**

```python
# 原子操作（单条字节码）：
x = y              # ✅ 赋值
x.append(y)        # ✅ append（CPython实现为原子）
x = x + 1          # ❌ 不是原子（3条字节码）
x += 1             # ❌ 不是原子（4条字节码）
x = x.copy()       # ❌ 不是原子
d[key] = value     # ✅ 字典赋值（原子）
lst[i] = value     # ✅ 列表赋值（原子）
```

---

##### 6. 如何绕过GIL？（4种方法）⭐⭐⭐

**方法1：多进程（multiprocessing）**

```python
from multiprocessing import Pool

def cpu_bound(n):
    return sum(i*i for i in range(n))

# 4个进程并行（每个进程独立的GIL）
with Pool(4) as pool:
    results = pool.map(cpu_bound, [10**7] * 4)

# 性能：4核CPU可以接近4倍加速
```

**方法2：使用C扩展（释放GIL）**

```python
# 使用NumPy（C实现，释放GIL）
import numpy as np

# 这个操作释放GIL，可以并行
arr = np.random.rand(10000, 10000)
result = arr @ arr.T  # 矩阵乘法（多线程加速）
```

**方法3：使用其他Python实现**

```python
# Jython（Java实现）：无GIL
# IronPython（.NET实现）：无GIL
# PyPy（JIT编译）：有GIL，但更快

# 缺点：部分Python库不兼容
```

**方法4：使用协程（asyncio）**

```python
import asyncio

# IO密集型任务，单线程高并发
async def fetch(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()

# 1000个并发请求，单线程
tasks = [fetch(url) for url in urls]
results = await asyncio.gather(*tasks)
```

---

##### 7. GIL的未来：PEP 703（去除GIL）⭐⭐

**Python 3.13+计划**：
- 实验性支持无GIL模式（`--disable-gil`）
- 引用计数改为无锁实现（deferred reference counting）
- 性能影响：单线程可能变慢5-10%，多线程提升显著

**使用建议**（当前）：

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| **IO密集型** | 多线程/asyncio | IO操作释放GIL，并发有效 |
| **CPU密集型** | 多进程/C扩展 | 绕过GIL限制 |
| **混合场景** | 多进程+多线程 | 进程间CPU并行，进程内IO并发 |

---

### 并发模型对比

| 方式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **多线程** | IO 密集型 | 共享内存，切换快 | 受 GIL 限制，不适合 CPU 密集 |
| **多进程** | CPU 密集型 | 绕过 GIL，利用多核 | 进程开销大，通信成本高 |
| **协程（asyncio）** | IO 密集型 | 单线程，开销小 | 需要异步库支持，学习曲线陡 |

---

### 多线程（threading）

**基础使用**：
```python
import threading
import time

def worker(name):
    print(f"{name} starting")
    time.sleep(2)
    print(f"{name} done")

threads = []
for i in range(5):
    t = threading.Thread(target=worker, args=(f"Worker-{i}",))
    threads.append(t)
    t.start()

for t in threads:
    t.join()  # 等待线程结束

print("All done")
```

---

**线程同步**：

**1. Lock（互斥锁）**
```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    with lock:  # 自动获取和释放锁
        counter += 1

threads = [threading.Thread(target=increment) for _ in range(1000)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # 1000（线程安全）
```

**2. RLock（可重入锁）**
```python
import threading

lock = threading.RLock()

def recursive_func(n):
    with lock:
        if n > 0:
            recursive_func(n - 1)  # 可重入
```

**3. Semaphore（信号量）**
```python
import threading

# 最多 3 个线程同时访问
semaphore = threading.Semaphore(3)

def worker():
    with semaphore:
        print(f"{threading.current_thread().name} acquired")
        time.sleep(1)

threads = [threading.Thread(target=worker) for _ in range(10)]
for t in threads:
    t.start()
```

**4. Event（事件）**
```python
import threading

event = threading.Event()

def waiter():
    print("waiting for event")
    event.wait()  # 阻塞直到 event.set()
    print("event received")

def setter():
    time.sleep(2)
    event.set()  # 唤醒所有等待的线程

threading.Thread(target=waiter).start()
threading.Thread(target=setter).start()
```

---

### 多进程（multiprocessing）

**基础使用**：
```python
import multiprocessing

def worker(name):
    print(f"{name} starting")
    time.sleep(2)
    print(f"{name} done")

if __name__ == "__main__":
    processes = []
    for i in range(5):
        p = multiprocessing.Process(target=worker, args=(f"Process-{i}",))
        processes.append(p)
        p.start()
    
    for p in processes:
        p.join()
    
    print("All done")
```

---

**进程间通信**：

**1. Queue（队列）**
```python
from multiprocessing import Process, Queue

def producer(queue):
    for i in range(5):
        queue.put(i)
    queue.put(None)  # 结束信号

def consumer(queue):
    while True:
        item = queue.get()
        if item is None:
            break
        print(f"consumed {item}")

if __name__ == "__main__":
    queue = Queue()
    p1 = Process(target=producer, args=(queue,))
    p2 = Process(target=consumer, args=(queue,))
    p1.start()
    p2.start()
    p1.join()
    p2.join()
```

**2. Pipe（管道）**
```python
from multiprocessing import Process, Pipe

def sender(conn):
    conn.send("Hello")
    conn.close()

def receiver(conn):
    msg = conn.recv()
    print(f"received: {msg}")
    conn.close()

if __name__ == "__main__":
    parent_conn, child_conn = Pipe()
    p1 = Process(target=sender, args=(parent_conn,))
    p2 = Process(target=receiver, args=(child_conn,))
    p1.start()
    p2.start()
    p1.join()
    p2.join()
```

**3. Manager（共享数据）**
```python
from multiprocessing import Process, Manager

def worker(shared_dict, key, value):
    shared_dict[key] = value

if __name__ == "__main__":
    with Manager() as manager:
        shared_dict = manager.dict()
        
        processes = []
        for i in range(5):
            p = Process(target=worker, args=(shared_dict, i, i*10))
            processes.append(p)
            p.start()
        
        for p in processes:
            p.join()
        
        print(dict(shared_dict))
```

---

**进程池（Pool）**：

```python
from multiprocessing import Pool

def square(x):
    return x * x

if __name__ == "__main__":
    with Pool(processes=4) as pool:
        # map：阻塞等待所有结果
        results = pool.map(square, range(10))
        print(results)  # [0, 1, 4, 9, 16, ...]
        
        # imap：返回迭代器，边计算边返回
        for result in pool.imap(square, range(10)):
            print(result)
        
        # apply_async：异步执行单个任务
        result = pool.apply_async(square, (10,))
        print(result.get())  # 100
```

---

## 装饰器与元编程

### 装饰器（Decorator）

**基础装饰器**：
```python
def log(func):
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

@log  # 等价于 add = log(add)
def add(a, b):
    return a + b

add(1, 2)
# 输出：
# calling add
# add returned 3
```

---

**保留元信息**：
```python
from functools import wraps

def log(func):
    @wraps(func)  # 保留原函数的 __name__、__doc__ 等
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log
def add(a, b):
    """Add two numbers"""
    return a + b

print(add.__name__)  # add（而非 wrapper）
print(add.__doc__)   # Add two numbers
```

---

**带参数的装饰器**：
```python
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def say_hello():
    print("Hello")

say_hello()
# 输出：
# Hello
# Hello
# Hello
```

---

**类装饰器**：
```python
class Counter:
    def __init__(self, func):
        self.func = func
        self.count = 0
    
    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"called {self.count} times")
        return self.func(*args, **kwargs)

@Counter
def say_hello():
    print("Hello")

say_hello()  # called 1 times \n Hello
say_hello()  # called 2 times \n Hello
```

---

**实战装饰器**：

**1. 超时装饰器**
```python
import signal
from functools import wraps

class TimeoutError(Exception):
    pass

def timeout(seconds):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            def handler(signum, frame):
                raise TimeoutError(f"{func.__name__} timeout")
            
            signal.signal(signal.SIGALRM, handler)
            signal.alarm(seconds)
            
            try:
                result = func(*args, **kwargs)
            finally:
                signal.alarm(0)  # 取消定时器
            
            return result
        return wrapper
    return decorator

@timeout(2)
def slow_function():
    time.sleep(3)

slow_function()  # 抛出 TimeoutError
```

---

**2. 缓存装饰器（记忆化）**
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(100))  # 快速返回

# 查看缓存信息
print(fibonacci.cache_info())
# CacheInfo(hits=98, misses=101, maxsize=128, currsize=101)
```

---

**3. 重试装饰器**
```python
import time
from functools import wraps

def retry(max_attempts=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempts = 0
            while attempts < max_attempts:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    attempts += 1
                    if attempts >= max_attempts:
                        raise
                    print(f"attempt {attempts} failed, retrying...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def unstable_function():
    import random
    if random.random() < 0.7:
        raise Exception("failed")
    return "success"
```

---

### 上下文管理器

**方式1：类实现**
```python
class DatabaseConnection:
    def __enter__(self):
        self.conn = connect_to_db()
        return self.conn
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False  # 不抑制异常

with DatabaseConnection() as conn:
    conn.execute("SELECT * FROM users")
```

**方式2：contextlib**
```python
from contextlib import contextmanager

@contextmanager
def database_connection():
    conn = connect_to_db()
    try:
        yield conn
    finally:
        conn.close()

with database_connection() as conn:
    conn.execute("SELECT * FROM users")
```

---

### 元类（Metaclass）

**元类**是创建类的类（类的模板）。

```python
# 类是对象的模板
class User:
    pass

user = User()  # User 是类，user 是对象

# 元类是类的模板
class Meta(type):
    def __new__(cls, name, bases, attrs):
        # name: 类名
        # bases: 父类
        # attrs: 属性字典
        print(f"creating class {name}")
        return super().__new__(cls, name, bases, attrs)

class User(metaclass=Meta):
    pass

# 输出：creating class User
```

**实战：ORM 自动注册**

```python
class ModelMeta(type):
    def __new__(cls, name, bases, attrs):
        # 自动添加 table_name 属性
        if 'table_name' not in attrs:
            attrs['table_name'] = name.lower() + 's'
        
        # 收集字段
        fields = {}
        for key, value in attrs.items():
            if isinstance(value, Field):
                fields[key] = value
        attrs['_fields'] = fields
        
        return super().__new__(cls, name, bases, attrs)

class Field:
    def __init__(self, field_type):
        self.field_type = field_type

class Model(metaclass=ModelMeta):
    pass

class User(Model):
    name = Field('VARCHAR')
    age = Field('INT')

print(User.table_name)  # users
print(User._fields)     # {'name': <Field>, 'age': <Field>}
```

---

## asyncio协程

### 基础概念

**协程（Coroutine）**：可以暂停和恢复的函数，适合 IO 密集型任务。

**关键字**：
- `async def`：定义协程函数
- `await`：等待协程或异步操作完成

```python
import asyncio

async def hello():
    print("Hello")
    await asyncio.sleep(1)  # 模拟 IO 操作，让出控制权
    print("World")

# 运行协程
asyncio.run(hello())
```

---

### 并发执行

**方式1：asyncio.gather**
```python
import asyncio

async def fetch_data(n):
    print(f"fetching {n}")
    await asyncio.sleep(1)
    return n * 10

async def main():
    # 并发执行多个协程
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3)
    )
    print(results)  # [10, 20, 30]

asyncio.run(main())
```

**方式2：asyncio.create_task**
```python
async def main():
    # 创建任务（立即开始执行）
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))
    
    # 等待完成
    result1 = await task1
    result2 = await task2
    
    print(result1, result2)

asyncio.run(main())
```

---

### 超时与取消

**超时**：
```python
async def slow_operation():
    await asyncio.sleep(5)
    return "done"

async def main():
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=2)
    except asyncio.TimeoutError:
        print("timeout")

asyncio.run(main())
```

**取消**：
```python
async def main():
    task = asyncio.create_task(fetch_data(1))
    
    await asyncio.sleep(0.5)
    task.cancel()  # 取消任务
    
    try:
        await task
    except asyncio.CancelledError:
        print("task cancelled")

asyncio.run(main())
```

---

### 异步上下文管理器

```python
class AsyncDatabaseConnection:
    async def __aenter__(self):
        self.conn = await connect_to_db_async()
        return self.conn
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.conn.close()
        return False

async def main():
    async with AsyncDatabaseConnection() as conn:
        await conn.execute("SELECT * FROM users")

asyncio.run(main())
```

---

### 异步迭代器

```python
class AsyncRange:
    def __init__(self, n):
        self.n = n
        self.i = 0
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.i >= self.n:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        self.i += 1
        return self.i

async def main():
    async for i in AsyncRange(5):
        print(i)

asyncio.run(main())
```

---

### 实战：aiohttp 并发爬虫

```python
import aiohttp
import asyncio

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    urls = [
        'https://example.com',
        'https://example.org',
        'https://example.net'
    ]
    
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        
        for url, result in zip(urls, results):
            print(f"{url}: {len(result)} bytes")

asyncio.run(main())
```

---

## Django ORM优化

### N+1 问题

**问题**：
```python
# ❌ N+1 查询
users = User.objects.all()  # 1 次查询
for user in users:
    print(user.profile.bio)  # N 次查询（每个 user 一次）
```

**解决方案1：select_related（一对一/多对一）**
```python
# ✅ 使用 JOIN，1 次查询
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # 不再额外查询
```

**解决方案2：prefetch_related（多对多/反向外键）**
```python
# ✅ 使用 IN 查询，2 次查询
users = User.objects.prefetch_related('groups').all()
for user in users:
    for group in user.groups.all():  # 不再额外查询
        print(group.name)
```

---

### 查询优化

**1. 只查询需要的字段**
```python
# ❌ 查询所有字段
users = User.objects.all()

# ✅ 只查询 id 和 name
users = User.objects.only('id', 'name')

# ✅ 排除某些字段
users = User.objects.defer('bio', 'avatar')
```

**2. 使用 values / values_list**
```python
# 返回字典
users = User.objects.values('id', 'name')
# [{'id': 1, 'name': 'Alice'}, ...]

# 返回元组
users = User.objects.values_list('id', 'name')
# [(1, 'Alice'), ...]

# 返回单列
ids = User.objects.values_list('id', flat=True)
# [1, 2, 3, ...]
```

**3. 聚合查询**
```python
from django.db.models import Count, Avg, Sum, Max, Min

# 统计数量
count = User.objects.count()

# 聚合
User.objects.aggregate(
    total=Count('id'),
    avg_age=Avg('age'),
    max_age=Max('age')
)
# {'total': 100, 'avg_age': 30.5, 'max_age': 80}

# 分组聚合
from django.db.models import Count
User.objects.values('city').annotate(count=Count('id'))
# [{'city': 'Beijing', 'count': 30}, {'city': 'Shanghai', 'count': 50}]
```

---

### 事务处理

**方式1：装饰器**
```python
from django.db import transaction

@transaction.atomic
def transfer(from_user, to_user, amount):
    from_user.balance -= amount
    from_user.save()
    
    to_user.balance += amount
    to_user.save()
```

**方式2：上下文管理器**
```python
from django.db import transaction

def transfer(from_user, to_user, amount):
    with transaction.atomic():
        from_user.balance -= amount
        from_user.save()
        
        to_user.balance += amount
        to_user.save()
```

**保存点（Savepoint）**：
```python
from django.db import transaction

def complex_operation():
    with transaction.atomic():
        # 创建保存点
        sid = transaction.savepoint()
        
        try:
            # 操作1
            user.save()
        except Exception:
            # 回滚到保存点
            transaction.savepoint_rollback(sid)
        else:
            # 提交保存点
            transaction.savepoint_commit(sid)
```

---

### 批量操作

**批量创建**：
```python
users = [User(name=f"User{i}") for i in range(1000)]
User.objects.bulk_create(users, batch_size=100)
```

**批量更新**：
```python
users = User.objects.filter(city='Beijing')
users.update(is_active=True)  # 1 条 UPDATE 语句
```

**批量删除**：
```python
User.objects.filter(is_active=False).delete()
```

---

## FastAPI实战

### 基础使用

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    name: str
    email: str

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in db:
        raise HTTPException(status_code=404, detail="User not found")
    return db[user_id]

@app.post("/users")
async def create_user(user: User):
    db[user.id] = user
    return user

@app.put("/users/{user_id}")
async def update_user(user_id: int, user: User):
    db[user_id] = user
    return user

@app.delete("/users/{user_id}")
async def delete_user(user_id: int):
    del db[user_id]
    return {"message": "User deleted"}
```

---

### 依赖注入

**基础依赖**：
```python
from fastapi import Depends

def get_db():
    db = connect_to_db()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
async def get_users(db=Depends(get_db)):
    return db.query("SELECT * FROM users")
```

**多层依赖**：
```python
def get_token_header(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid token")
    return authorization[7:]

def get_current_user(token: str = Depends(get_token_header), db=Depends(get_db)):
    user = db.get_user_by_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid user")
    return user

@app.get("/profile")
async def get_profile(user=Depends(get_current_user)):
    return user
```

---

### 中间件

```python
from fastapi import Request
import time

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

---

### 后台任务

```python
from fastapi import BackgroundTasks

def send_email(email: str, message: str):
    # 发送邮件（耗时操作）
    time.sleep(5)
    print(f"Email sent to {email}")

@app.post("/send-notification")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_email, email, "Hello")
    return {"message": "Notification sent"}
```

---

## 性能优化与调试

### cProfile 性能分析

```python
import cProfile
import pstats

def slow_function():
    total = 0
    for i in range(1000000):
        total += i
    return total

# 分析
profiler = cProfile.Profile()
profiler.enable()
slow_function()
profiler.disable()

# 输出统计
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)
```

---

### memory_profiler 内存分析

```python
from memory_profiler import profile

@profile
def memory_intensive():
    a = [i for i in range(1000000)]
    b = [i*2 for i in range(1000000)]
    return a, b

memory_intensive()
```

---

### 常见优化技巧

**1. 使用生成器**
```python
# ❌ 占用大量内存
def read_large_file():
    with open('large_file.txt') as f:
        return f.readlines()  # 一次性加载到内存

# ✅ 逐行读取
def read_large_file():
    with open('large_file.txt') as f:
        for line in f:  # 生成器
            yield line
```

**2. 使用 `__slots__`**
```python
# 默认使用 __dict__ 存储属性（占用内存多）
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

# 使用 __slots__（节省内存）
class Point:
    __slots__ = ['x', 'y']
    
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

**3. 使用局部变量**
```python
# ❌ 全局变量查找慢
import math

def calculate():
    result = 0
    for i in range(1000000):
        result += math.sin(i)
    return result

# ✅ 局部变量查找快
def calculate():
    sin = math.sin  # 缓存到局部变量
    result = 0
    for i in range(1000000):
        result += sin(i)
    return result
```

---


## 实战案例

### 游戏后台：异步任务队列

```python
import asyncio
from typing import Callable

class AsyncTaskQueue:
    def __init__(self, workers=4):
        self.queue = asyncio.Queue()
        self.workers = workers
    
    async def worker(self, name):
        while True:
            task, args = await self.queue.get()
            try:
                print(f"{name} processing {task.__name__}")
                await task(*args)
            except Exception as e:
                print(f"task failed: {e}")
            finally:
                self.queue.task_done()
    
    async def start(self):
        tasks = []
        for i in range(self.workers):
            task = asyncio.create_task(self.worker(f"Worker-{i}"))
            tasks.append(task)
        await self.queue.join()
        for task in tasks:
            task.cancel()
    
    def add_task(self, task: Callable, *args):
        self.queue.put_nowait((task, args))

# 使用
async def send_reward(user_id, reward):
    await asyncio.sleep(1)
    print(f"reward sent to {user_id}: {reward}")

async def main():
    queue = AsyncTaskQueue(workers=4)
    
    for i in range(10):
        queue.add_task(send_reward, i, 100)
    
    await queue.start()

asyncio.run(main())
```

---

### 金融支付：数据库事务重试

```python
from django.db import transaction
import time

def retry_on_deadlock(max_attempts=3):
    def decorator(func):
        def wrapper(*args, **kwargs):
            attempts = 0
            while attempts < max_attempts:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if 'deadlock' in str(e).lower():
                        attempts += 1
                        if attempts >= max_attempts:
                            raise
                        time.sleep(0.1 * attempts)
                    else:
                        raise
        return wrapper
    return decorator

@retry_on_deadlock(max_attempts=3)
@transaction.atomic
def transfer_money(from_account, to_account, amount):
    # 按 ID 顺序加锁，避免死锁
    accounts = sorted([from_account, to_account], key=lambda x: x.id)
    
    for account in accounts:
        account = Account.objects.select_for_update().get(id=account.id)
    
    from_account.balance -= amount
    from_account.save()
    
    to_account.balance += amount
    to_account.save()
```

---

**总结**

Python 面试重点：
- **核心特性**：魔法方法、可变/不可变、装饰器
- **并发模型**：GIL、多线程/多进程/协程的选择
- **ORM 优化**：N+1 问题、事务、批量操作
- **性能调优**：cProfile、memory_profiler、优化技巧
- **实战经验**：结合游戏/金融场景的工程能力

---

## 面试题自查

### Q1：Python 的内存管理机制是什么？引用计数有什么缺陷，如何解决？

**答**：CPython 使用**引用计数**作为主要机制——每个对象维护 `ob_refcnt`，归零时立即回收。缺陷是**无法处理循环引用**（两个对象互相引用，计数永不归零）。解决方案是**标记-清除 + 分代回收**：标记-清除从根对象出发遍历可达对象，清除不可达对象；分代回收将对象按存活时间分为三代（阈值 700/10/10），减少扫描范围。此外 CPython 还有 **PyMalloc 内存池**（≤512 字节的小对象走内存池，Arena→Pool→Block 三级结构），提升分配效率。

---

### Q2：Python 中 `is` 和 `==` 的区别？什么是小整数池？

**答**：`is` 比较的是两个对象的 **id（内存地址）**，判断是否为同一个对象；`==` 比较的是对象的 **值**（调用 `__eq__`）。CPython 对 `-5` 到 `256` 的小整数做了缓存（**小整数池**），这些整数在解释器启动时就创建好并复用，所以 `a = 42; b = 42; a is b` 为 `True`。超出此范围的整数每次都创建新对象。

---

### Q3：深拷贝和浅拷贝的区别？什么场景下必须用深拷贝？

**答**：**浅拷贝**（`copy.copy()` / `list.copy()`）只复制顶层对象，内部的嵌套可变对象仍然共享引用；**深拷贝**（`copy.deepcopy()`）递归复制所有层级的对象，完全独立。当对象包含嵌套的可变容器（如 list of list、dict of list）并且需要独立修改时，必须使用深拷贝。

---

### Q4：解释 Python 的 LEGB 作用域规则。什么是闭包？闭包的循环变量捕获陷阱是什么？

**答**：LEGB 是变量查找顺序：**L**ocal（函数内部）→ **E**nclosing（外层嵌套函数）→ **G**lobal（模块级）→ **B**uilt-in（内置）。**闭包**是内层函数引用了外层函数变量并被返回的函数对象，通过 cell 对象记住被引用的变量。**循环变量捕获陷阱**：在循环中创建的 lambda/函数捕获的是变量本身而非值，循环结束后所有闭包共享同一个变量的最终值。修复方法：使用默认参数 `lambda i=i: i` 在定义时绑定值。

---

### Q5：Python 的迭代器协议是什么？迭代器和可迭代对象有什么区别？

**答**：迭代器协议由 `__iter__()` 和 `__next__()` 两个方法定义。**可迭代对象**只需实现 `__iter__()`（返回迭代器），可以被多次遍历；**迭代器**同时实现两者，有状态（记住遍历位置），只能遍历一次。`for` 循环先调用 `iter()` 获取迭代器，再反复调用 `next()` 直到 `StopIteration`。

---

### Q6：生成器和普通函数有什么区别？`yield` 和 `return` 的区别？

**答**：生成器函数包含 `yield` 关键字，调用时**不执行函数体**，而是返回生成器对象。`yield` 暂停函数执行并返回值，保留函数状态（局部变量、执行位置），下次 `next()` 从暂停处继续。`return` 直接终止函数并返回值，函数状态丢失。生成器是**惰性求值**的，适合处理大数据流，内存友好。

---

### Q7：`yield from` 的作用是什么？它和普通 `yield` 有什么区别？

**答**：`yield from` 用于委托另一个可迭代对象或子生成器。与手动 `for item in sub: yield item` 相比，`yield from` 自动建立了调用方与子生成器之间的**透明双向通道**，转发 `send()`、`throw()`、`close()`，并且子生成器的 `return` 值会成为 `yield from` 表达式的返回值。这是 `asyncio` 协程的基础（`await` 本质是 `yield from`）。

---

### Q8：GIL 是什么？为什么 CPython 需要 GIL？有 GIL 还会有线程安全问题吗？

**答**：GIL（全局解释器锁）是 CPython 的一把全局互斥锁，确保同一时间只有一个线程执行 Python 字节码。需要 GIL 是因为 CPython 的引用计数不是线程安全的，如果没有 GIL 就需要给每个对象加锁（百万级对象=百万级锁）。**有 GIL 仍然有线程安全问题**——GIL 只保证单条字节码的原子性，`counter += 1` 等操作对应多条字节码，GIL 可能在字节码之间切换。

---

### Q9：如何绕过 GIL 限制？CPU 密集型和 IO 密集型分别用什么方案？

**答**：四种方法：① **多进程**（`multiprocessing`）——每个进程有独立 GIL，最常用；② **C 扩展**（如 NumPy，在 C 层释放 GIL）；③ 使用无 GIL 的解释器（Jython、IronPython）；④ Python 3.13+ `--disable-gil` 实验模式。**CPU 密集型**用多进程/C 扩展；**IO 密集型**用多线程或 asyncio（IO 操作会主动释放 GIL）。

---

### Q10：Python 的装饰器本质是什么？如何实现带参数的装饰器？`@wraps` 的作用？

**答**：装饰器本质是一个**接受函数并返回新函数的高阶函数**，`@decorator` 语法是 `func = decorator(func)` 的语法糖。带参数的装饰器需要多一层嵌套：外层接受参数返回真正的装饰器函数。`functools.wraps` 用于保留被装饰函数的元信息（`__name__`、`__doc__`、`__module__` 等），否则被装饰后函数名变成 `wrapper`。

---

### Q11：`asyncio` 的事件循环工作原理是什么？`async/await` 和多线程有什么区别？

**答**：`asyncio` 基于**单线程事件循环**：事件循环维护一个任务队列，轮询检查 IO 就绪状态，将就绪的协程恢复执行。`await` 让出控制权给事件循环（类似 `yield`），等 IO 完成后被回调恢复。与多线程相比：协程是**协作式调度**（在 `await` 处切换），没有线程切换开销和竞态条件；线程是**抢占式调度**（OS 随时切换），需要加锁。协程更轻量（KB 级内存 vs 线程 MB 级）。

---

### Q12：`select_related` 和 `prefetch_related` 的区别？分别适用于什么关系？

**答**：`select_related` 使用 SQL **JOIN** 查询，在一次查询中获取关联对象，适用于**一对一**（OneToOne）和**多对一**（ForeignKey 正向）关系。`prefetch_related` 使用**额外的 IN 查询**，先查主表再查关联表然后在 Python 中合并，适用于**多对多**（ManyToMany）和**反向外键**关系。`select_related` 一次 SQL 但可能 JOIN 过多，`prefetch_related` 多次 SQL 但每次更简单。

---

### Q13：Django 中如何解决 N+1 查询问题？如何检测 N+1？

**答**：使用 `select_related`/`prefetch_related` 预加载关联数据，将 N+1 次查询减少到 1~2 次。检测方法：① 使用 `django-debug-toolbar` 查看每个页面执行的 SQL 数量；② 使用 `django.db.connection.queries` 打印所有 SQL；③ 使用 `nplusone` 库自动检测。也可以用 `only()`/`defer()` 减少字段查询量。

---

### Q14：FastAPI 的依赖注入系统是如何工作的？`Depends` 嵌套依赖如何解析？

**答**：FastAPI 的 `Depends()` 声明依赖关系，框架在请求到达时自动解析依赖树：先解析最底层依赖，再逐层向上。依赖函数可以是普通函数或生成器（`yield` 实现类似 `with` 的上下文管理，`yield` 前为 setup，`yield` 后为 teardown）。同一请求内相同依赖默认缓存结果（避免重复创建 DB 连接等），可通过 `use_cache=False` 禁用。

---

### Q15：Python 中 `__new__` 和 `__init__` 的区别？如何用 `__new__` 实现单例模式？

**答**：`__new__` 是**类方法**，负责创建并返回实例对象（在 `__init__` 之前调用）；`__init__` 是**实例方法**，负责初始化已创建的对象。`__new__` 接收 `cls` 参数，`__init__` 接收 `self`。单例实现：在 `__new__` 中检查是否已有实例，有则直接返回，无则创建。

```python
class Singleton:
    _instance = None
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

---

### Q16：什么是元类（Metaclass）？它和类装饰器有什么区别？

**答**：元类是**创建类的类**（类是对象的模板，元类是类的模板），继承自 `type`。通过 `__new__` 或 `__init_subclass__` 可以在类创建时动态修改类的属性、方法、继承关系等。类装饰器在类创建后修改类，元类在类创建过程中介入。元类更强大（可以控制类的创建过程），但也更复杂。ORM 框架（如 Django Model）广泛使用元类自动注册模型和收集字段定义。

---

### Q17：`asyncio.gather` 和 `asyncio.create_task` 有什么区别？什么时候用哪个？

**答**：`create_task` 将单个协程包装为 Task 对象并**立即调度执行**（不需要 await 就已经开始跑了），返回 Task 可以后续 await 获取结果或取消。`gather` 接受多个协程/Task，**并发执行并等待所有完成**，返回结果列表。简单并发场景用 `gather` 更简洁；需要对单个任务做取消/超时/异常处理时用 `create_task`。

---

### Q18：Python 的 `with` 语句底层是如何工作的？如何同时管理多个资源？

**答**：`with` 语句调用上下文管理器的 `__enter__()` 进入、`__exit__()` 退出。执行流程：① 调用 `__enter__()` 获取资源；② `as` 绑定 `__enter__` 的返回值；③ 执行 `with` 块代码；④ 无论是否异常都调用 `__exit__(exc_type, exc_val, exc_tb)`。多资源管理：`with open('a') as f1, open('b') as f2:` 或使用 `contextlib.ExitStack` 动态管理多个资源。

---

### Q19：Python 多进程间通信有哪些方式？各自的特点和适用场景？

**答**：① **Queue**（多生产者-多消费者队列）——最常用，线程/进程安全，适合任务分发；② **Pipe**（双向管道）——只支持两个端点，速度比 Queue 快，适合一对一通信；③ **Manager**（共享对象）——基于代理模式，支持 dict、list、Namespace 等，方便但性能较低；④ **共享内存**（`multiprocessing.Value`/`Array`）——直接共享内存，性能最高但需要手动加锁。

---

### Q20：如何用 Python 进行性能分析？有哪些常用工具？

**答**：① **cProfile**——标准库的确定性分析器，统计每个函数的调用次数和耗时，用 `pstats` 排序分析；② **line_profiler**——逐行分析函数耗时（`@profile` 装饰器 + `kernprof` 命令）；③ **memory_profiler**——逐行分析内存使用；④ **py-spy**——采样分析器，无需修改代码，可以 attach 到正在运行的进程；⑤ **tracemalloc**——标准库，追踪内存分配来源。优化技巧包括：使用局部变量（查找快于全局）、`__slots__` 节省内存、生成器代替列表、`lru_cache` 缓存结果等。
