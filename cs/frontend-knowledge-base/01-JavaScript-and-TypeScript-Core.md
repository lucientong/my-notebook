Language: English | [中文](../前端知识库/01-JavaScript与TypeScript核心.md)

# JavaScript and TypeScript Core

---

## Table of Contents

### JavaScript Core

1. [Scope and Closures](#scope-and-closures)
2. [Prototypes and Inheritance](#prototypes-and-inheritance)
3. [Event Loop](#event-loop)
4. [Asynchronous Programming](#asynchronous-programming)
5. [Memory Management](#memory-management)
6. [ES6+ Core Features](#es6-core-features)

### TypeScript Core

7. [Type System](#type-system)
8. [Advanced Types](#advanced-types)
9. [Engineering Practice](#engineering-practice)

### Self-check and Practice

10. [Interview Self-check](#interview-self-check)
11. [Production Scenarios](#production-scenarios)

---

## Scope and Closures

### Core Concepts

JavaScript uses **lexical scope**. A function's scope is determined by where the
function is defined, not where it is called.

```javascript
const name = 'global';

function outer() {
  const name = 'outer';

  function inner() {
    console.log(name); // "outer", resolved by lexical scope
  }

  return inner;
}

const fn = outer();
fn();
```

A **closure** is a function that keeps access to variables from its lexical
environment even after the outer function has returned.

```javascript
function createCounter() {
  let count = 0;

  return {
    increment() {
      return ++count;
    },
    getCount() {
      return count;
    },
  };
}

const counter = createCounter();
counter.increment(); // 1
counter.getCount(); // 1
```

### Mechanism

- Function creation stores a reference to the surrounding lexical environment.
- The referenced variables cannot be garbage-collected while the closure is reachable.
- Closures are useful for encapsulation, currying, memoization, and event handlers.

### Common Pitfalls

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3
}

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
```

`var` is function-scoped, so all callbacks share the same binding. `let` creates
a new block-scoped binding for each iteration.

### Interview Answer

"A closure is not a special object. It is the normal result of lexical scoping
and first-class functions. The important production risk is retaining large
objects through a long-lived closure, for example in global event listeners or
caches. I would verify this with heap snapshots and allocation timelines."

## Prototypes and Inheritance

### Core Concepts

JavaScript objects delegate property lookup through a prototype chain. When a
property is not found on the object itself, the engine checks its prototype,
then the prototype's prototype, until `null`.

```javascript
function User(name) {
  this.name = name;
}

User.prototype.sayHi = function sayHi() {
  return `Hi, ${this.name}`;
};

const user = new User('Ada');
user.sayHi();
```

`class` syntax is primarily syntactic sugar over prototypes, but it also has
stricter semantics such as non-callable class constructors and lexical `super`.

### Trade-offs

- Prototype methods are memory-efficient because instances share functions.
- Class syntax is easier to read, but it can hide prototype delegation details.
- Composition is often easier to test and evolve than deep inheritance.

## Event Loop

### Browser Event Loop

The browser event loop coordinates macro tasks, microtasks, rendering, and user
interaction.

```javascript
console.log('A');

setTimeout(() => console.log('B'), 0);

Promise.resolve().then(() => console.log('C'));

console.log('D');

// A, D, C, B
```

Typical order:

1. Run the current synchronous call stack.
2. Drain the microtask queue, including Promise callbacks and `queueMicrotask`.
3. Potentially render a frame.
4. Run the next macro task, such as timers, I/O, or user events.

### Browser vs Node.js

Node.js has libuv phases: timers, pending callbacks, poll, check, and close
callbacks. `process.nextTick` has higher priority than Promise microtasks in
Node.js and should not be abused because it can starve the event loop.

### Production Practice

- Split long tasks above 50 ms.
- Avoid heavy synchronous JSON parsing on the main thread.
- Use Web Workers for CPU-heavy work in the browser.
- Use Worker Threads or separate services for CPU-heavy work in Node.js.

## Asynchronous Programming

### Promise

Promises model a value that may become available later. They support chaining
and error propagation.

```javascript
fetch('/api/user')
  .then((res) => {
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  })
  .then(renderUser)
  .catch(showError);
```

### async / await

`async` / `await` is syntax over Promise chaining. It improves readability but
does not make asynchronous code synchronous.

```javascript
async function loadUser(signal) {
  const res = await fetch('/api/user', { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

### Promise Combinators

- `Promise.all`: fail fast; use when all tasks are required.
- `Promise.allSettled`: collect all results; use for best-effort tasks.
- `Promise.race`: resolve or reject with the first settled promise.
- `Promise.any`: resolve with the first fulfilled promise.

### Production Practice

- Use `AbortController` for request cancellation.
- Add timeouts around external calls.
- Keep error boundaries clear between user-facing errors and developer errors.
- Avoid unbounded concurrency; use a queue or pool for bulk tasks.

## Memory Management

### Garbage Collection

Modern JavaScript engines use reachability-based garbage collection. Objects
reachable from roots such as global variables, the call stack, closures, DOM
references, or active timers are retained.

### Common Leak Sources

- Detached DOM nodes still referenced by JavaScript.
- Event listeners not removed from long-lived targets.
- Unbounded Maps used as caches.
- Timers or intervals that outlive a component.
- Large objects captured by closures.

```javascript
function mount() {
  const controller = new AbortController();

  window.addEventListener(
    'resize',
    () => {
      console.log('resize');
    },
    { signal: controller.signal },
  );

  return () => controller.abort();
}
```

### Interview Answer

"A memory leak in JavaScript means an object remains reachable when the business
logic no longer needs it. I would first reproduce it, take heap snapshots,
compare retained objects, inspect retaining paths, and then fix the lifecycle or
cache policy."

## ES6+ Core Features

### `var`, `let`, and `const`

- `var` is function-scoped and hoisted with `undefined`.
- `let` and `const` are block-scoped and have a temporal dead zone.
- `const` prevents reassignment, not mutation of object properties.

### `this` Binding

Priority from high to low:

1. `new` binding.
2. Explicit binding with `call`, `apply`, or `bind`.
3. Implicit binding through object method calls.
4. Default binding.

Arrow functions do not have their own `this`; they capture it lexically.

### Modules

ES modules are statically analyzable and support tree shaking. CommonJS is
dynamic and widely used in Node.js. Mixing them requires attention to default
exports, named exports, and package `type` configuration.

## Type System

### `any` vs `unknown`

`any` disables type checking. `unknown` represents an unknown value that must be
narrowed before use.

```typescript
function parse(input: string): unknown {
  return JSON.parse(input);
}

const value = parse('{"name":"Ada"}');

if (
  typeof value === 'object' &&
  value !== null &&
  'name' in value
) {
  console.log((value as { name: string }).name);
}
```

### `interface` vs `type`

- Prefer `interface` for public object shapes that may be extended.
- Prefer `type` for unions, intersections, mapped types, and function aliases.
- Be consistent with the team's style guide.

### Type Narrowing

TypeScript narrows types through control flow analysis, `typeof`, `instanceof`,
discriminated unions, custom type guards, and equality checks.

```typescript
type Result =
  | { status: 'success'; data: string }
  | { status: 'error'; error: Error };

function handle(result: Result) {
  if (result.status === 'success') {
    return result.data;
  }

  throw result.error;
}
```

## Advanced Types

### Generics

Generics make APIs reusable while preserving type information.

```typescript
function pick<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: 'Ada' };
const name = pick(user, 'name'); // string
```

### Utility Types

- `Partial<T>`: all properties optional.
- `Required<T>`: all properties required.
- `Pick<T, K>`: select keys.
- `Omit<T, K>`: remove keys.
- `Record<K, T>`: object map from keys to values.

### Production Practice

- Keep domain models explicit instead of overusing clever mapped types.
- Use runtime validation for untrusted external data.
- Avoid exporting broad `any` types from shared packages.
- Treat the type system as a design tool, not only a linting tool.

## Engineering Practice

### API Boundary

TypeScript types are erased at runtime. For network responses, form input,
localStorage, and third-party integrations, combine static typing with runtime
validation.

### Error Handling

Prefer explicit result types or thrown errors with clear ownership. Do not catch
errors only to log and continue silently.

### Maintainability

- Use strict mode where possible.
- Prefer small composable functions.
- Avoid global mutation.
- Use exhaustive checks for discriminated unions.

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}
```

## Interview Self-check

1. What is a closure, and when can it cause a memory leak?
2. Explain the execution order of synchronous code, Promise callbacks, and timers.
3. How does prototype lookup work?
4. What is the difference between `this` in a normal function and an arrow function?
5. How do `Promise.all` and `Promise.allSettled` differ?
6. Why should `unknown` be preferred over `any` for untrusted values?
7. How would you model API states with discriminated unions?
8. What are common JavaScript memory leak patterns?
9. How do ES modules differ from CommonJS?
10. When would you avoid advanced TypeScript types?
11. How do `var`, `let`, and `const` differ beyond syntax?
12. Why do many teams ban loose equality in shared codebases?
13. How do browser and Node.js event loops differ in production debugging?
14. How would you implement request timeout, cancellation, and retry together?
15. How do you design a type-safe SDK when backend responses are not fully trusted?
16. When should you use `Map`, `WeakMap`, or plain objects?
17. How do you diagnose a suspected retained closure in Chrome DevTools?
18. What TypeScript patterns improve large-team maintainability, and which ones hurt it?
19. How do you handle partial failure in batch asynchronous workflows?
20. How would you explain the trade-off between runtime validation and static typing?

## Production Scenarios

### Scenario 1: A Page Becomes Slower Over Time

Check for accumulating listeners, timers, detached DOM nodes, growing caches, and
large closures. Use Performance and Memory panels to compare heap snapshots
before and after repeated navigation.

### Scenario 2: A Type-safe API Client

Define request and response types, validate runtime data at the boundary, return
typed results, and keep transport errors separate from domain errors.

### Scenario 3: Large Batch Requests

Use a concurrency limit instead of firing thousands of Promises at once. Add
timeouts, cancellation, retry policy, and partial failure handling.

### Scenario 4: Untrusted API Data Breaks a Release

Do not rely on TypeScript alone because types disappear at runtime. Validate data
at the API boundary, log schema mismatch with release and endpoint metadata,
return safe fallback states, and coordinate with backend owners on contract tests.

### Scenario 5: Main Thread Jank During Data Processing

Capture a performance trace first. If long tasks come from parsing, sorting, or
formatting, split work into chunks, move CPU-heavy logic to a Web Worker, and
measure INP or custom interaction latency before and after the change.

## Summary

For interviews, JavaScript and TypeScript should be explained as a runtime plus
a type layer: lexical scope, prototype delegation, event loop scheduling,
garbage collection, and static modeling. Strong answers connect these concepts
to production risks such as memory leaks, blocked main threads, API uncertainty,
and maintainability.
