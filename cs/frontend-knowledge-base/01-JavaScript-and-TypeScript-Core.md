Language: English | [中文](../前端知识库/01-JavaScript与TypeScript核心.md)

# JavaScript and TypeScript Core

---

## Table of Contents

### JavaScript Core

1. [Scope and Closures](#scope-and-closures)
2. [this Binding](#this-binding)
3. [Prototypes and Inheritance](#prototypes-and-inheritance)
4. [Event Loop](#event-loop)
5. [Asynchronous Programming](#asynchronous-programming)
6. [Memory Management](#memory-management)
7. [ES6+ Core Features](#es6-core-features)

### TypeScript Core

8. [Type System](#type-system)
9. [Advanced Types](#advanced-types)
10. [Engineering Practice](#engineering-practice)

### Self-check and Practice

11. [Interview Self-check](#interview-self-check)
12. [Production Scenarios](#production-scenarios)

---

## JavaScript Core

### Scope and Closures

#### Worth Digging Into

**1. Lexical scope vs dynamic scope**

JavaScript uses **lexical scope** (static scope). A function's scope is fixed
where the function is **defined**, not where it is **called**.

```javascript
const name = 'global';

function outer() {
  const name = 'outer';

  function inner() {
    console.log(name); // 'outer' — resolved at definition time
  }

  return inner;
}

const fn = outer();
fn(); // prints 'outer', not 'global'
```

**Interview follow-ups**:
- Why is the output `'outer'` rather than `'global'`?
- If JavaScript used dynamic scope, what would it print?

---

**2. `var` / `let` / `const`**

| Trait | `var` | `let` | `const` |
|-------|-------|-------|---------|
| Scope | Function | Block | Block |
| Hoisting | Hoisted, initialized as `undefined` | Hoisted but in TDZ | Hoisted but in TDZ |
| Redeclaration | Allowed | Not allowed | Not allowed |
| Rebinding | Allowed | Allowed | Not allowed (object properties remain mutable) |

```javascript
console.log(a); // undefined — var hoisting
var a = 1;

// console.log(b); // ReferenceError — TDZ
let b = 2;

const obj = { x: 1 };
obj.x = 2;        // ✅ mutates a property
// obj = {};      // ❌ cannot rebind

for (var i = 0; i < 3; i++) {}
console.log(i);   // 3 — leaks to outer scope

for (let j = 0; j < 3; j++) {}
// console.log(j); // ReferenceError
```

Engineering default: prefer `const` → use `let` when reassignment is needed →
avoid `var` in new code.

---

**3. `==` vs `===`**

- `===`: strict equality, **no** type coercion.
- `==`: abstract equality with implicit conversion; rules are complex and easy to misuse.

```javascript
'' == 0          // true
0 == '0'         // true
null == undefined // true
null === undefined // false
NaN === NaN      // false — use Number.isNaN / Object.is
Object.is(+0, -0) // false
```

Teams ban `==` not for purity, but to reduce production ambiguity from coercion.
When you intentionally want “treat `null`/`undefined` the same,” prefer
`x == null` or an explicit check, and document that exception.

---

**4. Closures**

**Definition**: a function can access variables from its outer lexical
environment even after that outer function has returned.

**More precise mental model**:
- Creating a function stores a reference to its surrounding lexical environment
  (scope chain).
- While the closure remains reachable, **referenced bindings** cannot be GC'd.
- It is not “the entire execution context lives forever.” Only bindings still
  referenced by the closure are retained; unused locals may be optimized away
  (engine-dependent).

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
// count is only reachable through the methods
```

**Common uses**:

1. **Encapsulation / private state**
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

2. **Currying**
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

3. **Memoization**
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

**Classic pitfall**:

```javascript
// Shared binding with var
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 1000); // 5 5 5 5 5
}

// Fix 1: let — new binding per iteration
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 1000); // 0..4
}

// Fix 2: IIFE capture
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 1000);
  })(i);
}
```

**Interview follow-ups**:
- Do closures always leak memory? When do they?
- How would you prove with a Heap Snapshot that a closure still retains a large object?

### Interview Answer

"A closure is not a special object. It is the normal result of lexical scoping
and first-class functions. What production cares about is retention: if a
long-lived listener or cache closes over a large object that is no longer needed
by business logic, that object stays reachable. I verify with heap snapshots and
retainer paths, not by blaming 'closures' in general."

---

### this Binding

`this` is determined at **call time** by **how the function is invoked**
(except arrow functions). Priority high → low:

**Rule 1: `new` binding**
```javascript
function Person(name) {
  this.name = name;
}
const person = new Person('Alice');
console.log(person.name); // 'Alice'
```

**Rule 2: Explicit binding (`call` / `apply` / `bind`)**
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

**Rule 3: Implicit binding (method call)**
```javascript
const obj = {
  name: 'Charlie',
  greet() { console.log(`Hello, ${this.name}`); }
};
obj.greet(); // 'Hello, Charlie'

const fn = obj.greet;
fn(); // sloppy: this → global object, name undefined → 'Hello, undefined'
      // strict / modules / extracted class methods: this is undefined → TypeError on this.name
```

**Rule 4: Default binding**
```javascript
function showThis() {
  'use strict';
  console.log(this); // undefined
}
showThis();

function showThisSloppy() {
  console.log(this); // browser: window; Node CJS: global
                     // (do not confuse with module top-level this)
}
showThisSloppy();
```

**Arrow functions**: no own `this`; they capture the outer lexical `this`.
`call` / `apply` / `bind` cannot change an arrow's `this`.

```javascript
const obj = {
  name: 'David',
  regularFn() { console.log(this.name); },          // 'David'
  arrowFn: () => { console.log(this.name); },       // depends on outer this (often undefined)
  nested() {
    const arrow = () => console.log(this.name);     // 'David'
    arrow();
  }
};
```

**React class `this`** (historical interview topic; modern code prefers function
components + hooks):

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

**Interview follow-ups**:
- Why do `obj.method()` and `const f = obj.method; f()` differ?
- After `bind`, can `call` change `this`? What about `new boundFn()`?

### Production Practice

- Prefer arrow handlers in class fields or explicit `bind` when extracting methods.
- In TypeScript/`"use strict"`/ESM, assume default `this` is `undefined`.
- Do not rely on sloppy-mode global `this` in shared libraries.

---

### Prototypes and Inheritance

#### Worth Digging Into

**1. Prototype chain**

- Every object has an internal `[[Prototype]]` slot (`Object.getPrototypeOf` /
  legacy `__proto__`).
- Functions have a `prototype` used when instances are created with `new`.
- Property lookup walks the chain until `null`.

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

Lookup order: own → prototype → … → `null`.
`hasOwnProperty` / `Object.hasOwn` look at own properties only.

---

**2. Inheritance patterns (shallow → deep)**

**Pattern 1: Prototype-chain inheritance (not recommended)**
```javascript
function Parent() { this.colors = ['red', 'blue']; }
function Child() {}
Child.prototype = new Parent(); // reference types shared across instances
```

**Pattern 2: Constructor stealing (not recommended)**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayName = function () { console.log(this.name); };

function Child(name) {
  Parent.call(this, name); // instance props only
}
const child = new Child('Alice');
// child.sayName(); // TypeError — prototype methods missing
```

**Pattern 3: Combinatorial inheritance (classic; calls parent twice)**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayName = function () { console.log(this.name); };

function Child(name, age) {
  Parent.call(this, name); // 1st call: instance props
  this.age = age;
}
Child.prototype = new Parent(); // 2nd call: only to wire prototype (wasteful + polluted)
Child.prototype.constructor = Child;
```

Downsides: parent constructor runs twice; leftover instance props sit on
`Child.prototype`.

**Pattern 4: Parasitic combinatorial inheritance (ES5 best practice; semantic basis of `class`)**
```javascript
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayName = function () { console.log(this.name); };

function Child(name, age) {
  Parent.call(this, name); // call parent ONCE for instance props
  this.age = age;
}

// Object.create only links the prototype; it does NOT run Parent
Child.prototype = Object.create(Parent.prototype);
Child.prototype.constructor = Child;

Child.prototype.sayAge = function () { console.log(this.age); };

const child = new Child('Alice', 10);
child.sayName(); // 'Alice'
child.sayAge();  // 10
```

> Common mistake: calling `Object.create(Parent.prototype)` a “second constructor
> call.” `Object.create` does **not** invoke `Parent`. Only `Parent.call(this)`
> runs the constructor. The true “called twice” pattern is
> `Child.prototype = new Parent()`.

**Pattern 5: ES6 `class` (recommended)**
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
    super(name); // must call super first
    this.age = age;
  }
  sayAge() { console.log(this.age); }
}
```

`class` is still prototype inheritance under the hood, with cleaner static
inheritance, `super`, and non-enumerable methods by default.

### Trade-offs

- Prototype methods are memory-efficient (shared functions).
- `class` is clearer, but interviewers still expect the ES5 story.
- Prefer composition when inheritance depth starts hiding ownership of state.

---

### Event Loop

The runtime schedules work through the **event loop**. Rendering, timers, and
I/O callbacks are not one uninterrupted synchronous stack.

#### Browser model (simplified, interview-ready)

**Queues and phases — do not stuff rendering into the macrotask list**:

| Category | Typical sources | Notes |
|----------|-----------------|-------|
| Macrotask (task) | `setTimeout`, `setInterval`, `setImmediate` (Node), I/O, UI events, `postMessage` | Usually one task per turn |
| Microtask | `Promise.then/catch/finally`, `queueMicrotask`, `MutationObserver` | After a task, **drain the entire microtask queue** (including newly enqueued ones) |
| Rendering | style / layout / paint | **Not** a macrotask; happens when a frame opportunity arrives and the browser needs it |
| `requestAnimationFrame` | callbacks before the next paint | Aligned with the render pipeline; **not** a plain macrotask |

**Typical conceptual turn**:
1. Run one macrotask (or the script itself)
2. Drain the microtask queue
3. If needed and a frame is due: `rAF` → render
4. Take the next macrotask…

```javascript
console.log('1-sync');

setTimeout(() => console.log('2-macrotask'), 0);

Promise.resolve().then(() => console.log('3-microtask'));
queueMicrotask(() => console.log('3b-queueMicrotask'));

requestAnimationFrame(() => console.log('4-rAF')); // near paint time

console.log('5-sync');
// sync → microtasks → (then timeout / rAF; interleaving depends on implementation & paint)
```

**Example with microtasks between macrotasks**:

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

#### Node.js event-loop phases

Node (libuv) has finer phases (common interview topic):

`timers` → `pending callbacks` → `idle/prepare` → `poll` → `check` (`setImmediate`) → `close callbacks`

- `process.nextTick`: runs **before** Promise microtasks (drain nextTick queue, then Promise microtasks).
- The same snippet may print differently in browser vs Node — be ready with
  `setTimeout` vs `setImmediate` and `nextTick` examples.

```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
// typical: nextTick → promise → (timeout / immediate order depends on context)
```

### Production Practice

- Too many microtasks can stall paint frames → chunk work, use `scheduler` /
  `requestIdleCallback`, or Workers.
- Do not simulate long tasks with an endless `then` chain.

**Interview follow-ups**:
- Why is “UI rendering is a macrotask” wrong? How does it relate to microtasks and `rAF`?
- How does `queueMicrotask` differ from `Promise.resolve().then`?
- Why can recursive `process.nextTick` starve I/O?

---

### Asynchronous Programming

#### 1. Promise basics

Three states: `pending` → `fulfilled` / `rejected` (settlement is irreversible).

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    Math.random() > 0.5 ? resolve('ok') : reject(new Error('fail'));
  }, 1000);
});

promise
  .then((result) => console.log(result))
  .catch((error) => console.error(error))
  .finally(() => console.log('done'));
```

#### 2. Combinators (static methods)

```javascript
// all: every success; first failure rejects; empty iterable → resolve([])
Promise.all([fetch('/api/user'), fetch('/api/posts')])
  .then((responses) => Promise.all(responses.map((r) => r.json())))
  .catch((err) => console.error('one failed', err));

// allSettled: wait for all to settle
Promise.allSettled([fetch('/api/user'), fetch('/api/bad')]).then((results) => {
  results.forEach((r, i) => {
    console.log(i, r.status, r.status === 'fulfilled' ? r.value : r.reason);
  });
});

// race: first settled (fulfill or reject)
Promise.race([
  fetch('/api/fast'),
  new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 3000)),
]);

// any: first fulfilled; all rejected → AggregateError
Promise.any([fetch('/api/a'), fetch('/api/b')]);
```

| Method | Success when | Failure when |
|--------|--------------|--------------|
| `Promise.all` | all fulfilled | first rejected |
| `Promise.allSettled` | all settled | does not reject because a child rejected |
| `Promise.race` | first settled | that first settlement is a reject |
| `Promise.any` | first fulfilled | all rejected |

#### 3. Hand-rolled Promise (teaching version: microtasks, not `setTimeout`)

> The spec requires `then` callbacks to be scheduled as **microtasks**. Using
> `setTimeout` puts them on the macrotask queue and breaks ordering vs real Promises.

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
      // simplified: no full thenable assimilation
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
            // full impl also needs resolvePromise(promise2, x, ...)
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
        MyPromise.resolve(p)
          .then(
            (value) => {
              results[index] = { status: 'fulfilled', value };
            },
            (reason) => {
              results[index] = { status: 'rejected', reason };
            }
          )
          .then(() => {
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

Hand-write checklist:
- Empty iterable: `all` / `allSettled` immediately `resolve([])`
- Wrap non-thenables with `Promise.resolve`
- Preserve order with index assignment, not `push`
- Schedule with microtasks — never fake it with `setTimeout`

#### 4. async / await

An `async` function always returns a Promise. `await` suspends that async
function; under the hood it is still Promise + microtasks.

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

// ✅ parallel
async function fetchAll() {
  const [user, posts] = await Promise.all([
    fetch('/api/user').then((r) => r.json()),
    fetch('/api/posts').then((r) => r.json()),
  ]);
  return { user, posts };
}
```

**Top-level await (ESM)**: in an **ES Module**, you may `await` at the top
level without wrapping an `async` function. Supported by native browser modules
and Node `"type": "module"` / `.mjs`. Still unsupported in CJS and classic
`<script>`.

```javascript
// main.mjs
const config = await fetch('/config.json').then((r) => r.json());
export default config;
```

**Error-handling patterns**: unify try/catch and error taxonomy; isolate
concurrency with `allSettled`; optional helper `to(promise) → [err, data]`
(note weaker TS inference).

#### 5. AbortController, timeouts, and concurrency pools

```javascript
function fetchWithTimeout(url, ms, init = {}) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(new Error('timeout')), ms);
  return fetch(url, { ...init, signal: controller.signal }).finally(() =>
    clearTimeout(timer)
  );
}

// vs Promise.race timeout: AbortController can actually cancel fetch and cut waste

async function mapPool(items, limit, worker) {
  const ret = new Array(items.length);
  let i = 0;
  async function run() {
    while (i < items.length) {
      const cur = i++;
      ret[cur] = await worker(items[cur], cur);
    }
  }
  await Promise.all(
    Array.from({ length: Math.min(limit, items.length) }, () => run())
  );
  return ret;
}

// usage: at most 3 concurrent
await mapPool(urls, 3, (url) =>
  fetchWithTimeout(url, 5000).then((r) => r.json())
);
```

**Interview follow-ups**:
- After a `Promise.race` timeout, is the original request still in flight? How do you cancel?
- Why is a concurrency pool safer than blind `Promise.all` of 1000 tasks?

### Production Practice

- Prefer `AbortController` for cancellation; classify timeout vs abort vs HTTP errors.
- Bound concurrency for bulk work; combine with `allSettled` for partial failure UX.
- Keep user-facing errors separate from developer/ops diagnostics.

---

### Memory Management

#### 1. GC overview

Engines use **reachability analysis** plus mark-sweep / mark-compact style
algorithms: mark from roots (globals, call stack, registers), reclaim unreachable
objects.

#### 2. V8 lens (interview bonus)

**Hidden classes (Maps) and inline caches (IC)**
- Similar object shapes share a hidden class; property access takes a fast path.
- Method calls on monomorphic shapes can hit IC caches.
- **Deopt**: shape churn, type flips, `arguments` abuse, etc. can bail optimized
  code back to slower paths — often felt as “occasional jank.”

```javascript
// Good for stable shapes: isomorphic initialization
function Point(x, y) {
  this.x = x;
  this.y = y;
}

// Bad: add/delete properties at runtime or change property order
const a = { x: 1, y: 2 };
const b = { y: 2, x: 1 }; // may be a different shape
```

**Generational GC**
- **Young / Nursery**: short-lived objects; Scavenge (semi-space copy); frequent and cheap
- **Old**: long-lived objects; Mark-Sweep / Mark-Compact; more expensive
- Promotion: objects that survive young collections graduate to old space

**Heap Snapshot workflow**
1. Chrome DevTools → Memory → Heap snapshot
2. Snapshot before/after the interaction; use Comparison
3. Watch Detached HTMLElement and closure Retainer Paths
4. Use Allocation instrumentation / Timeline to see “who allocates”

#### 3. Common leak patterns

**Accidental globals**
```javascript
function leak() {
  accidentalGlobal = new Array(1e6); // missing var/let/const
}
```

**Closures retaining large objects**
```javascript
function createClosure() {
  const largeData = new Array(1e6).fill('data');
  const summary = largeData.length; // keep only what you need
  return () => console.log(summary);
}
```

**Listeners / timers not cleaned up**
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

#### 4. Object pools (hot allocation paths)

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

**Interview follow-ups**:
- Why does stable object shape help V8 performance?
- How do young vs old collection strategies differ?
- How do you read a Retainer path in a Heap Snapshot?

### Interview Answer

"A JS memory leak means something remains reachable after the product no longer
needs it. I reproduce, take heap snapshots, compare retained objects, walk
retainer paths, then fix lifecycle or cache policy. Hidden-class / IC / deopt
issues are performance, not necessarily leaks — diagnose with profiles, not only
snapshots."

---

### ES6+ Core Features

#### 1. Proxy and Reflect

`Proxy` intercepts fundamental object operations. `Reflect` provides the matching
default behaviors so traps can forward safely.

| Trap | Intercepts | Reflect |
|------|------------|---------|
| `get` | property read | `Reflect.get` |
| `set` | property write | `Reflect.set` |
| `has` | `in` | `Reflect.has` |
| `deleteProperty` | `delete` | `Reflect.deleteProperty` |
| `ownKeys` | `Object.keys` / `for...in` | `Reflect.ownKeys` |
| `getOwnPropertyDescriptor` | descriptors | `Reflect.getOwnPropertyDescriptor` |
| `defineProperty` | define | `Reflect.defineProperty` |
| `getPrototypeOf` / `setPrototypeOf` | prototype | matching Reflect APIs |
| `isExtensible` / `preventExtensions` | extensibility | matching Reflect APIs |
| `apply` / `construct` | call / `new` | matching Reflect APIs |

Simplified Vue 3 reactivity core:

```javascript
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
  if (deps) deps.forEach((effect) => effect());
}

function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key);
      const result = Reflect.get(target, key, receiver);
      if (typeof result === 'object' && result !== null) {
        return reactive(result);
      }
      return result;
    },
    set(target, key, value, receiver) {
      const oldValue = target[key];
      const result = Reflect.set(target, key, value, receiver);
      if (oldValue !== value) trigger(target, key);
      return result;
    },
    deleteProperty(target, key) {
      const hadKey = key in target;
      const result = Reflect.deleteProperty(target, key);
      if (hadKey && result) trigger(target, key);
      return result;
    }
  });
}

function effect(fn) {
  activeEffect = fn;
  fn();
  activeEffect = null;
}
```

**Interview points**:
- Proxy vs `Object.defineProperty`: Proxy can intercept new props, array index
  changes, and `delete` — the core reason Vue 3 reactivity is more complete.
- Prefer `Reflect.get` over `target[key]` so `receiver` / inheritance stay correct.

---

#### 2. Symbol

Seventh primitive type; each `Symbol()` is unique. Used as non-string property
keys to avoid name collisions.

```javascript
const s1 = Symbol('description');
const s2 = Symbol('description');
console.log(s1 === s2); // false

const s3 = Symbol.for('shared');
const s4 = Symbol.for('shared');
console.log(s3 === s4); // true — global registry

const ID = Symbol('id');
const user = { name: 'Alice', [ID]: 12345 };
console.log(Object.keys(user)); // ['name']
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
console.log(Reflect.ownKeys(user)); // ['name', Symbol(id)]
```

| Well-known Symbol | Role |
|-------------------|------|
| `Symbol.iterator` | default iterator (`for...of`, spread) |
| `Symbol.asyncIterator` | async iterator (`for await...of`) |
| `Symbol.hasInstance` | customize `instanceof` |
| `Symbol.toPrimitive` | object → primitive |
| `Symbol.toStringTag` | customize `Object.prototype.toString` |
| `Symbol.species` | derived constructor for subclassing builtins |

```javascript
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
        if (current <= end) return { value: current++, done: false };
        return { done: true };
      }
    };
  }
}
```

---

#### 3. Iterator and Generator

- **Iterator protocol**: object with `next()` returning `{ value, done }`.
- **Iterable protocol**: object with `[Symbol.iterator]()` returning an iterator.
- **Generators** (`function*` + `yield`) implement both conveniently.

```javascript
function* rangeGenerator(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

function* conversation() {
  const name = yield 'What is your name?';
  const hobby = yield `Hi ${name}, hobby?`;
  return `${name} likes ${hobby}`;
}

const talk = conversation();
talk.next();           // question
talk.next('Alice');    // greeting
talk.next('coding');   // done

function* outer() {
  yield 1;
  yield* (function* () { yield 'a'; yield 'b'; })();
  yield 2;
}
console.log([...outer()]); // [1, 'a', 'b', 2]
```

Async generators for streaming / paging:

```javascript
async function* fetchPages(baseUrl) {
  let page = 1;
  let hasMore = true;
  while (hasMore) {
    const response = await fetch(`${baseUrl}?page=${page}`);
    const data = await response.json();
    hasMore = data.hasMore;
    page++;
    yield data.items;
  }
}

async function loadAll() {
  const allItems = [];
  for await (const items of fetchPages('/api/posts')) {
    allItems.push(...items);
  }
  return allItems;
}
```

---

#### 4. WeakMap / WeakSet

Weak keys do not prevent GC. When the key becomes unreachable, the entry may be
cleared.

| Trait | Map / Set | WeakMap / WeakSet |
|-------|-----------|-------------------|
| Keys | any value | **objects**, and **unregistered Symbols** (GC-able; not `Symbol.for`) |
| References | strong | weak |
| Iterable / `size` | yes | no (GC timing is non-deterministic) |

```javascript
let obj = { name: 'temp' };
const wm = new WeakMap();
wm.set(obj, 'associated');
obj = null; // entry becomes eligible for GC

// ES2023+: non-registered Symbols may be WeakMap keys
const sym = Symbol('ephemeral');
wm.set(sym, 123);
```

Use cases: private per-instance data, DOM associations without strong retention.

---

#### 5. structuredClone / WeakRef / FinalizationRegistry

```javascript
const original = { d: new Date(), nested: { n: 1 } };
original.self = original;
const cloned = structuredClone(original);
console.log(cloned.self === cloned); // true
console.log(cloned.d instanceof Date); // true
// no functions, DOM nodes, Symbol keys as props, etc. — check environment support

let cache = new Map();
function getWidget(id) {
  const hit = cache.get(id)?.deref();
  if (hit) return hit;
  const widget = createExpensiveWidget(id);
  cache.set(id, new WeakRef(widget));
  return widget;
}

const registry = new FinalizationRegistry((heldValue) => {
  console.log('cleanup', heldValue);
});
(function () {
  const obj = {};
  registry.register(obj, 'resource-id');
})();
```

Deep-clone choice: prefer `structuredClone` at runtime; hand-roll when you need
functions / special `undefined` policy; use `JSON` only for simple dirty copies.

---

#### 6. Compact Promise combinator (native attach style)

Full `MyPromise` lives in Asynchronous Programming. Compact interview form:

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

Checklist: empty array, index order, `Promise.resolve` wrap, fail-fast.

---

#### 7. Module system and packaging boundaries

**ESM vs CJS**

| Trait | ESM | CJS |
|-------|-----|-----|
| Loading | static analysis | runtime `require` |
| Bindings | live bindings | value copy (exported object props are a separate story) |
| Tree shaking | friendly | hard |
| Top-level await | yes | no |

```javascript
// counter.mjs — live bindings
export let count = 0;
export function increment() { count++; }
```

**`package.json` `exports` and dual package hazard**

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

- `sideEffects: false` claims the whole package is safely tree-shakeable. If you
  ship CSS/polyfill side effects, use an **array whitelist** (as above).
- **Do not** put two `"sideEffects"` keys in the same JSON object — illegal /
  last-wins and confuses tooling.
- **Dual package hazard**: consumers loading both CJS and ESM copies of the same
  package → broken singletons / `instanceof`. Mitigate with a single entry
  surface, no deep imports, converged `exports`, and docs that pick one
  consumption style.

Dynamic `import()` returns a Promise of the module namespace — route-level lazy
load and optional polyfills.

Tree-shaking failures: top-level side effects, default-export mega-objects then
destructure, or incorrect `sideEffects` metadata.

**Interview follow-ups**:
- What is the dual package hazard, and how do library authors avoid it?
- What production bugs appear when `sideEffects` is wrong?

---

## TypeScript Core

### Type System

#### Worth Digging Into

**1. Base types and safety boundaries**

```typescript
let num: number = 42;
let tuple: [string, number] = ['Alice', 30];

let anything: any = 'hello';
anything = 42; // bypasses checking — use sparingly

let value: unknown = 'hello';
// value.toUpperCase(); // ❌
if (typeof value === 'string') {
  value.toUpperCase(); // ✅
}

function throwError(message: string): never {
  throw new Error(message);
}
```

**2. Object types: `interface` / `type`**

```typescript
interface User {
  id: number;
  name: string;
  email?: string;
  readonly createdAt: Date;
}

type Point = { x: number; y: number };
```

Choose `interface` for extensible object contracts and declaration merging;
choose `type` for unions / intersections / mapped / conditional forms.

**3. Function types and variance (`strictFunctionTypes`)**

Under `--strictFunctionTypes` (included in `strict`), **function parameters are
checked contravariantly** when assigning function types.

```typescript
type Handler<T> = (event: T) => void;

interface Animal { name: string }
interface Dog extends Animal { bark(): void }

const animalHandler: Handler<Animal> = (a) => console.log(a.name);
const dogHandler: Handler<Dog> = (d) => d.bark();

// dogHandler cannot be Handler<Animal>: a Cat might be passed
// let h: Handler<Animal> = dogHandler; // ❌ under strictFunctionTypes

// animalHandler can safely handle Dog (wider parameter)
let h2: Handler<Dog> = animalHandler; // ✅
```

Intuition: **return types covariant, parameters contravariant** — a function that
accepts a wider input may be placed where a narrower input is expected.

Method syntax vs function-property syntax historically differed; modern teaching
uses `strictFunctionTypes` + function-typed properties.

**4. Overloads**

```typescript
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}
```

### Interview Answer

"`any` turns the checker off; `unknown` forces narrowing. At API / storage /
third-party boundaries I model input as `unknown`, validate, then enter trusted
domain types. Variance mistakes usually show up as 'safe-looking' handler
assignments that accept too-narrow parameters."

---

### Advanced Types

#### Worth Digging Into

**1. Unions and intersections**

```typescript
type ID = string | number;

function printId(id: ID) {
  if (typeof id === 'string') {
    console.log(id.toUpperCase());
  } else {
    console.log(id.toFixed(2));
  }
}

type Named = { name: string };
type Aged = { age: number };
type Person = Named & Aged;
```

**2. Type guards**

```typescript
function padLeft(value: string, padding: string | number) {
  if (typeof padding === 'number') {
    return Array(padding + 1).join(' ') + value;
  }
  return padding + value;
}

type Animal =
  | { type: 'cat'; meow: () => void }
  | { type: 'dog'; bark: () => void };

function isCat(animal: Animal): animal is Extract<Animal, { type: 'cat' }> {
  return animal.type === 'cat';
}
```

**3. Generics**

```typescript
function identity<T>(arg: T): T {
  return arg;
}

interface Lengthwise { length: number }

function logLength<T extends Lengthwise>(arg: T): void {
  console.log(arg.length);
}

function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

**4. Utility types**

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type PartialUser = Partial<User>;
type RequiredUser = Required<PartialUser>;
type ReadonlyUser = Readonly<User>;
type UserPreview = Pick<User, 'id' | 'name'>;
type UserWithoutAge = Omit<User, 'age'>;
type UserMap = Record<number, User>;
type User2 = ReturnType<typeof getUser>;
type CreateParams = Parameters<typeof create>;
```

**5. Conditional types**

```typescript
type IsString<T> = T extends string ? true : false;
type ToArray<T> = T extends any ? T[] : never; // distributive
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

### Production Practice

- Prefer explicit domain models over clever mapped-type puzzles in shared APIs.
- Validate untrusted runtime data; types erase at runtime.
- Avoid exporting broad `any` from packages.

---

### Engineering Practice

#### 1. Important `tsconfig` knobs

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

| Option | Meaning |
|--------|---------|
| `moduleResolution` | `node16`/`nodenext` for Node ESM; `bundler` for Vite/Webpack |
| `paths` | compile-time path mapping; runtime needs bundler/tool sync |
| `composite` + `references` | Project References for large monorepo incremental builds |
| `strict` | enables the strict suite (including `strictFunctionTypes`) |

#### 2. Declaration files, `declare module`, Declaration Merging

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

// Declaration Merging: same-name interfaces merge
interface Window {
  __APP_VERSION__: string;
}

// Module augmentation
declare module './event-bus' {
  interface EventMap {
    'user:login': { userId: string };
  }
}
```

Library authors: emit `.d.ts` with `declaration: true`; pay attention to
`export =` / `export default` vs consumer `esModuleInterop`.

#### 3. Decorators: TS 5+ vs legacy `experimentalDecorators`

| | **Stage 3 standard decorators (TS 5+)** | **Legacy experimental** |
|--|-----------------------------------------|-------------------------|
| Flag | modern `decorators` semantics (evolving with versions) | `experimentalDecorators` + often `emitDecoratorMetadata` |
| Evaluation / signature | closer to TC39; different init semantics | heavy Angular / Nest history |
| Metadata | no default design-type emit | `emitDecoratorMetadata` + `reflect-metadata` |

Migration: new projects follow standard decorators; Nest/Angular legacy stacks
should explicitly stay on experimental assumptions — do not mix both mental models.

#### 4. API type modeling

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

Boundary rule: network data is `unknown` until validated into trusted domain types.

### Production Practice

- Prefer explicit Result / discriminated-union errors over silent catch-and-continue.
- Use exhaustive checks (`assertNever`) for state machines.
- Keep transport errors distinct from domain errors.

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}
```

---

## Interview Self-check

> Each question includes an answer plus 2–3 follow-ups. Senior interviewers
> escalate from definition → edge cases / production incidents / implementation trade-offs.

### Q1: What is a closure? Where is it used?

**Answer**: A closure is a function paired with its lexical environment. After the
outer function returns, the inner function can still access referenced outer
bindings. Uses: private state, currying, memoization, module pattern. Retention
is about **referenced bindings**, not a vague “whole context never dies.”

**Follow-ups**:
1. Why does `for (var i…)` + async callbacks print one final value? How does `let` fix it?
2. How do you locate a Retainer when a closure holds DOM / a large array?
3. Can engines drop unused closed-over locals? What does that mean for “leak” claims?

---

### Q2: What is the event-loop execution order?

**Answer**: Run one macrotask → drain microtasks → (optional) rAF / render → next
macrotask. Macrotasks include timers, I/O, UI events; microtasks include Promise,
`queueMicrotask`, MutationObserver. **Rendering is not a macrotask**; `rAF`
aligns with frames and should not be listed as a plain macrotask.

**Follow-ups**:
1. Why can a microtask infinite loop freeze the page? How does that differ from macrotask chunking?
2. Is `requestAnimationFrame` or `setTimeout(0)` better for animation?
3. In Node, what is `process.nextTick` priority vs Promise microtasks?

---

### Q3: What Promise static methods exist, and when?

**Answer**:

| Method | Use |
|--------|-----|
| `all` | all succeed; fail fast; `[]` → `[]` |
| `allSettled` | wait for all; status array |
| `race` | first settled |
| `any` | first fulfilled; all fail → AggregateError |
| `resolve` / `reject` | create settled promises |

**Follow-ups**:
1. How does `all` treat non-Promise values? Empty array?
2. Why use `allSettled` when you want “as many successes as possible”?
3. What side effect does `race` timeout have? How do you pair AbortController?

---

### Q4: What are `this` binding rules and priority?

**Answer**: `new` > explicit (`call`/`apply`/`bind`) > implicit (method call) >
default (sloppy → global; strict/`class`/ESM → `undefined`). Arrows inherit
lexical `this`.

**Follow-ups**:
1. Why do extracted method callbacks lose `this`? Show strict vs sloppy.
2. After `bind`, can `call` change `this`? What about `new boundFn()`?
3. Why did React class components need `bind` or class-field arrows?

---

### Q5: How does prototype-chain lookup work?

**Answer**: own props → `[[Prototype]]` chain → `null`. `Constructor.prototype`
is the instance prototype; prefer `Object.getPrototypeOf` over `__proto__`.

**Follow-ups**:
1. `hasOwnProperty` vs `in`?
2. Risks of mutating `Object.prototype`?
3. Uses of `Object.create(null)`?

---

### Q6: ES6 class inheritance vs ES5 inheritance?

**Answer**: Class is prototype sugar with stricter details (`super`, static
inheritance, non-enumerable methods). ES5 best practice is **parasitic
combinatorial inheritance** (`Parent.call` + `Object.create(Parent.prototype)`)
— parent runs **once**. Classic combinatorial inheritance (`new Parent()` on the
prototype) runs it twice.

**Follow-ups**:
1. Why is “`Object.create` = second parent call” wrong?
2. Class fields vs prototype methods regarding enumeration?
3. Caveats when `extends` Array / other builtins?

---

### Q7: How do async/await and Promise differ?

**Answer**: `async`/`await` is Promise sugar with better readability and
try/catch; it does not eliminate microtask scheduling. `async` functions return
Promises.

**Follow-ups**:
1. Is “await only inside async functions” still true? (ESM top-level await)
2. After `await`, what kind of task continues?
3. How do you explain serial `await` vs `Promise.all` to a product partner?

---

### Q8: `unknown` vs `any` in TypeScript?

**Answer**: `any` disables checking; `unknown` must be narrowed. Prefer `unknown`
for cross-boundary data.

**Follow-ups**:
1. How do you assign `unknown` to a concrete type?
2. Why is `any` contagious?
3. How do you combine with `zod` / `io-ts`?

---

### Q9: How do generic constraints work?

**Answer**: `T extends Constraint`; key constraints use `K extends keyof T`.

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

**Follow-ups**:
1. What does `extends` mean in constraints vs conditional types?
2. How do you require `T` to be a subset of a union?
3. When are default type parameters (`T = string`) useful?

---

### Q10: Common memory-leak scenarios and fixes?

**Answer**: accidental globals, closures retaining large objects, listeners/timers
not cleaned, Detached DOM still referenced from JS. Fix with snapshot diffs and
paired register/cleanup.

**Follow-ups**:
1. Leak vs reasonable cache — how do you tell?
2. When WeakMap vs WeakRef?
3. In V8 generational terms, how do you think about long-lived caches?

---

### Q11: Debounce vs throttle?

**Answer**:

| | Debounce | Throttle |
|--|----------|----------|
| Behavior | run after quiet period (retriggers reset timer) | at most once per interval |
| Scenes | search input, resize end | scroll, high-frequency clicks, **skill cooldown** |

**Follow-ups**:
1. trailing vs leading? How does lodash configure them?
2. Is skill cooldown closer to debounce or throttle? Why?
3. In React hooks, how do you keep latest props / avoid stale `this`?

---

### Q12: What are Partial, Pick, Omit?

**Answer**: `Partial` makes all optional; `Pick` selects keys; `Omit` excludes
keys. Common for form drafts, API patches, DTO shaping.

**Follow-ups**:
1. Hand-write `MyPartial<T>`?
2. `Omit` vs `Exclude`?
3. Deep Partial — how, and what are the traps?

---

### Q13: How do you deep-clone, and what are the edges?

**Answer**: Prefer `structuredClone`. Hand-rolled must handle cycles (WeakMap),
Date/RegExp/Map/Set, Symbol keys (`Reflect.ownKeys`), descriptors /
non-enumerables, and prototype policy. `JSON` drops functions, `undefined`, cycles.

**Follow-ups**:
1. Why is `for...in` + `hasOwnProperty` insufficient?
2. Functions, DOM, cross-realm objects?
3. Boundary vs shallow `{...obj}` / `Object.assign`?

---

### Q14: WeakMap vs Map?

**Answer**: WeakMap keys are objects or **GC-able unregistered Symbols**, weakly
held, non-iterable, no `size`. Good for private data and leak-safe associations.

**Follow-ups**:
1. Can `Symbol.for` keys be WeakMap keys?
2. Why no key enumeration?
3. Why does Vue 3 `targetMap` use WeakMap?

---

### Q15: Promise.all vs allSettled?

**Answer**: `all` fails fast; `allSettled` always waits and reports per-item
status. Dashboards that should render despite partial failure use the latter.

**Follow-ups**:
1. Why must hand-written `all` resolve `[]` immediately for empty input?
2. How do you still collect successes when `all` fails? (vs allSettled / wrappers)
3. Error strategy when combining with a concurrency pool?

---

### Q16: How do you check a value’s type?

**Answer**: `typeof` (primitives/functions; note `null`); `instanceof` (prototype
chain; careful across iframes); `Object.prototype.toString.call`; `Array.isArray`.

**Follow-ups**:
1. Why `typeof null === 'object'`?
2. Reliable Promise detection?
3. How do you turn a runtime check into a TS type guard?

---

### Q17: Core differences among `var` / `let` / `const`?

**Answer**: scope (function vs block), TDZ, redeclaration, `const` non-rebinding.
Engineering default: `const`.

**Follow-ups**:
1. What is the TDZ? Why can `typeof` on an undeclared `let` throw?
2. How do mutable `const` object properties reconcile with immutability practices?
3. Why do modern style guides ban new `var`?

---

### Q18: `==` vs `===`, and why ban `==`?

**Answer**: `===` does no coercion; `==` coerces and is footgun-prone. Ban reduces
ambiguity; `x == null` is a rare documented exception.

**Follow-ups**:
1. Walk through `[] == ![]` (show reasoning, not memorized boolean).
2. `Object.is` vs `===` for `NaN` and `+0/-0`?
3. How do you document an allowlist for `==`?

---

### Q19: Browser vs Node event-loop differences?

**Answer**: Both have macro/micro tasks, but Node has timers/poll/check phases;
`process.nextTick` precedes Promise microtasks; browsers emphasize paint frames
and `rAF`. Isomorphic code may print differently.

**Follow-ups**:
1. Inside an I/O callback, who wins between `setImmediate` and `setTimeout(0)`?
2. Why is recursive `nextTick` dangerous?
3. How do you avoid depending on one loop order in SSR isomorphic code?

---

### Q20: How to choose `interface` vs `type`?

**Answer**: object contracts / declaration merging → `interface`; unions, mapped
types, conditionals → `type`. Team consistency beats dogma.

**Follow-ups**:
1. Demo Declaration Merging extending `Window`.
2. Can `type` merge? If not, how do you extend?
3. For public library types, which choice helps downstream augmentation?

---

### Must-know hand-written snippets

**1. Debounce**

```typescript
function debounce<T extends (...args: any[]) => any>(fn: T, delay: number) {
  let timer: ReturnType<typeof setTimeout> | null = null;
  return function (this: unknown, ...args: Parameters<T>) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
```

**2. Throttle**

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

**3. Deep clone (cycles / special objects / Symbol keys)**

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

  const result = (
    Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value))
  ) as T;
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

**4. call / apply / bind (falsy context + bind + `new`)**

```typescript
Function.prototype.myCall = function (context: any, ...args: any[]) {
  // null/undefined → globalThis; do NOT use context || globalThis (0/''/false)
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
    const isNew = this instanceof bound;
    return fn.apply(
      isNew
        ? this
        : context === null || context === undefined
          ? globalThis
          : context,
      [...bindArgs, ...callArgs]
    );
  };
  if (fn.prototype) {
    bound.prototype = Object.create(fn.prototype);
  }
  return bound;
};
```

**Quality checklist**:
- Promise: `queueMicrotask`, empty array, index order
- Deep clone: WeakMap + `Reflect.ownKeys` + descriptors
- bind: `new` path and falsy `thisArg`

---

### Open-ended design prompts

**D1: Design a pluggable frontend analytics SDK — architecture?**

**Reference**:
- Micro-kernel + plugins: Core owns lifecycle / event bus; features via Plugin
- Protocol: `install`/`uninstall`, hooks (`beforeSend`/`afterSend`)
- Pipeline: collect → queue → batch → retry; idle reporting to protect main thread
- TS: plugin generics, Declaration Merging for event maps, strict event types
- Trade-offs: tree-shaking vs batteries-included; sync init vs lazy load

**Senior follow-ups**:
1. How do you avoid circular plugin init?
2. How do failure retry and privacy consent state changes compose?
3. How do you prevent dual-package double singletons for CJS/ESM consumers?

**D2: Suspected closure leak that won’t reproduce — investigation plan?**

**Reference**:
- Memory: snapshot diffs, Allocation timeline, Detached DOM
- Usual sources: listeners, timers, React effects without cleanup
- Path: Retainer Path → WeakMap / weak refs → regression test
- Prevention: lint (hooks deps), CI memory smoke tests

**Senior follow-ups**:
1. How do you prove “leak” vs “cache ceiling too high”?
2. Without DevTools in production, what signals can you still collect?
3. Why is “null everything” a bad first fix without finding the root retainer?

**D3: Design a request layer unifying timeout, cancel, idempotency, concurrency?**

**Reference**:
- AbortController end-to-end; timeout = abort + typed error
- Idempotency keys + in-flight Promise dedupe (payments)
- Concurrency pool + priority queue; `allSettled` for partial failure
- Types: DTO → validate → Domain; errors as discriminated unions

**Senior follow-ups**:
1. Client dedupe vs server idempotency for double-click pay — ownership?
2. After cancel, how do you avoid setState-after-unmount in React?
3. How do 429 / timeout backoff avoid retry stampedes?

---

## Production Scenarios

### Fintech payments: idempotent request wrapper

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

async function submitPayment(
  orderId: string,
  amount: number,
  signal?: AbortSignal
) {
  return client.request({
    url: '/api/payment',
    method: 'POST',
    data: { orderId, amount },
    idempotencyKey: `payment-${orderId}`,
    signal,
  });
}
```

### Scenario: page gets slower over time

Check accumulating listeners, timers, detached DOM, unbounded caches, and large
closures. Use Performance + Memory panels; compare heap snapshots across repeated
navigation.

### Scenario: type-safe API client

Define request/response types, validate at the boundary, return typed results,
and keep transport errors separate from domain errors.

### Scenario: large batch requests

Use a concurrency limit instead of firing thousands of Promises. Add timeouts,
cancellation, retry policy, and partial-failure handling (`allSettled`).

### Scenario: untrusted API data breaks a release

Types erase at runtime. Validate at the boundary, log schema mismatch with
release/endpoint metadata, return safe fallbacks, and align contract tests with
backend owners.

### Scenario: main-thread jank during data processing

Capture a performance trace. If long tasks come from parse/sort/format, chunk
work, move CPU-heavy logic to a Web Worker, and measure INP / custom interaction
latency before and after.

### Game scenario: skill cast with cooldown (throttle semantics, not debounce)

```typescript
interface Skill {
  id: string;
  name: string;
  cooldown: number; // ms
}

class SkillSystem {
  private cooldowns = new Map<string, number>();

  canCast(skill: Skill): boolean {
    const last = this.cooldowns.get(skill.id);
    if (!last) return true;
    return Date.now() - last >= skill.cooldown;
  }

  cast(skill: Skill, action: () => void): boolean {
    if (!this.canCast(skill)) {
      console.log(`${skill.name} is on cooldown...`);
      return false;
    }
    this.cooldowns.set(skill.id, Date.now());
    action();
    console.log(`Cast ${skill.name}!`);
    return true;
  }

  getRemainingCooldown(skill: Skill): number {
    const lastCast = this.cooldowns.get(skill.id);
    if (!lastCast) return 0;
    return Math.max(0, skill.cooldown - (Date.now() - lastCast));
  }
}

const system = new SkillSystem();
const fireball: Skill = { id: 'fireball', name: 'Fireball', cooldown: 3000 };

system.cast(fireball, () => console.log('launching...')); // ✅
system.cast(fireball, () => {}); // ❌ cooldown
```

Contrast:
- **Debounce**: cast only after the player stops pressing (timer resets) — wrong for skill CD
- **Throttle / cooldown**: after a successful cast, enter a fixed window — this model

---

## Summary

- **Foundations**: closures (referenced bindings), `this`, prototypes
  (combinatorial vs parasitic combinatorial), event loop (render / rAF / Node phases)
- **Hand-writes**: Promise (microtask + empty array), debounce/throttle, deep-clone
  edges, call/bind (falsy + `new`)
- **Engineering**: AbortController, package `exports` / `sideEffects`, TS variance
  and declaration files
- **Interviews**: definitions plus follow-up edges and production trade-offs

For interviews, explain JavaScript and TypeScript as a runtime plus a type layer:
lexical scope, prototype delegation, event-loop scheduling, garbage collection,
and static modeling — always connected to production risks such as leaks, main-thread
jank, API uncertainty, and maintainability.
