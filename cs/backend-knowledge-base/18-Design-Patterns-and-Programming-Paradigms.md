# Design Patterns and Programming Paradigms

Language: English | [中文](../后端知识库/18-设计模式与编程范式.md)

---

## Table of Contents

1. [SOLID Principles](#1-solid-principles)
2. [Creational Patterns](#2-creational-patterns)
3. [Structural Patterns](#3-structural-patterns)
4. [Behavioral Patterns](#4-behavioral-patterns)
5. [Go-Specific Patterns](#5-go-specific-patterns)
6. [Domain-Driven Design Basics](#6-domain-driven-design-basics)
7. [Dependency Injection](#7-dependency-injection)
8. [Anti-Patterns](#8-anti-patterns)
9. [Practical Refactoring Case](#9-practical-refactoring-case)
10. [Interview Self-Check](#10-interview-self-check)

---

## 1. SOLID Principles

### 1.1 Single Responsibility Principle

A module should have one reason to change.

In practice, this means separating business rules, persistence, transport, and external integration when they change for different reasons.

### 1.2 Open/Closed Principle

Software should be open for extension but closed for modification.

Use interfaces, strategies, registries, or configuration to add behavior without rewriting stable core logic.

### 1.3 Liskov Substitution Principle

Subtypes should be substitutable for their base types without breaking correctness.

Violation often appears when subclasses weaken preconditions, strengthen postconditions unexpectedly, or throw unsupported exceptions.

### 1.4 Interface Segregation Principle

Prefer small, focused interfaces over large general-purpose interfaces.

### 1.5 Dependency Inversion Principle

High-level modules should depend on abstractions, not concrete implementations.

This improves testability and reduces coupling to infrastructure.

---

## 2. Creational Patterns

### 2.1 Singleton

Singleton ensures one instance.

Use carefully. Global mutable singletons can hide dependencies and hurt testing.

### 2.2 Factory Method

Factory Method delegates object creation to a factory interface or method.

Useful when creation depends on type, configuration, or runtime environment.

### 2.3 Abstract Factory

Abstract Factory creates families of related objects.

Example: create storage, cache, and queue clients for a specific cloud provider.

### 2.4 Builder

Builder constructs complex objects step by step.

Useful when constructors have many optional parameters.

### 2.5 Prototype

Prototype creates new objects by cloning existing ones.

Useful when initialization is expensive and object state can be copied safely.

---

## 3. Structural Patterns

### 3.1 Adapter

Adapter converts one interface into another expected interface.

Common in third-party SDK integration.

### 3.2 Decorator

Decorator adds behavior without changing the wrapped object.

Examples:

- logging wrapper.
- retry wrapper.
- metrics wrapper.
- cache wrapper.

### 3.3 Proxy

Proxy controls access to another object.

Examples:

- remote proxy.
- lazy proxy.
- security proxy.
- AOP proxy.

### 3.4 Facade

Facade provides a simplified interface over complex subsystems.

### 3.5 Bridge

Bridge separates abstraction from implementation so both can evolve independently.

### 3.6 Composite

Composite represents tree structures where leaf and container share common interface.

### 3.7 Flyweight

Flyweight shares common immutable state to reduce memory usage.

---

## 4. Behavioral Patterns

### 4.1 Strategy

Strategy encapsulates interchangeable algorithms.

Good for payment methods, pricing rules, routing policies, and feature switches.

### 4.2 Observer

Observer notifies subscribers when events happen.

In backend systems, message queues and event buses are often distributed observer-like mechanisms.

### 4.3 Template Method

Template Method defines a fixed algorithm skeleton and lets subclasses or hooks customize steps.

### 4.4 Chain of Responsibility

Chain passes requests through handlers until one handles or rejects it.

Common in middleware, filters, and approval workflows.

### 4.5 Command

Command represents an operation as an object.

Useful for queues, retries, undo/redo, and audit.

### 4.6 State

State changes behavior based on internal state.

Useful for order/payment state machines.

### 4.7 Iterator

Iterator provides uniform traversal without exposing internal structure.

### 4.8 Mediator

Mediator centralizes interactions between components, reducing direct coupling.

---

## 5. Go-Specific Patterns

### 5.1 Functional Options

Functional options configure objects without long constructors.

```go
type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}
```

### 5.2 Middleware

Middleware composes request processing behavior.

Common uses:

- logging.
- auth.
- rate limiting.
- tracing.
- recovery.

### 5.3 Pipeline

Pipeline connects stages with channels or queues.

### 5.4 Fan-Out / Fan-In

Fan-out parallelizes work. Fan-in merges results.

Use context cancellation and bounded concurrency to avoid leaks.

---

## 6. Domain-Driven Design Basics

### 6.1 Core Building Blocks

- Entity.
- Value Object.
- Aggregate Root.
- Domain Service.
- Repository.
- Domain Event.

### 6.2 Entity

Entity has identity and lifecycle.

### 6.3 Value Object

Value object is defined by values and should be immutable.

### 6.4 Aggregate Root

Aggregate root protects invariants inside a consistency boundary.

Do not modify internal entities from outside the aggregate.

### 6.5 Domain Service

Domain service contains domain behavior that does not naturally belong to one entity.

### 6.6 Repository

Repository abstracts persistence for aggregates.

### 6.7 Domain Event

Domain event records something meaningful that happened in the domain.

---

## 7. Dependency Injection

DI provides dependencies from outside instead of constructing them internally.

Benefits:

- testability.
- decoupling.
- configuration flexibility.

Go commonly uses constructor injection or compile-time DI tools like Wire. Java Spring uses runtime IoC container and reflection/proxies.

---

## 8. Anti-Patterns

### 8.1 God Object

One object knows too much and does too much.

### 8.2 Spaghetti Code

Control flow and dependencies are tangled, making changes risky.

### 8.3 Over-Engineering

Too many abstractions for uncertain requirements.

### 8.4 Circular Dependency

Modules depend on each other in a cycle, making testing, deployment, and reasoning harder.

### 8.5 Recognition Checklist

Watch for:

- huge classes/functions.
- hidden global state.
- repeated switch by type.
- unstable interfaces.
- anemic domain model with scattered rules.
- abstractions with only one fake implementation and no clear need.

---

## 9. Practical Refactoring Case

Order system before refactoring:

- controller contains validation, pricing, payment, persistence, notifications.
- many `if` branches by payment method.
- retry and idempotency logic duplicated.

Refactoring approach:

- Strategy for payment methods.
- Chain of Responsibility for validation.
- Domain entity and aggregate for order state.
- Repository for persistence.
- Domain events for notifications.
- Decorator for metrics/retry/logging.

Goal is not to use patterns for their own sake. The goal is to make change safer and business rules clearer.

---

## 10. Interview Self-Check

### Q1: What is the purpose of design patterns?

**Answer:** They provide reusable names and structures for common design problems, helping teams communicate and avoid repeated mistakes.

### Q2: Strategy vs Factory?

**Answer:** Strategy chooses behavior. Factory creates objects.

### Q3: Decorator vs Proxy?

**Answer:** Decorator adds behavior while preserving the interface. Proxy controls access to the target, often for lazy loading, remote calls, security, or AOP.

### Q4: What is Dependency Injection?

**Answer:** Dependencies are provided from outside instead of constructed internally, improving decoupling and testability.

### Q5: What is an aggregate root?

**Answer:** It is the entry point of an aggregate and enforces invariants within a consistency boundary.

### Q6: What is a value object?

**Answer:** An immutable object identified by its values, not identity, such as Money or Address.

### Q7: Why can Singleton be harmful?

**Answer:** It introduces global state, hidden dependencies, and testing difficulty if overused.

### Q8: What is over-engineering?

**Answer:** Adding abstractions or complexity before there is a real need, making the system harder to understand and change.

### Q9: How do you choose a pattern?

**Answer:** Start from the change pressure and coupling problem. Use a pattern only when it makes the design simpler, more testable, or safer to evolve.

### Q10: How would you refactor a large order service?

**Answer:** Identify responsibilities, extract domain rules, define interfaces at boundaries, introduce strategies for variable behavior, add idempotency/state machine, and improve tests before large rewrites.

### Senior Interview Follow-Ups

### Q11: How do you tell whether a design pattern is helping or hurting?

**Answer:** A pattern helps when it matches a real change pressure, reduces coupling, protects invariants, or makes tests and ownership clearer. It hurts when it adds abstractions with one implementation, hides control flow, or forces developers to jump through layers for simple changes. In code review, ask what requirement is expected to vary and what concrete complexity the pattern removes.

### Q12: How do you avoid an anemic domain model in a backend service?

**Answer:** Keep important invariants and state transitions close to the domain objects or aggregate roots instead of scattering them across controllers and application services. Persistence and transport concerns should stay at boundaries. Application services orchestrate use cases, while entities/value objects enforce business rules. The trade-off is to avoid overloading entities with infrastructure details.

### Q13: How would you migrate from procedural transaction scripts to a cleaner domain design?

**Answer:** Start with the highest-change or highest-risk use case. Add characterization tests around current behavior, extract pure validation and state transition logic, introduce value objects for concepts like Money or Status, then move persistence behind repositories. Do not rewrite the whole module at once; use strangler-style refactoring and keep database transactions explicit until the new boundaries are proven.
