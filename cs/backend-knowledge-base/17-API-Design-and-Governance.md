# API Design and Governance

Language: English | [中文](../后端知识库/17-API设计与治理.md)

---

## Table of Contents

1. [RESTful API Design](#1-restful-api-design)
2. [gRPC In Depth](#2-grpc-in-depth)
3. [GraphQL](#3-graphql)
4. [API Idempotency](#4-api-idempotency)
5. [API Versioning](#5-api-versioning)
6. [API Gateway](#6-api-gateway)
7. [Documentation and Testing](#7-documentation-and-testing)
8. [API Performance](#8-api-performance)
9. [API Security](#9-api-security)
10. [Open Platform Design](#10-open-platform-design)
11. [Interview Self-Check](#11-interview-self-check)

---

## 1. RESTful API Design

### 1.1 REST Constraints

REST emphasizes resources, representations, statelessness, cacheability, uniform interface, and layered systems.

Good API design starts from resource modeling, not endpoint verbs.

### 1.2 URL Design

Good:

```text
GET /users/{userId}/orders
POST /orders
PATCH /orders/{orderId}
```

Avoid:

```text
POST /createOrder
GET /getUserInfo
```

### 1.3 HTTP Method Semantics

| Method | Semantics |
|--------|-----------|
| GET | read, safe, idempotent |
| POST | create or trigger processing |
| PUT | replace, idempotent |
| PATCH | partial update |
| DELETE | delete, idempotent by intent |

### 1.4 Status Codes

Use status codes consistently:

- 200 OK.
- 201 Created.
- 204 No Content.
- 400 Bad Request.
- 401 Unauthorized.
- 403 Forbidden.
- 404 Not Found.
- 409 Conflict.
- 429 Too Many Requests.
- 500 Internal Server Error.

---

## 2. gRPC In Depth

### 2.1 Core Concepts

gRPC uses HTTP/2 and Protobuf.

Strengths:

- efficient binary serialization.
- strong schema.
- streaming.
- code generation.
- good internal service-to-service communication.

### 2.2 Protobuf

Protobuf is compact and schema-driven.

Compatibility rules:

- Do not reuse field numbers.
- Add optional fields for compatibility.
- Reserve removed fields.
- Keep field types stable.

### 2.3 Communication Modes

- unary.
- server streaming.
- client streaming.
- bidirectional streaming.

### 2.4 Interceptors

Interceptors are middleware for:

- auth.
- logging.
- metrics.
- tracing.
- retry.
- rate limiting.

### 2.5 Error Handling

Use gRPC status codes and structured error details. Do not encode every error as OK with business code unless there is a strong compatibility reason.

---

## 3. GraphQL

GraphQL lets clients ask for exactly the data they need.

Strengths:

- flexible query shape.
- fewer over-fetching/under-fetching problems.
- strong schema.

Risks:

- N+1 query problem.
- complex query cost.
- caching complexity.
- authorization per field.

Use DataLoader or batching to solve N+1.

---

## 4. API Idempotency

Idempotency means repeated requests have the same effect as one request.

Common strategies:

- idempotency key.
- unique constraint.
- deduplication table.
- request token.
- state machine.

Example:

```text
POST /payments
Idempotency-Key: abc123
```

Server stores key, request hash, status, and response. Duplicate requests return the existing result.

---

## 5. API Versioning

Strategies:

- URL version: `/v1/orders`.
- header version.
- media type version.

Backward compatibility:

- add fields, do not remove required fields suddenly.
- keep enum compatibility.
- document deprecation.
- provide migration windows.

---

## 6. API Gateway

Gateway functions:

- routing.
- authentication.
- rate limiting.
- request signing.
- protocol translation.
- observability.
- canary routing.

Gateway should enforce cross-cutting policies while keeping business logic in services.

---

## 7. Documentation and Testing

### 7.1 OpenAPI

OpenAPI describes REST APIs in machine-readable format.

Benefits:

- generated docs.
- client SDK generation.
- contract validation.
- mock server.

### 7.2 Contract Testing

Contract tests ensure consumer expectations and provider behavior remain compatible.

### 7.3 API Test Types

- unit tests.
- integration tests.
- contract tests.
- E2E tests.
- performance tests.
- security tests.

---

## 8. API Performance

### 8.1 Batch API

Batch APIs reduce round trips but must handle partial failures and request size limits.

### 8.2 Pagination

Prefer cursor/keyset pagination for large datasets.

Avoid deep offset pagination for high-volume tables.

### 8.3 Caching

Cache layers:

- browser/CDN.
- gateway.
- service local cache.
- Redis.

Use clear invalidation and TTL strategy.

### 8.4 Compression and Connection Pooling

Compression reduces bandwidth but costs CPU. Connection pooling reduces handshake overhead and improves latency.

---

## 9. API Security

Security controls:

- TLS.
- authentication and authorization.
- request signing.
- anti-replay timestamp/nonce.
- rate limiting.
- input validation.
- output encoding.
- data masking.
- audit logs.

Sensitive APIs should be designed with abuse cases in mind.

---

## 10. Open Platform Design

Components:

- developer portal.
- application registration.
- API key and secret management.
- OAuth2 authorization.
- HMAC signing.
- quota and rate limits.
- SDKs.
- webhook callbacks.
- audit and billing.

Operational requirements:

- key rotation.
- sandbox environment.
- backward compatibility.
- SLA and error code documentation.

---

## 11. Interview Self-Check

### Q1: What makes an API RESTful?

**Answer:** It models resources, uses HTTP methods semantically, stays stateless, uses representations, and leverages HTTP status codes and caching where appropriate.

### Q2: PUT vs PATCH?

**Answer:** PUT replaces a resource and is idempotent. PATCH partially updates a resource.

### Q3: gRPC vs REST?

**Answer:** gRPC is schema-first, binary, efficient, and supports streaming, making it good for internal service calls. REST is simpler, human-readable, and browser/tooling friendly.

### Q4: What is API idempotency?

**Answer:** Repeating the same request produces the same business effect. It is usually implemented with idempotency keys, unique constraints, and stored responses.

### Q5: How do you version APIs?

**Answer:** Use URL/header/media type versioning, preserve backward compatibility, add fields safely, document deprecation, and provide migration windows.

### Q6: What is contract testing?

**Answer:** It verifies that providers satisfy consumer expectations, reducing API drift between services.

### Q7: How do you prevent replay attacks?

**Answer:** Use timestamp, nonce, request signature, short validity window, and server-side nonce replay cache.

### Q8: How do you design pagination?

**Answer:** Use cursor/keyset pagination for large or frequently changing datasets. Offset pagination is simple but performs poorly at deep pages.

### Q9: What belongs in an API gateway?

**Answer:** Cross-cutting concerns like routing, auth, rate limiting, logging, tracing, and traffic control. Business logic should stay in services.

### Q10: How do you design public API error responses?

**Answer:** Use stable error codes, human-readable messages, request IDs, documentation links, and avoid leaking sensitive internals.

### Senior Interview Follow-Ups

### Q11: How do you design an API idempotency key for payments or order creation?

**Answer:** Scope the key by tenant/user and operation, store request hash, processing status, response, expiration time, and conflict metadata. A repeated request with the same key and same request body returns the original result; the same key with a different body should be rejected. The implementation needs a unique constraint or atomic insert, clear handling for in-progress requests, and observability for duplicate and conflict rates.

### Q12: How do you evolve an API without breaking clients?

**Answer:** Prefer additive changes: new optional fields, new endpoints, and compatible enum evolution. Avoid silently changing semantics. Use contract tests, deprecation windows, traffic analysis, SDK version tracking, and migration guides. For public APIs, versioning is not enough; you need telemetry to know who still depends on old behavior and a rollback plan if a compatible-looking change breaks clients.

### Q13: What belongs in a gateway versus a service?

**Answer:** Put cross-cutting concerns in the gateway: authentication handoff, coarse authorization, rate limiting, routing, protocol translation, request IDs, logging, and traffic shaping. Keep business rules, fine-grained authorization, domain validation, and transaction decisions inside services. A gateway with deep business logic becomes a bottleneck and a hidden distributed monolith.
