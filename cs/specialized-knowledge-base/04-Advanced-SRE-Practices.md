# Advanced SRE Practices

Language: English | [中文](../专项知识库/04-SRE高级实践.md)

---

## Table of Contents

### Part 1: Observability Foundations
1. [Three Pillars of Observability](#1-three-pillars-of-observability)
2. [Metrics System with Prometheus](#2-metrics-system-with-prometheus)
3. [Log Management](#3-log-management)
4. [Distributed Tracing](#4-distributed-tracing)
5. [Alerting System](#5-alerting-system)
6. [Incident Troubleshooting SOP](#6-incident-troubleshooting-sop)

### Part 2: Advanced SRE Practices
7. [Disaster Recovery Architecture](#7-disaster-recovery-architecture)
8. [Incident Management and Self-Healing](#8-incident-management-and-self-healing)
9. [Monitoring System Design](#9-monitoring-system-design)
10. [Performance Optimization](#10-performance-optimization)
11. [Cost Optimization](#11-cost-optimization)
12. [Reliability Governance](#12-reliability-governance)
13. [Chaos Engineering](#13-chaos-engineering)
14. [Capacity Planning and FinOps](#14-capacity-planning-and-finops)
15. [Interview Self-Check](#15-interview-self-check)

---

# Part 1: Observability Foundations

## 1. Three Pillars of Observability

| Pillar | Purpose | Typical Tools |
|--------|---------|---------------|
| Metrics | Health and trend signals such as QPS, latency, errors, saturation | Prometheus, VictoriaMetrics, InfluxDB |
| Logs | Detailed event records and business context | ELK, Loki, Cloud Logging |
| Traces | Per-request call graph across services | Jaeger, Zipkin, SkyWalking, OpenTelemetry |

Monitoring answers "is the system healthy?" Observability answers "why is it unhealthy and what changed?" Mature SRE work combines both: stable alerting for known failure modes and exploratory signals for unknown failure modes.

### 1.1 Monitoring vs Observability

| Dimension | Monitoring | Observability |
|-----------|------------|---------------|
| Main use | Detect known bad states | Investigate unknown states |
| Data shape | Predefined dashboards and alerts | Metrics, logs, traces, events, profiles |
| User | On-call and service owners | On-call, developers, incident commanders |
| Failure mode | Alert noise if thresholds are naive | High cost and low value if data is ungoverned |

Senior viewpoint: observability is not collecting everything. It is collecting the smallest useful set of signals that can support fast decisions.

---

## 2. Metrics System with Prometheus

### 2.1 Four Golden Signals

| Signal | Meaning | Example |
|--------|---------|---------|
| Latency | How long requests take | P99 < 200 ms |
| Traffic | How much demand the system receives | QPS, requests per minute |
| Errors | How often requests fail | 5xx ratio, business failure ratio |
| Saturation | How full resources are | CPU, memory, queue depth, connection pools |

Use golden signals for user-facing services. Use resource-oriented signals for infrastructure.

### 2.2 Prometheus Metric Types

| Type | Meaning | Use |
|------|---------|-----|
| Counter | Monotonic increasing value | Requests, errors, retries |
| Gauge | Point-in-time value | Memory, goroutine count, queue depth |
| Histogram | Bucketed distribution | Latency percentiles through `histogram_quantile` |
| Summary | Client-side quantiles | Local percentile, less aggregatable |

Example PromQL:

```promql
# QPS
sum(rate(http_requests_total[1m])) by (service)

# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)

# P99 latency from histogram
histogram_quantile(
  0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

### 2.3 RED and USE

RED is best for request-serving systems:

- Rate
- Errors
- Duration

USE is best for resources:

- Utilization
- Saturation
- Errors

During incidents, connect them:

1. Use RED to prove user impact.
2. Use USE to locate saturated resources.
3. Use logs and traces to explain why the resource became saturated.

Avoid the common mistake of only watching CPU and memory. Many outages happen with normal CPU because connection pools, queues, threads, or downstream dependencies are saturated.

### 2.4 Cardinality Governance

High-cardinality labels can destroy a metrics system.

Bad labels:

- `user_id`
- `request_id`
- raw URL with IDs
- unbounded error message

Better labels:

- `service`
- `method`
- route template such as `/users/:id`
- normalized status or error class

Rule of thumb: metrics labels should support aggregation. Put per-user or per-request detail into logs or traces.

---

## 3. Log Management

### 3.1 Structured Logs

Prefer structured JSON logs with consistent fields:

```json
{
  "level": "error",
  "service": "payment-api",
  "trace_id": "abc-123",
  "user_id": "u-1001",
  "path": "/payments",
  "status": 500,
  "duration_ms": 842,
  "error_code": "BANK_TIMEOUT"
}
```

Important fields:

- `trace_id`
- `request_id`
- `service`
- `version`
- `region` or `az`
- normalized error code
- latency and business status

### 3.2 Log Pipeline

Typical architecture:

```text
Application -> Agent/Filebeat -> Kafka -> Log Processor -> Elasticsearch/Loki -> Query UI
```

Design concerns:

- Backpressure: logging should not take down the application.
- Sampling: high-volume info logs should be sampled.
- Security: mask secrets, tokens, and personally sensitive data.
- Retention: hot data for fast investigation, cold data for compliance and postmortems.

### 3.3 Incident Use

Useful queries:

```text
level:ERROR AND service:payment-api AND @timestamp:[now-15m TO now]
trace_id:"abc-123"
error_code:BANK_TIMEOUT
version:v2026.06.10
```

Senior viewpoint: logs should explain individual events. Metrics explain aggregate impact. Traces explain causality across services.

---

## 4. Distributed Tracing

### 4.1 Core Concepts

| Concept | Meaning |
|---------|---------|
| Trace | A full request journey |
| Span | One operation in the journey |
| Trace ID | Shared correlation ID |
| Span ID | Operation ID |
| Baggage | Cross-service context, used sparingly |

### 4.2 What Tracing Solves

Tracing is especially useful when:

- P99 latency rises but CPU is normal.
- A request crosses many services.
- Dependencies have partial failure.
- You need to find the slowest span and its owner.

### 4.3 OpenTelemetry Practice

Good span attributes:

- `service.name`
- `http.method`
- `http.route`
- `http.status_code`
- `db.system`
- `db.operation`
- `rpc.system`
- normalized business error code

Avoid:

- raw request bodies
- secrets
- high-cardinality user labels on every span

### 4.4 Trace Sampling

Sampling strategies:

- Head sampling: decide at request start; cheap but may miss rare errors.
- Tail sampling: decide after observing outcome; better for errors and high latency.
- Dynamic sampling: keep more traces for anomalous services or high-priority customers.

---

## 5. Alerting System

### 5.1 Good Alert Rules

An alert should be:

- User-impacting or actionably predictive.
- Owned by a team.
- Attached to a runbook.
- Tuned for signal, not dashboard decoration.

Example:

```yaml
- alert: HighErrorBudgetBurn
  expr: |
    (
      sum(rate(http_requests_total{service="payment-api",status=~"5.."}[5m]))
      /
      sum(rate(http_requests_total{service="payment-api"}[5m]))
    ) / 0.0005 > 14.4
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "payment-api is burning error budget too fast"
```

### 5.2 Alert Routing and Noise Control

Core mechanisms:

- Grouping: combine similar alerts.
- Inhibition: suppress upstream symptom alerts when a root dependency is down.
- Deduplication: avoid paging repeatedly for the same incident.
- Escalation: increase urgency when there is no acknowledgement or no mitigation.

Alert fatigue is a reliability problem. If engineers learn to ignore alerts, the alerting system has failed.

### 5.3 Multi-Window Burn Rate

Use multiple windows to catch both fast outages and slow burns:

| Window | Purpose |
|--------|---------|
| 5m + 1h | Large sudden incidents |
| 30m + 6h | Sustained degradation |
| 6h + 3d | Long-term quality decline |

Burn rate formula:

```text
Burn Rate = current error rate / allowed error rate
```

---

## 6. Incident Troubleshooting SOP

### 6.1 Slow API

1. Confirm user impact with RED metrics.
2. Check whether the problem is global or segmented by region, version, tenant, or route.
3. Use traces to locate the slow span.
4. Check saturation: thread pool, connection pool, DB, cache, queue, CPU, disk, network.
5. Stop the bleeding: rollback, rate-limit, disable feature, scale, or fail over.
6. Verify recovery and capture evidence for postmortem.

### 6.2 Service Down

1. Confirm health and availability signals.
2. Check recent deployments and config changes.
3. Inspect logs around the first failure.
4. Check dependency availability.
5. Roll back or switch traffic if service-level objectives are being violated.
6. Keep status communication synchronized with technical actions.

### 6.3 P0/P1 Incident Command

Key roles:

- Incident commander: coordinates, decides escalation and communication.
- Operations lead: executes mitigations.
- Investigation lead: finds root cause evidence.
- Communications lead: updates stakeholders.
- Scribe: records timeline and decisions.

Senior SREs separate mitigation from diagnosis. Restoring service comes first; the full explanation can follow.

---

# Part 2: Advanced SRE Practices

## 7. Disaster Recovery Architecture

### 7.1 RTO and RPO

| Term | Meaning |
|------|---------|
| RTO | How quickly service must recover |
| RPO | How much data loss is acceptable |

Architecture choices:

| Pattern | RPO | RTO | Cost | Use |
|---------|-----|-----|------|-----|
| Backup restore | Hours to days | Hours to days | Low | Non-critical systems |
| Active-passive | Minutes | Minutes | Medium | Important internal systems |
| Same-city active-active | Seconds | Minutes | High | User-facing services |
| Multi-region active-active | Near zero | Seconds to minutes | Very high | Payment, finance, core commerce |

### 7.2 Active-Active Trade-Offs

Benefits:

- Better availability.
- Lower regional latency.
- Controlled failover.

Costs:

- Data consistency complexity.
- Traffic routing complexity.
- Conflict resolution.
- More expensive observability and drills.

The senior answer should include when not to use active-active. If the business does not require it, the complexity can reduce reliability.

### 7.3 Data Strategy

Common patterns:

- Single-writer per shard.
- User-based or tenant-based routing.
- Async replication with read-after-write handling.
- Conflict detection through versioning.
- Stronger consistency only for critical paths.

---

## 8. Incident Management and Self-Healing

### 8.1 Incident Severity

| Severity | Example | Response |
|----------|---------|----------|
| P0 | Full outage, data loss, severe financial risk | Immediate paging and command room |
| P1 | Core function unavailable or severely degraded | Fast escalation |
| P2 | Partial degradation | Business-hours or lower urgency depending on impact |
| P3 | Non-critical issue | Normal backlog |

Severity should be based on user impact, not internal component names.

### 8.2 Automatic Mitigation

Useful mechanisms:

- Circuit breakers.
- Adaptive rate limits.
- Bulkheads for thread and connection pools.
- Automatic rollback.
- Auto scaling.
- Read traffic fallback to primary when replica lag is high.
- Feature flags and kill switches.

Self-healing must include guardrails. An automation that acts on the wrong signal can amplify an incident.

### 8.3 Runbook and Playbook

| Document | Purpose |
|----------|---------|
| Runbook | Concrete execution steps for a known alert |
| Playbook | Coordination strategy for broader incident classes |

A good runbook includes:

- Alert meaning.
- Impact check.
- First commands or dashboards.
- Safe mitigation.
- Rollback.
- Validation.
- Escalation conditions.

### 8.4 Escalation Policy

Mature escalation is threshold-based:

- No mitigation within 10-15 minutes.
- Core SLI continues to degrade.
- Incident crosses team boundaries.
- Financial, security, legal, or public communication risk appears.

Escalation is not blame. It is a reliability control for time-sensitive decisions.

---

## 9. Monitoring System Design

### 9.1 Layered Monitoring

```text
User Experience
  -> Application
  -> Middleware
  -> Infrastructure
  -> Cloud Provider / Hardware
```

Examples:

- User side: synthetic monitoring, RUM, conversion.
- Application: QPS, latency, errors, business success.
- Middleware: DB, Redis, Kafka, RPC, search.
- Infrastructure: CPU, memory, disk, network, containers, nodes.

### 9.2 Business Metrics

For critical systems, technical success is not enough. Track business-level SLIs:

- payment success rate
- order creation success rate
- login success rate
- callback delay
- fraud decision latency

An API can return 200 while the business operation is wrong. Senior SRE design must include business correctness signals.

### 9.3 Data Governance

Govern:

- metric cardinality
- retention
- downsampling
- dashboard ownership
- alert ownership
- unused metrics
- cost per service or team

Observability without governance becomes expensive noise.

---

## 10. Performance Optimization

### 10.1 Methodology

1. Define target: P99, throughput, cost, or saturation.
2. Build a baseline.
3. Locate the bottleneck with profiles, traces, and resource metrics.
4. Change one major variable.
5. Validate improvement and regression risk.

### 10.2 Application-Level Optimization

Common areas:

- Reduce allocation and GC pressure.
- Reuse connections.
- Batch small operations.
- Avoid serialization hotspots.
- Add timeouts and cancellation.
- Control retries with budget and jitter.

### 10.3 Database and Cache

Database:

- Index critical queries.
- Avoid full scans.
- Control transaction size.
- Separate read and write paths carefully.
- Monitor connection pool wait, not just active connections.

Cache:

- Prevent penetration with negative caching or bloom filters.
- Prevent breakdown with singleflight or locks.
- Prevent avalanche with randomized TTL and staged warming.
- Track hit rate by route and data class.

### 10.4 Network and Protocols

HTTP/3 and QUIC help most when:

- Mobile or unstable networks are common.
- RTT is high.
- Connection migration matters.
- Packet loss creates long-tail issues.

They help less when:

- The bottleneck is application or database time.
- The path is stable internal RPC.
- Security, WAF, tracing, and packet analysis tooling do not support UDP/QUIC well.

---

## 11. Cost Optimization

### 11.1 Resource Rightsizing

Use actual utilization over meaningful windows:

- P50/P90/P99 CPU.
- P50/P90/P99 memory.
- peak QPS and seasonality.
- latency and saturation correlation.

Avoid lowering resources just because average CPU is low. Check burst behavior, GC, cold starts, and failure-mode capacity.

### 11.2 Storage Cost

Common techniques:

- Log retention tiers.
- Index lifecycle management.
- Metrics downsampling.
- Object storage for cold data.
- Compression and deduplication.

### 11.3 Traffic Cost

Optimize:

- CDN cache hit rate.
- cross-region traffic.
- cross-AZ data transfer.
- image and static asset size.
- unnecessary replication.

Cost work should not silently remove reliability margin. Make the risk visible.

---

## 12. Reliability Governance

### 12.1 SLI, SLO, SLA

| Abbr | Full name | One line |
|------|-----------|----------|
| **SLI** | Service Level **Indicator** | What you **measure** (observable, calculable) |
| **SLO** | Service Level **Objective** | Internal **target** for that SLI |
| **SLA** | Service Level **Agreement** | Customer **contract** with consequences if breached |

```text
SLI (what to measure)  →  SLO (internal target)  →  SLA (external contract)
```

**Payment API example**

| Layer | What it defines | Example |
|-------|-----------------|---------|
| SLI | Metric name + formula + window | `payment_success_rate = success / total`, 30d rolling |
| SLO | Internal target | success rate ≥ 99.95% |
| SLA | Customer commitment + remedy | monthly availability < 99.9% → 10% service credit |

**SLI definition vs measured value**

| | What it is | Example |
|--|------------|---------|
| SLI definition | metric + formula + window | `api_request_success_rate` over 30d |
| Measured value | current result of that SLI | 99.97% over last 30 days |
| SLO | threshold the value must meet | ≥ 99.95% |
| SLA | customer-facing floor (often below SLO) | ≥ 99.9% or compensation |

| Term | Audience | Meaning | Example |
|------|----------|---------|---------|
| SLI | Engineering | Measured service quality **indicator** | metric: `api_request_success_rate`; value: 99.97% |
| SLO | Internal team | **Target** the SLI must hit | availability ≥ 99.95%, P99 < 200ms |
| SLA | External customer | Contract commitment with **remedies** | monthly availability < 99.9% → 10% credit |

Common SLI types (user-facing, not machine vanity metrics):

| Type | Examples | Poor SLI |
|------|----------|----------|
| Availability | request success rate, health check pass rate | CPU < 80% |
| Latency | P99 response time, end-to-end duration | average QPS |
| Throughput | successful orders per second | pod count ≥ 3 |
| Correctness | payment callback on-time rate | dashboard is green |

Principles:

- SLIs must be precisely measurable and tied to user experience.
- SLOs are usually stricter than SLAs to leave internal buffer.
- Do not define an SLA for every internal component; start with SLI/SLO on critical user journeys.

### 12.2 Error Budget Policy

An error budget policy should map budget state to actions:

| State | Budget | Release Strategy |
|-------|--------|------------------|
| Healthy | > 50% remaining | normal releases |
| Watch | 25-50% remaining | stricter review and canary |
| Risk | < 25% remaining | only low-risk or reliability work |
| Exhausted | <= 0 | freeze non-fix releases |

If nothing changes when budget is exhausted, the SLO program is not governing engineering behavior.

### 12.3 Change Management

Release safeguards:

- canary release
- automatic rollback
- success rate gate
- P99 gate
- business metric gate
- capacity and dependency check
- rollback rehearsal for high-risk changes

Database and infrastructure changes require extra care because rollback can be much harder than application rollback.

### 12.4 DORA and SRE Metrics

DORA metrics describe delivery performance:

- deployment frequency
- lead time for changes
- change failure rate
- time to restore service

SRE metrics describe service reliability:

- SLO compliance
- error budget consumption
- availability
- latency
- alert quality

They are complementary. Fast delivery is not success if it burns reliability.

---

## 13. Chaos Engineering

### 13.1 Principles

1. Define steady state.
2. Form a hypothesis.
3. Inject a realistic failure with controlled blast radius.
4. Observe whether the system stays within SLO.
5. Turn findings into engineering action.

### 13.2 Useful Experiments

Examples:

- Kill one pod or VM.
- Add network delay to a dependency.
- Block DNS resolution.
- Fill disk in a controlled environment.
- Introduce database replica lag.
- Simulate cloud zone failure.

### 13.3 Common Failure of Chaos Programs

Chaos work fails when it becomes a demo:

- no clear SLO hypothesis
- no action item after discovery
- experiments only run in low-risk environments
- no incident response rehearsal
- no measurement of improved resilience

The value is not injecting failure. The value is proving and improving resilience.

---

## 14. Capacity Planning and FinOps

### 14.1 Capacity Estimation

Basic formula:

```text
Required instances = peak QPS * safety factor / single-instance capacity
```

Inputs:

- measured single-instance capacity
- seasonal traffic
- business growth
- burst profile
- startup time
- dependency capacity
- failover capacity

Do not size only for normal load. Size for failure mode: one AZ down, one dependency degraded, or a major release rollback.

### 14.2 Elastic Scaling

Common mechanisms:

- HPA: horizontal scaling for stateless services.
- VPA: vertical resizing, often better for stateful or proxy workloads but may restart pods.
- KEDA: event-driven scaling for queues and async work.
- Predictive scaling: pre-scale before expected peaks.

Pitfalls:

- scale-up lag
- cold start
- dependency bottleneck
- scale-down too aggressive
- metrics delayed or missing

### 14.3 FinOps

FinOps is the operating model for cloud cost:

- rightsizing
- reserved or savings commitments for baseline load
- spot instances for interruptible load
- idle resource cleanup
- storage lifecycle
- cost attribution by service or team

Good FinOps decisions include reliability constraints. Spot is excellent for stateless workers; it is dangerous for stateful databases or critical payment paths.

---

## 15. Interview Self-Check

### Q1: What are the four golden signals?

**Answer**: latency, traffic, errors, and saturation. Explain what each means and how it maps to user experience.

### Q2: How do you design SLOs for a payment API?

**Answer**:

- Pick user-facing SLIs: payment success rate, callback completion, P99 latency.
- Use a rolling window such as 30 days.
- Set realistic targets based on history and business need.
- Define burn-rate alerts.
- Connect error budget state to release policy.

### Q3: Why is burn-rate alerting better than a static error-rate threshold?

**Answer**: burn rate normalizes impact by SLO. A 1% error rate means different things for 99% and 99.99% SLOs. Multi-window burn rate catches both fast outages and slow degradation.

### Q4: How do RED and USE fit together in incident response?

**Answer**: RED proves whether user requests are harmed; USE finds the saturated resource. For example, API P99 rises, then DB connection pool wait and disk queue depth reveal the bottleneck.

### Q5: What is toil, and how would you handle a team spending 70% of time firefighting?

**Answer**: toil is repetitive, manual, automatable operational work. Measure top toil sources, automate the highest-frequency recovery actions, remove unactionable alerts, create runbooks, and track repeat incident rate.

### Q6: What makes a runbook high quality?

**Answer**: it is executable under pressure. It includes impact check, exact first steps, safe mitigation, rollback, validation, escalation criteria, and links to dashboards, logs, and traces.

### Q7: What is change failure rate and how do you govern it?

**Answer**: change failure rate is the percentage of changes that cause incidents, rollback, or hotfixes. Track it through the release platform and tighten gates when error budget is under pressure.

### Q8: What is cell-based architecture?

**Answer**: it partitions a system into independent cells with their own application capacity and data slice. It improves isolation and scalable operations but adds routing, data, and capacity complexity.

### Q9: How do cascading failures happen?

**Answer**: a slow dependency causes upstream queues and pools to fill, retries amplify traffic, and more services become saturated. Prevent it with timeouts, retry budgets, bulkheads, circuit breakers, and rate limits.

### Q10: Payment success drops from 99.99% to 99.9%, but CPU and memory are normal. What do you inspect?

**Answer**:

1. Segment failures by route, region, version, tenant, and dependency.
2. Use traces to locate the failing span.
3. Check business error codes, connection pools, timeouts, and external provider responses.
4. Check recent changes and feature flags.
5. Mitigate through rollback, provider routing, rate limiting, or degradation.

### Q11: What should happen automatically when error budget is exhausted?

**Answer**: freeze non-fix releases, raise release approval level, tighten canary gates, page service owners, and focus engineering work on reliability fixes until the budget state improves.

### Q12: Why can SLO be green while users complain?

**Answer**: the SLI may be too coarse, averaged over the wrong population, or detached from the real user journey. Fix by adding journey-level SLIs, segmentation, tail latency, and business correctness signals.

### Q13: How do you design an escalation policy for P1?

**Answer**: define trigger, time threshold, roles, contact path, required updates, and exit criteria. Example: if core success rate stays below threshold for 10 minutes without mitigation, escalate to the second-line owner; after 20 minutes, escalate to service lead and business stakeholder.

### Q14: How do you validate postmortem action items?

**Answer**: action items need owner, deadline, acceptance criteria, and evidence. Validate by deployment, rehearsal or real-traffic verification, metric improvement, and no recurrence over a review window.

### Q15: Why do some chaos engineering programs fail?

**Answer**: they inject faults without hypotheses, SLO validation, action tracking, or incident response rehearsal. That creates theater instead of resilience.

---

### Open-Ended Design Questions

**D1: Design an alerting system for a thousand-node cluster while avoiding alert storms.**

Reference answer:

- Define severity by user impact.
- Aggregate by root cause and service dependency.
- Use inhibition to suppress downstream symptoms.
- Use burn-rate alerts for user-facing services.
- Require runbook ownership.
- Track MTTA, MTTR, false-positive rate, and repeat alerts.

**D2: A core service SLO drops from 99.95% to 99.8%. What do you do?**

Reference answer:

- Identify when and where the budget burned.
- Separate one large incident from repeated smaller degradations.
- Freeze or tighten risky changes.
- Fix top contributors by user impact.
- Improve alerts, runbooks, auto rollback, and capacity.
- Review whether the SLI definition matches real user pain.

---

## Summary

Advanced SRE is the engineering discipline that connects user experience, system design, operational response, and governance. Strong SRE practice is visible in five behaviors: clear SLIs, actionable alerts, fast mitigation, disciplined postmortems, and reliability policies that actually influence engineering decisions.
