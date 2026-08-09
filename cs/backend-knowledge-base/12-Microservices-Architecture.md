# Microservices Architecture

Language: English | [中文](../后端知识库/12-微服务架构.md)

> **Scope**: how to split, connect, stay alive, stay consistent, and stay observable.
> Rate-limit / circuit-breaker *algorithms* → [09 System Design](./09-System-Design-and-Architecture.md).
> Istio / Envoy detail → [11 Cloud Native](./11-Cloud-Native-and-Containers.md). This doc keeps selection and microservice usage.

---

## Table of Contents

1. [Split and Migration](#1-split-and-migration)
2. [Service Communication](#2-service-communication)
3. [Discovery](#3-discovery)
4. [Configuration](#4-configuration)
5. [Gateway, BFF, Mesh Boundaries](#5-gateway-bff-mesh-boundaries)
6. [Resilience Quartet](#6-resilience-quartet)
7. [Distributed Transactions](#7-distributed-transactions)
8. [Tracing and Observability](#8-tracing-and-observability)
9. [Release Governance and Mesh](#9-release-governance-and-mesh)
10. [Testing](#10-testing)
11. [Interview Self-Check](#11-interview-self-check)

---

## 1. Split and Migration

Monolith = one building, shared utilities. Microservices = separate meters per building: independent scale and failure domains, but you must build roads (network), reconciliation (consistency), and a property office (platform).

**Topology (top-down):** external traffic → API gateway → services (each with its own DB). Do not draw services above the gateway.

Split by **business capability**, not by technical layer (`UserController` service is an anti-pattern). Hard criteria: single responsibility, high cohesion, low coupling (no cross-service joins into another team's tables).

### 1.1 DDD, Data Ownership, Migration

- **Bounded context**: same term means the same thing inside one boundary.
- **Data ownership**: one write owner per aggregate; others read via API/events/replicas.
- **Strangler**: put a façade in front, peel hot paths first, cut over gradually.
- **Anti-corruption layer**: translate legacy models at the boundary.
- **Order**: platform gates → extract service → **then** split the database → reconcile before weakening sync transactions.

---

## 2. Service Communication

| Style | When |
|-------|------|
| Sync RPC (HTTP/gRPC) | Need immediate answer; keep hop budget |
| Async messaging | Decouple, absorb spikes, eventual consistency |

**REST ≠ HTTP/1.1.** REST is an architectural style; it often rides HTTP/1.1 or HTTP/2. gRPC defaults to HTTP/2 + Protobuf. Avoid unverifiable claims like “gRPC is always 10× faster.”

**Distributed IDs**: UUID/ULID (no center), segment allocators, Snowflake-style. Trade trend-ordering vs central allocator dependency.

---

## 3. Discovery

Problem: instances come and go; clients need healthy targets.

| Approach | Notes |
|----------|-------|
| K8s Service DNS + readiness | Enough for many in-cluster east-west cases |
| Registry (Nacos/Eureka/Consul…) | Heterogeneous runtimes, rich metadata routing |
| Mesh control plane | Platform-owned discovery + traffic policy |

Label CAP carefully: e.g. Consul registration on Raft is closer to **CP** for the catalog; do not paste a one-cell “AP/CP” slogan without saying *which subsystem*.

**Graceful drain**: deregister / readiness=false → finish in-flight → stop process. Startup is the reverse plus warm-up.

---

## 4. Configuration

Central config beats hard-coded files for multi-instance consistency, audit, and rollback. Hot refresh must rebuild half-initialized resources (pools, clients), not only swap a string in memory.

---

## 5. Gateway, BFF, Mesh Boundaries

| Layer | Job |
|-------|-----|
| **Gateway** | North-south entry: route, authn/z, edge limit, protocol adapt, canary entry |
| **BFF** | Per-client aggregation when mobile/web need different shapes/rhythms |
| **Mesh** | East-west policy: mTLS, retries, outlier detection, traffic split |

Do not turn the gateway into a business god-object; put domain aggregation in BFF/app services.

---

## 6. Resilience Quartet

Algorithm detail lives in doc 09. Here: **how to chain them**.

1. **Timeout** — hard wait budget so threads/connections are not pinned forever.
2. **Rate limit** — shed load at gateway / inbound / outbound.
3. **Circuit break** — fail fast on a sick dependency; stop the cascade.
4. **Retry** — last, only if idempotent, capped, with backoff.

Plus **bulkheads** (separate pools/semaphores per dependency) and **dependency tiers** (strong vs weak/degradable).

### 6.1 Circuit breaker mainline (not Hystrix)

States still: Closed / Open / Half-Open.

| Stack | Choice |
|-------|--------|
| Java | **Resilience4j** or **Spring Cloud CircuitBreaker** |
| Traffic rules / console | Sentinel |
| Multi-language platform | Envoy/Istio outlier detection |
| Legacy | Hystrix — **historical only**, Netflix stopped maintaining it |

Sentinel vs Resilience4j: dynamic rules / hotspot limiting / ops console → Sentinel; library-level breaker/retry/bulkhead in Spring → Resilience4j/SCCB. Isolation strategy ≠ breaker granularity — do not confuse the two in comparison tables.

---

## 7. Distributed Transactions

Default engineering path: **local transaction + reliable messaging (Outbox) / Saga / TCC (funds)**. Prefer business-acceptable eventual consistency over XA/2PC.

**Why not 2PC as default:** blocking, coordinator risk, fights real-world timeouts and partitions.

### 7.1 Outbox / Inbox (correct local message table)

Producer does **not** wait for consumer ACK to mark “consumed.”

```text
[Producer Outbox]
BEGIN
  write business row
  insert outbox(pending)
COMMIT
→ forwarder publishes to MQ → mark sent

[Consumer Inbox]
dedupe by unique key / inbox table
→ apply business change (idempotent state machine)
```

| Pattern | Idea |
|---------|------|
| Outbox | Same DB tx as business write; async publish |
| Inbox | Consumer-side dedupe storage |
| Saga | Committed steps + compensations; readers may see **business intermediate state** (not DB dirty read) |
| TCC | Try / Confirm / Cancel — stronger reservation, higher intrusion |
| Seata AT/TCC/… | Pick only when Outbox is not enough; XA last |

**Idempotency triad:** unique key + dedupe store + re-entrant state machine.

---

## 8. Tracing and Observability

W3C `traceparent`:

```text
00-<32-hex-trace-id>-<16-hex-parent-id>-<flags>
```

Propagate at each hop; export via **OTLP → OpenTelemetry Collector** → Jaeger/Tempo/…
Jaeger Agent mode is legacy for new designs — standardize on Collector.

Metrics: latency / error / saturation per dependency; logs carry trace id; events carry business keys for replay.

---

## 9. Release Governance and Mesh

Canary vs blue/green: small progressive risk → canary; instant cutover with spare capacity → blue/green.
Traffic dyeing: propagate headers/metadata; orthogonal tags for canary vs stress tests; shadow DB/table/topic for load tests so production data is not polluted.

Mesh is not automatically better than SDK: multi-language governance and platform maturity vs sidecar cost. Details in doc 11.

---

## 10. Testing

Unit → contract (Pact-style) → integration → e2e sparingly.
Contract tests matter more once services only collaborate through APIs. Chaos: steady state → hypothesis → inject → verify breaker/degrade → fix → repeat.

---

## 11. Interview Self-Check

1. **Monolith vs microservices?** Deploy unit, data boundary, and team topology change—not just folders. *Follow-up:* What is a distributed monolith?
2. **Why not shared DB?** Hidden schema/transaction coupling kills boundaries.
3. **Registry vs K8s DNS?** DNS+readiness often enough in-cluster; registry/Mesh for heterogeneous or metadata routing.
4. **Gateway duties without becoming a god?** Route/auth/limit/protocol/canary; aggregation → BFF.
5. **Resilience order?** Timeout → limit → break → retry (+ bulkhead/tiers).
6. **Modern breaker stack?** Resilience4j/SCCB or Sentinel/Mesh; Hystrix history only.
7. **REST vs gRPC? REST = HTTP/1.1?** No. Style vs protocol; no fake “10×” claims.
8. **`traceparent` + export path?** W3C hex format; OTLP → Collector.
9. **Outbox flow?** pending→sent on producer; Inbox idempotency on consumer; no wait-for-consumer-ack on producer.
10. **Saga dirty read?** Misnomer—business intermediate state, not uncommitted DB dirty read.
11. **Why avoid 2PC?** Blocking/coordinator/partition reality; prefer Outbox.
12. **Idempotency triad / distributed IDs?** Unique key + store + state machine; UUID vs segment vs Snowflake.
13. **Graceful shutdown?** Drain traffic first, then stop.
14. **Stress dyeing without polluting prod?** Orthogonal tags + shadow stores + sandbox third parties.

### Design prompts

- **D1** Strangle a monolith: context → façade → service-before-DB → Outbox over blind 2PC.
- **D2** Five sync hops under 100ms: parallelize, cut hops, cache, budget timeouts, degrade weak deps.
- **D3** Order–inventory eventual consistency with Outbox/Inbox and reconciliation.
- **D4** Still need a registry on Kubernetes? When DNS/readiness is enough vs when not.
- **D5** Coexist canary tags and stress-test tags without metric pollution.

---

## Summary

1. **Boundary** first (DDD + data ownership); Strangler / ACL; split DB last.
2. **Communication** with budgets; REST ≠ HTTP/1.1; ID schemes by ordering needs.
3. **Platform triangle**: discovery (incl. vs K8s DNS) / config / gateway (vs BFF/Mesh).
4. **Resilience quartet** with Resilience4j/SCCB as Java mainline; Hystrix historical.
5. **Consistency**: Outbox/Inbox correctly split; Saga intermediate ≠ dirty read; idempotency triad.
6. **Observe & release**: W3C `traceparent` + OTLP/Collector; dyeing; Mesh detail in 11, algorithms in 09.
