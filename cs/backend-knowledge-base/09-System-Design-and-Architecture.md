# System Design and Architecture

Language: English | [中文](../后端知识库/09-系统设计与架构.md)

---

## Table of Contents

1. [Rate Limiting](#1-rate-limiting)
2. [Circuit Breaking and Degradation](#2-circuit-breaking-and-degradation)
3. [Distributed Transactions](#3-distributed-transactions)
4. [Distributed Consistency](#4-distributed-consistency)
5. [High Availability Architecture](#5-high-availability-architecture)
6. [Microservices Architecture](#6-microservices-architecture)
7. [System Design Cases](#7-system-design-cases)
8. [Distributed ID and Locking](#8-distributed-id-and-locking)
9. [Interview Self-Check](#9-interview-self-check)

---

## 1. Rate Limiting

### 1.1 Why Rate Limiting Matters

Rate limiting protects systems from overload, abuse, traffic spikes, and cascading failures.

It should answer:

- Who is limited: user, IP, API key, tenant, service?
- What is limited: QPS, concurrency, bytes, cost?
- Where is it applied: client, gateway, service, dependency?
- What happens when exceeded: reject, queue, degrade, or challenge?

### 1.2 Algorithms

| Algorithm | Strength | Weakness |
|-----------|----------|----------|
| Fixed window | simple | boundary burst |
| Sliding window log | accurate | memory cost |
| Sliding window counter | balanced | approximate |
| Token bucket | allows burst | needs token state |
| Leaky bucket | smooth output | may increase latency |

Token bucket is common because it supports controlled bursts.

### 1.3 Distributed Rate Limiting

Common designs:

- Redis atomic counter or Lua script.
- Local limiter + central quota.
- Gateway-level limiting.
- Sharded quota by key.

Trade-off:

- Strong global accuracy costs latency and availability.
- Local limiting is faster but approximate.

---

## 2. Circuit Breaking and Degradation

### 2.1 Circuit Breaker

States:

```text
Closed -> Open -> Half-Open -> Closed
```

When downstream failures exceed threshold, breaker opens and fails fast. Half-open probes recovery.

Metrics:

- error rate.
- timeout rate.
- slow call rate.
- request volume.

### 2.2 Degradation

Degradation keeps critical paths alive by reducing non-essential functionality.

Examples:

- return cached data.
- hide recommendation module.
- disable expensive ranking.
- use default config.

Good degradation is product-aware, not just engineering-level failure handling.

---

## 3. Distributed Transactions

### 3.1 2PC

Two-phase commit:

```text
prepare -> commit/rollback
```

It provides stronger consistency but has blocking and coordinator failure problems.

### 3.2 TCC

Try-Confirm-Cancel:

- Try reserves resources.
- Confirm commits.
- Cancel releases reservation.

Useful for business-level resource reservation, but implementation cost is high.

### 3.3 Saga

Saga splits a long transaction into local transactions with compensating actions.

Good for long-running workflows where eventual consistency is acceptable.

### 3.4 Local Message Table

Pattern:

```text
local DB transaction writes business row + outbox message
-> async relay publishes message
-> consumer idempotently handles event
```

This is often practical for backend systems because it avoids distributed XA while preserving reliable event publication.

---

## 4. Distributed Consistency

### 4.1 CAP

CAP says when a network partition happens, a distributed system must choose between consistency and availability.

Interview nuance:

- Partition tolerance is not optional in distributed systems.
- CAP is about behavior under partition, not normal operation.

### 4.2 BASE

BASE:

- Basically Available.
- Soft state.
- Eventually consistent.

It is a pragmatic model for large-scale distributed systems.

### 4.3 Consistent Hashing

Consistent hashing reduces remapping when nodes join or leave.

Virtual nodes improve load balance.

### 4.4 Raft ⭐⭐⭐

Raft roles:

- leader.
- follower.
- candidate.

Core ideas:

- leader election.
- log replication.
- majority quorum.
- term numbers.
- committed entries.

Raft is easier to explain than Paxos and is used by systems such as etcd.

---

## 5. High Availability Architecture

### 5.1 Failure Domain Isolation

Isolate failures by:

- process.
- host.
- availability zone.
- region.
- dependency.
- tenant.

Avoid shared bottlenecks and single points of failure.

### 5.2 Timeout and Retry

Bad retries can amplify failures.

Guidelines:

- set timeouts at every network boundary.
- retry only idempotent operations or with idempotency key.
- use exponential backoff and jitter.
- cap retry count and budget.
- avoid retry storms.

### 5.3 Backpressure

Backpressure tells upstream to slow down or fail fast when downstream is saturated.

Mechanisms:

- bounded queues.
- concurrency limits.
- rate limiting.
- load shedding.
- circuit breakers.

---

## 6. Microservices Architecture

Microservices should be justified by team boundaries, deployment independence, scalability, and ownership.

They introduce:

- network latency.
- distributed failure.
- data consistency complexity.
- observability requirements.
- deployment and governance overhead.

Do not split services only by nouns. Split by business capability and ownership boundaries.

---

## 7. System Design Cases

### 7.1 Short URL Service

Core design:

- Generate unique short code.
- Store mapping in database.
- Cache hot mappings in Redis.
- Redirect with 301/302 depending on product requirement.
- Track analytics asynchronously.

Key topics:

- ID generation.
- collision handling.
- cache strategy.
- abuse prevention.

### 7.2 Flash Sale System

Core design:

- CDN/static page.
- rate limit and queue traffic.
- pre-deduct inventory in Redis.
- async order creation.
- idempotency key.
- database unique constraints.
- eventual reconciliation.

The key is protecting inventory correctness and downstream capacity.

### 7.3 Feed System

Common models:

- push: write fanout.
- pull: read fanout.
- hybrid: push for normal users, pull for celebrities.

Trade-off:

- write amplification vs read latency.

### 7.4 Distributed Rate Limiter

Design:

- local limiter for fast path.
- Redis or centralized service for global quota.
- token bucket algorithm.
- fallback policy when central store is unavailable.
- metrics for allowed/rejected/degraded requests.

---

## 8. Distributed ID and Locking

### 8.1 Distributed ID

Options:

- UUID.
- database auto-increment with segment allocation.
- Snowflake-like ID.
- Redis increment.

Snowflake-like ID usually contains timestamp, worker ID, and sequence. Watch clock rollback.

### 8.2 Distributed Lock

Distributed locks are coordination tools, not correctness boundaries by themselves.

Use:

- unique constraints.
- state machines.
- fencing tokens.
- idempotency.

Redlock has controversy because timing assumptions and network partitions are hard. For critical correctness, prefer consensus systems or database constraints.

---

## 9. Interview Self-Check

### Q1: Token bucket vs leaky bucket?

**Answer:** Token bucket allows bursts up to bucket capacity and controls average rate. Leaky bucket smooths output at a fixed rate but can increase waiting.

### Q2: What is circuit breaker?

**Answer:** It fails fast when downstream is unhealthy, preventing resource exhaustion and cascading failures. It usually has closed, open, and half-open states.

### Q3: How do you design retries safely?

**Answer:** Use timeout, bounded retry count, exponential backoff, jitter, retry budget, and idempotency guarantees.

### Q4: TCC vs Saga?

**Answer:** TCC reserves and confirms/cancels resources explicitly. Saga chains local transactions with compensation. TCC is stricter but more expensive.

### Q5: What does CAP really mean?

**Answer:** Under network partition, a distributed system must choose consistency or availability. Partition tolerance is unavoidable in distributed systems.

### Q6: How does Raft commit a log entry?

**Answer:** The leader replicates the entry to followers. Once a majority acknowledges it, the entry is committed and can be applied to state machines.

### Q7: How would you design a flash sale?

**Answer:** Use CDN, rate limiting, queueing, inventory pre-deduction, async order processing, idempotency, DB constraints, and reconciliation.

### Q8: How do you avoid retry storms?

**Answer:** Add jitter, retry budgets, circuit breakers, concurrency limits, and fail-fast behavior when downstream is saturated.

### Q9: What is a good system design answer structure?

**Answer:** Clarify requirements, estimate scale, define APIs/data model, propose architecture, discuss bottlenecks, consistency, failure handling, observability, and trade-offs.

### Q10: Why are distributed locks risky?

**Answer:** Network partitions, clock issues, lock expiration, and client pauses can break assumptions. Critical writes need fencing tokens or stronger correctness mechanisms.

### Senior Interview Follow-Ups

### Q11: How do you estimate capacity for a system design problem?

**Answer:** Start from business traffic assumptions: DAU, peak QPS, read/write ratio, payload size, retention, and growth rate. Convert them into storage, bandwidth, cache memory, database write volume, and queue backlog. Then identify the first bottleneck and add headroom for peak traffic, retries, failover, and reprocessing. A good answer states assumptions explicitly and revises them when the interviewer changes constraints.

### Q12: How would you debug a sudden cross-service latency spike?

**Answer:** Check whether the spike is global or isolated by service, region, tenant, or endpoint. Correlate P50/P95/P99 latency with deploys, traffic, dependency errors, saturation, queue length, retry volume, and trace spans. Then mitigate before perfect root cause: reduce retries, open circuit breakers, shed non-critical traffic, roll back risky changes, or scale the saturated dependency. After recovery, preserve traces and write a postmortem with guardrail metrics.

### Q13: When is eventual consistency acceptable?

**Answer:** It is acceptable when user impact is bounded, reconciliation is possible, and the product can explain temporary inconsistency. Examples include search indexes, analytics counters, feeds, and notifications. It is risky for account balances, inventory finalization, entitlement checks, or compliance records unless the design has explicit state machines, idempotency, audit logs, and repair jobs.
