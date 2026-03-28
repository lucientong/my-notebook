# Rust 系统编程

---

## 📑 目录

### 核心概念
1. [Rust 语言概述](#1-rust-语言概述)
2. [所有权系统](#2-所有权系统)
3. [生命周期](#3-生命周期)
4. [Trait 系统](#4-trait-系统)

### 高级特性
5. [智能指针](#5-智能指针)
6. [并发编程](#6-并发编程)
7. [异步编程](#7-异步编程)
8. [宏系统](#8-宏系统)

### 实践
9. [错误处理](#9-错误处理)
10. [面试题自查](#10-面试题自查)

---

## 1. Rust 语言概述

### 1.1 Rust 核心特点

| 特性 | 说明 | 对比 C/C++ |
|------|------|-----------|
| **内存安全** | 编译时防止悬垂指针、数据竞争 | 运行时错误→编译时错误 |
| **零成本抽象** | 高级抽象无运行时开销 | 与手写优化代码性能相当 |
| **无GC** | 所有权系统管理内存 | 无 GC 暂停 |
| **线程安全** | 编译时防止数据竞争 | 类型系统保证 |

### 1.2 适用场景

- 系统编程（OS、驱动、嵌入式）
- Web3/区块链（Solana、Polkadot、Near）
- 高性能服务（Cloudflare、Discord）
- WebAssembly
- 命令行工具

---

## 2. 所有权系统

### 2.1 所有权规则

```rust
// Rust 所有权三大规则：
// 1. 每个值有且只有一个所有者（owner）
// 2. 当所有者离开作用域，值被丢弃（drop）
// 3. 同一时刻，要么一个可变引用，要么多个不可变引用

fn main() {
    let s1 = String::from("hello");  // s1 是 "hello" 的所有者
    let s2 = s1;                     // 所有权转移（move）给 s2
    // println!("{}", s1);           // 编译错误！s1 已无效
    println!("{}", s2);              // OK
}  // s2 离开作用域，"hello" 被释放
```

### 2.2 借用（Borrowing）

```rust
fn main() {
    let s = String::from("hello");
    
    // 不可变借用（可以有多个）
    let r1 = &s;
    let r2 = &s;
    println!("{}, {}", r1, r2);  // OK
    
    // 可变借用（同时只能有一个）
    let mut s = String::from("hello");
    let r3 = &mut s;
    r3.push_str(" world");
    // let r4 = &mut s;  // 编译错误！已有可变借用
    // let r5 = &s;      // 编译错误！已有可变借用
    println!("{}", r3);
}

// 借用规则：
// - 不可变借用期间，不能有可变借用
// - 可变借用期间，不能有其他任何借用
// - 引用必须始终有效（不能悬垂）
```

### 2.3 Move 与 Copy

```rust
// Copy trait：栈上简单类型，赋值时复制
let x: i32 = 5;
let y = x;  // 复制，x 仍有效
println!("{}, {}", x, y);  // OK

// 默认 Move：堆上数据，赋值时转移所有权
let s1 = String::from("hello");
let s2 = s1;  // Move
// println!("{}", s1);  // 错误！

// 显式克隆
let s1 = String::from("hello");
let s2 = s1.clone();  // 深拷贝
println!("{}, {}", s1, s2);  // OK

// 实现 Copy 的类型：
// - 所有整数、浮点数、布尔、字符
// - 元组（如果所有元素都是 Copy）
// - 数组（如果元素是 Copy）
```

---

## 3. 生命周期

### 3.1 生命周期标注

```rust
// 生命周期确保引用有效
// 编译器需要知道返回的引用与哪个输入引用关联

// 显式标注生命周期
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xyz");
        result = longest(&s1, &s2);
        println!("{}", result);  // OK，s2 还有效
    }
    // println!("{}", result);  // 错误！s2 已失效，result 可能指向 s2
}
```

### 3.2 结构体中的生命周期

```rust
// 结构体包含引用时必须标注生命周期
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
    
    // 生命周期省略规则
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention: {}", announcement);
        self.part  // 返回与 self 相同的生命周期
    }
}
```

### 3.3 生命周期省略规则

```rust
// 编译器自动推断的规则：
// 1. 每个引用参数获得独立的生命周期
// 2. 如果只有一个输入生命周期，它被赋给所有输出
// 3. 如果有 &self 或 &mut self，self 的生命周期赋给所有输出

// 无需标注（规则2）
fn first_word(s: &str) -> &str { ... }
// 等价于
fn first_word<'a>(s: &'a str) -> &'a str { ... }

// 无需标注（规则3）
impl Context {
    fn parse(&self) -> Result<(), &str> { ... }
}
```

### 3.4 'static 生命周期

```rust
// 'static：整个程序运行期间有效

// 字符串字面量是 'static
let s: &'static str = "I live forever";

// 泄漏内存创建 'static（慎用）
let leaked: &'static String = Box::leak(Box::new(String::from("leaked")));

// 常见于 trait bound
fn print_it<T: std::fmt::Display + 'static>(input: T) {
    println!("{}", input);
}
```

---

## 4. Trait 系统

### 4.1 定义与实现

```rust
// 定义 trait
trait Summary {
    fn summarize(&self) -> String;
    
    // 默认实现
    fn summarize_author(&self) -> String {
        String::from("(unknown)")
    }
}

// 实现 trait
struct Article {
    headline: String,
    author: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}, by {}", self.headline, self.author)
    }
}
```

### 4.2 Trait Bound

```rust
// trait 作为参数
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// 完整语法
fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}

// 多个 trait bound
fn notify(item: &(impl Summary + Display)) { }
fn notify<T: Summary + Display>(item: &T) { }

// where 子句（复杂情况）
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{ ... }
```

### 4.3 常用标准库 Trait

| Trait | 说明 | 示例 |
|-------|------|------|
| Clone | 显式深拷贝 | `x.clone()` |
| Copy | 隐式复制 | `let y = x;` |
| Drop | 析构函数 | 离开作用域时调用 |
| Default | 默认值 | `T::default()` |
| Debug | 调试打印 | `println!("{:?}", x)` |
| Display | 用户友好打印 | `println!("{}", x)` |
| PartialEq/Eq | 相等比较 | `x == y` |
| PartialOrd/Ord | 顺序比较 | `x < y` |
| Hash | 哈希 | HashMap 键 |
| From/Into | 类型转换 | `String::from("hi")` |
| Deref | 解引用 | `*x` |
| Iterator | 迭代器 | `for x in iter` |

### 4.4 Trait Object 与动态分发

```rust
// 静态分发（泛型，编译时确定）
fn notify<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}

// 动态分发（trait object，运行时确定）
fn notify(item: &dyn Summary) {
    println!("{}", item.summarize());
}

// trait object 存储
let articles: Vec<Box<dyn Summary>> = vec![
    Box::new(Article { ... }),
    Box::new(Tweet { ... }),
];

// 对象安全规则：
// - 返回类型不能是 Self
// - 没有泛型类型参数
```

---

## 5. 智能指针

### 5.1 Box<T>

```rust
// Box：堆上分配
let b = Box::new(5);  // 5 在堆上
println!("{}", b);

// 用途1：大数据避免栈溢出
let large = Box::new([0u8; 1_000_000]);

// 用途2：递归类型
enum List {
    Cons(i32, Box<List>),
    Nil,
}

let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));
```

### 5.2 Rc<T> 与 Arc<T>

```rust
use std::rc::Rc;

// Rc：单线程引用计数
let a = Rc::new(String::from("hello"));
let b = Rc::clone(&a);  // 增加引用计数，不是深拷贝
let c = Rc::clone(&a);

println!("count = {}", Rc::strong_count(&a));  // 3

// Arc：多线程安全的引用计数
use std::sync::Arc;
use std::thread;

let a = Arc::new(5);
let b = Arc::clone(&a);

thread::spawn(move || {
    println!("{}", b);
});
```

### 5.3 RefCell<T> 与内部可变性

```rust
use std::cell::RefCell;

// RefCell：运行时借用检查（单线程）
let data = RefCell::new(5);

{
    let mut borrowed = data.borrow_mut();  // 运行时可变借用
    *borrowed += 1;
}

println!("{}", data.borrow());  // 不可变借用

// Rc<RefCell<T>> 组合：多所有者 + 可变
use std::rc::Rc;
use std::cell::RefCell;

let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
let clone1 = Rc::clone(&shared);

shared.borrow_mut().push(4);
clone1.borrow_mut().push(5);
```

### 5.4 智能指针对比

| 类型 | 所有权 | 可变性 | 线程安全 | 场景 |
|------|--------|--------|---------|------|
| Box<T> | 单一 | 继承外部 | 看 T | 堆分配 |
| Rc<T> | 共享 | 不可变 | 否 | 单线程共享 |
| Arc<T> | 共享 | 不可变 | 是 | 多线程共享 |
| RefCell<T> | 单一 | 内部可变 | 否 | 运行时借用检查 |
| Mutex<T> | 共享 | 内部可变 | 是 | 多线程互斥 |

---

## 6. 并发编程

### 6.1 线程

```rust
use std::thread;

// 创建线程
let handle = thread::spawn(|| {
    println!("Hello from thread!");
});

handle.join().unwrap();  // 等待线程结束

// move 捕获
let v = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("{:?}", v);  // v 所有权移入线程
});
```

### 6.2 Send 与 Sync

```rust
// Send：可以安全地在线程间转移所有权
// Sync：可以安全地在线程间共享引用（&T 是 Send 则 T 是 Sync）

// 自动实现：大多数类型
// 不实现 Send：Rc<T>（引用计数非原子）
// 不实现 Sync：RefCell<T>、Cell<T>（非线程安全内部可变）

// 手动实现（unsafe，需保证安全性）
unsafe impl Send for MyType {}
unsafe impl Sync for MyType {}
```

### 6.3 Mutex 与 RwLock

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());

// RwLock：读写锁，多读单写
use std::sync::RwLock;

let lock = RwLock::new(5);

// 多个读者
{
    let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();
    println!("{}, {}", *r1, *r2);
}

// 单个写者
{
    let mut w = lock.write().unwrap();
    *w += 1;
}
```

### 6.4 Channel

```rust
use std::sync::mpsc;  // multi-producer, single-consumer
use std::thread;

// 创建通道
let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send("hello").unwrap();
});

let received = rx.recv().unwrap();
println!("Got: {}", received);

// 多生产者
let (tx, rx) = mpsc::channel();
let tx2 = tx.clone();

thread::spawn(move || { tx.send(1).unwrap(); });
thread::spawn(move || { tx2.send(2).unwrap(); });

for received in rx {
    println!("Got: {}", received);
}
```

---

## 7. 异步编程

### 7.1 async/await 基础

```rust
// async 函数返回 Future
async fn fetch_data() -> String {
    // 模拟异步操作
    tokio::time::sleep(Duration::from_secs(1)).await;
    String::from("data")
}

// await 等待 Future 完成
async fn main() {
    let data = fetch_data().await;
    println!("{}", data);
}

// Future trait
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),    // 完成
    Pending,     // 未完成
}
```

### 7.2 Tokio 运行时

```rust
use tokio;

#[tokio::main]
async fn main() {
    // 并发执行
    let (result1, result2) = tokio::join!(
        fetch_data("url1"),
        fetch_data("url2"),
    );
    
    // spawn 独立任务
    let handle = tokio::spawn(async {
        // 后台任务
    });
    
    // 超时
    let result = tokio::time::timeout(
        Duration::from_secs(5),
        fetch_data("url"),
    ).await;
}
```

### 7.3 异步与同步边界

```rust
// 在同步代码中运行异步
let rt = tokio::runtime::Runtime::new().unwrap();
let result = rt.block_on(async {
    fetch_data().await
});

// spawn_blocking：在异步中运行阻塞代码
async fn process() {
    let result = tokio::task::spawn_blocking(|| {
        // CPU 密集型或阻塞 I/O
        std::thread::sleep(Duration::from_secs(1));
        42
    }).await.unwrap();
}
```

---

## 8. 宏系统

### 8.1 声明式宏 (macro_rules!)

```rust
// 定义宏
macro_rules! vec {
    // 模式匹配
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}

// 使用
let v = vec![1, 2, 3];

// 常用片段类型：
// expr  - 表达式
// ident - 标识符
// ty    - 类型
// tt    - token tree
// stmt  - 语句
// block - 代码块
```

### 8.2 过程宏

```rust
// 派生宏（最常用）
use proc_macro::TokenStream;

#[proc_macro_derive(MyTrait)]
pub fn my_trait_derive(input: TokenStream) -> TokenStream {
    // 解析输入
    let ast = syn::parse(input).unwrap();
    // 生成代码
    impl_my_trait(&ast)
}

// 使用
#[derive(MyTrait)]
struct MyStruct { ... }

// 属性宏
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
    // ...
}

// 使用
#[route(GET, "/")]
fn index() { ... }
```

---

## 9. 错误处理

### 9.1 Result 与 Option

```rust
// Result<T, E>
enum Result<T, E> {
    Ok(T),
    Err(E),
}

fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

// Option<T>
enum Option<T> {
    Some(T),
    None,
}

fn find(vec: &[i32], target: i32) -> Option<usize> {
    for (i, &v) in vec.iter().enumerate() {
        if v == target {
            return Some(i);
        }
    }
    None
}
```

### 9.2 错误传播

```rust
// ? 操作符：自动传播错误
fn read_file() -> Result<String, io::Error> {
    let mut file = File::open("file.txt")?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}

// 等价于
fn read_file() -> Result<String, io::Error> {
    let mut file = match File::open("file.txt") {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    // ...
}
```

### 9.3 自定义错误

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),
    
    #[error("Custom error: {message}")]
    Custom { message: String },
}

fn process() -> Result<(), MyError> {
    let content = std::fs::read_to_string("file.txt")?;
    let num: i32 = content.trim().parse()?;
    if num < 0 {
        return Err(MyError::Custom { 
            message: "negative number".into() 
        });
    }
    Ok(())
}
```

---

## 10. 面试题自查

**Q1: Rust 所有权三大规则是什么？**

每个值有且只有一个所有者；所有者离开作用域时值被丢弃；同一时刻要么一个可变引用，要么多个不可变引用。

**Q2: Move 和 Copy 的区别？**

Copy 是栈上简单类型的隐式复制，原值仍有效；Move 是所有权转移，原值失效。实现 Copy 的类型：整数、浮点、布尔、字符、以及元素都是 Copy 的元组/数组。

**Q3: 什么是生命周期？为什么需要标注？**

生命周期是引用有效的作用域范围。标注帮助编译器理解多个引用之间的关系，确保返回的引用在使用时仍然有效，防止悬垂引用。

**Q4: Rc<T> 和 Arc<T> 的区别？**

Rc 是单线程引用计数（非原子），不能在线程间共享；Arc 是原子引用计数，实现了 Send + Sync，可安全地在线程间共享。

**Q5: RefCell<T> 的作用是什么？**

提供运行时借用检查的内部可变性。在编译时无法确定借用安全性时使用，违反借用规则会在运行时 panic 而非编译错误。

**Q6: Send 和 Sync trait 的含义？**

Send 表示类型可以安全地在线程间转移所有权；Sync 表示类型可以安全地在线程间共享引用（&T 是 Send 则 T 是 Sync）。

**Q7: async/await 的底层原理？**

async 函数返回实现 Future trait 的状态机。await 调用 poll 方法，返回 Ready 时继续执行，返回 Pending 时让出控制权。运行时负责在 Future 就绪时重新调度。

**Q8: Box<T>、Rc<T>、Arc<T> 分别在什么场景使用？**

Box：堆分配单一所有者数据、递归类型；Rc：单线程多所有者共享不可变数据；Arc：多线程共享数据（通常配合 Mutex 使用）。

**Q9: 声明式宏和过程宏的区别？**

声明式宏（macro_rules!）基于模式匹配替换；过程宏操作 TokenStream，可以解析和生成任意代码，功能更强大（派生宏、属性宏、函数式宏）。

**Q10: ? 操作符做了什么？**

自动传播错误。成功时提取 Ok/Some 的值继续执行；失败时调用 From trait 转换错误类型并提前返回 Err/None。

**Q11: 什么是零成本抽象？**

高级抽象（如迭代器、闭包）在编译后与手写底层代码性能相同。编译器会进行内联、单态化等优化，无运行时开销。

**Q12: Rust 如何防止数据竞争？**

通过所有权和借用规则：同一时刻只能有一个可变引用，或多个不可变引用。Sync/Send trait 保证只有线程安全的类型才能跨线程共享。

**Q13: Pin<T> 的作用是什么？**

防止值被移动。用于自引用结构和 async 状态机，确保内存地址稳定。Pin<&mut T> 的值不能被 move，除非 T: Unpin。

**Q14: 闭包如何捕获环境变量？**

三种方式：&T（借用）、&mut T（可变借用）、T（获取所有权）。编译器自动推断最少权限，使用 move 强制获取所有权。

**Q15: Rust 的 trait object 和泛型分别何时使用？**

泛型：编译时单态化，零运行时开销，代码可能膨胀；trait object（dyn）：运行时动态分发，有虚表开销，但代码更小且支持异构集合。
