# Modularity

> **Modularity** = the system is divided into **parts with clear boundaries**, where each part hides its internals behind an interface, so parts can be understood, changed, tested and replaced **independently**.
> "Can I change how orders are priced without touching, re-testing or even understanding shipping, emails and reports?"

In **ISO/IEC 25010**, modularity is a sub-characteristic of maintainability: *"a change to one component has minimal impact on other components"*.

Related: [Maintainability §3](../maintainability/README.md#3-modularity-coupling-and-cohesion) (coupling, cohesion, SOLID, hexagonal: the short version), [Maintainability §5](../maintainability/README.md#5-domain-driven-design-and-boundaries) (DDD and bounded contexts), [Simplicity](../simplicity/README.md), [Testability](../testability/README.md), [Evolvability](../evolvability/README.md), [Modifiability](../modifiability/README.md).

---

## Table of contents

1. [What a module is](#1-what-a-module-is)
2. [Coupling and cohesion in depth](#2-coupling-and-cohesion-in-depth)
3. [Information hiding and deep modules](#3-information-hiding-and-deep-modules)
4. [Where to draw boundaries](#4-where-to-draw-boundaries)
5. [Module interfaces and data ownership](#5-module-interfaces-and-data-ownership)
6. [Dependency rules](#6-dependency-rules)
7. [The modular monolith](#7-the-modular-monolith)
8. [Modules vs services](#8-modules-vs-services)
9. [Measuring modularity](#9-measuring-modularity)
10. [Approaches by system type](#10-approaches-by-system-type)
11. [Trade-offs](#11-trade-offs)
12. [Anti-patterns](#12-anti-patterns)
13. [Real-world stories](#13-real-world-stories)
14. [Checklist](#14-checklist)
15. [Exercises](#15-exercises)
16. [Further reading](#16-further-reading)

---

## 1. What a module is

A **module** is any unit with an **interface** (what others may use) and an **implementation** (what's hidden). The same ideas apply at every level:

```
function → class → package/namespace → module/component → service → system
```

A module is good when:
- You can **use** it by reading only its interface.
- You can **change** its implementation without changing its users.
- You can **test** it on its own.
- It has **one clear purpose** you can say in a sentence.

The goal isn't "many small pieces". It's **independence**: being able to work on one part while safely ignoring the rest. That's what lets a codebase (and a team) grow without slowing down.

---

## 2. Coupling and cohesion in depth

- **Cohesion**: how strongly the things *inside* a module belong together. High cohesion = things that change together live together.
- **Coupling**: how much modules *depend on each other*. Low coupling = a change in one rarely forces a change in another.

### 2.1 Connascence: a finer vocabulary for coupling

Two pieces of code are **connascent** if changing one requires changing the other. Weaker forms are better, and any form is worse when the two pieces are **far apart** (different modules, teams, services).

| Form (weak → strong) | Meaning | Example |
|---|---|---|
| **Name** | Must agree on a name | Calling `calculateTotal()` |
| **Type** | Must agree on a type | Passing a `Money`, not a `float` |
| **Meaning** | Must agree on what values mean | `status = 3` means "shipped" (use an enum instead) |
| **Position** | Must agree on order | `create_user("Ana", "ana@x.com", True, False)` |
| **Algorithm** | Must use the same algorithm | Both sides compute the same hash/checksum format |
| **Execution order** | Must run in a specific order | `init()` must be called before `send()` |
| **Timing** | Must happen within some time | Race conditions, timeouts |
| **Identity** | Must refer to the same instance | Two modules sharing one mutable object |

Rules of thumb: **convert strong forms into weak ones** (magic numbers → named enums; positional args → named args/objects), and **keep strong connascence inside a module**, never across boundaries.

### 2.2 Coupling across systems

| Coupling (worst → better) | Example |
|---|---|
| Shared mutable database tables | Two services read and write `orders` |
| Synchronous runtime dependency | A fails whenever B is down or slow |
| Implementation details | Importing another module's internal class or private table |
| Shared data format | Both depend on the same schema |
| Published contract (API/events) | Versioned API or event schema owned by one side |

---

## 3. Information hiding and deep modules

**Information hiding** (David Parnas, 1972): decompose a system so that each module hides **one design decision likely to change** (a data format, an algorithm, a vendor, a hardware detail). When that decision changes, only one module changes.

Parnas's key insight was that the "obvious" decomposition, **one module per processing step** (read input → process → write output), is usually *worse*: every step knows the shared data format, so changing the format touches everything. Decomposing by **design decisions** instead keeps changes local.

**Deep modules** (John Ousterhout): a module's value = functionality it provides − complexity of its interface.

```
Deep module                    Shallow module
┌──────────┐ small interface   ┌──────────────────────┐ big interface
│          │                   │                      │
│          │ lots of           └──────────────────────┘ little behind it
│          │ functionality
│          │
└──────────┘
```

- Unix file I/O: 5 calls (`open`, `read`, `write`, `lseek`, `close`) hide disks, file systems, caching, permissions, devices.
- A shallow example: a `UserRepository` with 30 methods that each wrap one SQL query. The caller still has to know everything.

Prefer **fewer, deeper** modules over many thin pass-through layers.

---

## 4. Where to draw boundaries

### 4.1 Split by business capability, not by technical layer

```
Package by layer (common, weak)          Package by feature / capability (better)
controllers/                             orders/
  order_controller                         api, service, repository, models
  payment_controller                     payments/
services/                                  api, service, repository, models
  order_service                          shipping/
  payment_service                          ...
repositories/
  order_repository
  payment_repository
```

A typical feature change ("orders can have a gift message") touches the **controller, service, repository, model and migration** of orders. Package-by-layer spreads that across 5 directories mixed with unrelated code; package-by-feature keeps it in one place.

### 4.2 Good boundary heuristics

- **Bounded contexts** (DDD): where the meaning of words changes ("customer" in billing vs support), there's a boundary ([Maintainability §5](../maintainability/README.md#5-domain-driven-design-and-boundaries)).
- **Rate of change**: separate stable parts from volatile ones (tax rules change yearly; the order state machine rarely).
- **Data ownership**: one module owns each piece of data.
- **Team ownership** (Conway's law): boundaries that match teams stay clean; boundaries that cut across teams erode.
- **Different quality needs**: a part with very different security, scaling or availability needs is a good separate module.

### 4.3 Worked example: counting the files a change touches

Feature: "Add a gift message to orders" in a mid-size app.

| Structure | Files touched | Directories | Modules/teams involved |
|---|---|---|---|
| Package by layer, shared `models.py` used everywhere | 9 (5 layers + shared model + 3 places reading `Order` fields directly) | 6 | Orders, Shipping, Emails (they touch `Order` internals) |
| Package by feature, `Order` internals hidden behind the orders API | 4 (orders API, service, repository, migration) | 1 | Orders only |

Shipping and emails didn't need to change in the second design because they get order data through the orders module's interface (or an `OrderPlaced` event), not by reading its tables and classes. That difference, multiplied by every change for years, is the value of modularity.

---

## 5. Module interfaces and data ownership

- **Small public surface**: export only what others need. Everything else is internal (language visibility rules, `internal` packages, `__init__.py` exports, `package.json` `exports`).
- **Stable interface, free implementation**: treat a module's public API like a published API ([Maintainability §6](../maintainability/README.md#6-apis-and-contracts)).
- **Each module owns its data**: only the owning module reads and writes its tables. Others go through its interface or subscribe to its events. Shared tables are the most common way modularity silently dies.
- **Communication between modules** (in the same process):
  - direct calls through the module's public interface (simple, synchronous)
  - **domain events** published in-process (`OrderPlaced`) so new subscribers don't change the publisher
- **Don't leak internal types**: return the module's own DTOs, not ORM entities that drag the database model across the boundary.

---

## 6. Dependency rules

- **Acyclic Dependencies Principle**: the module dependency graph must have **no cycles**. With a cycle, the modules effectively become one big module: you can't build, test or reason about one without the other. Break cycles with dependency inversion (an interface owned by one side) or events.
- **Stable Dependencies Principle**: depend in the direction of **stability**. Volatile modules may depend on stable ones, not the reverse.
- **Dependency inversion**: high-level policy (domain logic) shouldn't depend on low-level details (database, HTTP, vendor SDKs). Define a port in the domain; implement it in an adapter ([Maintainability §3.3](../maintainability/README.md#33-layered-and-ports-and-adapters-designs)).

**Instability** (Robert C. Martin) helps check the direction:

```
I = Ce / (Ca + Ce)
Ce = efferent coupling (modules this one depends on)
Ca = afferent coupling (modules that depend on this one)
I = 0 → maximally stable (many depend on it, it depends on nothing)
I = 1 → maximally unstable (depends on others, nothing depends on it)
```

Example: `money` (used by 8 modules, uses nothing) → I = 0: keep it very stable and well tested. `admin-reports` (uses 6 modules, used by none) → I = 1: free to change often. A **stable module depending on an unstable one** is a warning sign.

---

## 7. The modular monolith

One deployable application, internally split into **strictly separated modules**. It's the recommended default for most systems ([Maintainability §4](../maintainability/README.md#4-architecture-styles)).

```
app/
  modules/
    orders/        public: api.py (functions, DTOs), events.py   internal: everything else
    payments/
    shipping/
    catalog/
  shared/kernel/   tiny: Money, IDs, clock interface (stable, rarely changes)
  main.py          wiring
```

Make the boundaries **real**, or they erode within months:
- **Enforce dependency rules in CI**: import-linter (Python), dependency-cruiser (JS/TS), ArchUnit (Java), Spring Modulith, Packwerk (Ruby, from Shopify), Nx module boundaries, .NET `internal` + NetArchTest.
- **Separate schemas per module** in the same database (`orders.*`, `payments.*`), with no cross-schema foreign keys or joins.
- **Module-level tests** that only use the module's public interface.
- **Owners** per module (CODEOWNERS).

Benefits: monolith simplicity (one deploy, in-process calls, easy refactoring, local transactions where needed) plus clear boundaries, and a cheap path to extract a service later if one module really needs it.

---

## 8. Modules vs services

A service is a module with a **network boundary**. That adds:
- latency and new failure modes (timeouts, partial failure, retries)
- independent deployment and scaling (the benefit)
- data consistency problems (no shared transactions)
- operational cost: pipelines, monitoring, on-call, versioning

**Get the boundaries right as modules first.** Moving a well-isolated module to a service is mostly mechanical. Splitting a tangled codebase into services gives you a **distributed monolith**: all the costs of services plus all the coupling of the mess.

Extract a module into a service when it has a clear reason: a separate team, very different scaling or availability needs, a different security boundary, a different release cadence, or a different technology.

---

## 9. Measuring modularity

| Measure | How | Healthy sign |
|---|---|---|
| **Dependency graph** | Tools above, or `pydeps`, `madge`, `jdeps` | No cycles; arrows point toward stable modules |
| **Boundary violations** | Architecture tests in CI | Zero, and the rules exist |
| **Change coupling** | From Git history: files/modules that change in the same commits (CodeScene, code-maat) | Changes mostly stay within one module |
| **Cross-module change rate** | % of PRs touching more than one module | Low and stable |
| **Instability vs abstractness** | Martin's metrics per module | Stable modules are abstract; concrete modules are unstable |
| **Team signal** | How often teams must coordinate to ship | Rare |

Change coupling is the most honest measure: it shows what *actually* changes together, whatever the diagram says.

---

## 10. Approaches by system type

| System | Modularity focus |
|---|---|
| **Small app / MVP** | Package by feature from day one; a few clear modules; no need for enforcement tooling yet |
| **Growing SaaS** | Modular monolith with enforced boundaries, schema per module, module owners |
| **Microservices** | Service boundaries along bounded contexts; database per service; contracts and events; group services into domains |
| **Frontend apps** | Feature folders, a shared design system, clear state ownership; micro-frontends only for large multi-team apps |
| **Mobile apps** | Feature modules (Gradle modules, Swift packages) for build speed and ownership |
| **Libraries / SDKs** | Small public API, everything else internal; semantic versioning of the public surface |
| **Data platforms** | Data products owned by domains (data mesh ideas); clear contracts for datasets |
| **Infrastructure code** | Terraform modules with clear inputs/outputs; no giant "everything" state file |

---

## 11. Trade-offs

| Choice | Cost |
|---|---|
| Strict boundaries | Some duplication; extra mapping code (DTOs); slower "quick hacks" |
| Many small modules | Navigation overhead, shallow modules, more interfaces to maintain |
| Few big modules | Low coordination cost, but internal tangling |
| Events between modules | Looser coupling, harder to follow the flow |
| Network boundaries (services) | Independence vs latency, failure modes, operational cost |
| Enforcement tooling | Setup and occasional friction vs boundaries that actually hold |

---

## 12. Anti-patterns

- **Big ball of mud**: no recognizable structure; everything can call everything.
- **God module / `utils` / `common` dumping ground** that everything depends on and that changes constantly.
- **Cyclic dependencies** between modules.
- **Shared tables** across modules or services.
- **Package by layer** in a large codebase, so every feature touches every directory.
- **Leaking ORM entities** across boundaries.
- **Entity services**: one service per noun (`UserService`, `OrderService`, `ProductService`) that all need each other for any real use case.
- **Distributed monolith**: services that must deploy together or share a database.
- **Boundaries only on the diagram**, not enforced in code.

---

## 13. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Parnas (1972)** | "On the Criteria To Be Used in Decomposing Systems into Modules" compared two decompositions of the same program and showed that hiding design decisions beats splitting by processing steps. | Still the best single idea about modularity. |
| **Amazon's API mandate (~2002)** | As retold by Steve Yegge (2011), Jeff Bezos required all teams to expose their data and functionality **only** through service interfaces, with no direct database access and no back doors. | Hard boundaries and data ownership enabled independent teams and, later, AWS. |
| **Shopify (2019–2020)** | Instead of splitting its huge Rails monolith into microservices, Shopify reorganised it into components with enforced boundaries, and built **Packwerk** to check dependencies. | A modular monolith can scale to a very large codebase and team. |
| **Segment (2018)** | Segment's "Goodbye Microservices" post described moving from 100+ microservices (one per destination integration) back to a single service, because operating and updating them all had become the bottleneck. | Splitting too fine multiplies operational cost without adding real independence. |
| **Amazon Prime Video (2023)** | A monitoring service built from serverless functions and step-function orchestration was consolidated into a single process, cutting infrastructure cost by about 90%. | Network boundaries have a cost; the right granularity depends on the workload. |
| **Uber DOMA (2020)** | With around 2,000+ microservices, Uber grouped them into **domains** with gateways and clear layers ("Domain-Oriented Microservice Architecture"). | At scale you need modularity *above* the service level too. |

---

## 14. Checklist

- [ ] Modules organised by business capability (package by feature), each with a one-sentence purpose
- [ ] Each module has a small, explicit public interface; internals not importable
- [ ] Each module owns its data; no cross-module table access
- [ ] No dependency cycles; dependencies point toward stable modules
- [ ] Boundary rules enforced in CI
- [ ] Change coupling reviewed periodically from Git history
- [ ] Owners for every module
- [ ] Services extracted only for clear reasons, along existing module boundaries

---

## 15. Exercises

1. **Connascence.** Find 5 examples of connascence in code you wrote (meaning, position, execution order...). Rewrite each into a weaker form.
2. **Parnas.** Take a small pipeline (read a CSV of orders → validate → compute totals → write a report). Decompose it by processing steps, then by design decisions likely to change (file format, tax rules, report format). Which handles "switch from CSV to JSON" better?
3. **Instability.** For 5 modules of a project, count Ca and Ce and compute I. Does any stable module depend on an unstable one?
4. **Boundaries.** For an online bookstore, list the modules you'd create, what data each owns, and which events they publish.
5. **Shared table.** Reports read `orders` and `payments` tables directly. Propose two ways to remove that coupling and compare them.
6. **Change coupling.** Run `git log --name-only` on a repo you know and find the 5 file pairs that change together most often. Are they in the same module?
7. **Extract or not?** Your notifications module sends 2M emails/day and keeps getting slowed down by the main app's deploys. Should it become a service? What would you check first?

---

## 16. Further reading

- David Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (1972)
- John Ousterhout, *A Philosophy of Software Design* (deep modules, information hiding)
- Robert C. Martin, *Clean Architecture* (component principles, stability metrics)
- Meilir Page-Jones, *What Every Programmer Should Know About Object-Oriented Design* (connascence); connascence.io
- Eric Evans, *Domain-Driven Design*; Vaughn Vernon, *Implementing Domain-Driven Design*
- Simon Brown, "Modular monoliths" talk and the C4 model
- Shopify Engineering: "Deconstructing the Monolith" and "Enforcing Modularity in Rails Apps with Packwerk"
- Adam Tornhill, *Your Code as a Crime Scene* / *Software Design X-Rays* (change coupling, hotspots)
- Sam Newman, *Monolith to Microservices*
