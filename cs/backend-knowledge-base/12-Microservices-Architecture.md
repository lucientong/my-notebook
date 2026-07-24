# Microservices Architecture

Language: English | [中文](../后端知识库/12-微服务架构.md)

---

## Table of Contents

1. [Microservice Basics](#1-microservice-basics)
2. [Service Discovery](#2-service-discovery)
3. [API Gateway](#3-api-gateway)
4. [Circuit Breaking and Degradation](#4-circuit-breaking-and-degradation)
5. [Distributed Tracing](#5-distributed-tracing)
6. [Configuration Center](#6-configuration-center)
7. [Distributed Transactions](#7-distributed-transactions)
8. [Service Communication](#8-service-communication)
9. [Rate Limiting](#9-rate-limiting)
10. [Microservice Testing](#10-microservice-testing)
11. [Service Governance Practice](#11-service-governance-practice)
12. [Interview Self-Check](#12-interview-self-check)

---

## 1. Microservice Basics

### 1.1 Monolith vs Microservices

Monolith:

- simple deployment.
- local calls.
- easier transactions.
- can become hard to change at scale.

Microservices:

- independent ownership and deployment.
- independent scaling.
- clearer bounded contexts.
- distributed failure and consistency complexity.

Do not use microservices just because the system is large. Use them when organization, deployment, and ownership boundaries justify the cost.

### 1.2 Decomposition Principles

Good boundaries:

- business capability.
- data ownership.
- team ownership.
- change frequency.
- failure isolation.

Avoid splitting by technical layers such as controller/service/DAO.

---

## 2. Service Discovery

Service discovery maps logical service names to available instances.

Models:

- client-side discovery.
- server-side discovery.
- DNS-based discovery.
- service mesh discovery.

Consul, etcd, and Nacos are common registry/configuration systems.

Important concerns:

- health checking.
- TTL/lease.
- stale instances.
- multi-zone awareness.
- graceful shutdown.

---

## 3. API Gateway

Gateway responsibilities:

- routing.
- authentication.
- rate limiting.
- protocol translation.
- request signing.
- observability.
- canary routing.

Gateway should not contain complex business logic, otherwise it becomes a distributed monolith.

---

## 4. Circuit Breaking and Degradation

Circuit breaker prevents cascading failure by failing fast when downstream is unhealthy.

Degradation keeps critical user flows alive by disabling non-critical functions.

Examples:

- return cached profile.
- hide recommendations.
- use fallback config.
- skip analytics write.

---

## 5. Distributed Tracing

Distributed tracing connects requests across services.

Core concepts:

- trace.
- span.
- trace ID.
- parent-child span relationship.
- baggage/context propagation.

OpenTelemetry provides a standard model for traces, metrics, and logs.

Tracing helps answer:

- where latency is spent.
- which dependency failed.
- whether errors correlate with deployment or traffic.

---

## 6. Configuration Center

Configuration center manages dynamic configuration.

Requirements:

- versioning.
- audit.
- rollout.
- rollback.
- permission control.
- client caching.
- safe defaults.

Do not use dynamic config as a hidden deployment system without governance.

---

## 7. Distributed Transactions

Common patterns:

- Saga.
- TCC.
- local message table/outbox.
- best-effort notification.

For many backend systems, outbox/event-driven consistency is more practical than strong distributed transactions.

Design requirements:

- idempotent consumers.
- retry.
- DLQ.
- reconciliation.
- explicit state machine.

---

## 8. Service Communication

### 8.1 Sync vs Async

Synchronous RPC:

- simple request-response model.
- easier immediate result.
- tighter coupling and failure propagation.

Asynchronous messaging:

- decoupling.
- buffering.
- eventual consistency.
- harder debugging and ordering.

### 8.2 gRPC vs REST

| Dimension | REST | gRPC |
|-----------|------|------|
| Protocol | HTTP/JSON usually | HTTP/2 + Protobuf |
| Human readability | high | lower |
| Performance | good | usually better |
| Streaming | limited | strong |
| Browser support | native | needs gateway for many cases |

---

## 9. Rate Limiting

Algorithms:

- fixed window.
- sliding window.
- token bucket.
- leaky bucket.

Rate limiting in microservices can be applied at:

- gateway.
- service.
- dependency client.
- tenant quota layer.

---

## 10. Microservice Testing

Testing pyramid:

- unit tests.
- integration tests.
- contract tests.
- end-to-end tests.

Contract testing prevents provider/consumer API drift without relying only on expensive E2E tests.

Chaos engineering validates resilience by injecting controlled failures.

---

## 11. Service Governance Practice

### 11.1 Canary Release

Canary rollout gradually sends traffic to a new version.

Watch:

- error rate.
- latency.
- business metrics.
- resource usage.
- logs/traces.

Rollback must be faster than rollout.

### 11.2 Full-Link Load Testing

Full-link load testing validates the whole production-like path.

Must protect real users and data:

- traffic isolation.
- test data tagging.
- capacity guardrails.
- kill switch.

### 11.3 Service Mesh

Service mesh moves communication concerns into sidecars or data plane proxies.

Capabilities:

- traffic management.
- mTLS.
- retries/timeouts.
- observability.
- policy enforcement.

Trade-off: operational complexity and proxy overhead.

---

## 12. Interview Self-Check

### Q1: When should you choose microservices?

**Answer:** When independent ownership, deployment, scaling, and failure isolation outweigh distributed complexity.

### Q2: How do you split services?

**Answer:** Split by bounded context, business capability, data ownership, and team ownership, not by technical layers.

### Q3: What is service discovery?

**Answer:** It lets clients or gateways find healthy service instances dynamically through registry, DNS, or service mesh.

### Q4: What should an API gateway do?

**Answer:** Routing, auth, rate limiting, protocol translation, observability, and traffic control. It should avoid deep business logic.

### Q5: Why is distributed tracing important?

**Answer:** It reveals latency and failures across service boundaries, which logs alone often cannot reconstruct.

### Q6: How do you handle distributed transactions?

**Answer:** Prefer local transactions plus reliable events/outbox when eventual consistency is acceptable. Use TCC or Saga for business workflows that require compensation.

### Q7: How do you prevent cascading failure?

**Answer:** Timeouts, retries with budgets, circuit breakers, rate limits, bulkheads, fallback, and backpressure.

### Q8: What is contract testing?

**Answer:** It verifies that service providers and consumers agree on API contracts, reducing integration failures.

### Q9: What is canary release?

**Answer:** A staged rollout that sends a small percentage of traffic to a new version and expands only if metrics remain healthy.

### Q10: What is the trade-off of service mesh?

**Answer:** It standardizes traffic governance and security, but adds operational complexity, latency overhead, and debugging layers.

### Senior Interview Follow-Ups

### Q11: How do you decide whether to split one service into two?

**Answer:** Look for independent business ownership, different change cadence, separate scaling needs, clear data ownership, and failure isolation value. Do not split only because a codebase is large. If the split creates cross-service transactions, chatty RPC, duplicated models, or unclear ownership, a modular monolith may be better until boundaries stabilize.

### Q12: What is your incident SOP for cascading failure in microservices?

**Answer:** First identify the dependency causing saturation with traces, error budgets, queue length, and client retry metrics. Stop amplification by disabling aggressive retries, opening circuit breakers, shedding low-priority traffic, and rolling back the triggering deploy if correlated. Then protect core flows with degradation and bulkheads. After recovery, tune timeouts, retry budgets, concurrency limits, and dashboards so the same failure becomes visible earlier.

### Q13: How do you keep service contracts from drifting?

**Answer:** Use schema-first APIs where possible, consumer-driven contract tests, versioned protobuf/OpenAPI definitions, backward-compatible field changes, and CI checks that run against both provider and consumer expectations. Runtime observability should include unknown field usage, deprecated endpoint traffic, and error-code distribution. Governance should make breaking changes explicit instead of discovering them in E2E tests or production.
