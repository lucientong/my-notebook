# JavaScript 与 TypeScript 核心知识

---

## 📑 目录

- [JavaScript 核心](#javascript-核心)
  - [作用域与闭包](#作用域与闭包)
  - [原型与继承](#原型与继承)
  - [事件循环](#事件循环)
  - [异步编程](#异步编程)
  - [内存管理](#内存管理)
- [TypeScript 核心](#typescript-核心)
  - [类型系统](#类型系统)
  - [高级类型](#高级类型)
  - [工程实践](#工程实践)
- [常见面试题](#常见面试题)
- [实战案例](#实战案例)

---

## JavaScript 核心

### 作用域与闭包

#### 值得深挖

**1. 词法作用域 vs 动态作用域**

JavaScript 使用词法作用域（静态作用域），函数的作用域在函数**定义时**确定，而非**调用时**。

```javascript
// 词法作用域示例
const name = 'global';

function outer() {
  const name = 'outer';
  
  function inner() {
    console.log(name); // 'outer' - 定义时确定
  }
  
  return inner;
}

const fn = outer();
fn(); // 输出 'outer'，而非 'global'
```

**面试追问**：
- 解释为什么输出 'outer' 而非 'global'？
- 如果 JavaScript 是动态作用域，输出会是什么？

---

**2. 闭包（Closure）**

**定义**：函数能够访问其外部作用域的变量，即使外部函数已经返回。

**核心原理**：
- 函数是一等公民，可以作为返回值
- 内部函数保持对外部作用域的引用（作用域链）
- 外部函数的执行上下文不会被销毁

```javascript
// 经典闭包示例
function createCounter() {
  let count = 0; // 私有变量
  
  return {
    increment() {
      return ++count;
    },
    decrement() {
      return --count;
    },
    getCount() {
      return count;
    }
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount());  // 2
// count 变量无法直接访问，只能通过方法
```

**闭包的应用场景**：

1. **数据封装 / 私有变量**
```javascript
function createUser(name) {
  let balance = 0; // 私有
  
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

// 使用
const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6
```

3. **延迟执行 / 记忆化**
```javascript
function memoize(fn) {
  const cache = new Map();
  
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// 示例：斐波那契数列
const fib = memoize((n) => {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
});

console.log(fib(40)); // 快速返回
```

**闭包常见陷阱**：

```javascript
// 陷阱1：循环中的闭包
for (var i = 0; i < 5; i++) {
  setTimeout(() => {
    console.log(i); // 输出 5, 5, 5, 5, 5
  }, 1000);
}

// 解决方案1：使用 let（块级作用域）
for (let i = 0; i < 5; i++) {
  setTimeout(() => {
    console.log(i); // 输出 0, 1, 2, 3, 4
  }, 1000);
}

// 解决方案2：立即执行函数（IIFE）
for (var i = 0; i < 5; i++) {
  (function(j) {
    setTimeout(() => {
      console.log(j); // 输出 0, 1, 2, 3, 4
    }, 1000);
  })(i);
}

// 解决方案3：bind 绑定参数
for (var i = 0; i < 5; i++) {
  setTimeout(console.log.bind(null, i), 1000);
}
```

**面试追问**：
- 闭包会导致内存泄漏吗？什么情况下会？
- 如何检测和修复闭包引起的内存泄漏？

---

**3. this 绑定规则**

JavaScript 的 `this` 在**运行时**绑定，取决于函数的调用方式。

**四种绑定规则**（优先级从高到低）：

**规则1：new 绑定**
```javascript
function Person(name) {
  this.name = name;
}

const person = new Person('Alice');
console.log(person.name); // 'Alice'
// new 创建新对象，this 指向该对象
```

**规则2：显式绑定（call / apply / bind）**
```javascript
function greet() {
  console.log(`Hello, ${this.name}`);
}

const user = { name: 'Bob' };

greet.call(user);  // 'Hello, Bob'
greet.apply(user); // 'Hello, Bob'

const boundGreet = greet.bind(user);
boundGreet(); // 'Hello, Bob'
```

**规则3：隐式绑定（对象方法调用）**
```javascript
const obj = {
  name: 'Charlie',
  greet() {
    console.log(`Hello, ${this.name}`);
  }
};

obj.greet(); // 'Hello, Charlie'

// 陷阱：赋值后 this 丢失
const fn = obj.greet;
fn(); // 'Hello, undefined' (严格模式下报错)
```

**规则4：默认绑定**
```javascript
function showThis() {
  console.log(this);
}

showThis(); // 非严格模式：window/global，严格模式：undefined
```

**箭头函数的 this**：
- 箭头函数没有自己的 `this`，继承外层作用域的 `this`
- 无法通过 `call`/`apply`/`bind` 改变 `this`

```javascript
const obj = {
  name: 'David',
  regularFn: function() {
    console.log(this.name); // 'David'
  },
  arrowFn: () => {
    console.log(this.name); // undefined (继承全局作用域)
  },
  nested: function() {
    const arrow = () => {
      console.log(this.name); // 'David' (继承 nested 的 this)
    };
    arrow();
  }
};

obj.regularFn();
obj.arrowFn();
obj.nested();
```

**实战场景：React 组件中的 this**

```javascript
class MyComponent extends React.Component {
  constructor() {
    super();
    this.state = { count: 0 };
    
    // 方案1：bind 绑定
    this.handleClick = this.handleClick.bind(this);
  }
  
  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }
  
  // 方案2：箭头函数（class field）
  handleClickArrow = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  render() {
    return (
      <div>
        <button onClick={this.handleClick}>Click</button>
        <button onClick={this.handleClickArrow}>Click Arrow</button>
      </div>
    );
  }
}
```

---

### 原型与继承

#### 值得深挖

**1. 原型链（Prototype Chain）**

JavaScript 是基于原型的面向对象语言。

**核心概念**：
- 每个对象都有一个内部属性 `[[Prototype]]`（可通过 `__proto__` 或 `Object.getPrototypeOf()` 访问）
- 函数有 `prototype` 属性，用于构造实例的原型
- 原型链用于属性查找

```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.sayHello = function() {
  console.log(`Hello, ${this.name}`);
};

const alice = new Person('Alice');

// 原型链：alice -> Person.prototype -> Object.prototype -> null
console.log(alice.__proto__ === Person.prototype); // true
console.log(Person.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__); // null

alice.sayHello(); // 'Hello, Alice'
```

**属性查找机制**：
1. 在对象自身查找
2. 在 `__proto__` 指向的原型查找
3. 沿原型链向上查找，直到 `null`

```javascript
const obj = { a: 1 };
obj.b = 2;
Object.prototype.c = 3;

console.log(obj.a); // 1 (自身属性)
console.log(obj.b); // 2 (自身属性)
console.log(obj.c); // 3 (原型链上找到)
console.log(obj.d); // undefined (找不到)

// 检测自身属性
console.log(obj.hasOwnProperty('a')); // true
console.log(obj.hasOwnProperty('c')); // false
```

---

**2. 继承模式**

**方案1：原型链继承（不推荐）**
```javascript
function Parent() {
  this.colors = ['red', 'blue'];
}

function Child() {}

Child.prototype = new Parent();

const child1 = new Child();
const child2 = new Child();

child1.colors.push('green');
console.log(child2.colors); // ['red', 'blue', 'green'] ❌ 共享引用
```

**问题**：
- 引用类型属性被所有实例共享
- 无法向父类构造函数传参

---

**方案2：构造函数继承（不推荐）**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}

Parent.prototype.sayName = function() {
  console.log(this.name);
};

function Child(name) {
  Parent.call(this, name); // 调用父类构造函数
}

const child = new Child('Alice');
console.log(child.colors); // ['red', 'blue'] ✅ 独立
child.sayName(); // ❌ TypeError: child.sayName is not a function
```

**问题**：
- 无法继承原型上的方法
- 每个实例都会复制方法，内存浪费

---

**方案3：组合继承（常用）**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}

Parent.prototype.sayName = function() {
  console.log(this.name);
};

function Child(name, age) {
  Parent.call(this, name); // 继承属性
  this.age = age;
}

// 继承方法
Child.prototype = Object.create(Parent.prototype);
Child.prototype.constructor = Child;

Child.prototype.sayAge = function() {
  console.log(this.age);
};

const child = new Child('Alice', 10);
child.sayName(); // 'Alice' ✅
child.sayAge();  // 10 ✅
```

**优点**：
- 属性独立，方法共享
- 可以向父类传参

**缺点**：
- 调用了两次父类构造函数（`Parent.call` 和 `Object.create`）

---

**方案4：ES6 Class（推荐）**
```javascript
class Parent {
  constructor(name) {
    this.name = name;
    this.colors = ['red', 'blue'];
  }
  
  sayName() {
    console.log(this.name);
  }
}

class Child extends Parent {
  constructor(name, age) {
    super(name); // 必须先调用 super
    this.age = age;
  }
  
  sayAge() {
    console.log(this.age);
  }
}

const child = new Child('Alice', 10);
child.sayName(); // 'Alice' ✅
child.sayAge();  // 10 ✅
```

**优点**：
- 语法清晰，更接近传统 OOP
- 内部实现优化

---

### 事件循环

#### 值得深挖

**JavaScript 是单线程的**，但通过事件循环（Event Loop）实现异步非阻塞。

**核心概念**：
- **调用栈（Call Stack）**：同步代码执行
- **宏任务队列（Macrotask Queue）**：`setTimeout`、`setInterval`、I/O、UI 渲染
- **微任务队列（Microtask Queue）**：`Promise.then`、`MutationObserver`、`queueMicrotask`

**执行顺序**：
1. 执行同步代码（调用栈）
2. 执行**所有微任务**（直到微任务队列清空）
3. 执行**一个宏任务**
4. 执行**所有微任务**
5. 渲染（如果需要）
6. 回到步骤3

```javascript
console.log('1-同步');

setTimeout(() => {
  console.log('2-宏任务');
}, 0);

Promise.resolve().then(() => {
  console.log('3-微任务');
});

console.log('4-同步');

// 输出顺序：
// 1-同步
// 4-同步
// 3-微任务
// 2-宏任务
```

**复杂示例**：

```javascript
console.log('start');

setTimeout(() => {
  console.log('timeout1');
  Promise.resolve().then(() => {
    console.log('promise3');
  });
}, 0);

Promise.resolve().then(() => {
  console.log('promise1');
  setTimeout(() => {
    console.log('timeout2');
  }, 0);
}).then(() => {
  console.log('promise2');
});

console.log('end');

// 输出顺序：
// start
// end
// promise1
// promise2
// timeout1
// promise3
// timeout2
```

**面试追问**：
- 微任务为什么会影响 UI 卡顿？
  - 答：微任务在一次事件循环中全部执行完，如果微任务过多或耗时过长，会阻塞渲染
- 如何避免微任务阻塞？
  - 答：拆分任务、使用 `requestIdleCallback`、使用 Web Worker

---

### 异步编程

#### 值得深挖

**1. Promise**

**三种状态**：
- `pending`：初始状态
- `fulfilled`：操作成功
- `rejected`：操作失败

**状态转换**：
- `pending` → `fulfilled` 或 `rejected`（**不可逆**）

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = Math.random() > 0.5;
    if (success) {
      resolve('成功');
    } else {
      reject('失败');
    }
  }, 1000);
});

promise
  .then(result => console.log(result))
  .catch(error => console.error(error))
  .finally(() => console.log('完成'));
```

**Promise 链式调用**：

```javascript
fetch('/api/user')
  .then(response => response.json())
  .then(user => {
    console.log(user);
    return fetch(`/api/posts?userId=${user.id}`);
  })
  .then(response => response.json())
  .then(posts => console.log(posts))
  .catch(error => console.error(error));
```

**Promise 并发**：

```javascript
// Promise.all：所有成功才成功，一个失败就失败
Promise.all([
  fetch('/api/user'),
  fetch('/api/posts'),
  fetch('/api/comments')
])
  .then(responses => Promise.all(responses.map(r => r.json())))
  .then(([user, posts, comments]) => {
    console.log({ user, posts, comments });
  })
  .catch(error => console.error('某个请求失败', error));

// Promise.allSettled：等待所有完成（无论成功失败）
Promise.allSettled([
  fetch('/api/user'),
  fetch('/api/posts-invalid') // 会失败
])
  .then(results => {
    results.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        console.log(`请求${index}成功:`, result.value);
      } else {
        console.log(`请求${index}失败:`, result.reason);
      }
    });
  });

// Promise.race：第一个完成的决定结果
Promise.race([
  fetch('/api/fast'),
  new Promise((_, reject) => 
    setTimeout(() => reject('超时'), 3000)
  )
])
  .then(result => console.log('最快的:', result))
  .catch(error => console.error('超时或失败:', error));

// Promise.any：第一个成功的决定结果（ES2021）
Promise.any([
  fetch('/api/slow-but-success'),
  fetch('/api/fast-but-fail')
])
  .then(result => console.log('第一个成功:', result))
  .catch(error => console.error('全部失败:', error));
```

---

**手写 Promise（简化版）**

```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'pending'; // pending | fulfilled | rejected
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];
    
    const resolve = (value) => {
      if (this.state === 'pending') {
        this.state = 'fulfilled';
        this.value = value;
        this.onFulfilledCallbacks.forEach(fn => fn());
      }
    };
    
    const reject = (reason) => {
      if (this.state === 'pending') {
        this.state = 'rejected';
        this.reason = reason;
        this.onRejectedCallbacks.forEach(fn => fn());
      }
    };
    
    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }
  
  then(onFulfilled, onRejected) {
    // 参数可选
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v;
    onRejected = typeof onRejected === 'function' ? onRejected : e => { throw e; };
    
    const promise2 = new MyPromise((resolve, reject) => {
      if (this.state === 'fulfilled') {
        setTimeout(() => {
          try {
            const x = onFulfilled(this.value);
            resolve(x);
          } catch (error) {
            reject(error);
          }
        });
      }
      
      if (this.state === 'rejected') {
        setTimeout(() => {
          try {
            const x = onRejected(this.reason);
            resolve(x);
          } catch (error) {
            reject(error);
          }
        });
      }
      
      if (this.state === 'pending') {
        this.onFulfilledCallbacks.push(() => {
          setTimeout(() => {
            try {
              const x = onFulfilled(this.value);
              resolve(x);
            } catch (error) {
              reject(error);
            }
          });
        });
        
        this.onRejectedCallbacks.push(() => {
          setTimeout(() => {
            try {
              const x = onRejected(this.reason);
              resolve(x);
            } catch (error) {
              reject(error);
            }
          });
        });
      }
    });
    
    return promise2;
  }
  
  catch(onRejected) {
    return this.then(null, onRejected);
  }
  
  finally(callback) {
    return this.then(
      value => MyPromise.resolve(callback()).then(() => value),
      reason => MyPromise.resolve(callback()).then(() => { throw reason; })
    );
  }
  
  static resolve(value) {
    if (value instanceof MyPromise) return value;
    return new MyPromise(resolve => resolve(value));
  }
  
  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }
  
  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const results = [];
      let count = 0;
      
      promises.forEach((promise, index) => {
        MyPromise.resolve(promise).then(value => {
          results[index] = value;
          count++;
          if (count === promises.length) {
            resolve(results);
          }
        }, reject);
      });
    });
  }
}

// 测试
const p = new MyPromise((resolve) => {
  setTimeout(() => resolve('success'), 1000);
});

p.then(value => {
  console.log(value); // 'success'
  return 'next';
}).then(value => {
  console.log(value); // 'next'
});
```

---

**2. async/await**

`async/await` 是 Promise 的语法糖，让异步代码看起来像同步代码。

```javascript
// Promise 写法
function fetchUser() {
  return fetch('/api/user')
    .then(response => response.json())
    .then(user => {
      console.log(user);
      return user;
    })
    .catch(error => console.error(error));
}

// async/await 写法
async function fetchUser() {
  try {
    const response = await fetch('/api/user');
    const user = await response.json();
    console.log(user);
    return user;
  } catch (error) {
    console.error(error);
  }
}
```

**并发执行**：

```javascript
// ❌ 串行执行（慢）
async function fetchAll() {
  const user = await fetch('/api/user'); // 等待1秒
  const posts = await fetch('/api/posts'); // 再等待1秒
  return { user, posts }; // 总共2秒
}

// ✅ 并行执行（快）
async function fetchAll() {
  const [user, posts] = await Promise.all([
    fetch('/api/user'),
    fetch('/api/posts')
  ]); // 同时请求，总共1秒
  return { user, posts };
}
```

**错误处理**：

```javascript
async function fetchWithRetry(url, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return await response.json();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      console.log(`重试 ${i + 1}/${maxRetries}...`);
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
}
```

---

### 内存管理

#### 值得深挖

**1. 垃圾回收（GC）**

JavaScript 使用**标记清除（Mark-and-Sweep）**算法。

**基本原理**：
1. 从根对象（全局对象、调用栈）开始，标记所有可达对象
2. 清除未标记的对象

**常见内存泄漏场景**：

**场景1：全局变量**
```javascript
// ❌ 泄漏
function leak() {
  global = new Array(1000000); // 全局变量不会被回收
}

// ✅ 正确
function noLeak() {
  const local = new Array(1000000); // 函数结束后可回收
}
```

**场景2：闭包持有大对象**
```javascript
// ❌ 泄漏
function createClosure() {
  const largeData = new Array(1000000).fill('data');
  
  return function() {
    console.log('closure'); // 即使不用 largeData，也会持有引用
  };
}

// ✅ 改进
function createClosure() {
  const largeData = new Array(1000000).fill('data');
  const summary = largeData.length; // 只保留需要的
  
  return function() {
    console.log(summary);
  };
}
```

**场景3：事件监听器未移除**
```javascript
// ❌ 泄漏
class Component {
  constructor(element) {
    this.element = element;
    this.handleClick = () => {
      console.log(this.data);
    };
    this.element.addEventListener('click', this.handleClick);
    this.data = new Array(1000000);
  }
}

// 即使 element 从 DOM 移除，监听器仍持有 Component 引用

// ✅ 正确
class Component {
  constructor(element) {
    this.element = element;
    this.handleClick = () => {
      console.log(this.data);
    };
    this.element.addEventListener('click', this.handleClick);
    this.data = new Array(1000000);
  }
  
  destroy() {
    this.element.removeEventListener('click', this.handleClick);
    this.data = null;
  }
}
```

**场景4：定时器未清理**
```javascript
// ❌ 泄漏
function startPolling() {
  const data = new Array(1000000);
  setInterval(() => {
    console.log(data.length); // 持有 data 引用
  }, 1000);
}

// ✅ 正确
function startPolling() {
  const data = new Array(1000000);
  const timer = setInterval(() => {
    console.log(data.length);
  }, 1000);
  
  return () => clearInterval(timer); // 返回清理函数
}
```

---

**2. 性能优化技巧**

**对象池（Object Pool）**

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

// 使用
const vectorPool = new ObjectPool(
  () => ({ x: 0, y: 0 }),
  (v) => { v.x = 0; v.y = 0; }
);

const v = vectorPool.acquire();
v.x = 10;
v.y = 20;
// ... 使用
vectorPool.release(v); // 归还复用
```

---

## TypeScript 核心

### 类型系统

#### 值得深挖

**1. 基础类型**

```typescript
// 基本类型
let num: number = 42;
let str: string = 'hello';
let bool: boolean = true;
let nul: null = null;
let undef: undefined = undefined;
let sym: symbol = Symbol('key');
let bigint: bigint = 100n;

// 数组
let arr1: number[] = [1, 2, 3];
let arr2: Array<number> = [1, 2, 3];

// 元组
let tuple: [string, number] = ['Alice', 30];

// 枚举
enum Color {
  Red,
  Green,
  Blue
}
let c: Color = Color.Green;

// any（慎用）
let anything: any = 'hello';
anything = 42; // OK

// unknown（更安全的 any）
let value: unknown = 'hello';
// value.toUpperCase(); // ❌ 错误
if (typeof value === 'string') {
  value.toUpperCase(); // ✅ 类型收窄后可用
}

// void
function log(message: string): void {
  console.log(message);
}

// never（永不返回）
function throwError(message: string): never {
  throw new Error(message);
}
```

---

**2. 对象类型**

```typescript
// 接口
interface User {
  id: number;
  name: string;
  email?: string; // 可选
  readonly createdAt: Date; // 只读
}

const user: User = {
  id: 1,
  name: 'Alice',
  createdAt: new Date()
};

// user.createdAt = new Date(); // ❌ 只读属性

// 类型别名
type Point = {
  x: number;
  y: number;
};

// 索引签名
interface StringMap {
  [key: string]: string;
}

const map: StringMap = {
  hello: 'world',
  foo: 'bar'
};
```

---

**3. 函数类型**

```typescript
// 函数声明
function add(a: number, b: number): number {
  return a + b;
}

// 函数表达式
const multiply: (a: number, b: number) => number = (a, b) => a * b;

// 可选参数
function greet(name: string, greeting?: string): string {
  return `${greeting || 'Hello'}, ${name}`;
}

// 默认参数
function greet2(name: string, greeting: string = 'Hello'): string {
  return `${greeting}, ${name}`;
}

// 剩余参数
function sum(...numbers: number[]): number {
  return numbers.reduce((a, b) => a + b, 0);
}

// 重载
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

#### 值得深挖

**1. 配置 tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM"],
    "strict": true, // 开启所有严格检查
    "noUncheckedIndexedAccess": true, // 索引访问可能为 undefined
    "noImplicitReturns": true, // 函数必须有返回
    "noFallthroughCasesInSwitch": true, // switch 必须 break
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

---

**2. API 类型建模**

```typescript
// 后端返回的原始数据（不可信）
interface UserDTO {
  id: number;
  name: string;
  email: string | null;
  created_at: string; // 注意：字符串格式
}

// 前端使用的数据（可信）
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
}

// 类型守卫 + 转换
function parseUser(dto: unknown): User {
  // 运行时校验
  if (
    typeof dto !== 'object' ||
    dto === null ||
    typeof (dto as any).id !== 'number' ||
    typeof (dto as any).name !== 'string'
  ) {
    throw new Error('Invalid user data');
  }
  
  const data = dto as UserDTO;
  
  return {
    id: data.id,
    name: data.name,
    email: data.email || '',
    createdAt: new Date(data.created_at)
  };
}

// 使用 zod 库（推荐）
import { z } from 'zod';

const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().nullable(),
  created_at: z.string()
});

function parseUserWithZod(data: unknown): User {
  const dto = UserSchema.parse(data); // 自动校验
  return {
    id: dto.id,
    name: dto.name,
    email: dto.email || '',
    createdAt: new Date(dto.created_at)
  };
}
```

---

## 常见面试题

### 必背手写题

**1. 手写防抖（Debounce）**

```typescript
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout> | null = null;
  
  return function(this: any, ...args: Parameters<T>) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

// 测试
const log = debounce((msg: string) => console.log(msg), 1000);
log('a'); // 不输出
log('b'); // 不输出
log('c'); // 1秒后输出 'c'
```

---

**2. 手写节流（Throttle）**

```typescript
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let last = 0;
  
  return function(this: any, ...args: Parameters<T>) {
    const now = Date.now();
    if (now - last >= delay) {
      last = now;
      fn.apply(this, args);
    }
  };
}

// 测试
const log = throttle((msg: string) => console.log(msg), 1000);
log('a'); // 输出 'a'
log('b'); // 不输出（未到1秒）
setTimeout(() => log('c'), 1100); // 输出 'c'
```

---

**3. 手写深拷贝**

```typescript
function deepClone<T>(obj: T, map = new WeakMap()): T {
  // 处理基本类型和 null
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }
  
  // 处理循环引用
  if (map.has(obj)) {
    return map.get(obj);
  }
  
  // 处理特殊对象
  if (obj instanceof Date) return new Date(obj) as any;
  if (obj instanceof RegExp) return new RegExp(obj) as any;
  if (obj instanceof Map) {
    const clonedMap = new Map();
    map.set(obj, clonedMap as any);
    obj.forEach((value, key) => {
      clonedMap.set(deepClone(key, map), deepClone(value, map));
    });
    return clonedMap as any;
  }
  if (obj instanceof Set) {
    const clonedSet = new Set();
    map.set(obj, clonedSet as any);
    obj.forEach(value => {
      clonedSet.add(deepClone(value, map));
    });
    return clonedSet as any;
  }
  
  // 处理数组和对象
  const clone = (Array.isArray(obj) ? [] : {}) as T;
  map.set(obj, clone);
  
  for (const key in obj) {
    if (obj.hasOwnProperty(key)) {
      clone[key] = deepClone(obj[key], map);
    }
  }
  
  return clone;
}

// 测试
const original = {
  a: 1,
  b: { c: 2 },
  d: [1, 2, { e: 3 }],
  f: new Date(),
  g: /test/g
};
original.h = original; // 循环引用

const cloned = deepClone(original);
console.log(cloned.b === original.b); // false
console.log(cloned.h === cloned); // true
```

---

**4. 手写 call/apply/bind**

```typescript
// call
Function.prototype.myCall = function(context: any, ...args: any[]) {
  context = context || globalThis;
  const fnSymbol = Symbol();
  context[fnSymbol] = this;
  const result = context[fnSymbol](...args);
  delete context[fnSymbol];
  return result;
};

// apply
Function.prototype.myApply = function(context: any, args: any[] = []) {
  context = context || globalThis;
  const fnSymbol = Symbol();
  context[fnSymbol] = this;
  const result = context[fnSymbol](...args);
  delete context[fnSymbol];
  return result;
};

// bind
Function.prototype.myBind = function(context: any, ...bindArgs: any[]) {
  const fn = this;
  return function(...callArgs: any[]) {
    return fn.apply(context, [...bindArgs, ...callArgs]);
  };
};

// 测试
const obj = { name: 'Alice' };
function greet(greeting: string, punctuation: string) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

greet.myCall(obj, 'Hello', '!'); // 'Hello, Alice!'
greet.myApply(obj, ['Hi', '?']); // 'Hi, Alice?'
const boundGreet = greet.myBind(obj, 'Hey');
boundGreet('~'); // 'Hey, Alice~'
```

---

## 实战案例

### 金融支付场景

**幂等请求封装**

```typescript
interface RequestConfig {
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  data?: any;
  idempotencyKey?: string; // 幂等键
}

class HttpClient {
  private pendingRequests = new Map<string, Promise<any>>();
  
  async request<T>(config: RequestConfig): Promise<T> {
    const { idempotencyKey } = config;
    
    // 幂等保护：相同请求正在进行时，返回同一个 Promise
    if (idempotencyKey && this.pendingRequests.has(idempotencyKey)) {
      return this.pendingRequests.get(idempotencyKey)!;
    }
    
    const promise = this.executeRequest<T>(config);
    
    if (idempotencyKey) {
      this.pendingRequests.set(idempotencyKey, promise);
      promise.finally(() => {
        this.pendingRequests.delete(idempotencyKey);
      });
    }
    
    return promise;
  }
  
  private async executeRequest<T>(config: RequestConfig): Promise<T> {
    const response = await fetch(config.url, {
      method: config.method,
      headers: {
        'Content-Type': 'application/json',
        ...(config.idempotencyKey && {
          'Idempotency-Key': config.idempotencyKey
        })
      },
      body: config.data ? JSON.stringify(config.data) : undefined
    });
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    
    return response.json();
  }
}

// 使用
const client = new HttpClient();

async function submitPayment(orderId: string, amount: number) {
  // 用订单ID作为幂等键，避免重复提交
  return client.request({
    url: '/api/payment',
    method: 'POST',
    data: { orderId, amount },
    idempotencyKey: `payment-${orderId}`
  });
}

// 即使快速点击多次，也只会发送一次请求
submitPayment('order-123', 100);
submitPayment('order-123', 100); // 返回同一个 Promise
```

---

### 游戏场景

**防抖的技能释放**

```typescript
interface Skill {
  id: string;
  name: string;
  cooldown: number; // 冷却时间（毫秒）
}

class SkillSystem {
  private cooldowns = new Map<string, number>();
  
  canCast(skill: Skill): boolean {
    const lastCast = this.cooldowns.get(skill.id);
    if (!lastCast) return true;
    
    const now = Date.now();
    return now - lastCast >= skill.cooldown;
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
    
    const elapsed = Date.now() - lastCast;
    return Math.max(0, skill.cooldown - elapsed);
  }
}

// 使用
const system = new SkillSystem();
const fireball: Skill = { id: 'fireball', name: '火球术', cooldown: 3000 };

system.cast(fireball, () => {
  // 发送技能释放请求
  console.log('发送火球...');
}); // ✅ 释放成功

system.cast(fireball, () => {}); // ❌ 冷却中

setTimeout(() => {
  system.cast(fireball, () => {}); // ✅ 3秒后可再次释放
}, 3000);
```

---

**总结**

JavaScript 和 TypeScript 是前端开发的基石，面试中：
- **基础扎实**：闭包、原型、事件循环要讲清原理
- **手写题必备**：Promise、防抖节流、深拷贝必须熟练
- **TypeScript 加分**：类型体操、泛型、工具类型能体现工程能力
- **结合业务**：用金融/游戏场景的例子，展示你的实战经验

面试官最看重的是：**你是否理解底层原理，能否解决实际问题**。💪
