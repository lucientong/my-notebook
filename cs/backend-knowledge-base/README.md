# Backend Knowledge Base

> This English backend knowledge base mirrors the Chinese `后端知识库` with the same 01-19 numbering. It is designed for preparing backend interviews in English, especially for international companies where you need to explain fundamentals, system design trade-offs, and production experience clearly.

Language: English | [中文](../后端知识库/README.md)

---

## Who This Is For

- Backend engineers preparing for English technical interviews
- Engineers who know the concepts in Chinese but want accurate English terminology and answer structure
- Candidates preparing for system design, distributed systems, infrastructure, cloud-native, or platform engineering roles
- Developers who want to connect language internals, databases, middleware, and production architecture into one coherent backend map

---

## What Backend Engineering Covers

Backend engineering is not just about a programming language or a framework. A strong interview answer usually connects several layers:

- `Systems fundamentals`: operating systems, networking, concurrency, memory, IO, and failure modes
- `Programming languages`: Go, Java, Python, and C++ runtime behavior, performance, and concurrency models
- `Data systems`: MySQL, Redis, PostgreSQL, ClickHouse, MongoDB, Elasticsearch, and vector databases
- `Middleware`: message queues, caching, search, API gateways, service discovery, and configuration
- `Architecture`: rate limiting, circuit breaking, distributed transactions, consistency, high availability, and microservices
- `Engineering governance`: API design, authentication, authorization, observability, deployment, and long-term maintainability

For interviews, avoid only naming technologies. A stronger answer explains what problem the technology solves, what trade-off it introduces, and how you would operate it in production.

---

## Recommended Reading Paths

### Path A: Backend Fundamentals

1. [13-Operating-Systems-In-Depth.md](./13-Operating-Systems-In-Depth.md)
2. [06-Networking-Fundamentals-and-Protocols.md](./06-Networking-Fundamentals-and-Protocols.md)
3. [05-Concurrency-Programming-Models.md](./05-Concurrency-Programming-Models.md)
4. [02-MySQL-In-Depth.md](./02-MySQL-In-Depth.md)
5. [07-Redis-and-Caching.md](./07-Redis-and-Caching.md)
6. [08-Message-Queues.md](./08-Message-Queues.md)

This path builds the foundation for explaining why services become slow, unstable, inconsistent, or hard to scale.

### Path B: Main Programming Stack

Choose the language that matches your target role:

1. Go roles: [01-Go-In-Depth.md](./01-Go-In-Depth.md) -> [05-Concurrency-Programming-Models.md](./05-Concurrency-Programming-Models.md)
2. Java roles: [04-Java-In-Depth.md](./04-Java-In-Depth.md) -> [05-Concurrency-Programming-Models.md](./05-Concurrency-Programming-Models.md)
3. Python backend / AI engineering roles: [03-Python-Development.md](./03-Python-Development.md) -> [06-Networking-Fundamentals-and-Protocols.md](./06-Networking-Fundamentals-and-Protocols.md)
4. Infrastructure / performance roles: [15-Cpp-System-Programming.md](./15-Cpp-System-Programming.md) -> [13-Operating-Systems-In-Depth.md](./13-Operating-Systems-In-Depth.md)

Do not read language documents as syntax notes. Focus on runtime behavior, memory management, concurrency, error handling, and profiling.

### Path C: System Design and Architecture

1. [09-System-Design-and-Architecture.md](./09-System-Design-and-Architecture.md)
2. [12-Microservices-Architecture.md](./12-Microservices-Architecture.md)
3. [11-Cloud-Native-and-Containers.md](./11-Cloud-Native-and-Containers.md)
4. [16-Authentication-and-Authorization.md](./16-Authentication-and-Authorization.md)
5. [17-API-Design-and-Governance.md](./17-API-Design-and-Governance.md)
6. [18-Design-Patterns-and-Programming-Paradigms.md](./18-Design-Patterns-and-Programming-Paradigms.md)

This path is most useful for mid-level and senior interviews. Draw the architecture, identify bottlenecks, and explicitly discuss trade-offs.

### Path D: Data and Search

1. [02-MySQL-In-Depth.md](./02-MySQL-In-Depth.md)
2. [10-Databases-Comprehensive.md](./10-Databases-Comprehensive.md)
3. [14-Big-Data-Processing-Fundamentals.md](./14-Big-Data-Processing-Fundamentals.md)
4. [19-Elasticsearch-Search-Engine.md](./19-Elasticsearch-Search-Engine.md)

This path is useful for data-intensive applications, search systems, recommendation systems, logging pipelines, and analytics platforms.

### Path E: English Interview Sprint

1. Review the Chinese main path first: `13 -> 06 -> 02 -> 07 -> 08 -> 09`.
2. Read the English documents with the same numbers.
3. Practice each interview question out loud using a simple structure:
   - `Short answer`: state the core idea first.
   - `Mechanism`: explain how it works.
   - `Trade-off`: mention what can go wrong or what you give up.
   - `Production practice`: describe monitoring, retry, rollback, or operational safeguards.

---

## Difficulty Levels

### L1: Fundamentals

Goal: explain processes, threads, TCP, HTTP, indexes, transactions, cache, queues, and basic failure modes in your own words.

Recommended documents:

- [13-Operating-Systems-In-Depth.md](./13-Operating-Systems-In-Depth.md)
- [06-Networking-Fundamentals-and-Protocols.md](./06-Networking-Fundamentals-and-Protocols.md)
- [02-MySQL-In-Depth.md](./02-MySQL-In-Depth.md)
- [07-Redis-and-Caching.md](./07-Redis-and-Caching.md)
- [08-Message-Queues.md](./08-Message-Queues.md)

### L2: Engineering Practice

Goal: design APIs, handle retries, make consumers idempotent, govern cache behavior, split services, and diagnose performance problems.

Recommended documents:

- [01-Go-In-Depth.md](./01-Go-In-Depth.md) / [04-Java-In-Depth.md](./04-Java-In-Depth.md) / [03-Python-Development.md](./03-Python-Development.md)
- [09-System-Design-and-Architecture.md](./09-System-Design-and-Architecture.md)
- [12-Microservices-Architecture.md](./12-Microservices-Architecture.md)
- [17-API-Design-and-Governance.md](./17-API-Design-and-Governance.md)

### L3: Deep Architecture and Specialized Topics

Goal: reason about high availability, distributed consistency, cloud-native deployment, auth systems, search systems, big data processing, and long-term maintainability.

Recommended documents:

- [11-Cloud-Native-and-Containers.md](./11-Cloud-Native-and-Containers.md)
- [14-Big-Data-Processing-Fundamentals.md](./14-Big-Data-Processing-Fundamentals.md)
- [15-Cpp-System-Programming.md](./15-Cpp-System-Programming.md)
- [16-Authentication-and-Authorization.md](./16-Authentication-and-Authorization.md)
- [18-Design-Patterns-and-Programming-Paradigms.md](./18-Design-Patterns-and-Programming-Paradigms.md)
- [19-Elasticsearch-Search-Engine.md](./19-Elasticsearch-Search-Engine.md)

---

## Document Map

| # | English Document | Chinese Source | Focus | Status |
|---|------------------|----------------|-------|--------|
| 01 | [Go In Depth](./01-Go-In-Depth.md) | [Go语言深入](../后端知识库/01-Go语言深入.md) | core types, GMP scheduler, goroutines, channels, context, sync primitives, GC, Gin | Done |
| 02 | [MySQL In Depth](./02-MySQL-In-Depth.md) | [MySQL深入](../后端知识库/02-MySQL深入.md) | B+Tree indexes, execution plans, transactions, locks, MVCC, replication | Done |
| 03 | [Python Development](./03-Python-Development.md) | [Python开发](../后端知识库/03-Python开发.md) | data model, memory/GC, GIL, metaprogramming, asyncio, Django ORM, FastAPI | Done |
| 04 | [Java In Depth](./04-Java-In-Depth.md) | [Java深入](../后端知识库/04-Java深入.md) | JVM, GC, JUC, Spring, MyBatis, tuning, virtual threads | Done |
| 05 | [Concurrency Programming Models](./05-Concurrency-Programming-Models.md) | [并发编程模型](../后端知识库/05-并发编程模型.md) | Go/Python/Java concurrency models, primitives, pitfalls | Done |
| 06 | [Networking Fundamentals and Protocols](./06-Networking-Fundamentals-and-Protocols.md) | [网络基础与协议](../后端知识库/06-网络基础与协议.md) | OSI, TCP, HTTP/HTTPS, WebSocket, DNS, QUIC, BBR, troubleshooting | Done |
| 07 | [Redis and Caching](./07-Redis-and-Caching.md) | [Redis与缓存](../后端知识库/07-Redis与缓存.md) | Redis structures, cache penetration/breakdown/avalanche, locks, HA, Redis 7 | Done |
| 08 | [Message Queues](./08-Message-Queues.md) | [消息队列](../后端知识库/08-消息队列.md) | Kafka, RabbitMQ, reliability, ordering, DLQ, selection | Done |
| 09 | [System Design and Architecture](./09-System-Design-and-Architecture.md) | [系统设计与架构](../后端知识库/09-系统设计与架构.md) | rate limiting, circuit breaking, distributed transactions, consistency | Done |
| 10 | [Databases Comprehensive](./10-Databases-Comprehensive.md) | [数据库综合](../后端知识库/10-数据库综合.md) | PostgreSQL, ClickHouse, InfluxDB, Milvus, MongoDB, selection | Done |
| 11 | [Cloud Native and Containers](./11-Cloud-Native-and-Containers.md) | [云原生与容器](../后端知识库/11-云原生与容器.md) | Docker, Kubernetes, Service Mesh | Done |
| 12 | [Microservices Architecture](./12-Microservices-Architecture.md) | [微服务架构](../后端知识库/12-微服务架构.md) | service discovery, gateway, resilience, tracing, config center | Done |
| 13 | [Operating Systems In Depth](./13-Operating-Systems-In-Depth.md) | [操作系统深入](../后端知识库/13-操作系统深入.md) | processes, threads, virtual memory, IO, filesystems, network stack | Done |
| 14 | [Big Data Processing Fundamentals](./14-Big-Data-Processing-Fundamentals.md) | [大数据处理基础](../后端知识库/14-大数据处理基础.md) | batch processing, stream processing, log analytics, alerting | Done |
| 15 | [C++ System Programming](./15-Cpp-System-Programming.md) | [C++系统编程](../后端知识库/15-C++系统编程.md) | modern C++, RAII, move semantics, memory model, STL, performance | Done |
| 16 | [Authentication and Authorization](./16-Authentication-and-Authorization.md) | [认证授权专题](../后端知识库/16-认证授权专题.md) | Session, JWT, OAuth2, OIDC, RBAC, ABAC, MFA, password security | Done |
| 17 | [API Design and Governance](./17-API-Design-and-Governance.md) | [API设计与治理](../后端知识库/17-API设计与治理.md) | REST, gRPC, GraphQL, idempotency, gateways, versioning, security | Done |
| 18 | [Design Patterns and Programming Paradigms](./18-Design-Patterns-and-Programming-Paradigms.md) | [设计模式与编程范式](../后端知识库/18-设计模式与编程范式.md) | SOLID, Go/Java patterns, DDD, DI, anti-patterns | Done |
| 19 | [Elasticsearch Search Engine](./19-Elasticsearch-Search-Engine.md) | [Elasticsearch搜索引擎](../后端知识库/19-Elasticsearch搜索引擎.md) | inverted index, FST, BM25, DSL, aggregations, clusters, tuning | Done |

---

## How To Answer Backend Questions In English

Use this structure for most technical questions:

```text
In short, ...
The key mechanism is ...
The main trade-off is ...
In production, I would also ...
```

Examples:

- For reliability questions, mention the producer, broker, and consumer sides separately.
- For consistency questions, state the desired guarantee first, then explain what can be relaxed.
- For performance questions, separate latency, throughput, resource usage, and tail latency.
- For system design questions, start with requirements and constraints before naming components.

---

## Common Interview Pitfalls

- Translating Chinese answers word by word instead of using a clear English answer structure
- Only naming tools like Kafka, Redis, or Kubernetes without explaining the failure modes they introduce
- Ignoring idempotency, retry budgets, DLQ handling, replay, and observability in distributed systems
- Treating system design as a fixed template instead of a set of trade-offs under constraints
- Over-optimizing early without first clarifying correctness, reliability, and operational complexity

---

## Maintenance Notes

- The English documents keep the same numbering as the Chinese backend documents.
- Code examples are preserved unless comments need to be translated.
- Interview sections use `### Qx:` and `**Answer:**` for consistency.
- Senior follow-up questions emphasize trade-offs, failure modes, debugging SOPs, capacity, consistency, security, and observability.
- Cross-references should point to English documents when available.
- Completed documents: 01-19 are now fully mirrored in English.
