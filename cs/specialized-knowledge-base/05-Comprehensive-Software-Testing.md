# Comprehensive Software Testing - Full-Stack Testing Practice

Language: English | [中文](../专项知识库/05-软件测试综合.md)

---

## Table of Contents

### Foundations
1. [Test Pyramid](#1-test-pyramid)
2. [Testing Theory Foundations](#2-testing-theory-foundations)

### Language and Framework Practice
3. [Frontend Testing](#3-frontend-testing)
4. [Go Testing](#4-go-testing)
5. [Python Testing](#5-python-testing)
6. [Java Testing](#6-java-testing)

### Advanced Testing Types
7. [API Testing](#7-api-testing)
8. [Contract Testing](#8-contract-testing)
9. [Mutation Testing](#9-mutation-testing)
10. [Test Data Management](#10-test-data-management)
11. [Performance Testing](#11-performance-testing)
12. [CI/CD Testing Strategy](#12-cicd-testing-strategy)

### Interview Self-Check
13. [Interview Self-Check](#13-interview-self-check)

---

## 1. Test Pyramid

```text
        +-----------+
        | E2E Tests |    few, expensive, fragile
        +-----------+
        |Integration|    moderate, verifies boundaries
        +-----------+
        |Unit Tests |    many, fast, stable
        +-----------+
```

| Test Type | Typical Share | Speed | Cost | Feedback |
|-----------|---------------|-------|------|----------|
| Unit | 70% | milliseconds | low | fast |
| Integration | 20% | seconds | medium | medium |
| E2E | 10% | minutes | high | slow |

### 1.1 Unit Tests

Unit tests verify one function, class, or module in isolation.

```javascript
function add(a, b) {
  return a + b;
}

test('adds numbers', () => {
  expect(add(1, 2)).toBe(3);
});
```

Good unit tests are fast, deterministic, focused, and easy to diagnose.

### 1.2 Integration Tests

Integration tests verify collaboration between modules or external boundaries:

- API plus database.
- service plus message broker.
- repository plus real storage.
- provider plus contract verification.

```javascript
test('creates a user', async () => {
  const response = await request(app)
    .post('/api/users')
    .send({ name: 'Alice', email: 'alice@example.com' });

  expect(response.status).toBe(201);
  const user = await db.query('SELECT * FROM users WHERE email = ?', ['alice@example.com']);
  expect(user.name).toBe('Alice');
});
```

### 1.3 E2E Tests

E2E tests verify the most important user journeys:

```javascript
test('user login flow', async ({ page }) => {
  await page.goto('/login');
  await page.fill('input[name="email"]', 'test@example.com');
  await page.fill('input[name="password"]', 'password');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/dashboard');
});
```

E2E should cover critical happy paths and a few high-risk failure paths, not every business rule.

### 1.4 The Ice Cream Cone Anti-Pattern

The anti-pattern is too many E2E tests and too few unit tests. Symptoms:

- slow CI
- flaky releases
- failures hard to diagnose
- expensive maintenance
- late feedback

Fix by pushing logic into testable units, using contract tests for service boundaries, and limiting E2E to critical journeys.

---

## 2. Testing Theory Foundations

### 2.1 Coverage

| Coverage Type | Meaning | Caveat |
|---------------|---------|--------|
| Statement | executed statements / total statements | can miss branches |
| Branch | executed branches / total branches | better for conditionals |
| Function | called functions / total functions | may hide weak assertions |
| Line | executed lines / total lines | useful but not sufficient |

Coverage proves code was executed, not that behavior was correct.

Example:

```javascript
function isAdult(age) {
  return age > 18; // bug: should be >= 18
}

test('adult', () => {
  expect(isAdult(20)).toBe(true);
});
```

This can produce high coverage while missing the boundary bug.

### 2.2 Instrumentation

Coverage tools inject counters into code:

```javascript
var __coverage__ = { add: { statements: 0, functions: 0 } };

function add(a, b) {
  __coverage__.add.functions++;
  __coverage__.add.statements++;
  return a + b;
}
```

The test run records which counters were hit, then reports statement, branch, function, and line coverage.

### 2.3 Mocking and Dependency Injection

Mocking is useful when real dependencies are:

- slow
- unstable
- expensive
- unavailable
- hard to force into edge cases

Dependency injection makes code testable:

```javascript
async function getUserProfile(userId, fetcher = fetch) {
  const response = await fetcher(`/api/users/${userId}`);
  return response.json();
}

test('gets user profile', async () => {
  const mockFetch = jest.fn().mockResolvedValue({
    json: async () => ({ id: 1, name: 'Alice' })
  });

  const profile = await getUserProfile(1, mockFetch);
  expect(profile.name).toBe('Alice');
  expect(mockFetch).toHaveBeenCalledWith('/api/users/1');
});
```

### 2.4 Test Doubles

| Type | Role | Example |
|------|------|---------|
| Dummy | passed but not used | unused logger argument |
| Stub | returns fixed values | fake user service response |
| Spy | records calls | verify logger called |
| Mock | predefines expectations | verify email service called with exact payload |
| Fake | simplified working implementation | in-memory database |

Stub versus mock:

- Stub supports state verification.
- Mock supports interaction verification.

### 2.5 Test Isolation

Common isolation techniques:

- reset process state in `beforeEach`
- transaction rollback for database tests
- unique test data per case
- fake timers for time-dependent code
- stable random seed
- isolated containers for external dependencies

Flaky tests often come from hidden shared state.

### 2.6 TDD

TDD loop:

```text
Red -> Green -> Refactor
```

Benefits:

- naturally testable design
- fast feedback
- safer refactoring
- executable documentation

Costs:

- slower early implementation
- team learning curve
- less suitable when requirements are still being explored

---

## 3. Frontend Testing

### 3.1 Jest and Component Tests

Use unit tests for pure logic and component tests for UI behavior.

```javascript
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from './Counter';

test('increments count', () => {
  render(<Counter />);
  fireEvent.click(screen.getByText('Increment'));
  expect(screen.getByTestId('count')).toHaveTextContent('Count: 1');
});
```

Prefer user-visible selectors when possible. Use stable `data-testid` only when semantic selectors are insufficient.

### 3.2 Async UI Tests

Use proper waits, not arbitrary sleeps:

```javascript
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});
```

Common async pitfalls:

- not waiting for state updates
- unresolved promises
- timers not controlled
- network mocks leaking across tests

### 3.3 Playwright E2E

```javascript
import { test, expect } from '@playwright/test';

test('successful login', async ({ page }) => {
  await page.goto('/login');
  await page.fill('input[name="email"]', 'test@example.com');
  await page.fill('input[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  await page.waitForURL('/dashboard');
  await expect(page.locator('h1')).toHaveText('Dashboard');
});
```

E2E best practices:

- isolate data per test
- use stable locators
- mock unstable third-party dependencies
- record screenshots and videos on failure
- keep tests independent

---

## 4. Go Testing

### 4.1 Basic and Table-Driven Tests

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Fatalf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

Table-driven tests are readable, easy to extend, and make boundary cases visible.

### 4.2 Race and Concurrency Tests

```bash
go test -race ./...
go test -run TestName -count=100
```

Use:

- `sync.WaitGroup`
- `atomic`
- deterministic scheduling where possible
- repeated runs for suspected race cases

### 4.3 HTTP Tests

```go
func TestHealthHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/health", nil)
    w := httptest.NewRecorder()

    healthHandler(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("status = %d", w.Code)
    }
}
```

### 4.4 Benchmarks

```go
func BenchmarkFibonacci(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Fibonacci(10)
    }
}
```

Run with:

```bash
go test -bench=. -benchmem
```

Benchmark results should be interpreted with allocation count, CPU profile, and realistic workload shape.

---

## 5. Python Testing

### 5.1 pytest Basics

```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)
```

Parameterized tests:

```python
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

### 5.2 Fixtures

```python
@pytest.fixture
def db_session():
    session = db.create_session()
    session.begin()
    yield session
    session.rollback()
    session.close()
```

Fixtures should own setup and cleanup. Shared fixtures must not leak mutable state across tests.

### 5.3 Mocking

```python
from unittest.mock import patch, Mock

def test_get_user():
    with patch("api.requests.get") as mock_get:
        response = Mock()
        response.json.return_value = {"id": 1, "name": "Alice"}
        mock_get.return_value = response

        user = get_user(1)
        assert user["name"] == "Alice"
        mock_get.assert_called_once_with("https://api.example.com/users/1")
```

Patch where the dependency is used, not where it is originally defined.

---

## 6. Java Testing

### 6.1 JUnit 5

```java
@Test
void testDivideByZero() {
    Exception exception = assertThrows(
        IllegalArgumentException.class,
        () -> calculator.divide(10, 0)
    );
    assertEquals("Cannot divide by zero", exception.getMessage());
}
```

### 6.2 Mockito

```java
@Test
void testGetUserById() {
    UserService mockService = mock(UserService.class);
    when(mockService.getUser(1L)).thenReturn(new User(1L, "Alice"));

    UserController controller = new UserController(mockService);
    User user = controller.getUserById(1L);

    assertEquals("Alice", user.getName());
    verify(mockService, times(1)).getUser(1L);
}
```

### 6.3 Spring Boot Tests

Use test slices for fast feedback:

- `@WebMvcTest` for controllers.
- `@DataJpaTest` for repositories.
- `@SpringBootTest` for full integration.

Full application tests are valuable but expensive; use them intentionally.

---

## 7. API Testing

API tests verify:

- status code
- response schema
- error codes
- authentication and authorization
- idempotency
- backward compatibility

Example with SuperTest:

```javascript
describe('User API', () => {
  test('GET /api/users/:id', async () => {
    const response = await request(app)
      .get('/api/users/1')
      .expect(200);

    expect(response.body.name).toBe('Alice');
  });
});
```

Senior focus: API tests should include failure paths such as invalid input, permission failure, timeout behavior, duplicate requests, and backward-compatible schema evolution.

---

## 8. Contract Testing

### 8.1 What Contract Testing Solves

Contract testing verifies that service consumers and providers agree on API behavior.

It solves:

- expensive full-environment integration tests
- broken provider changes
- slow parallel development
- unclear ownership when integration fails

### 8.2 Consumer-Driven Contract Flow

```text
Consumer defines expected request/response
  -> Consumer test generates contract
  -> Contract is published to broker
  -> Provider verifies implementation against contract
  -> Deployment checks compatibility
```

### 8.3 What to Put in a Contract

Include:

- method
- path
- headers
- request schema
- response schema
- status code
- error response shape
- provider state

Avoid over-specifying incidental fields that consumers do not rely on.

---

## 9. Mutation Testing

Mutation testing introduces small code changes to verify whether tests detect them.

Example mutations:

- `>=` to `>`
- `+` to `-`
- `&&` to `||`
- `return x` to `return 0`
- remove a statement

Mutation score:

```text
Mutation Score = killed mutants / total mutants
```

Why it matters:

- Coverage can be high with weak assertions.
- Mutation testing reveals whether assertions actually protect behavior.

Best practice:

- Run incrementally on changed code.
- Do not demand 100%; equivalent mutants exist.
- Use survived mutants to improve tests.

---

## 10. Test Data Management

### 10.1 Strategies

| Strategy | Use |
|----------|-----|
| In-memory data | unit tests |
| Transaction rollback | integration tests |
| Database snapshot | E2E or complex scenarios |
| Data factory | dynamic and readable data |
| TestContainers | realistic external dependencies |

### 10.2 Data Factory

```javascript
const createUser = (overrides = {}) => ({
  id: crypto.randomUUID(),
  name: 'Alice',
  email: 'alice@example.com',
  role: 'user',
  ...overrides
});

const admin = createUser({ role: 'admin' });
```

### 10.3 Principles

- Keep data minimal.
- Make data independent.
- Make tests repeatable.
- Avoid magic numbers.
- Tie test data to the behavior being verified.

---

## 11. Performance Testing

### 11.1 Types

| Type | Goal |
|------|------|
| Benchmark | baseline single operation cost |
| Load test | validate expected traffic |
| Stress test | find breaking point |
| Soak test | detect leaks over time |
| Spike test | validate burst resilience |

### 11.2 k6 Example

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const response = http.get('http://localhost:8080/api/users');
  check(response, { 'status is 200': r => r.status === 200 });
  sleep(1);
}
```

### 11.3 Load Model Design

A realistic model includes:

- concurrency
- arrival rate
- ramp-up
- peak duration
- think time
- traffic mix
- data distribution
- dependency behavior

Performance testing without production-like data and dependency behavior often gives false confidence.

---

## 12. CI/CD Testing Strategy

### 12.1 Pipeline Shape

```text
Build
  -> Unit tests
  -> Integration tests
  -> Contract verification
  -> Security/static checks
  -> E2E for critical paths
  -> Performance baseline for high-risk changes
```

Suggested gates:

- no new critical bugs
- changed-code coverage threshold
- contract compatibility
- no high-severity security findings
- performance regression below agreed threshold
- rollback path verified for high-risk changes

### 12.2 Parallelization

Parallelize tests that do not share resources. Run shared-resource tests serially or isolate dependencies per worker.

### 12.3 Flaky Test Governance

Detect:

```bash
for i in {1..10}; do npm test; done
```

Governance:

- quarantine flaky tests so they do not mask real failures
- keep quarantine visible
- assign owners
- fix or delete within a short window
- track flaky rate as a quality metric

Flaky tests are not harmless. They train teams to distrust CI.

---

## 13. Interview Self-Check

### Q1: What is the test pyramid and why does it matter?

**Answer**: it recommends many fast unit tests, fewer integration tests, and a small number of E2E tests. This balances confidence, speed, and maintainability.

### Q2: What is the difference between stub and mock?

**Answer**: a stub provides predefined data for state verification. A mock defines and verifies interactions, such as whether a dependency was called with specific arguments.

### Q3: Does 100% coverage mean there are no bugs?

**Answer**: no. Coverage means execution, not correctness. Boundary bugs, missing assertions, concurrency issues, security issues, and performance issues can remain.

### Q4: What is contract testing?

**Answer**: it verifies API agreements between consumer and provider. Consumer-driven contracts let consumers define expectations and providers validate implementation without requiring a full integration environment.

### Q5: How do you test external dependencies?

**Answer**:

- Unit tests: mock interfaces.
- Integration tests: use TestContainers, embedded services, or WireMock.
- Service boundaries: use contract tests.
- E2E: use only stable or controlled dependencies.

### Q6: What is mutation testing?

**Answer**: mutation testing injects artificial bugs and checks whether tests fail. It measures assertion strength better than coverage alone.

### Q7: How do you handle flaky E2E tests?

**Answer**:

1. Identify and tag flaky tests through repeated runs and CI history.
2. Classify cause: timing, data, environment, ordering, dependency.
3. Fix with explicit waits, isolated data, stable selectors, and controlled dependencies.
4. Quarantine temporarily but keep ownership and deadline.

### Q8: Why do many teams have many tests but still see frequent production bugs?

**Answer**: tests may protect low-risk happy paths while missing real failure modes: timeouts, retries, rollback, concurrency, dirty data, schema compatibility, and dependency degradation.

### Q9: Why are async workflows easy to miss in tests?

**Answer**: message delay, duplicate consumption, out-of-order events, dead letters, compensation failure, and eventual consistency are not naturally covered by synchronous API tests. Cover them through unit tests for idempotency, integration tests with real message brokers, and system-level trace-based verification.

### Q10: Release tests passed but production rolled back. What should be reviewed?

**Answer**:

- environment differences
- production data differences
- missing non-functional gates
- rollback rehearsal
- observability and alert gaps
- dependency versions and configuration drift

### Q11: How do you measure testing ROI?

**Answer**:

- defect escape rate
- test execution time
- flaky rate
- test maintenance cost
- MTTR after failed tests
- confidence during refactoring
- repeated defect rate

### Q12: What is shift-left testing?

**Answer**: moving quality activities earlier: requirement review, acceptance criteria, API contract review, TDD, static checks, pre-commit tests, and PR gates.

### Q13: How would you design a rollback-aware test strategy?

**Answer**: bind test results to artifact version, config version, schema version, and release baseline. Validate rollback compatibility for database and message changes, and keep previous stable baselines for canary comparison.

### Q14: How should defect recurrence be managed?

**Answer**: classify each production defect by test gap, add templates or checks for recurring classes, track escape rate and recurrence rate, and verify that the same category decreases over time.

### Q15: As an interviewer, how do you identify real testing experience?

**Answer**: ask for a specific production bug that tests missed, how the team traced the gap, what test or process was added, and how they proved the same class of bug decreased.

---

### Open-Ended Design Questions

**D1: Design an automated testing system for a hundred-engineer team.**

Reference answer:

- Unit tests required on PRs, fast and isolated.
- Integration tests for service boundaries with TestContainers.
- Contract tests for microservices.
- E2E tests only for critical user journeys.
- Performance baselines for high-risk paths.
- Clear quality gates and ownership.
- Dashboards for coverage, execution time, flaky rate, and defect escape rate.

**D2: E2E tests are constantly flaky. How do you govern them?**

Reference answer:

- Identify top flaky tests through repeated runs and CI history.
- Classify root causes.
- Replace sleeps with explicit waits.
- Isolate test data.
- Mock unstable third parties.
- Quarantine only temporarily.
- Track flaky rate and close recurring causes.

---

## Summary

A mature testing system is not measured by test count alone. It is measured by whether high-risk behavior is protected, feedback is fast, tests are trusted, and production failures lead to stronger regression protection instead of repeated firefighting.
