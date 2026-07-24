Language: English | [中文](../前端知识库/03-Node.js后端开发.md)

# Node.js Backend Development

---

## Table of Contents

### Basics and Frameworks

1. [Node.js Fundamentals](#1-nodejs-fundamentals)
2. [Express](#2-express)
3. [Koa](#3-koa)
4. [Middleware Internals](#4-middleware-internals)

### Service Capabilities

5. [Database Integration](#5-database-integration)
6. [Authentication](#6-authentication)
7. [File Upload](#7-file-upload)
8. [WebSocket Realtime Communication](#8-websocket-realtime-communication)

### Engineering Practice and Self-check

9. [Performance Optimization](#9-performance-optimization)
10. [Process Management and Deployment](#10-process-management-and-deployment)
11. [Streams and Buffer](#11-streams-and-buffer)
12. [Interview Self-check](#12-interview-self-check)
13. [Production Scenarios](#13-production-scenarios)

---

## 1. Node.js Fundamentals

Node.js is a JavaScript runtime built on V8 and libuv. It is single-threaded at
the JavaScript execution level, but it uses the operating system and libuv thread
pool for asynchronous I/O.

### Strengths

- High concurrency for I/O-heavy workloads.
- Shared language across frontend and backend.
- Rich npm ecosystem.
- Strong fit for BFF, API gateway, SSR, realtime, and tooling.

### Weaknesses

- CPU-heavy tasks can block the event loop.
- Memory leaks and unhandled exceptions can affect the whole process.
- Dependency governance is important because npm supply-chain risk is real.

### Event Loop

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);
setImmediate(() => console.log('3'));
Promise.resolve().then(() => console.log('4'));

console.log('5');
```

Microtasks run after the current stack. In Node.js, `process.nextTick` has even
higher priority than Promise microtasks, so excessive use can starve I/O.

## 2. Express

Express is a minimalist middleware-based web framework.

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ ok: true });
});

app.post('/api/users', async (req, res, next) => {
  try {
    const user = await createUser(req.body);
    res.status(201).json(user);
  } catch (error) {
    next(error);
  }
});

app.use((error, req, res, next) => {
  res.status(500).json({ message: 'Internal Server Error' });
});
```

Production practice:

- Centralize error handling.
- Validate input at the boundary.
- Add request IDs and structured logs.
- Use rate limiting and body size limits.
- Keep route handlers thin and business logic testable.

## 3. Koa

Koa uses an async middleware composition model, often described as an onion
model.

```javascript
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  ctx.set('X-Response-Time', `${Date.now() - start}ms`);
});
```

Express is callback-oriented and route-centric. Koa is more minimal and composes
async middleware naturally. The choice depends on team familiarity, ecosystem,
and architecture.

## 4. Middleware Internals

Middleware is a chain of functions that can read the request, modify context,
call the next middleware, or terminate the response.

Key concerns:

- Ordering matters.
- Error middleware must be registered after normal middleware.
- Async errors must be propagated.
- Shared mutable request context should be controlled.

Interview phrasing:

"Middleware is a structured way to separate cross-cutting concerns such as
logging, auth, parsing, rate limiting, and error handling. The main risks are
incorrect ordering and swallowed asynchronous errors."

## 5. Database Integration

For database access, use connection pooling, parameterized queries, migrations,
and transaction boundaries.

Production checklist:

- Configure pool size based on database capacity, not only application traffic.
- Set query timeouts.
- Use indexes and inspect slow queries.
- Avoid N+1 query patterns.
- Keep transaction scope small.
- Never concatenate user input into SQL.

```javascript
await db.query('SELECT * FROM users WHERE id = ?', [userId]);
```

## 6. Authentication

Common strategies:

- **Session + Cookie**: good for traditional web apps; server controls session
  invalidation.
- **JWT**: good for stateless APIs; harder to revoke immediately.
- **OAuth/OIDC**: delegated identity and enterprise SSO.

Cookie security:

- `HttpOnly` prevents JavaScript access.
- `Secure` requires HTTPS.
- `SameSite` reduces CSRF risk.

Trade-off answer:

"JWT reduces server-side session storage but shifts complexity to token expiry,
rotation, revocation, and key management. For browser apps, I still prefer
secure cookies over storing tokens in localStorage."

## 7. File Upload

For large files:

- Stream data instead of buffering the entire file.
- Enforce size and type limits.
- Store files in object storage.
- Use resumable or multipart upload for unreliable networks.
- Scan or validate untrusted files.

```javascript
req.pipe(fs.createWriteStream('/tmp/upload.bin'));
```

Avoid keeping large uploads in memory. Backpressure is critical.

## 8. WebSocket Realtime Communication

WebSocket provides full-duplex communication over a long-lived connection.

Production concerns:

- Authentication during connection upgrade.
- Heartbeat and reconnect strategy.
- Backpressure and message rate limits.
- Horizontal scaling with Redis Pub/Sub, message queues, or a gateway layer.
- Sticky sessions or shared connection state.

## 9. Performance Optimization

Main principles:

- Do not block the event loop.
- Use streams for large data.
- Cache expensive reads.
- Limit concurrency for downstream services.
- Move CPU-heavy tasks to Worker Threads, child processes, or separate services.

Monitoring:

- Event loop lag.
- Heap usage and GC pauses.
- Request latency percentiles.
- Error rate and saturation.
- Downstream dependency latency.

## 10. Process Management and Deployment

Node.js applications should handle signals and graceful shutdown.

```javascript
const server = app.listen(3000);

process.on('SIGTERM', () => {
  server.close(() => {
    process.exit(0);
  });

  setTimeout(() => process.exit(1), 10000).unref();
});
```

Production deployment often uses containers, process managers, or orchestrators.
Key practices include health checks, readiness checks, immutable releases,
rollback, and configuration management.

## 11. Streams and Buffer

`Buffer` represents binary data. Streams process data chunk by chunk.

Stream types:

- Readable.
- Writable.
- Duplex.
- Transform.

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const zlib = require('zlib');

await pipeline(
  fs.createReadStream('access.log'),
  zlib.createGzip(),
  fs.createWriteStream('access.log.gz'),
);
```

`pipeline` is preferred because it handles backpressure and error propagation.

## 12. Interview Self-check

1. Why is Node.js suitable for I/O-heavy services?
2. How does Node.js use multiple CPU cores?
3. Explain the Node.js event loop phases.
4. How do Express and Koa middleware differ?
5. JWT vs session: how would you choose?
6. How would you diagnose a memory leak?
7. How do you handle large file uploads?
8. What is backpressure?
9. How do you implement graceful shutdown?
10. How do you avoid blocking the event loop?
11. When should you use Worker Threads?
12. How do you design rollback-safe configuration changes?
13. How do you choose between `process.nextTick`, Promise microtasks, and `setImmediate`?
14. How would you diagnose high p99 latency when average latency looks normal?
15. How do you size a database pool for Node.js services?
16. What are the security risks of storing browser tokens in localStorage?
17. How do you design idempotency for retryable APIs?
18. How do you prevent stream pipeline errors from becoming silent data loss?
19. How would you scale WebSocket connections across multiple instances?
20. What metrics prove that a Node.js service is saturated?

## 13. Production Scenarios

### Scenario 1: Event Loop Lag Spikes

Check CPU-heavy synchronous code, large JSON parsing, regex backtracking,
compression, crypto operations, and uncontrolled concurrency. Measure event loop
lag and capture CPU profiles.

### Scenario 2: Memory Keeps Growing

Compare heap snapshots, inspect retained objects, check caches, timers,
listeners, request-scoped data stored globally, and stream buffers.

### Scenario 3: Realtime Service Scaling

Separate connection gateway from business processing, use a shared pub/sub
layer, design heartbeat and reconnection, and monitor connection count, message
rate, and send queue size.

### Scenario 4: p99 Latency Spikes Under Moderate Traffic

Correlate event loop lag, GC pauses, downstream latency, pool wait time, and CPU
profiles. Average latency can hide saturation, so inspect percentiles by route,
dependency, release, and payload size.

### Scenario 5: Safe Configuration Rollout

Treat config as a versioned dependency. Validate it before activation, roll out
by instance or user segment, keep backward-compatible defaults, and provide a
fast rollback path when error rate or latency crosses a guardrail.

## Summary

Node.js interview strength comes from connecting runtime mechanics to service
engineering. Explain the event loop, middleware, streams, authentication, and
deployment in terms of reliability, observability, and operational trade-offs.
