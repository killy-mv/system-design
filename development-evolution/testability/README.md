# Testability

> **Testability** = how easily, quickly and reliably you can **check that the system behaves correctly**, at every level from one function to the whole system.
> "Can I know within minutes, with confidence, that my change didn't break anything?"

In **ISO/IEC 25010**, testability is a sub-characteristic of maintainability: how effectively test criteria can be established and tests performed.

Related: [Maintainability §9](../maintainability/README.md#9-testing-strategy) (testing strategy: the short version), [Reliability §16](../../runtime-operational/reliability/README.md#16-testing-for-reliability) (testing for reliability: fuzzing, Jepsen, simulation), [Modularity](../modularity/README.md) (isolated modules are testable modules), [Deployability](../../operations-devops/deployability/README.md) (tests are what make frequent deploys safe).

---

## Table of contents

1. [Why testability matters](#1-why-testability-matters)
2. [What makes code testable](#2-what-makes-code-testable)
3. [What makes code hard to test](#3-what-makes-code-hard-to-test)
4. [Designing for testability](#4-designing-for-testability)
5. [Test doubles and real dependencies](#5-test-doubles-and-real-dependencies)
6. [Test levels and the shape of a test suite](#6-test-levels-and-the-shape-of-a-test-suite)
7. [Testing distributed systems](#7-testing-distributed-systems)
8. [Test suite health: speed, flakiness, quality](#8-test-suite-health-speed-flakiness-quality)
9. [Approaches by system type](#9-approaches-by-system-type)
10. [Trade-offs](#10-trade-offs)
11. [Anti-patterns](#11-anti-patterns)
12. [Real-world stories](#12-real-world-stories)
13. [Checklist](#13-checklist)
14. [Exercises](#14-exercises)
15. [Further reading](#15-further-reading)

---

## 1. Why testability matters

- **Tests make change safe.** Without them, every refactor is a gamble, so people stop refactoring, and the code decays.
- **Fast feedback = fast development.** A 3-minute suite gets run all the time; a 2-hour suite gets skipped.
- **Testable design is usually better design**: code that's easy to test tends to have clear inputs and outputs, explicit dependencies and small responsibilities.
- **Continuous delivery depends on it**: you can't deploy many times a day without automated confidence.

Testability is a property of the **code and architecture**, not just of the test suite. Hard-to-test code stays hard to test no matter how much effort goes into tests.

---

## 2. What makes code testable

| Property | Meaning |
|---|---|
| **Controllability** | You can put the code into the state you want and give it any input (including time, randomness, failures) |
| **Observability** | You can see the result: return values, state changes, emitted events, calls made |
| **Isolation** | You can test a part without starting everything else |
| **Determinism** | The same input gives the same result every time |
| **Speed** | Tests run in milliseconds to seconds |
| **Understandability** | It's clear what the code is supposed to do, so you know what to assert |

---

## 3. What makes code hard to test

- **Hidden dependencies**: globals, singletons, static calls, `new HttpClient()` inside a function.
- **I/O mixed with logic**: the pricing calculation is inside the HTTP handler next to SQL and an email call.
- **Time and randomness** read directly (`now()`, `random()`, `uuid4()`).
- **Hidden state**: module-level caches, mutable class attributes, shared test databases.
- **Constructors that do work** (connect to the DB, read files, call APIs).
- **Concurrency and async** without control over scheduling.
- **Huge functions and classes** with many responsibilities.
- **Framework-heavy code** where business logic only runs inside the framework.

---

## 4. Designing for testability

- **Dependency injection**: pass collaborators in (constructor or function arguments) instead of creating or importing them inside.
- **Inject time, randomness and IDs**: a `Clock`, a seeded random generator, an ID generator.
- **Functional core, imperative shell** (Gary Bernhardt): business decisions in **pure functions**; a thin outer layer does the I/O.
- **Ports and adapters** ([Maintainability §3.3](../maintainability/README.md#33-layered-and-ports-and-adapters-designs)): the domain talks to interfaces; tests use in-memory adapters.
- **Small, explicit interfaces** that are easy to fake.
- **Return values and events** instead of hidden side effects.
- **Idempotent, deterministic operations**: easier to test and to retry.
- **Configurable endpoints and timeouts** so tests can point at local fakes and run fast.

### Worked example: untestable → testable

```python
# Before: business rule mixed with time, DB and email
def apply_black_friday_discount(order_id):
    order = db.query("SELECT * FROM orders WHERE id = %s", order_id)
    if datetime.now().month == 11 and datetime.now().day >= 24 and order.total > 100:
        new_total = order.total * 0.8
        db.execute("UPDATE orders SET total = %s WHERE id = %s", new_total, order_id)
        smtp.send(order.email, f"You saved {order.total - new_total}!")
```

To test it you need a database, an SMTP server, and to **wait until late November** (or patch `datetime` globally). Every test is slow and fragile.

```python
# After: functional core
def black_friday_discount(total: Money, today: date) -> Money:
    in_window = today.month == 11 and today.day >= 24
    return total * Decimal("0.8") if in_window and total > Money(100) else total

# Imperative shell
def apply_discount(order_id, orders: OrderRepo, mailer: Mailer, clock: Clock):
    order = orders.get(order_id)
    new_total = black_friday_discount(order.total, clock.today())
    if new_total != order.total:
        orders.update_total(order_id, new_total)
        mailer.send_savings(order.email, order.total - new_total)
```

Now the rule is tested with plain values, in microseconds:

```python
assert black_friday_discount(Money(200), date(2026, 11, 25)) == Money(160)
assert black_friday_discount(Money(200), date(2026, 11, 23)) == Money(200)  # before window
assert black_friday_discount(Money(100), date(2026, 11, 25)) == Money(100)  # not > 100
```

The shell gets **one or two** integration tests with a real database and a fake mailer. The edge cases (dates, boundaries, rounding) live in fast unit tests. Same behaviour; tests go from "needs infrastructure and a calendar" to milliseconds.

---

## 5. Test doubles and real dependencies

| Double | What it does | Use for |
|---|---|---|
| **Dummy** | Passed but never used | Filling parameters |
| **Stub** | Returns canned answers | Controlling inputs ("the API returns 500") |
| **Spy** | Records how it was called | Checking a side effect happened |
| **Mock** | Pre-programmed expectations, fails if not met | Interaction-heavy protocols (use sparingly) |
| **Fake** | Working lightweight implementation | In-memory repository, fake payment provider, local email catcher |

Guidelines:
- Prefer **fakes and real dependencies** over deep mocks. Heavily mocked tests test your mocks, and break on every refactor.
- Use **real databases and brokers** in integration tests via **Testcontainers**: they catch SQL errors, constraint violations, transaction behaviour and serialisation issues that mocks never will.
- **Don't mock what you don't own**: wrap third-party APIs in your own adapter, test the adapter against a sandbox/contract, and fake *your* adapter elsewhere.
- Use **service virtualisation** (WireMock, Mock Service Worker, Prism for OpenAPI) for external HTTP dependencies.

---

## 6. Test levels and the shape of a test suite

| Level | Scope | Speed | Catches |
|---|---|---|---|
| **Unit** | A function/class/module in isolation | ms | Logic errors, edge cases |
| **Integration** | Your code + a real dependency (DB, queue) | seconds | Queries, mappings, transactions, configuration |
| **Component / service** | One service with its real DB, external services faked | seconds | The service's behaviour through its API |
| **Contract** | Agreement between consumer and provider | seconds | Breaking API/event changes between teams |
| **End-to-end** | The whole system through the UI/API | minutes | Wiring, critical user journeys |
| **Production checks** | Canaries, synthetic tests, feature-flag rollouts | continuous | What only real traffic and data reveal |

Shapes:
- **Pyramid**: many unit, fewer integration, few E2E. Classic, good for logic-heavy code.
- **Trophy** (Kent C. Dodds): emphasis on integration tests, plus static analysis at the base. Good for web apps where most bugs are in wiring.
- **Honeycomb** (Spotify): mostly service-level integration tests for microservices.
- **Ice-cream cone** (anti-pattern): mostly manual and E2E tests, few unit tests. Slow and flaky.

Pick by asking: **which tests give the most confidence per second of runtime and minute of maintenance?** Google's "test sizes" are a practical rule: *small* (single process, no I/O), *medium* (single machine, localhost I/O), *large* (multiple machines). Size limits what a test may do and keeps the suite fast.

---

## 7. Testing distributed systems

- **Contract tests** (Pact, Spring Cloud Contract; schema compatibility checks for events) let services deploy independently without a full E2E environment.
- **Component tests** per service with Testcontainers and faked neighbours.
- **A few E2E tests** for the most critical journeys only (checkout, signup).
- **Ephemeral environments** per pull request for realistic integration.
- **Testing in production, safely**: canaries, feature flags, synthetic users, shadow traffic ([Deployability](../../operations-devops/deployability/README.md)).
- **Fault injection**: timeouts, errors, partitions with Toxiproxy and chaos tools ([Availability](../../runtime-operational/availability/README.md)).
- **Deterministic simulation and property-based testing** for the hardest correctness problems ([Reliability §16](../../runtime-operational/reliability/README.md#16-testing-for-reliability)).

---

## 8. Test suite health: speed, flakiness, quality

### 8.1 Flaky tests

A **flaky** test sometimes passes and sometimes fails without any code change. Common causes: timing and `sleep()`, test order dependence, shared state or databases, real network calls, unseeded randomness, time zones and dates, concurrency, resource leaks.

### Worked example: why "just a little flaky" isn't OK

A suite of **500 tests**, each flaky only **0.1%** of the time:

```
P(entire suite green) = 0.999^500 ≈ 0.61
```

**39% of CI runs fail for no reason.** People start clicking "retry" by reflex, and soon real failures get ignored too. With 2,000 such tests: 0.999^2000 ≈ 14% green.

Fix: track flakiness per test, **quarantine** flaky tests immediately (they still run, but don't block), and fix the root cause (inject time, wait for conditions instead of sleeping, isolate data per test, random ports).

### 8.2 Speed

- Budget: unit tests in seconds, the full CI suite in **under 10 minutes**.
- Parallelise; run only affected tests (build tools like Bazel, Nx, Turborepo, pytest-testmon); keep large tests few.

### 8.3 Quality: are the tests any good?

- **Coverage** shows what's *not* tested; high coverage doesn't prove good tests.
- **Mutation testing** (Stryker, mutmut, PIT, cargo-mutants): changes your code in small ways and checks tests fail. Surviving mutants = weak assertions.
- **Test behaviour, not implementation**: tests should survive refactoring.
- **Readable tests**: Arrange–Act–Assert, test data builders, one reason to fail, good names ("rejects refund after 30 days").

---

## 9. Approaches by system type

| System | Testability focus |
|---|---|
| **Backend APIs** | Functional core for business rules; component tests with Testcontainers; contract tests with consumers |
| **Frontend** | Testing Library (test what users see), Mock Service Worker for APIs, a few Playwright E2E tests, visual regression for design systems |
| **Mobile** | Unit tests for view models, UI tests for critical flows on real devices/emulators, snapshot tests |
| **Microservices** | Contract tests, component tests, few E2E, production canaries |
| **Data pipelines** | Unit tests on transformations, data tests (dbt tests, Great Expectations), data contracts, sample datasets |
| **Infrastructure as code** | `plan` checks, policy tests (OPA/conftest), Terratest for modules, ephemeral test environments |
| **ML systems** | Evaluation datasets as regression tests, data validation, deterministic seeds, tests for feature pipelines |
| **LLM apps** | Eval suites with graded examples, schema validation of outputs, fakes for model calls in unit tests |
| **Distributed databases / protocols** | Property-based tests, deterministic simulation, Jepsen-style tests, formal models (TLA+) |
| **Legacy code** | Characterisation tests first, then find seams to break dependencies ([Maintainability §16](../maintainability/README.md#16-evolving-legacy-systems)) |

---

## 10. Trade-offs

| Choice | Cost |
|---|---|
| Many unit tests | Can lock in implementation details if written against internals |
| Mostly integration tests | Slower, but closer to reality |
| E2E tests | High confidence per test, but slow, flaky and costly to maintain |
| Heavy mocking | Fast, but tests can pass while the real system fails |
| Testcontainers / real deps | Realistic, but slower and needs Docker in CI |
| Dependency injection everywhere | Testable, but more indirection; keep it to real boundaries |
| High coverage targets | Encourage tests, but also encourage low-value tests written to hit numbers |

---

## 11. Anti-patterns

- Tests that break on every refactor because they assert internal calls.
- Mocking everything, including your own simple value objects.
- `sleep(2)` in tests instead of waiting for a condition.
- A shared test database that tests modify, so order matters.
- Ignoring or re-running flaky tests instead of fixing them.
- The ice-cream cone: mostly manual or E2E testing.
- Coverage targets gamed with assertion-free tests.
- Business logic only reachable through the UI or a running framework.
- No tests for the scariest code, because it's the hardest to test.

---

## 12. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Ariane 5 (1996)** | Software reused from Ariane 4 converted a 64-bit floating point value into a 16-bit integer. Ariane 5's faster trajectory produced a value too large, the conversion overflowed, and the rocket self-destructed about 40 seconds after launch. The reused code had not been tested with Ariane 5 flight data. | Reused code must be re-tested against the new system's real inputs and assumptions. |
| **Google flaky tests (2016)** | Google reported that about 1.5% of all test runs were flaky, and that around 16% of their tests showed some flakiness. Flaky tests were a major cost and source of lost trust. | Flakiness is a systemic problem that needs tooling and policy, not just individual fixes. |
| **Google test sizes** | Google classifies tests as small, medium or large by what they may do (processes, I/O, network), with time limits for each. This keeps the vast majority of tests fast and hermetic. | Constraints on what tests may touch keep suites fast at scale. |
| **SQLite** | SQLite is tested with a test suite many hundreds of times larger than the library itself, including 100% branch coverage (MC/DC), fuzzing, out-of-memory and I/O-failure simulation. It's one of the most widely deployed pieces of software in the world. | For foundational software, extreme testability and testing are worth it. |
| **FoundationDB** | FoundationDB was built from the start to run inside a deterministic simulator that injects faults, so years of simulated failures could be replayed exactly from a seed. | Designing for testability can make even distributed systems testable. |

---

## 13. Checklist

- [ ] Business rules in pure functions or a domain core, testable without I/O
- [ ] Time, randomness, IDs and external services injected
- [ ] Integration tests with real dependencies (Testcontainers) for data access
- [ ] Contract tests between services; a small set of E2E tests for critical journeys
- [ ] CI suite under 10 minutes; tests parallelised
- [ ] Flaky tests tracked, quarantined and fixed
- [ ] Mutation testing or similar used on critical code
- [ ] Tests assert behaviour, not implementation details

---

## 14. Exercises

1. **Refactor for testability.** Find a function in your code that mixes logic with I/O. Split it into a functional core and a shell, and write tests for the core.
2. **Flakiness math.** Your suite has 1,200 tests; 30 of them fail 2% of the time each, the rest never. What's the chance a run is green? How many retries on average until green?
3. **Choose the level.** For each, which test level fits best: tax calculation, a SQL query with window functions, an API used by another team, the checkout journey, a retry-with-backoff helper, a Kafka consumer's deserialisation.
4. **Mocks vs fakes.** Write the same test for an order service three ways: with mocks, with an in-memory fake repository, and with Testcontainers Postgres. Introduce a bug in the SQL. Which tests catch it?
5. **Contract test.** Two services exchange an `OrderCreated` event. Design a contract test that would catch the provider renaming a field.
6. **Legacy.** You inherit a 2,000-line untested function. Describe how you'd get it under test before changing it.

---

## 15. Further reading

- *Growing Object-Oriented Software, Guided by Tests* (Freeman, Pryce)
- *Working Effectively with Legacy Code* (Michael Feathers): seams, characterisation tests
- *xUnit Test Patterns* (Gerard Meszaros): test doubles vocabulary
- *Unit Testing Principles, Practices, and Patterns* (Vladimir Khorikov)
- *Software Engineering at Google* ch. 11–14 (testing overview, unit testing, test doubles, larger testing) — free online
- Gary Bernhardt, "Boundaries" talk (functional core, imperative shell)
- Martin Fowler, "TestPyramid", "Mocks Aren't Stubs"
- Kent C. Dodds, "The Testing Trophy"
- Google Testing Blog, "Flaky Tests at Google and How We Mitigate Them" (2016)
- SQLite, "How SQLite Is Tested" (sqlite.org/testing.html)
