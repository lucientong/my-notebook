# JavaScript 与 TypeScript 核心知识

---

## 📑 目录

### JavaScript 核心
1. [作用域与闭包](#作用域与闭包)
2. [this 绑定](#this-绑定)
3. [原型与继承](#原型与继承)
4. [事件循环](#事件循环)
5. [异步编程](#异步编程)
6. [内存管理](#内存管理)
7. [ES6+ 核心特性深入](#es6-核心特性深入)

### TypeScript 核心
8. [类型系统](#类型系统)
9. [高级类型](#高级类型)
10. [工程实践](#工程实践)

### 自查与实战
11. [面试题自查](#面试题自查)
12. [实战案例](#实战案例)

---

## JavaScript 核心

### 作用域与闭包

#### 值得深挖

**1. 词法作用域 vs 动态作用域**

JavaScript 使用词法作用域（静态作用域），函数的作用域在函数**定义时**确定，而非**调用时**。

```javascript
const name = 'global';

function outer() {
  const name = 'outer';

  function inner() {
    console.log(name); // 'outer' —— 定义时确定
  }

  return inner;
}

const fn = outer();
fn(); // 输出 'outer'，而非 'global'
```

**面试追问**：
- 解释为什么输出 `'outer'` 而非 `'global'`？
- 如果 JavaScript 是动态作用域，输出会是什么？

---

**2. `var` / `let` / `const`**

| 特性 | `var` | `let` | `const` |
|------|-------|-------|---------|
| 作用域 | 函数作用域 | 块级作用域 | 块级作用域 |
| 变量提升 | 提升并初始化为 `undefined` | 提升但处于 TDZ | 提升但处于 TDZ |
| 重复声明 | 允许 | 不允许 | 不允许 |
| 重新绑定 | 允许 | 允许 | 不允许（对象内部属性仍可变） |

```javascript
console.log(a); // undefined —— var 提升
var a = 1;

// console.log(b); // ReferenceError —— TDZ
let b = 2;

const obj = { x: 1 };
obj.x = 2;        // ✅ 改的是属性
// obj = {};      // ❌ 不能重新绑定

for (var i = 0; i < 3; i++) {}
console.log(i);   // 3 —— 泄漏到外层

for (let j = 0; j < 3; j++) {}
// console.log(j); // ReferenceError
```

工程默认：`const` → 需要重新赋值再用 `let` → 尽量不用 `var`。

---

**3. `==` 与 `===`**

- `===`：严格相等，**不做**类型转换。
- `==`：会按抽象相等算法做隐式转换，规则复杂。

```javascript
'' == 0          // true
0 == '0'         // true
null == undefined // true
null === undefined // false
NaN === NaN      // false —— 用 Number.isNaN / Object.is
Object.is(+0, -0) // false
```

团队禁用 `==` 的核心不是洁癖，而是减少隐式转换带来的线上歧义。明确需要「`null`/`undefined` 一并处理」时，优先写 `x == null` 或显式判断，并在约定中说明。

---

**4. 闭包（Closure）**

**定义**：函数能够访问其外部词法环境中的变量，即使外部函数已经返回。

**更准确的原理表述**：
- 函数创建时会保存对周围词法环境的引用（作用域链）
- 只要闭包仍可达，**被引用的变量**就不能被 GC
- 不是「整个执行上下文永远不销毁」，而是：**外部环境中仍被闭包引用的绑定会保留**；未被引用的局部变量可以被优化掉（引擎视实现而定）

```javascript
function createCounter() {
  let count = 0;

  return {
    increment() { return ++count; },
    decrement() { return --count; },
    getCount() { return count; }
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.getCount());  // 1
// count 只能通过方法访问
```

**闭包的应用场景**：

1. **数据封装 / 私有变量**
```javascript
function createUser(name) {
  let balance = 0;
  return {
    getName() { return name; },
    deposit(amount) {
      if (amount > 0) balance += amount;
      return balance;
    },
    getBalance() { return balance; }
  };
}
```

2. **柯里化（Currying）**
```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return (...moreArgs) => curried(...args, ...moreArgs);
  };
}

const add = (a, b, c) => a + b + c;
console.log(curry(add)(1)(2)(3)); // 6
```

3. **记忆化（Memoization）**
```javascript
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

**闭包常见陷阱**：

```javascript
// 陷阱：循环 + var —— 共享同一个绑定
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 1000); // 5 5 5 5 5
}

// 方案1：let —— 每次迭代新绑定
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 1000); // 0..4
}

// 方案2：IIFE 捕获
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 1000);
  })(i);
}
```

**面试追问**：
- 闭包一定导致内存泄漏吗？什么情况下才会？
- 如何用 Heap Snapshot 证明某个闭包仍持有大对象？

---

### this 绑定

`this` 在**运行时**由**调用方式**决定（箭头函数除外）。优先级从高到低：

**规则1：`new` 绑定**
```javascript
function Person(name) {
  this.name = name;
}
const person = new Person('Alice');
console.log(person.name); // 'Alice'
```

**规则2：显式绑定（`call` / `apply` / `bind`）**
```javascript
function greet() {
  console.log(`Hello, ${this.name}`);
}
const user = { name: 'Bob' };
greet.call(user);  // 'Hello, Bob'
greet.apply(user); // 'Hello, Bob'
const bound = greet.bind(user);
bound();           // 'Hello, Bob'
```

**规则3：隐式绑定（对象方法调用）**
```javascript
const obj = {
  name: 'Charlie',
  greet() { console.log(`Hello, ${this.name}`); }
};
obj.greet(); // 'Hello, Charlie'

const fn = obj.greet;
fn(); // 非严格：this → 全局对象，name 为 undefined，输出 'Hello, undefined'
      // 严格模式 / 模块 / class 方法抽出后：this 为 undefined，访问 this.name 会 TypeError
```

**规则4：默认绑定**
```javascript
function showThis() {
  'use strict';
  console.log(this); // undefined
}
showThis();

function showThisSloppy() {
  console.log(this); // 浏览器：window；Node CJS：global（勿与模块顶层 this 混淆）
}
showThisSloppy();
```

**箭头函数**：没有自己的 `this`，捕获外层词法 `this`；`call`/`apply`/`bind` 不能改其 `this`。

```javascript
const obj = {
  name: 'David',
  regularFn() { console.log(this.name); },          // 'David'
  arrowFn: () => { console.log(this.name); },       // 取决于外层 this（常为 undefined）
  nested() {
    const arrow = () => console.log(this.name);     // 'David'
    arrow();
  }
};
```

**React class 组件中的 this**（历史考点，现代多用函数组件 + hooks）：

```javascript
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }
  handleClickArrow = () => {
    this.setState({ count: this.state.count + 1 });
  };
}
```

**面试追问**：
- `obj.method()` 与 `const f = obj.method; f()` 为什么结果不同？
- `bind` 后的函数再用 `call` 能改 `this` 吗？和 `new boundFn()` 呢？

---
### 原型与继承

#### 值得深挖

**1. 原型链（Prototype Chain）**

- 每个对象有内部槽 `[[Prototype]]`（`Object.getPrototypeOf` / 遗留的 `__proto__`）
- 函数有 `prototype`，供 `new` 出来的实例链接
- 属性查找沿原型链向上，直到 `null`

```javascript
function Person(name) {
  this.name = name;
}
Person.prototype.sayHello = function () {
  console.log(`Hello, ${this.name}`);
};

const alice = new Person('Alice');
// alice → Person.prototype → Object.prototype → null
console.log(Object.getPrototypeOf(alice) === Person.prototype); // true
alice.sayHello(); // 'Hello, Alice'
```

**属性查找**：自身 → 原型 → … → `null`；`hasOwnProperty` / `Object.hasOwn` 只看自身。

---

**2. 继承模式（由浅入深）**

**方案1：原型链继承（不推荐）**
```javascript
function Parent() { this.colors = ['red', 'blue']; }
function Child() {}
Child.prototype = new Parent(); // 引用类型被实例共享
```

**方案2：盗用构造函数 / 构造函数继承（不推荐）**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayName = function () { console.log(this.name); };

function Child(name) {
  Parent.call(this, name); // 仅实例属性
}
const child = new Child('Alice');
// child.sayName(); // TypeError —— 拿不到原型方法
```

**方案3：组合继承（经典写法，会调两次父构造）**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayName = function () { console.log(this.name); };

function Child(name, age) {
  Parent.call(this, name); // 第 1 次：实例属性
  this.age = age;
}
Child.prototype = new Parent(); // 第 2 次：仅为了挂原型（多余且污染）
Child.prototype.constructor = Child;
```

缺点：父构造被调用两次；`Child.prototype` 上会留下多余的实例属性。

**方案4：寄生组合继承（ES5 最佳实践，Class 的语义基础）**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayName = function () { console.log(this.name); };

function Child(name, age) {
  Parent.call(this, name); // 只调用一次父构造，拿实例属性
  this.age = age;
}

// Object.create 只建立原型链接，不会执行 Parent
Child.prototype = Object.create(Parent.prototype);
Child.prototype.constructor = Child;

Child.prototype.sayAge = function () { console.log(this.age); };

const child = new Child('Alice', 10);
child.sayName(); // 'Alice'
child.sayAge();  // 10
```

> 常见误区：把「`Object.create(Parent.prototype)`」仍说成「组合继承调两次构造」。
> `Object.create` **不会**调用 `Parent`；只有 `Parent.call(this)` 那一次。
> 真正「调两次」的是 `Child.prototype = new Parent()` 那种经典组合继承。

**方案5：ES6 `class`（推荐）**
```javascript
class Parent {
  constructor(name) {
    this.name = name;
    this.colors = ['red', 'blue'];
  }
  sayName() { console.log(this.name); }
}

class Child extends Parent {
  constructor(name, age) {
    super(name); // 必须先 super
    this.age = age;
  }
  sayAge() { console.log(this.age); }
}
```

`class` 仍是原型继承的语法糖，但静态方法继承、`super`、不可枚举方法等细节更规范。

---

### 事件循环

JavaScript 运行时靠**事件循环**调度任务；渲染、计时器、I/O 回调都不在「纯同步栈」里一口气跑完。

#### 浏览器模型（简化但面试够用）

**队列与阶段（不要把渲染塞进宏任务列表）**：

| 类别 | 典型来源 | 说明 |
|------|----------|------|
| 宏任务（task / macrotask） | `setTimeout`、`setInterval`、`setImmediate`（Node）、I/O、UI 事件、`postMessage` | 每次循环通常取 **一个** task |
| 微任务（microtask） | `Promise.then/catch/finally`、`queueMicrotask`、`MutationObserver` | **一个 task 结束后清空整条微任务队列**（清空过程中新产生的微任务也会继续跑） |
| 渲染相关 | 样式计算、布局、绘制 | **不是**宏任务；在帧机会到来且浏览器认为需要时进行 |
| `requestAnimationFrame` | 下一帧绘制前的回调 | 与渲染管线对齐，**不要**说成普通宏任务 |

**一轮典型顺序（概念模型）**：
1. 执行一个宏任务（或脚本本身）
2. 清空微任务队列
3. 如需要且到了帧时机：`rAF` → 渲染
4. 取下一个宏任务……

```javascript
console.log('1-同步');

setTimeout(() => console.log('2-宏任务'), 0);

Promise.resolve().then(() => console.log('3-微任务'));
queueMicrotask(() => console.log('3b-queueMicrotask'));

requestAnimationFrame(() => console.log('4-rAF')); // 相对宏/微，时机在「即将渲染」

console.log('5-同步');
// 同步 → 微任务 →（之后才是 timeout / rAF，具体交错依赖实现与是否渲染）
```

**复杂示例（宏任务之间夹微任务）**：

```javascript
console.log('start');

setTimeout(() => {
  console.log('timeout1');
  Promise.resolve().then(() => console.log('promise3'));
}, 0);

Promise.resolve().then(() => {
  console.log('promise1');
  setTimeout(() => console.log('timeout2'), 0);
}).then(() => console.log('promise2'));

console.log('end');
// start → end → promise1 → promise2 → timeout1 → promise3 → timeout2
```

#### Node.js 事件循环阶段

Node 基于 libuv，阶段更细（常见考点）：

`timers` → `pending callbacks` → `idle/prepare` → `poll` → `check`（`setImmediate`）→ `close callbacks`

- `process.nextTick`：**先于** Promise 微任务（nextTick 队列清空后再跑 Promise microtask）
- 同样代码在浏览器与 Node 输出可能不同——要能举 `setTimeout` vs `setImmediate`、nextTick 例子

```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
// 典型：nextTick → promise →（timeout / immediate 相对顺序视上下文而定）
```

**生产实践**：
- 微任务过多会拖住渲染帧 → 拆分任务、`scheduler`/`requestIdleCallback`、Worker
- 不要用「死循环 `then`」模拟长任务

**面试追问**：
- 为什么说「UI 渲染不是宏任务」？它和微任务、`rAF` 的先后关系是什么？
- `queueMicrotask` 和 `Promise.resolve().then` 有何差异？
- 为什么 `process.nextTick` 递归可能导致 I/O 饿死？

---

### 异步编程

#### 1. Promise 基础

三种状态：`pending` → `fulfilled` / `rejected`（不可逆）。

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    Math.random() > 0.5 ? resolve('成功') : reject(new Error('失败'));
  }, 1000);
});

promise
  .then((result) => console.log(result))
  .catch((error) => console.error(error))
  .finally(() => console.log('完成'));
```

#### 2. 组合子（静态方法）

```javascript
// all：全部成功；一个失败立即失败；空可迭代 → resolve([])
Promise.all([fetch('/api/user'), fetch('/api/posts')])
  .then((responses) => Promise.all(responses.map((r) => r.json())))
  .catch((err) => console.error('某个失败', err));

// allSettled：等全部结束
Promise.allSettled([fetch('/api/user'), fetch('/api/bad')]).then((results) => {
  results.forEach((r, i) => {
    console.log(i, r.status, r.status === 'fulfilled' ? r.value : r.reason);
  });
});

// race：最先 settled（成功或失败）
Promise.race([
  fetch('/api/fast'),
  new Promise((_, reject) => setTimeout(() => reject(new Error('超时')), 3000)),
]);

// any：最先 fulfilled；全部失败 → AggregateError
Promise.any([fetch('/api/a'), fetch('/api/b')]);
```

| 方法 | 成功条件 | 失败条件 |
|------|----------|----------|
| `Promise.all` | 全部 fulfilled | 第一个 rejected |
| `Promise.allSettled` | 全部 settled | 本身不因子 Promise 失败而 reject |
| `Promise.race` | 第一个 settled | 第一个若是 reject 则失败 |
| `Promise.any` | 第一个 fulfilled | 全部 rejected |

#### 3. 手写 Promise（教学版：用微任务，不用 setTimeout）

> 规范要求 `then` 回调以 **microtask** 调度。用 `setTimeout` 会落到宏任务，顺序与真 Promise 不一致。

```javascript
const runMicrotask =
  typeof queueMicrotask === 'function'
    ? queueMicrotask
    : (fn) => Promise.resolve().then(fn);

class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];

    const resolve = (value) => {
      if (this.state !== 'pending') return;
      // 简化：未实现完整 thenable 递归解析
      this.state = 'fulfilled';
      this.value = value;
      this.onFulfilledCallbacks.forEach((fn) => fn());
    };

    const reject = (reason) => {
      if (this.state !== 'pending') return;
      this.state = 'rejected';
      this.reason = reason;
      this.onRejectedCallbacks.forEach((fn) => fn());
    };

    try {
      executor(resolve, reject);
    } catch (e) {
      reject(e);
    }
  }

  then(onFulfilled, onRejected) {
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : (v) => v;
    onRejected =
      typeof onRejected === 'function'
        ? onRejected
        : (e) => {
            throw e;
          };

    const promise2 = new MyPromise((resolve, reject) => {
      const handle = (cb, arg) => {
        runMicrotask(() => {
          try {
            const x = cb(arg);
            // 完整实现还需 resolvePromise(promise2, x, ...)
            resolve(x);
          } catch (err) {
            reject(err);
          }
        });
      };

      if (this.state === 'fulfilled') handle(onFulfilled, this.value);
      else if (this.state === 'rejected') handle(onRejected, this.reason);
      else {
        this.onFulfilledCallbacks.push(() => handle(onFulfilled, this.value));
        this.onRejectedCallbacks.push(() => handle(onRejected, this.reason));
      }
    });

    return promise2;
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }

  finally(callback) {
    return this.then(
      (value) => MyPromise.resolve(callback()).then(() => value),
      (reason) =>
        MyPromise.resolve(callback()).then(() => {
          throw reason;
        })
    );
  }

  static resolve(value) {
    if (value instanceof MyPromise) return value;
    return new MyPromise((resolve) => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const list = Array.from(promises);
      if (list.length === 0) {
        resolve([]);
        return;
      }
      const results = new Array(list.length);
      let count = 0;
      list.forEach((p, index) => {
        MyPromise.resolve(p).then((value) => {
          results[index] = value;
          count++;
          if (count === list.length) resolve(results);
        }, reject);
      });
    });
  }

  static allSettled(promises) {
    return new MyPromise((resolve) => {
      const list = Array.from(promises);
      if (list.length === 0) {
        resolve([]);
        return;
      }
      const results = new Array(list.length);
      let count = 0;
      list.forEach((p, index) => {
        MyPromise.resolve(p).then(
          (value) => {
            results[index] = { status: 'fulfilled', value };
          },
          (reason) => {
            results[index] = { status: 'rejected', reason };
          }
        ).then(() => {
          count++;
          if (count === list.length) resolve(results);
        });
      });
    });
  }

  static race(promises) {
    return new MyPromise((resolve, reject) => {
      for (const p of promises) {
        MyPromise.resolve(p).then(resolve, reject);
      }
    });
  }
}
```

手写要点：
- 空数组：`all` / `allSettled` 立即 `resolve([])`
- `Promise.resolve` 包装非 thenable
- 用 index 保序，不要 `push`
- 微任务调度，禁止用 `setTimeout` 冒充

#### 4. async / await

`async` 函数始终返回 Promise；`await` 暂停的是该异步函数的执行，背后仍是 Promise + 微任务。

```javascript
async function fetchUser() {
  try {
    const response = await fetch('/api/user');
    return await response.json();
  } catch (error) {
    console.error(error);
    throw error;
  }
}

// ✅ 并行
async function fetchAll() {
  const [user, posts] = await Promise.all([
    fetch('/api/user').then((r) => r.json()),
    fetch('/api/posts').then((r) => r.json()),
  ]);
  return { user, posts };
}
```

**顶层 await（ESM）**：在 **ES Module** 顶层可以直接 `await`，不必包一层 `async` 函数。浏览器原生 module、Node 的 `"type": "module"` / `.mjs` 均支持。CJS、普通 `<script>` 仍不行。

```javascript
// main.mjs
const config = await fetch('/config.json').then((r) => r.json());
export default config;
```

**错误处理模式**：统一 try/catch、错误分类；并发用 `allSettled` 做隔离；可用小工具 `to(promise) → [err, data]`（注意 TS 推断变弱）。

#### 5. AbortController、超时与并发池

```javascript
function fetchWithTimeout(url, ms, init = {}) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(new Error('timeout')), ms);
  return fetch(url, { ...init, signal: controller.signal }).finally(() =>
    clearTimeout(timer)
  );
}

// 与 Promise.race 超时相比：AbortController 能真正取消 fetch，减少浪费

async function mapPool(items, limit, worker) {
  const ret = new Array(items.length);
  let i = 0;
  async function run() {
    while (i < items.length) {
      const cur = i++;
      ret[cur] = await worker(items[cur], cur);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, () => run()));
  return ret;
}

// 使用：最多 3 并发
await mapPool(urls, 3, (url) => fetchWithTimeout(url, 5000).then((r) => r.json()));
```

**面试追问**：
- `Promise.race` 超时后原请求还在飞吗？如何取消？
- 并发池为什么比无脑 `Promise.all(1000个)` 更稳？

---
### 内存管理

#### 1. 垃圾回收（GC）概览

引擎普遍使用**可达性分析** + **标记清除 / 标记整理** 一类算法：从根（全局、调用栈、寄存器等）出发标记可达对象，不可达对象可回收。

#### 2. V8 视角（面试加分）

**隐藏类（Hidden Class / Maps）与内联缓存（IC）**
- 对象形状相似时共享隐藏类，属性访问走快速路径
- 同形对象上的方法调用可通过 IC 缓存命中
- **deopt（去优化）**：形状突变、类型反转、`arguments` 滥用等会让优化代码回退，表现为「偶发卡顿」

```javascript
// 利于隐藏类稳定：同构初始化
function Point(x, y) {
  this.x = x;
  this.y = y;
}

// 不利于：运行时乱加属性 / 删属性 / 改变属性顺序
const a = { x: 1, y: 2 };
const b = { y: 2, x: 1 }; // 形状可能不同
```

**分代 GC（Generational）**
- **新生代（Young / Nursery）**：短命对象，Scavenge（半空间拷贝）为主，频繁且快
- **老生代（Old）**：长寿对象，Mark-Sweep / Mark-Compact，开销更大
- 晋升：熬过几次新生代回收的对象进入老生代

**Heap Snapshot 排查**
1. Chrome DevTools → Memory → Heap snapshot
2. 操作前后各拍一张，Comparison 看增长
3. 关注 Detached HTMLElement、闭包 Retainer Path
4. Allocation instrumentation / Timeline 找「谁在分配」

#### 3. 常见泄漏场景

**全局变量**
```javascript
function leak() {
  accidentalGlobal = new Array(1e6); // 缺 var/let/const
}
```

**闭包误持有大对象**
```javascript
function createClosure() {
  const largeData = new Array(1e6).fill('data');
  const summary = largeData.length; // 只保留需要的
  return () => console.log(summary);
}
```

**事件监听未移除 / 定时器未清理**
```javascript
class Component {
  constructor(el) {
    this.el = el;
    this.onClick = () => {};
    el.addEventListener('click', this.onClick);
  }
  destroy() {
    this.el.removeEventListener('click', this.onClick);
  }
}
```

#### 4. 对象池（热点分配场景）

```javascript
class ObjectPool {
  constructor(factory, reset, initialSize = 10) {
    this.factory = factory;
    this.reset = reset;
    this.pool = Array.from({ length: initialSize }, () => factory());
  }
  acquire() {
    return this.pool.pop() || this.factory();
  }
  release(obj) {
    this.reset(obj);
    this.pool.push(obj);
  }
}
```

**面试追问**：
- 为什么「对象形状稳定」能提升 V8 性能？
- 新生代和老生代的回收策略差异是什么？
- Heap Snapshot 里 Retainer 链路怎么读？

---
### ES6+ 核心特性深入
#### 1. Proxy 与 Reflect

**原理说明**

`Proxy` 用于创建对象的**代理层**，可拦截并自定义对象的基本操作（属性读取、赋值、函数调用等）。`Reflect` 是与 Proxy trap 一一对应的**静态方法集合**，提供了对象操作的默认行为，确保在自定义 trap 内能安全地执行原始操作。

**设计哲学**：
- Proxy 实现了"元编程"——在语言层面拦截和修改对象的默认行为
- Reflect 将原本分散在 `Object`、`Function`、`delete` 等处的操作统一到一个命名空间
- 两者配合使用，形成"拦截 + 转发"的标准范式

**13 种 trap 一览**：

| trap | 拦截操作 | Reflect 对应 |
|------|----------|-------------|
| `get` | 属性读取 | `Reflect.get` |
| `set` | 属性赋值 | `Reflect.set` |
| `has` | `in` 操作符 | `Reflect.has` |
| `deleteProperty` | `delete` 操作 | `Reflect.deleteProperty` |
| `ownKeys` | `Object.keys` / `for...in` | `Reflect.ownKeys` |
| `getOwnPropertyDescriptor` | 属性描述符获取 | `Reflect.getOwnPropertyDescriptor` |
| `defineProperty` | 定义属性 | `Reflect.defineProperty` |
| `getPrototypeOf` | 原型获取 | `Reflect.getPrototypeOf` |
| `setPrototypeOf` | 原型设置 | `Reflect.setPrototypeOf` |
| `isExtensible` | 可扩展性检查 | `Reflect.isExtensible` |
| `preventExtensions` | 阻止扩展 | `Reflect.preventExtensions` |
| `apply` | 函数调用 | `Reflect.apply` |
| `construct` | `new` 调用 | `Reflect.construct` |

**代码示例：Vue3 响应式核心原理（简化版）**

```javascript
// ---------- 依赖收集系统 ----------
let activeEffect = null;
const targetMap = new WeakMap(); // target -> Map<key, Set<effect>>

function track(target, key) {
  if (!activeEffect) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    depsMap = new Map();
    targetMap.set(target, depsMap);
  }
  let deps = depsMap.get(key);
  if (!deps) {
    deps = new Set();
    depsMap.set(key, deps);
  }
  deps.add(activeEffect);
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const deps = depsMap.get(key);
  if (deps) {
    deps.forEach(effect => effect());
  }
}

// ---------- 响应式代理 ----------
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key);                    // 读取时收集依赖
      const result = Reflect.get(target, key, receiver);
      // 深层代理：如果值是对象，递归转换
      if (typeof result === 'object' && result !== null) {
        return reactive(result);
      }
      return result;
    },
    set(target, key, value, receiver) {
      const oldValue = target[key];
      const result = Reflect.set(target, key, value, receiver);
      if (oldValue !== value) {
        trigger(target, key);                // 赋值时触发更新
      }
      return result;
    },
    deleteProperty(target, key) {
      const hadKey = key in target;
      const result = Reflect.deleteProperty(target, key);
      if (hadKey && result) {
        trigger(target, key);
      }
      return result;
    }
  });
}

// ---------- 副作用函数 ----------
function effect(fn) {
  activeEffect = fn;
  fn(); // 首次执行，触发 get → track
  activeEffect = null;
}

// ---------- 使用 ----------
const state = reactive({ count: 0, user: { name: 'Alice' } });

effect(() => {
  console.log('count 变化:', state.count);
});

state.count++;   // 控制台输出: count 变化: 1
state.count = 5; // 控制台输出: count 变化: 5
```

**面试考点**：
- **Proxy vs Object.defineProperty**：Proxy 可拦截新增属性、数组下标变化、`delete` 操作，而 `Object.defineProperty` 不能，这就是 Vue3 比 Vue2 响应式更完善的根本原因
- **为什么要用 `Reflect.get` 而不是 `target[key]`**：`Reflect.get` 能正确处理 `receiver`（代理链中的 this 指向），保证继承场景下行为正确
- **手写题**：实现一个 `reactive` 函数，支持依赖收集和触发更新

---

#### 2. Symbol

**原理说明**

`Symbol` 是 ES6 引入的**第七种原始数据类型**。每个 Symbol 值都是唯一的，主要用于创建对象的非字符串属性键，避免属性名冲突。

```javascript
// ---- 基础：唯一性 ----
const s1 = Symbol('description');
const s2 = Symbol('description');
console.log(s1 === s2); // false —— 描述相同但值不同

// ---- Symbol.for：全局注册表（可复用） ----
const s3 = Symbol.for('shared');
const s4 = Symbol.for('shared');
console.log(s3 === s4); // true —— 全局注册表中同一个

// ---- 作为对象属性键 ----
const ID = Symbol('id');
const user = {
  name: 'Alice',
  [ID]: 12345   // Symbol 属性不会被常规遍历
};

console.log(user[ID]);              // 12345
console.log(Object.keys(user));     // ['name'] —— Symbol 属性不可见
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
console.log(Reflect.ownKeys(user)); // ['name', Symbol(id)]
```

**Well-known Symbols（内置 Symbol）**

| Symbol | 用途 | 典型场景 |
|--------|------|---------|
| `Symbol.iterator` | 定义默认迭代器 | `for...of`、展开运算符 |
| `Symbol.asyncIterator` | 定义异步迭代器 | `for await...of` |
| `Symbol.hasInstance` | 自定义 `instanceof` | 类型判断定制 |
| `Symbol.toPrimitive` | 对象转原始值 | `+obj`、模板字符串 |
| `Symbol.toStringTag` | 自定义 `Object.prototype.toString` | 调试标识 |
| `Symbol.species` | 派生对象的构造函数 | 子类化内置对象 |

**代码示例：Symbol.iterator 实现可迭代协议**

```javascript
// 自定义区间对象，支持 for...of
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return {
      next() {
        if (current <= end) {
          return { value: current++, done: false };
        }
        return { done: true };
      }
    };
  }
}

const range = new Range(1, 5);
for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}
console.log([...range]); // [1, 2, 3, 4, 5]

// Symbol.toPrimitive 定制类型转换
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }

  [Symbol.toPrimitive](hint) {
    if (hint === 'number') return this.amount;
    if (hint === 'string') return `${this.amount} ${this.currency}`;
    return this.amount; // default
  }
}

const price = new Money(99.9, 'CNY');
console.log(+price);        // 99.9
console.log(`${price}`);    // '99.9 CNY'
console.log(price + 1);     // 100.9
```

**面试考点**：
- Symbol 的唯一性保证和 `Symbol.for` 全局注册表的区别
- `for...in` / `Object.keys` / `Object.getOwnPropertySymbols` / `Reflect.ownKeys` 对 Symbol 属性的可见性差异
- 手写一个可迭代对象（实现 `Symbol.iterator`）

---

#### 3. Iterator 与 Generator

**原理说明**

**迭代器协议**：任何对象只要实现 `next()` 方法（返回 `{value, done}`），就是一个迭代器。
**可迭代协议**：任何对象只要实现 `[Symbol.iterator]()` 方法（返回一个迭代器），就是可迭代对象。

**Generator 函数**是一种特殊函数，使用 `function*` 声明，内部通过 `yield` 暂停执行、惰性产出值，天然满足迭代器协议。

```javascript
// ---- 手动实现迭代器 ----
function createRangeIterator(start, end) {
  let current = start;
  return {
    next() {
      if (current <= end) {
        return { value: current++, done: false };
      }
      return { value: undefined, done: true };
    },
    [Symbol.iterator]() { return this; } // 自身也是可迭代的
  };
}

const iter = createRangeIterator(1, 3);
console.log(iter.next()); // { value: 1, done: false }
console.log(iter.next()); // { value: 2, done: false }
console.log(iter.next()); // { value: 3, done: false }
console.log(iter.next()); // { value: undefined, done: true }

// ---- Generator 函数（等价但更简洁） ----
function* rangeGenerator(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

for (const num of rangeGenerator(1, 5)) {
  console.log(num); // 1, 2, 3, 4, 5
}

// ---- yield 的双向通信 ----
function* conversation() {
  const name = yield '你叫什么名字？';
  const hobby = yield `你好 ${name}，你有什么爱好？`;
  return `${name} 喜欢 ${hobby}`;
}

const talk = conversation();
console.log(talk.next());          // { value: '你叫什么名字？', done: false }
console.log(talk.next('Alice'));   // { value: '你好 Alice，你有什么爱好？', done: false }
console.log(talk.next('编程'));    // { value: 'Alice 喜欢 编程', done: true }

// ---- yield* 委托 ----
function* inner() {
  yield 'a';
  yield 'b';
}

function* outer() {
  yield 1;
  yield* inner(); // 委托给另一个 generator
  yield 2;
}

console.log([...outer()]); // [1, 'a', 'b', 2]
```

**异步 Generator：处理异步数据流**

```javascript
// 异步分页数据加载
async function* fetchPages(baseUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${baseUrl}?page=${page}`);
    const data = await response.json();
    hasMore = data.hasMore;
    page++;
    yield data.items; // 每次 yield 一页数据
  }
}

// 使用 for await...of 消费
async function loadAll() {
  const allItems = [];
  for await (const items of fetchPages('/api/posts')) {
    allItems.push(...items);
    console.log(`已加载 ${allItems.length} 条`);
  }
  return allItems;
}

// 实战：可取消的轮询
async function* poll(fn, interval) {
  while (true) {
    yield await fn();
    await new Promise(resolve => setTimeout(resolve, interval));
  }
}

async function startPolling() {
  for await (const status of poll(() => fetch('/api/status').then(r => r.json()), 3000)) {
    console.log('状态:', status);
    if (status.completed) break; // 条件满足时退出
  }
}
```

**面试考点**：
- 迭代器协议与可迭代协议的区别：`next()` vs `[Symbol.iterator]()`
- Generator 的暂停/恢复机制是如何实现 async/await 的基础（co 库原理）
- 手写题：用 Generator 实现一个惰性求值的无限序列（如斐波那契）
- 异步 Generator 在流式数据处理中的应用

---
#### 4. WeakMap / WeakSet

**原理**：弱引用键 —— 不阻止键被 GC；键不可达时条目可被清理。

| 特性 | Map / Set | WeakMap / WeakSet |
|------|-----------|-------------------|
| 键类型 | 任意值 | **对象**，以及**未注册的 Symbol**（可被 GC 的 Symbol；`Symbol.for` 注册的不行） |
| 引用 | 强引用 | 弱引用 |
| 可遍历 / `size` | 有 | 无（GC 时机不确定） |

```javascript
let obj = { name: 'temp' };
const wm = new WeakMap();
wm.set(obj, '关联数据');
obj = null; // 条目可随 GC 清理

// ES2023+：非注册 Symbol 也可作 WeakMap 键
const sym = Symbol('ephemeral');
wm.set(sym, 123);
```

**私有数据 / DOM 关联**（避免强引用泄漏）见前文章节示例思路：`WeakMap` 以实例或 DOM 节点为键。

**面试追问**：
- 为什么曾经说「键只能是对象」？现在为何允许某些 Symbol？
- 为什么不能遍历 WeakMap？

---

#### 5. structuredClone / WeakRef / FinalizationRegistry

```javascript
// structuredClone：结构化克隆（支持循环引用、Date/Map/Set 等）
const original = { d: new Date(), nested: { n: 1 } };
original.self = original;
const cloned = structuredClone(original);
console.log(cloned.self === cloned); // true
console.log(cloned.d instanceof Date); // true
// 不支持函数、DOM 节点、Symbol 作属性键等 —— 需查阅当前环境能力

// WeakRef：持有对象但不阻止 GC（谨慎使用）
let cache = new Map();
function getWidget(id) {
  const ref = cache.get(id);
  const hit = ref?.deref();
  if (hit) return hit;
  const widget = createExpensiveWidget(id);
  cache.set(id, new WeakRef(widget));
  return widget;
}

// FinalizationRegistry：对象被 GC 后的清理回调（不保证及时，勿做关键业务逻辑）
const registry = new FinalizationRegistry((heldValue) => {
  console.log('清理', heldValue);
});
(function () {
  const obj = {};
  registry.register(obj, 'resource-id');
})();
```

深拷贝选型：`structuredClone`（运行时）优先；需要函数/`undefined` 特殊策略时再手写；`JSON` 仅作简单脏拷。

---

#### 6. Promise 组合子手写（对照规范行为）

正文「异步编程」已含完整 `MyPromise`。以下是挂到原生上的精简版考点：

```javascript
Promise.myAll = function (promises) {
  return new Promise((resolve, reject) => {
    const list = Array.from(promises);
    if (list.length === 0) return resolve([]);
    const results = [];
    let count = 0;
    list.forEach((p, i) => {
      Promise.resolve(p).then((v) => {
        results[i] = v;
        if (++count === list.length) resolve(results);
      }, reject);
    });
  });
};
```

要点：空数组、保序、`Promise.resolve` 包装、快速失败。

---

#### 7. 模块系统与工程边界

**ESM vs CJS**

| 特性 | ESM | CJS |
|------|-----|-----|
| 加载 | 静态分析 | 运行时 `require` |
| 绑定 | 活绑定 | 值拷贝（导出对象属性可变是另一回事） |
| Tree Shaking | 友好 | 困难 |
| 顶层 await | 支持 | 不支持 |

```javascript
// ESM 活绑定
// counter.mjs
export let count = 0;
export function increment() { count++; }
```

**`package.json` 的 `exports` 与 dual package hazard**

```json
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./package.json": "./package.json"
  },
  "sideEffects": ["*.css", "*.scss", "./src/polyfill.js"]
}
```

- `sideEffects: false`：声明整包可安全 tree-shake；若有 CSS/polyfill 副作用，应改用**数组白名单**（上例）。
- **不要**在 JSON 里写两个同名 `"sideEffects"` 键——非法/后者覆盖前者，语义混乱。
- **Dual package hazard**：同一包同时走 CJS 与 ESM，可能加载两份副本 → 单例失效、`instanceof` 失败。缓解：统一入口、避免深路径、用 `exports` 收敛、文档规定一种消费方式。

**动态 `import()`**：返回 Promise，resolve 为模块命名空间对象；适合路由懒加载、按需 polyfill。

**Tree Shaking 失效**：模块顶层副作用、默认导出巨型对象再解构、被标记有副作用却未正确配置。

**面试追问**：
- 什么是 dual package hazard？如何在库设计里规避？
- `sideEffects` 配错会导致什么线上问题？

---

## TypeScript 核心

### 类型系统

#### 值得深挖

**1. 基础类型与安全边界**

```typescript
let num: number = 42;
let tuple: [string, number] = ['Alice', 30];

let anything: any = 'hello';
anything = 42; // 绕过检查，慎用

let value: unknown = 'hello';
// value.toUpperCase(); // ❌
if (typeof value === 'string') {
  value.toUpperCase(); // ✅
}

function throwError(message: string): never {
  throw new Error(message);
}
```

**2. 对象类型：`interface` / `type`**

```typescript
interface User {
  id: number;
  name: string;
  email?: string;
  readonly createdAt: Date;
}

type Point = { x: number; y: number };
```

选型：可扩展对象契约、声明合并 → `interface`；联合/交叉/映射/条件运算 → `type`。

**3. 函数类型与方差（`strictFunctionTypes`）**

在 `--strictFunctionTypes`（`strict` 已包含）下，**函数参数是抗变（contravariant）检查**（比较的是函数类型赋值兼容性时）。

```typescript
type Handler<T> = (event: T) => void;

interface Animal { name: string }
interface Dog extends Animal { bark(): void }

const animalHandler: Handler<Animal> = (a) => console.log(a.name);
const dogHandler: Handler<Dog> = (d) => d.bark();

// dogHandler 不能赋给 Handler<Animal>：否则可能传入 Cat
// let h: Handler<Animal> = dogHandler; // ❌ 在 strictFunctionTypes 下报错

// animalHandler 可以更安全地处理 Dog（参数更宽）
let h2: Handler<Dog> = animalHandler; // ✅
```

直觉：**返回值协变、参数抗变**——「能处理更宽输入」的函数，才能放到期望更窄输入的位置。

方法语法与函数属性在旧行为上曾有差异；以 `strictFunctionTypes` + 函数类型属性为准理解现代规则。

**4. 函数重载**

```typescript
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}
```

---
### 高级类型

#### 值得深挖

**1. 联合类型与交叉类型**

```typescript
// 联合类型（或）
type ID = string | number;

function printId(id: ID) {
  if (typeof id === 'string') {
    console.log(id.toUpperCase());
  } else {
    console.log(id.toFixed(2));
  }
}

// 交叉类型（且）
type Named = { name: string };
type Aged = { age: number };
type Person = Named & Aged;

const person: Person = {
  name: 'Alice',
  age: 30
};
```

---

**2. 类型守卫**

```typescript
// typeof 类型守卫
function padLeft(value: string, padding: string | number) {
  if (typeof padding === 'number') {
    return Array(padding + 1).join(' ') + value;
  }
  return padding + value;
}

// instanceof 类型守卫
class Bird {
  fly() { console.log('flying'); }
}

class Fish {
  swim() { console.log('swimming'); }
}

function move(animal: Bird | Fish) {
  if (animal instanceof Bird) {
    animal.fly();
  } else {
    animal.swim();
  }
}

// 自定义类型守卫
interface Cat {
  type: 'cat';
  meow: () => void;
}

interface Dog {
  type: 'dog';
  bark: () => void;
}

type Animal = Cat | Dog;

function isCat(animal: Animal): animal is Cat {
  return animal.type === 'cat';
}

function makeSound(animal: Animal) {
  if (isCat(animal)) {
    animal.meow();
  } else {
    animal.bark();
  }
}
```

---

**3. 泛型**

```typescript
// 基础泛型
function identity<T>(arg: T): T {
  return arg;
}

const num = identity(42); // T = number
const str = identity('hello'); // T = string

// 泛型约束
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(arg: T): void {
  console.log(arg.length);
}

logLength('hello'); // ✅
logLength([1, 2, 3]); // ✅
// logLength(42); // ❌ 没有 length 属性

// 泛型接口
interface GenericIdentityFn<T> {
  (arg: T): T;
}

const myIdentity: GenericIdentityFn<number> = identity;

// 泛型类
class GenericNumber<T> {
  zeroValue: T;
  add: (x: T, y: T) => T;
}

const myNumber = new GenericNumber<number>();
myNumber.zeroValue = 0;
myNumber.add = (x, y) => x + y;
```

---

**4. 工具类型**

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

// Partial：所有属性变可选
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; age?: number; }

// Required：所有属性变必填
type RequiredUser = Required<PartialUser>;

// Readonly：所有属性变只读
type ReadonlyUser = Readonly<User>;

// Pick：选取部分属性
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: number; name: string; }

// Omit：排除部分属性
type UserWithoutAge = Omit<User, 'age'>;
// { id: number; name: string; email: string; }

// Record：创建对象类型
type UserMap = Record<number, User>;
// { [key: number]: User; }

// ReturnType：获取函数返回类型
function getUser() {
  return { id: 1, name: 'Alice' };
}
type User2 = ReturnType<typeof getUser>;
// { id: number; name: string; }

// Parameters：获取函数参数类型
function create(name: string, age: number) {}
type CreateParams = Parameters<typeof create>;
// [string, number]
```

---

**5. 条件类型**

```typescript
// 基础条件类型
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false

// 分布式条件类型
type ToArray<T> = T extends any ? T[] : never;

type C = ToArray<string | number>; // string[] | number[]

// infer 关键字
type ReturnType2<T> = T extends (...args: any[]) => infer R ? R : never;

function foo() { return { x: 10, y: 20 }; }
type FooReturn = ReturnType2<typeof foo>; // { x: number; y: number; }

// 实战：深度 Readonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};

interface Config {
  db: {
    host: string;
    port: number;
  };
}

type ReadonlyConfig = DeepReadonly<Config>;
// {
//   readonly db: {
//     readonly host: string;
//     readonly port: number;
//   };
// }
```

---
### 工程实践

#### 1. tsconfig 关键项

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "strictFunctionTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "declaration": true,
    "declarationMap": true,
    "composite": true
  },
  "references": [
    { "path": "./packages/ui" },
    { "path": "./packages/core" }
  ]
}
```

| 选项 | 含义 |
|------|------|
| `moduleResolution` | `node16`/`nodenext` 贴近 Node ESM；`bundler` 适合 Vite/Webpack |
| `paths` | 编译期路径映射；运行时需打包器/工具同步 |
| `composite` + `references` | Project References，大型 monorepo 增量构建 |
| `strict` | 打开一整套严格检查（含 `strictFunctionTypes` 等） |

#### 2. 声明文件、`declare module`、Declaration Merging

```typescript
// types/shims.d.ts
declare module '*.vue' {
  import type { DefineComponent } from 'vue';
  const component: DefineComponent<object, object, unknown>;
  export default component;
}

declare module 'legacy-sdk' {
  export function init(key: string): void;
}

// Declaration Merging：同名 interface 合并
interface Window {
  __APP_VERSION__: string;
}

// 命名空间合并、模块增强（module augmentation）
declare module './event-bus' {
  interface EventMap {
    'user:login': { userId: string };
  }
}
```

库作者：`declaration: true` 产出 `.d.ts`；使用 `export =` / `export default` 时注意消费者的 `esModuleInterop`。

#### 3. 装饰器：TS 5+ 与旧 `experimentalDecorators`

| | **Stage 3 标准装饰器（TS 5+）** | **旧实验装饰器** |
|--|-------------------------------|------------------|
| 开关 | 较新目标/`decorators` 语义（随版本演进） | `experimentalDecorators` + 常配 `emitDecoratorMetadata` |
| 求值时机 / 写法 | 贴近 TC39，参数与初始化语义不同 | 历史 Angular/Nest 生态大量依赖 |
| 元数据 | 不默认 emit设计型元数据 | `emitDecoratorMetadata` 依赖 `reflect-metadata` |

迁移建议：新项目优先跟进标准装饰器；维护 Nest/Angular 老项目时明确仍在用 experimental，不要混用两套假设。

#### 4. API 类型建模

```typescript
interface UserDTO {
  id: number;
  name: string;
  email: string | null;
  created_at: string;
}

interface User {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
}

function parseUser(dto: unknown): User {
  if (
    typeof dto !== 'object' ||
    dto === null ||
    typeof (dto as UserDTO).id !== 'number' ||
    typeof (dto as UserDTO).name !== 'string'
  ) {
    throw new Error('Invalid user data');
  }
  const data = dto as UserDTO;
  return {
    id: data.id,
    name: data.name,
    email: data.email || '',
    createdAt: new Date(data.created_at),
  };
}

import { z } from 'zod';
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().nullable(),
  created_at: z.string(),
});

function parseUserWithZod(data: unknown): User {
  const dto = UserSchema.parse(data);
  return {
    id: dto.id,
    name: dto.name,
    email: dto.email || '',
    createdAt: new Date(dto.created_at),
  };
}
```

边界原则：网络数据是 `unknown`，校验后再进入可信领域类型。

---
## 面试题自查

> 每题含答案 + 2～3 个追问。资深面试官常从「定义」追到「边界 / 线上事故 / 实现取舍」。

### Q1: 闭包是什么？有哪些应用场景？

**答案**：闭包是函数与其词法环境的组合；外部函数返回后，内部函数仍可访问被引用的外部变量。应用：私有状态、柯里化、记忆化、模块模式。注意：保留的是**被引用的绑定**，不是笼统的「整个执行上下文永不销毁」。

**追问**：
1. 为何 `for (var i…)` + 异步回调会打印同一个最终值？`let` 如何修复？
2. 闭包持有 DOM / 大数组时，如何用 Heap Snapshot 定位 Retainer？
3. 引擎是否可能优化掉闭包未使用的变量？这对「泄漏」判断意味着什么？

---

### Q2: 事件循环的执行顺序是什么？

**答案**：执行一个宏任务 → 清空微任务 →（可选）rAF / 渲染 → 下一宏任务。宏任务含 timer、I/O、UI 事件等；微任务含 Promise、`queueMicrotask`、MutationObserver。**渲染不是宏任务**；`rAF` 对齐帧，也不应简单列进 macrotask 清单。

**追问**：
1. 微任务死循环为何会卡住页面？和宏任务拆分有何不同？
2. `requestAnimationFrame` 与 `setTimeout(0)` 谁更适合动画？
3. Node 里 `process.nextTick` 相对 Promise 微任务的优先级？

---

### Q3: Promise 有哪些静态方法？各自用途？

**答案**：

| 方法 | 用途 |
|------|------|
| `all` | 全成功；一失败即失败；`[]` → `[]` |
| `allSettled` | 等全部结束，返回 status 数组 |
| `race` | 最先 settled |
| `any` | 最先 fulfilled；全失败 → AggregateError |
| `resolve` / `reject` | 创建已决议 Promise |

**追问**：
1. `all` 对非 Promise 值如何处理？空数组呢？
2. 多个请求要「尽量多成功」为什么常用 `allSettled`？
3. `race` 做超时有何副作用？如何配合 AbortController？

---

### Q4: this 的绑定规则与优先级？

**答案**：`new` > 显式（`call`/`apply`/`bind`）> 隐式（对象调用）> 默认（非严格 → 全局；严格/`class`/ESM → `undefined`）。箭头函数词法继承 `this`。

**追问**：
1. 抽出的方法作回调时为何丢 `this`？举严格/非严格两种表现。
2. `bind` 后再 `call` 能否改 `this`？`new boundFn()` 呢？
3. React class 里为何曾需要 `bind` 或 class field 箭头？

---

### Q5: 原型链查找机制是什么？

**答案**：自身属性 → `[[Prototype]]` 链 → `null`。`Constructor.prototype` 是实例的原型；`Object.getPrototypeOf` 优于 `__proto__`。

**追问**：
1. `hasOwnProperty` 与 `in` 的区别？
2. 修改 `Object.prototype` 有何风险？
3. `Object.create(null)` 的用途？

---

### Q6: ES6 Class 继承与 ES5 继承有何区别？

**答案**：Class 是原型继承语法糖，但更规范：`super`、静态继承、方法默认不可枚举。ES5 推荐**寄生组合继承**（`Parent.call` + `Object.create(Parent.prototype)`），只调一次父构造；经典组合继承（`new Parent()` 挂原型）才会调两次。

**追问**：
1. 指出「`Object.create` = 第二次调用父构造」错在哪里？
2. Class 字段与原型方法在实例枚举上有何差异？
3. `extends` 内置类型（如 Array）时要注意什么？

---

### Q7: async/await 与 Promise 有何区别？

**答案**：`async/await` 是 Promise 语法糖，可读性与 try/catch 更友好；并不消灭微任务调度。`async` 函数返回 Promise。

**追问**：
1. 「await 只能写在 async 函数里」在今天还成立吗？（ESM 顶层 await）
2. `await` 之后的代码以什么任务类型继续？
3. 串行 `await` 与 `Promise.all` 的性能差异如何讲给产品听？

---

### Q8: TypeScript 的 unknown 和 any 有何区别？

**答案**：`any` 关闭检查；`unknown` 必须收窄后才能用。跨边界数据优先 `unknown`。

**追问**：
1. `unknown` 如何赋给具体类型？
2. 为何说 `any` 会「传染」？
3. 与 `zod`/`io-ts` 运行时校验如何配合？

---

### Q9: 泛型约束怎么用？

**答案**：`T extends Constraint`；键约束用 `K extends keyof T`。

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

**追问**：
1. `extends` 在泛型约束与条件类型里分别表示什么？
2. 如何写「T 必须是某联合的子集」？
3. 泛型默认类型 `T = string` 的适用场景？

---

### Q10: 常见内存泄漏场景？如何避免？

**答案**：意外全局、闭包误持大对象、监听器/定时器未清理、Detached DOM 仍被 JS 引用。用 Snapshot 对比 + 成对注册/清理。

**追问**：
1. 如何区分「泄漏」与「合理缓存」？
2. WeakMap / WeakRef 各适合什么？
3. V8 分代里，长生命周期缓存应放哪一代视角理解？

---

### Q11: 防抖和节流的区别是什么？

**答案**：

| | 防抖 Debounce | 节流 Throttle |
|--|---------------|---------------|
| 行为 | 停止触发后再执行（重触即重计时） | 固定间隔最多执行一次 |
| 场景 | 搜索输入、resize 结束 | 滚动、高频点击、**技能冷却** |

**追问**：
1. trailing / leading 分别是什么？ lodash 如何配？
2. 技能冷却更接近防抖还是节流？为什么？
3. React 里用 hooks 封装时如何保证 `this`/最新 props？

---

### Q12: Partial、Pick、Omit 分别是什么？

**答案**：`Partial` 全可选；`Pick` 取键；`Omit` 排除键。常用于表单草稿、API patch、DTO 裁剪。

**追问**：
1. 手写一个 `MyPartial<T>`？
2. `Omit` 与 `Exclude` 差别？
3. 深度 Partial 怎么写？有何坑？

---

### Q13: 如何实现深拷贝？边界有哪些？

**答案**：优先 `structuredClone`。手写需处理：循环引用（WeakMap）、Date/RegExp/Map/Set、Symbol 键（`Reflect.ownKeys`）、描述符/不可枚举、原型（是否保留看需求）。`JSON` 方案丢失函数、`undefined`、循环等。

**追问**：
1. 为何 `for...in` + `hasOwnProperty` 不够？
2. 函数、DOM、跨 realm 对象怎么办？
3. 和浅拷贝 `{...obj}` / `Object.assign` 的分界？

---

### Q14: WeakMap 和 Map 有何区别？

**答案**：WeakMap 键为对象或**可 GC 的未注册 Symbol**，弱引用、不可遍历、无 `size`；适合私有数据与避免泄漏的关联。

**追问**：
1. `Symbol.for` 的 Symbol 能作 WeakMap 键吗？
2. 为什么不可枚举键集合？
3. Vue3 `targetMap` 为何用 WeakMap？

---

### Q15: Promise.all 与 allSettled 的区别？

**答案**：`all` 快速失败；`allSettled` 总等全部并给出每项 status。看板类「部分失败仍渲染」用后者。

**追问**：
1. 手写 `all` 空数组为何必须立即 resolve？
2. 如何在 `all` 失败时仍拿到已成功的结果？（对比 allSettled / 自行包装）
3. 与并发池组合时错误策略怎么设计？

---

### Q16: 如何判断变量类型？

**答案**：`typeof`（基本类型/函数，注意 `null`）；`instanceof`（原型链，跨 iframe 慎用）；`Object.prototype.toString.call`；`Array.isArray`。

**追问**：
1. 为什么 `typeof null === 'object'`？
2. 怎样可靠判断 Promise？
3. TS 里如何把运行时判断做成类型守卫？

---

### Q17: `var`、`let`、`const` 的核心区别？

**答案**：作用域（函数 vs 块）、TDZ、重复声明、`const` 不可重新绑定。工程默认 `const`。

**追问**：
1. 什么是暂时性死区？为何 `typeof` 未声明 `let` 会抛错？
2. `const` 对象属性可变与「不可变数据」实践如何统一？
3. 为何现代规范禁止新代码使用 `var`？

---

### Q18: `==` 和 `===` 有何区别？为何常禁用 `==`？

**答案**：`===` 不做类型转换；`==` 隐式转换易踩坑。禁用是为了降歧义；`x == null` 是少数可读例外。

**追问**：
1. 解释 `[] == ![]` 的推导过程（展示你真懂，不是背结论）。
2. `Object.is` 与 `===` 对 `NaN`、`+0/-0` 的差异？
3. 文档里如何约定「允许的 ==」白名单？

---

### Q19: 浏览器和 Node.js 事件循环关键差异？

**答案**：都有宏/微任务，但 Node 有 timers/poll/check 等阶段；`process.nextTick` 优先于 Promise 微任务；浏览器更强调渲染帧与 `rAF`。同构代码输出可能不同。

**追问**：
1. `setImmediate` 与 `setTimeout(0)` 在 I/O 回调内谁更先？
2. 为何 nextTick 递归危险？
3. 前端 SSR 同构时如何避免依赖某一种循环顺序？

---

### Q20: `interface` 和 `type` 如何选型？

**答案**：对象契约 / 声明合并 → `interface`；联合、映射、条件类型 → `type`。团队一致比教条更重要。

**追问**：
1. 演示 Declaration Merging 扩 `Window`。
2. `type` 能否合并？不能时怎么扩展？
3. 库的公共 API 导出类型时如何选，利于下游增强？

---

### 必背手写题

**1. 手写防抖（Debounce）**

```typescript
function debounce<T extends (...args: any[]) => any>(fn: T, delay: number) {
  let timer: ReturnType<typeof setTimeout> | null = null;
  return function (this: unknown, ...args: Parameters<T>) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
```

**2. 手写节流（Throttle）**

```typescript
function throttle<T extends (...args: any[]) => any>(fn: T, delay: number) {
  let last = 0;
  return function (this: unknown, ...args: Parameters<T>) {
    const now = Date.now();
    if (now - last >= delay) {
      last = now;
      fn.apply(this, args);
    }
  };
}
```

**3. 手写深拷贝（对齐边界：循环引用 / 特殊对象 / Symbol 键）**

```typescript
function deepClone<T>(value: T, map = new WeakMap()): T {
  if (value === null || typeof value !== 'object') return value;

  if (map.has(value as object)) return map.get(value as object);

  if (value instanceof Date) return new Date(value) as T;
  if (value instanceof RegExp) return new RegExp(value) as T;

  if (value instanceof Map) {
    const result = new Map();
    map.set(value, result);
    value.forEach((v, k) => result.set(deepClone(k, map), deepClone(v, map)));
    return result as T;
  }
  if (value instanceof Set) {
    const result = new Set();
    map.set(value, result);
    value.forEach((v) => result.add(deepClone(v, map)));
    return result as T;
  }

  const result = (Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value))) as T;
  map.set(value as object, result);

  for (const key of Reflect.ownKeys(value as object)) {
    const desc = Object.getOwnPropertyDescriptor(value as object, key)!;
    if ('value' in desc) {
      desc.value = deepClone((value as any)[key], map);
    }
    Object.defineProperty(result as object, key, desc);
  }
  return result;
}
```

**4. 手写 call / apply / bind（处理 falsy context、bind + new）**

```typescript
Function.prototype.myCall = function (context: any, ...args: any[]) {
  // null/undefined → globalThis；注意不能用 context || globalThis（否则 0/''/false 被误替换）
  const ctx =
    context === null || context === undefined ? globalThis : Object(context);
  const key = Symbol();
  Object.defineProperty(ctx, key, { value: this, configurable: true });
  const result = ctx[key](...args);
  delete ctx[key];
  return result;
};

Function.prototype.myApply = function (context: any, args: any[] = []) {
  return this.myCall(context, ...args);
};

Function.prototype.myBind = function (context: any, ...bindArgs: any[]) {
  const fn = this;
  const bound = function (this: unknown, ...callArgs: any[]) {
    // new bound() 时忽略传入的 context，this 指向新实例
    const isNew = this instanceof bound;
    return fn.apply(isNew ? this : context === null || context === undefined ? globalThis : context, [
      ...bindArgs,
      ...callArgs,
    ]);
  };
  if (fn.prototype) {
    bound.prototype = Object.create(fn.prototype);
  }
  return bound;
};
```

**手写题质量对齐检查**：
- Promise：`queueMicrotask`、空数组、保序
- 深拷贝：WeakMap + `Reflect.ownKeys` + 描述符
- bind：`new` 场景与 falsy `thisArg`

---

### 开放式设计题

**D1：设计可插件化的前端 SDK（埋点类），核心架构怎么考虑？**

**参考思路**：
- 微内核 + 插件：Core 管生命周期与事件总线，功能经 Plugin 注入
- 协议：`install/uninstall`、Hook（`beforeSend`/`afterSend`）
- 管道：采集 → 队列 → 批量上报 → 失败重试；空闲时段上报避免抢主线程
- TS：插件泛型、Declaration Merging 扩展事件表、严格事件类型
- 权衡：Tree Shaking 与开箱能力；同步初始化 vs 异步懒加载

**资深追问**：
1. 插件互相依赖时如何避免环形初始化？
2. 上报失败与用户隐私同意（consent）状态变更如何编排？
3. 如何保证双端（CJS/ESM）消费不出现 dual package 双单例？

**D2：怀疑闭包泄漏但无法稳定复现，排查策略？**

**参考思路**：
- Memory：Snapshot 对比、Allocation timeline、Detached DOM
- 常见源：监听未移除、定时器、React effect 未清理
- 路径：Retainer Path → 改 WeakMap/弱引用 → 回归用例
- 预防：lint（hooks deps）、CI 内存烟测

**资深追问**：
1. 如何证明「泄漏」而不是「缓存上限不合理」？
2. 生产环境不能开 DevTools 时，你还能采集什么信号？
3. 修复时为什么忌讳「先 null 一切」而不找根引用？

**D3：设计前端请求层：超时、取消、幂等、并发限制如何统一？**

**参考思路**：
- AbortController 贯穿；超时 = abort + 错误类型区分
- 幂等键与 in-flight Promise 去重（支付场景）
- 并发池 + 优先级队列；`allSettled` 做部分失败
- 类型：DTO → 校验 → Domain；错误用可判别联合

**资深追问**：
1. 用户连点支付，客户端去重与服务端幂等各自负责什么？
2. 取消后，React 组件 setState 如何避免「卸载后更新」？
3. 网关 429 与超时重试的退避如何设计，避免雪崩？

---
## 实战案例

### 金融支付场景

**幂等请求封装**

```typescript
interface RequestConfig {
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  data?: unknown;
  idempotencyKey?: string;
  signal?: AbortSignal;
}

class HttpClient {
  private pendingRequests = new Map<string, Promise<unknown>>();

  async request<T>(config: RequestConfig): Promise<T> {
    const { idempotencyKey } = config;

    if (idempotencyKey && this.pendingRequests.has(idempotencyKey)) {
      return this.pendingRequests.get(idempotencyKey) as Promise<T>;
    }

    const promise = this.executeRequest<T>(config);

    if (idempotencyKey) {
      this.pendingRequests.set(idempotencyKey, promise);
      promise.finally(() => this.pendingRequests.delete(idempotencyKey!));
    }

    return promise;
  }

  private async executeRequest<T>(config: RequestConfig): Promise<T> {
    const response = await fetch(config.url, {
      method: config.method,
      headers: {
        'Content-Type': 'application/json',
        ...(config.idempotencyKey && {
          'Idempotency-Key': config.idempotencyKey,
        }),
      },
      body: config.data ? JSON.stringify(config.data) : undefined,
      signal: config.signal,
    });

    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  }
}

const client = new HttpClient();

async function submitPayment(orderId: string, amount: number, signal?: AbortSignal) {
  return client.request({
    url: '/api/payment',
    method: 'POST',
    data: { orderId, amount },
    idempotencyKey: `payment-${orderId}`,
    signal,
  });
}
```

---

### 游戏场景

**节流 / 冷却的技能释放**（这是冷却窗口，语义上接近**节流**，不是防抖）

```typescript
interface Skill {
  id: string;
  name: string;
  cooldown: number; // 冷却毫秒
}

class SkillSystem {
  private cooldowns = new Map<string, number>();

  canCast(skill: Skill): boolean {
    const lastCast = this.cooldowns.get(skill.id);
    if (!lastCast) return true;
    return Date.now() - lastCast >= skill.cooldown;
  }

  cast(skill: Skill, action: () => void): boolean {
    if (!this.canCast(skill)) {
      console.log(`${skill.name} 冷却中...`);
      return false;
    }
    this.cooldowns.set(skill.id, Date.now());
    action();
    console.log(`释放 ${skill.name}！`);
    return true;
  }

  getRemainingCooldown(skill: Skill): number {
    const lastCast = this.cooldowns.get(skill.id);
    if (!lastCast) return 0;
    return Math.max(0, skill.cooldown - (Date.now() - lastCast));
  }
}

const system = new SkillSystem();
const fireball: Skill = { id: 'fireball', name: '火球术', cooldown: 3000 };

system.cast(fireball, () => console.log('发送火球...')); // ✅
system.cast(fireball, () => {}); // ❌ 冷却中
```

对比：
- **防抖**：松手后才施法（连续按键重计时）——不适合技能 CD
- **节流 / 冷却**：成功释放后进入固定窗口——本例模型

---

**总结**

- **基础**：闭包（精确到「被引用绑定」）、this、原型（分清组合 vs 寄生组合）、事件循环（含渲染 / rAF / Node 阶段）
- **手写**：Promise（微任务 + 空数组）、防抖节流、深拷贝边界、call/bind（falsy + `new`）
- **工程**：AbortController、模块 `exports` / sideEffects、TS 方差与声明文件
- **面试**：能答定义，更能答追问里的边界与线上取舍

---
