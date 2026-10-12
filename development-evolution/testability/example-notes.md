# Testability: hands-on practice notes

Planned labs for learning testability by **making untestable code testable, and making test suites fast and trustworthy**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)). Pick a test stack: **pytest** (Python), **Vitest/Jest** (TS), or **JUnit** (Java), plus **Testcontainers** and **Playwright**.

Keep a small table per lab: number of tests, suite duration, and what bugs each kind of test caught.

**Suggested order:** 1 → 2 → 3 → 5 → 6 → 4 → 7 → 8.

**Related labs elsewhere:** property-based testing of a ledger (reliability lab 13), Maelstrom/Gossip Glomers (reliability bonus), performance regression tests in CI (performance lab 14).

---

## Labs

### 1. Untestable → testable ⭐
- **Do:** pick messy-shop's discount/pricing code that mixes `now()`, SQL and email. Try to test it first, and write down what's painful.
- **Refactor:** functional core + imperative shell with an injected `Clock`, repository and mailer (README §4 worked example).
- **Measure:** number of edge cases tested and suite time, before vs after.

### 2. Mocks vs fakes vs real dependencies ⭐
- **Do:** test the order repository three ways: mocked DB client, in-memory fake, **Testcontainers** Postgres.
- **Break:** introduce real bugs: a typo in SQL, a violated unique constraint, a wrong transaction boundary, a timezone conversion bug.
- **See:** which approach catches which bug, and how long each suite takes. README §5.

### 3. Flaky test clinic ⭐
- **Do:** write 5 deliberately flaky tests: one using `sleep()` with async work, one depending on test order, one sharing DB rows, one using the real current time, one using unseeded randomness.
- **Detect:** run the suite 100 times in a loop (`pytest --count=100` with pytest-repeat, `vitest --repeat`, or a shell loop) and record pass rates.
- **Fix** each one at the root cause. Then compute the README §8.1 math with your own numbers.

### 4. Contract testing between services
- **Do:** split messy-shop's shipping into a separate service (or reuse modularity lab 7). Write consumer-driven contract tests with **Pact** (orders = consumer, shipping = provider).
- **Break:** rename a field in shipping's response. The provider's CI should fail before deployment.

### 5. Mutation testing
- **Do:** run **mutmut** (Python), **Stryker** (JS/TS) or **PIT** (Java) on the pricing module.
- **See:** surviving mutants despite high coverage. Strengthen assertions until most mutants are killed. Note the runtime cost.

### 6. Property-based tests
- **Do:** with **Hypothesis** or **fast-check**, test properties of the cart and pricing: total never negative, discount never increases the price, adding then removing an item returns the same cart, totals are the same whatever the order items are added in.
- **See:** shrunk counterexamples for bugs you didn't think of.

### 7. A few good E2E tests
- **Do:** **Playwright** tests for 2 critical journeys only (sign up, checkout), with test data created via API, not via UI clicks.
- **Measure:** runtime and flakiness over 50 runs. Use Playwright's trace viewer to debug a failure.
- **Discuss:** which other journeys deserve E2E tests, and which are better covered lower down?

### 8. Make CI fast
- **Do:** measure the full suite time. Then: parallelise (pytest-xdist, Vitest threads), split tests by size (small/medium/large), run only affected tests on PRs, cache dependencies and containers.
- **Goal:** under 5 minutes for the PR pipeline. Record each step's saving.

---

## Progress

- [ ] 1. Untestable → testable
- [ ] 2. Mocks vs fakes vs real dependencies
- [ ] 3. Flaky test clinic
- [ ] 4. Contract testing between services
- [ ] 5. Mutation testing
- [ ] 6. Property-based tests
- [ ] 7. A few good E2E tests
- [ ] 8. Make CI fast
