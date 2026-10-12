# Modularity: hands-on practice notes

Planned labs for learning modularity by **taking a tangled codebase apart, enforcing boundaries, and measuring how far changes spread**. Theory is in [README.md](README.md).

## The shared practice codebase: `messy-shop`

All development-evolution labs (modularity, simplicity, testability, evolvability, and the short ones) use the same deliberately messy app, built once:

- A small shop backend: **catalog, cart, orders, payments (fake provider), shipping, emails, loyalty points, admin reports**.
- One language you want to practise (Python + FastAPI/Django or TypeScript + Express/NestJS work well), one Postgres database.
- Written **badly on purpose**:
  - package by layer (`controllers/`, `services/`, `models/`), with one shared `models` file
  - every module reads every table; reports join across everything
  - a `utils.py` / `helpers.ts` that everything imports
  - a dependency cycle (orders ↔ payments)
  - business logic mixed with HTTP, SQL, `now()`, random IDs and email sending
  - an `Order` with boolean flags (`is_paid`, `is_shipped`, `is_cancelled`)
  - a `switch` on payment method in 4 places
- A small set of end-to-end tests so refactoring is safe.

Alternative if you don't want to write it: start from a **RealWorld ("Conduit")** example app implementation in your language and add the mess.

Keep a `CHANGELOG-labs.md`: for each lab, what you changed and the before/after numbers.

**Suggested order:** 1 → 2 → 3 → 4 → 5 → 6, then 7.

**Planned folder layout:**

```
development-evolution/labs/
  messy-shop/              ← the shared codebase (git history matters for lab 2)
development-evolution/modularity/labs/
  01-dependency-graph/README.md
  ...
```

---

## Labs

### 1. Draw the real dependency graph ⭐
- **Do:** generate the module/import graph of messy-shop with **pydeps** or **import-linter** (Python), **dependency-cruiser** or **madge** (JS/TS), or **jdeps** (Java).
- **Find:** cycles, the most-depended-on files, and modules that reach into other modules' internals.
- **Compute:** instability `I = Ce / (Ca + Ce)` for each top-level package (README §6). Does anything stable depend on something unstable?

### 2. Change coupling from Git history
- **Do:** on messy-shop (after a few weeks of commits) **and** on a real open-source project, find files that change together. Use **code-maat** (Adam Tornhill) or a small script over `git log --name-only`.
- **See:** pairs that always change together but live in different directories = hidden coupling. Compare with what the dependency graph from lab 1 shows.
- **See:** README §9.

### 3. Package by layer → package by feature ⭐
- **Before:** implement "gift message on orders" and count files and directories touched.
- **Refactor:** reorganise into `orders/`, `payments/`, `shipping/`, `catalog/`... each with a small public `api` module; move code and break the cycle.
- **After:** implement a similar feature ("delivery instructions") and count again.
- **See:** README §4.3 worked example.

### 4. Enforce the boundaries in CI ⭐
- **Do:** add rules with **import-linter**, **dependency-cruiser**, **ArchUnit**, or **Spring Modulith**: modules may only import other modules' `api`; no cycles; `shared/kernel` depends on nothing.
- **Break:** deliberately import `payments/internal/...` from orders and watch CI fail.
- **See:** README §7.

### 5. Each module owns its data
- **Do:** move tables into per-module schemas (`orders.*`, `payments.*`). Replace the admin report's cross-schema join with either (a) calls to each module's API, or (b) a reporting read model updated from events.
- **See:** what breaks, what gets slower, what gets clearer. README §5.

### 6. Domain events inside the monolith
- **Do:** make orders publish an in-process `OrderPlaced` event. Move email sending and loyalty points into subscribers.
- **Test:** add a new subscriber ("notify warehouse") without changing a single line in `orders/`.
- **Discuss:** what happens if a subscriber fails? (Link: outbox in reliability lab 4.)

### 7. Extract one module into a service
- **Do:** move `shipping` out of the monolith into its own service (it already has a clean interface after labs 3–6).
- **Measure what you paid:** added latency per request, new failure modes (shipping down → ?), a second pipeline, tracing needed to follow a request.
- **See:** README §8. Then decide honestly whether it was worth it.

---

## Bonus: read and compare

Read Shopify's "Deconstructing the Monolith" and Segment's "Goodbye Microservices". Write half a page: why did they move in **opposite** directions, and what did both care about?

---

## Tools

| Tool | Used for |
|---|---|
| pydeps / import-linter (Python), dependency-cruiser / madge (JS/TS), jdeps / ArchUnit (Java) | Dependency graphs and boundary rules |
| code-maat or a `git log` script | Change coupling |
| Postgres schemas | Data ownership per module |
| Docker Compose | Running the extracted service (lab 7) |

## Progress

- [ ] Build messy-shop
- [ ] 1. Draw the real dependency graph
- [ ] 2. Change coupling from Git history
- [ ] 3. Package by layer → package by feature
- [ ] 4. Enforce the boundaries in CI
- [ ] 5. Each module owns its data
- [ ] 6. Domain events inside the monolith
- [ ] 7. Extract one module into a service
- [ ] Bonus: read and compare
