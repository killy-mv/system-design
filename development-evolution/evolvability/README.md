# Evolvability

> **Evolvability** = the system's architecture can **adapt to changes over years**: new requirements, more users, new team structures, new technologies, new regulations, without a rewrite.
> "In three years the business, the team and the technology will all be different. Will this system bend, or break?"

Related: [Maintainability](../maintainability/README.md) (Kleppmann's three principles: operability, simplicity, **evolvability**), [Maintainability §16](../maintainability/README.md#16-evolving-legacy-systems) (evolving legacy systems), [Modularity](../modularity/README.md), [Modifiability](../modifiability/README.md) (the day-to-day, code-level version), [Deployability](../../operations-devops/deployability/README.md) (compatibility and migrations during rollouts).

---

## Table of contents

1. [Evolvability vs modifiability](#1-evolvability-vs-modifiability)
2. [What changes over a system's life](#2-what-changes-over-a-systems-life)
3. [Principles](#3-principles)
4. [Reversible and irreversible decisions](#4-reversible-and-irreversible-decisions)
5. [Architecture fitness functions](#5-architecture-fitness-functions)
6. [Evolving interfaces and data](#6-evolving-interfaces-and-data)
7. [Migration patterns](#7-migration-patterns)
8. [Recording decisions](#8-recording-decisions)
9. [Technology lifecycle](#9-technology-lifecycle)
10. [Teams and architecture evolve together](#10-teams-and-architecture-evolve-together)
11. [Approaches by system type and stage](#11-approaches-by-system-type-and-stage)
12. [Trade-offs](#12-trade-offs)
13. [Anti-patterns](#13-anti-patterns)
14. [Real-world stories](#14-real-world-stories)
15. [Checklist](#15-checklist)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Evolvability vs modifiability

| | Modifiability | Evolvability |
|---|---|---|
| Scope | Code: a function, a module | Architecture: data, interfaces, deployment, technology, teams |
| Time scale | Days to weeks | Months to years |
| Typical change | Add a field, a payment method | Go multi-region, split a monolith, replace the database, support a new market, re-org teams |
| Main tools | Clean code, modularity, tests | Boundaries, contracts, migration patterns, fitness functions, decision records |

**Evolutionary architecture** (Ford, Parsons, Kua): an architecture that supports **guided, incremental change across multiple dimensions**. Guided = automated checks keep the important properties intact while it changes. Incremental = small steps, each one deployable and reversible.

---

## 2. What changes over a system's life

| Change | Example | Typical horizon |
|---|---|---|
| **Business requirements** | New pricing model, new product line, marketplace instead of a shop | Months |
| **Scale** | 1k → 1M users; 10 GB → 10 TB | 1–3 years |
| **Team and organisation** | 4 engineers → 80 across 10 teams | 1–3 years |
| **Markets and regulation** | GDPR, data residency, accessibility laws, a new country's tax rules | Months–years |
| **Integrations** | New payment providers, partners, a public API | Months |
| **Technology** | Framework end of life, language versions, cloud services deprecated | 2–5 years |
| **Data** | New entities, changed meanings, new analytics needs | Continuous |

You can't predict **which** changes come. You can predict that **many will**. So the goal isn't to build in every possible flexibility (that's premature generality, see [Simplicity](../simplicity/README.md)); it's to make change **cheap and safe in general**.

---

## 3. Principles

- **Isolate volatility.** Put things likely to change (vendors, pricing rules, data formats, external APIs) behind stable interfaces. This is Parnas's information hiding at the architecture level ([Modularity §3](../modularity/README.md#3-information-hiding-and-deep-modules)).
- **Stable contracts, flexible internals.** Interfaces between teams and systems change slowly and compatibly; implementations change freely.
- **Data outlives code.** Code gets rewritten; data stays for decades. Invest most care in the data model, IDs, formats and ownership.
- **Small, reversible steps.** Every step of a migration should be deployable on its own and able to be rolled back.
- **Replaceability over reusability.** Code that's easy to delete is easier to evolve than code that's reused everywhere ("Write code that is easy to delete, not easy to extend").
- **Last responsible moment.** Make irreversible decisions as late as you safely can, when you know the most.
- **Automate the guardrails** (fitness functions, §5) so the architecture doesn't erode while it changes.

---

## 4. Reversible and irreversible decisions

Jeff Bezos's **one-way vs two-way doors**: most decisions are two-way doors (easy to undo): make them quickly. A few are one-way doors (very costly to undo): slow down, research, prototype.

| Usually two-way doors | Usually one-way doors (or close to it) |
|---|---|
| Library choice inside one module | Public API shape and semantics |
| Internal module structure | ID formats and sizes, primary keys |
| Cache strategy | Data model of core entities; what you store at all |
| Logging format | Choice of primary database for core data (expensive, not impossible) |
| Framework for a new internal service | Event formats consumed by many teams |
| Instance types, scaling settings | Contracts with customers (SLAs, data retention promises) |

### Worked example: the 32-bit ID

A table uses a signed 32-bit integer primary key (`INT`, max **2,147,483,647**). It was a fine choice at launch.

```
Launch: 50,000 rows/day → would take ~117 years to fill up
Year 3: the product took off: 4,000,000 rows/day
Remaining IDs: ~2.1B − 1.0B used ≈ 1.1B → full in ~275 days
```

When it fills, **inserts fail**: the feature (often the core one) is down. Changing the column to `BIGINT` on a billion-row table means rewriting the table and every foreign key and index that references it, plus every service, cache and client that assumed a 32-bit number. Done under pressure, that's days of risky work. Done early, it's a migration you plan.

Basecamp hit exactly this in 2018: an events table reached the 32-bit limit and part of Basecamp 3 was read-only for hours. Lessons:
- For core tables, **use 64-bit IDs from the start** (or UUIDv7/ULID). The cost is a few bytes per row.
- **Monitor sequences** ("ID usage > 50% of max") like any capacity limit ([Capacity](../../runtime-operational/capacity/README.md)).
- Treat data formats as one-way doors and decide them deliberately.

---

## 5. Architecture fitness functions

A **fitness function** is an automated check that an important architectural property still holds. It turns "we should keep X" into "CI fails if X breaks".

| Property | Fitness function |
|---|---|
| Module boundaries | Architecture tests (import-linter, ArchUnit, dependency-cruiser): no forbidden imports, no cycles |
| API compatibility | Breaking-change detection on OpenAPI (oasdiff) or Protobuf (`buf breaking`) in CI |
| Event compatibility | Schema registry compatibility checks |
| Performance | k6 thresholds (`p(95) < 300ms`) on each PR or nightly |
| Frontend weight | Bundle size budgets (size-limit, Lighthouse CI) |
| Security | Dependency vulnerability scans, no secrets in code, container policies |
| Operability | Every service has health checks, dashboards and an owner (service catalog checks) |
| Data | Migrations must be backward compatible (lint rules for dangerous DDL, e.g. squawk for Postgres) |
| Cost | Cost per request tracked; alerts on regressions |

Start with 3–5 that protect what matters most. Each one makes future change **safer**, because you'll know immediately if a change breaks a property you care about.

---

## 6. Evolving interfaces and data

Interfaces and data are where evolution is hardest, because other people and systems depend on them.

- **APIs**: additive changes, tolerant readers, deprecation with `Sunset` headers, versioning only for real breaking changes ([Maintainability §6](../maintainability/README.md#6-apis-and-contracts)).
- **Events**: schema registry, compatibility rules, never reuse field names/numbers, consumers ignore unknown fields.
- **Database schemas**: expand–contract migrations ([Deployability §9](../../operations-devops/deployability/README.md#9-database-and-schema-changes)), online schema changes, backfills.
- **Data meaning**: when a concept changes ("customer" now includes businesses), introduce the new concept explicitly rather than silently changing what a column means.
- **Version translation layers**: like Stripe, keep the core on the latest model and translate requests/responses for older API versions at the edge, so old clients keep working without forking the code.

---

## 7. Migration patterns

Big changes should happen **incrementally**, with the system running and a rollback path at every step.

| Pattern | Idea | Use for |
|---|---|---|
| **Strangler fig** | Put a façade/proxy in front of the old system; route one capability at a time to the new one; retire the old piece by piece | Replacing a legacy system or monolith |
| **Branch by abstraction** | Introduce an interface over the thing to replace; build the new implementation behind it; switch with a flag; delete the old | Replacing a library, ORM, data store or vendor inside a codebase |
| **Parallel run / Scientist** | Run old and new on the same inputs; use the old result; log differences | Rewrites where correctness is critical (pricing, permissions, billing) |
| **Expand–contract** | Add the new, migrate users/data, remove the old | Schemas, APIs, events |
| **Dual write / CDC + backfill** | Keep the new data store in sync, backfill history, verify, switch reads, then writes | Database migrations |
| **Dark launch** | Run the new path in production without showing results | Validating performance and correctness under real load |
| **Anticorruption layer** | Translate between the legacy model and the new one | Stopping legacy concepts from leaking into new code |

### Worked example: strangling a billing module

A monolith's billing code needs to move to a new service (new pricing model, separate team).

```
Step 1  Put a routing layer in front of billing calls (inside the monolith: a BillingPort interface)
Step 2  Build the new billing service; run it in shadow mode: the monolith calls both,
        uses the old result, logs differences                         (weeks: fix mismatches)
Step 3  Route 1% of customers' invoices to the new service (flag per customer), compare totals
Step 4  10% → 50% → 100%, with automatic fallback to the old path on errors
Step 5  Migrate historical invoices (backfill), verify totals match per customer
Step 6  Remove the old billing code and the shadow comparison
```

Every step is deployable on its own and reversible (flip the flag back). Compare with a big-bang cut-over: months of building in isolation, one risky weekend migration, and no way back if totals are wrong on Monday.

---

## 8. Recording decisions

Future engineers (including you) need to know **why** things are the way they are, or they can't safely change them.

- **Architecture Decision Records (ADRs)**: short documents (context, decision, consequences, status), stored in the repo next to the code. Templates: Michael Nygard's original, MADR.
- **RFCs / design docs** for larger changes, reviewed before building.
- **Tech radar**: which technologies to adopt, trial, assess or hold.
- **Revisit decisions**: an ADR can be superseded by a newer one; that's the point.

---

## 9. Technology lifecycle

Every framework, language version, OS and managed service eventually reaches **end of life**: Python 2 (2020), AngularJS (2021), CentOS Linux 8 (2021), old Node.js and database versions every year.

- **Upgrade continuously**: small regular upgrades (Renovate/Dependabot + good tests) are far cheaper than a forced multi-year jump.
- **Track versions and EOL dates** in a service catalog; plan upgrades before support ends.
- **Abstract over a vendor only where volatility is real** (payments, email, LLM providers), not over everything "just in case".
- **Avoid deep coupling to framework internals**; keep business logic framework-independent where practical.

---

## 10. Teams and architecture evolve together

- **Conway's law**: systems mirror the communication structure of the organisation that builds them.
- **Inverse Conway manoeuvre**: shape teams to match the architecture you want.
- When teams grow or re-organise, module and service boundaries need to follow, or ownership blurs and the architecture erodes ([Maintainability §14](../maintainability/README.md#14-teams-and-organisation)).
- **Team cognitive load** limits how much of the system one team can own and evolve well (*Team Topologies*).

---

## 11. Approaches by system type and stage

| Context | Evolvability focus |
|---|---|
| **Startup / MVP** | Optimise for learning; keep it simple and modular; protect only the one-way doors (data model, IDs, public API); easy to throw parts away |
| **Growing product** | Modular monolith, ADRs, a few fitness functions, continuous dependency upgrades, expand–contract discipline |
| **Large organisation** | Bounded contexts, contracts between teams, platform golden paths, migration patterns as standard practice, tech radar |
| **Public APIs / platforms** | Backward compatibility above all; version translation layers; long deprecation periods; usage analytics before removal |
| **Data platforms** | Data contracts, schema evolution rules, lineage, versioned datasets |
| **Mobile apps** | Old app versions live for years: server-driven UI/config, API compatibility windows, minimum-version enforcement |
| **Embedded / IoT** | Remote update capability designed in from day one; protocol versioning; hardware lives 10+ years |
| **Legacy systems** | Characterisation tests, strangler fig, anticorruption layers; avoid big-bang rewrites |

---

## 12. Trade-offs

| Choice | Cost |
|---|---|
| Abstractions for volatile parts | Indirection; wasted if the change never comes |
| Strict compatibility | Slower API evolution; old code paths kept alive longer |
| Incremental migrations | Longer total duration; two systems running in parallel for a while |
| Big-bang rewrite | Faster on paper; very high risk, long period with no user value |
| Fitness functions | Setup and maintenance; occasional false positives |
| Continuous upgrades | Constant small effort vs rare huge, risky effort |
| 64-bit IDs / richer data models up front | Slightly more storage; avoids painful migrations later |

---

## 13. Anti-patterns

- **Big-bang rewrite** of a working system, with no incremental path or rollback.
- **Speculative generality**: plugin systems and abstractions for changes that never come.
- **Frozen architecture**: decisions nobody dares to revisit because nobody remembers why they were made.
- **Version pinning forever**, then a forced emergency upgrade at end of life.
- **Breaking changes without deprecation** in APIs or events.
- **Changing the meaning** of existing data fields silently.
- **Architecture only on slides**: no fitness functions, so the real system drifts.
- **Two systems forever**: a migration started but never finished, so both must be maintained.

---

## 14. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Netscape (late 1990s)** | Netscape decided to rewrite its browser from scratch. The rewrite took about three years with no major release in between, while competitors moved ahead. Joel Spolsky's essay "Things You Should Never Do" (2000) made this the classic warning. | Big-bang rewrites throw away years of accumulated fixes and stop delivery. |
| **Amazon (early 2000s)** | Amazon moved from a large monolith (Obidos) to services owned by small teams, step by step, over several years. | Large architecture evolution is possible incrementally, aligned with team structure. |
| **Stripe API versioning (2017 blog)** | Stripe pins each account to an API version and keeps the core on the newest model, translating responses for older versions through small "version change" modules. Old integrations from years ago keep working. | A translation layer lets you evolve while keeping strict backward compatibility. |
| **Basecamp (Nov 2018)** | An events table hit the 32-bit integer ID limit, and parts of Basecamp 3 became read-only for several hours while the column was migrated. | ID sizes are one-way doors; monitor limits before they hit. |
| **Python 2 → 3 (2008–2020)** | The incompatible Python 3 release took the ecosystem about 12 years to migrate; Python 2 reached end of life in 2020. Companies like Dropbox described multi-year migrations. | Breaking changes in foundations cost the whole ecosystem; incremental compatibility tools matter. |
| **Shopify** | Instead of rewriting into microservices, Shopify evolved its Rails monolith into a modular monolith with enforced component boundaries. | You can evolve the architecture you have rather than replacing it. |

---

## 15. Checklist

- [ ] One-way-door decisions (data model, IDs, public APIs, event formats) made deliberately and documented
- [ ] Volatile dependencies (vendors, external APIs) isolated behind interfaces
- [ ] APIs and events evolved compatibly, with deprecation policies
- [ ] Migrations done incrementally with a rollback path at every step
- [ ] ADRs for significant decisions, stored with the code
- [ ] 3–5 fitness functions running in CI
- [ ] Dependencies upgraded continuously; EOL dates tracked
- [ ] Capacity limits of data formats (sequences, ID sizes) monitored

---

## 16. Exercises

1. **One-way doors.** List 10 decisions in a project you built. Mark each as one-way or two-way. Were any one-way doors made casually?
2. **ID math.** Your `orders` table uses `INT` IDs, has 600M rows, and grows by 1.5M/day. When does it run out? Plan the migration to `BIGINT` without downtime.
3. **Fitness functions.** Choose 4 architectural properties of a system you know and write how you'd check each automatically in CI.
4. **Strangler fig.** Plan the incremental replacement of a monolith's search feature (SQL `LIKE` queries) with a search engine. What are the steps, and how do you roll back at each?
5. **API evolution.** A public API returns `price` as a float in dollars. You need integer cents and multiple currencies. Design the change so existing clients keep working.
6. **Write an ADR** for a real decision you made in one of the labs (e.g. "Use a modular monolith instead of microservices"), with context, decision, consequences.
7. **Rewrite or evolve?** A team wants to rewrite a 7-year-old app "because the code is bad". List the questions you'd ask before agreeing, and an incremental alternative.

---

## 17. Further reading

- *Building Evolutionary Architectures* (Neal Ford, Rebecca Parsons, Patrick Kua, Pramod Sadalage), 2nd ed.: fitness functions
- *Monolith to Microservices* (Sam Newman): strangler fig, branch by abstraction, data migration patterns
- *Working Effectively with Legacy Code* (Michael Feathers)
- Martin Fowler: "StranglerFigApplication", "BranchByAbstraction", "ParallelChange"
- Joel Spolsky, "Things You Should Never Do, Part I" (2000)
- Stripe blog: "APIs as infrastructure: future-proofing Stripe with versioning" (2017)
- tef, "Write code that is easy to delete, not easy to extend" (programmingisterrible.com, 2016)
- Michael Nygard, "Documenting Architecture Decisions" (2011); adr.github.io
- *Team Topologies* (Skelton, Pais)
- GitHub's Scientist library (parallel run / experiment pattern)
