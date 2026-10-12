# Simplicity

> **Simplicity** = the system has **no more complexity than the problem requires**: few moving parts, few concepts to learn, few ways for things to interact unexpectedly.
> "Can a new engineer understand this part well enough to change it safely in their first week?"

Related: [Maintainability §2](../maintainability/README.md#2-simplicity-managing-complexity) (short version: principles), [Modularity](../modularity/README.md) (deep modules hide complexity), [Analyzability](../analyzability/README.md) (understanding code), [Evolvability](../evolvability/README.md).

---

## Table of contents

1. [Why simplicity matters](#1-why-simplicity-matters)
2. [Essential vs accidental complexity](#2-essential-vs-accidental-complexity)
3. [Simple is not the same as easy](#3-simple-is-not-the-same-as-easy)
4. [Where complexity comes from](#4-where-complexity-comes-from)
5. [Simplicity in code](#5-simplicity-in-code)
6. [Simplicity in system design](#6-simplicity-in-system-design)
7. [The cost of adding a component](#7-the-cost-of-adding-a-component)
8. [Measuring complexity](#8-measuring-complexity)
9. [Approaches by system type and stage](#9-approaches-by-system-type-and-stage)
10. [Trade-offs](#10-trade-offs)
11. [Anti-patterns](#11-anti-patterns)
12. [Real-world stories](#12-real-world-stories)
13. [Checklist](#13-checklist)
14. [Exercises](#14-exercises)
15. [Further reading](#15-further-reading)

---

## 1. Why simplicity matters

> "There are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies, and the other way is to make it so complicated that there are no obvious deficiencies." — Tony Hoare

Complexity is the main reason software gets **slower to change** over time. It shows up as (Ousterhout):
- **Change amplification**: a simple change needs edits in many places.
- **Cognitive load**: you must hold a lot in your head to change anything.
- **Unknown unknowns**: you can't even tell what you need to know. The worst one.

It also hurts every runtime attribute: more components = more failure modes ([Availability](../../runtime-operational/availability/README.md)), more interactions = more bugs ([Reliability](../../runtime-operational/reliability/README.md)), more systems = more to secure and operate.

Complexity is **incremental**: no single change makes a system complex. Hundreds of small "it's just one more flag / one more service / one more special case" decisions do. So simplicity is a habit applied to every change, not a one-time cleanup.

---

## 2. Essential vs accidental complexity

From Fred Brooks, "No Silver Bullet" (1986):
- **Essential complexity** comes from the problem itself: tax rules, payment states, regulations, real-world edge cases. You can't remove it, only organise it well.
- **Accidental complexity** comes from our solution: unnecessary layers, tools, abstractions, inconsistent conventions, premature distribution, workarounds for earlier workarounds.

Goal: **eliminate accidental complexity**, and **contain essential complexity** in well-named, deep modules so most of the code doesn't need to know about it.

---

## 3. Simple is not the same as easy

From Rich Hickey's talk "Simple Made Easy" (2011):

| | Simple | Easy |
|---|---|---|
| Opposite | Complex (intertwined, "complected") | Hard (unfamiliar, far from what I know) |
| About | The **thing**: how many concerns are tangled together | **Me**: how familiar or close at hand it is |
| Example | A pure function with clear inputs/outputs | A framework that does magic in one line |

"Easy" tools often bring hidden complexity (magic, implicit behaviour, global state). "Simple" things may take more effort to learn but are easier to reason about and change for years. Choose **simple**, and invest in making simple things easy (docs, templates, tooling).

Also: **simple ≠ less code**. A clever one-liner can be complex; three explicit lines can be simple.

---

## 4. Where complexity comes from

| Source | Example |
|---|---|
| **Dependencies between parts** | Changing one module forces understanding five others |
| **Obscurity** | Important information isn't obvious: unclear names, hidden side effects, implicit ordering |
| **State** | Mutable shared state, caches, copies of the same data in several places |
| **Special cases** | `if customer == "ACME"`, flags for one-off behaviours |
| **Configurability** | Every option doubles the possible behaviours |
| **Premature generality** | Frameworks and plugin systems for one use case |
| **Too many technologies** | Five languages, four databases, three queues |
| **Distribution** | Network calls where function calls would do |
| **Time and concurrency** | Async flows, retries, ordering, races |
| **Organisation** | Systems mirror team structure (Conway's law); unclear ownership |
| **Legacy layers** | Workarounds that outlived the problem they solved |

---

## 5. Simplicity in code

- **Deep modules**: simple interfaces over powerful implementations ([Modularity §3](../modularity/README.md#3-information-hiding-and-deep-modules)).
- **Pull complexity downward**: a module should handle hard cases itself instead of pushing them to every caller.
- **Define errors out of existence**: design APIs so error cases can't happen or don't matter (e.g. "delete if exists" instead of "delete, error if missing"; return an empty list instead of `null`).
- **Make illegal states unrepresentable** with types and state machines.
- **Explicit over implicit**: no hidden global state, magic or surprising side effects.
- **Pure functions and immutability** where practical: easier to reason about and test.
- **Consistency**: one way to do logging, errors, config, naming across the codebase.
- **Good names**: a name that needs a comment is usually the wrong name.
- **Delete code**: dead code, unused features, stale flags and old workarounds are pure cost.
- **Avoid cleverness**: optimise for the reader, not the writer.

### Worked example: booleans vs states

An order modelled with flags:

```python
class Order:
    is_paid: bool
    is_shipped: bool
    is_cancelled: bool
    is_refunded: bool
```

4 booleans = **2⁴ = 16** possible combinations. Only about 5 make sense. The other 11 (shipped but not paid, cancelled and shipped, refunded but never paid...) are bugs waiting to happen, and every function must defend against them.

```python
class OrderStatus(Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"
    REFUNDED = "refunded"

S = OrderStatus
ALLOWED = {
    S.PENDING: {S.PAID, S.CANCELLED},
    S.PAID:    {S.SHIPPED, S.REFUNDED},
    S.SHIPPED: {S.REFUNDED},
}

def transition(order, new_status):
    if new_status not in ALLOWED.get(order.status, set()):
        raise InvalidTransition(order.status, new_status)
    order.status = new_status
```

**5 states, explicit transitions**, invalid combinations impossible. Less code in every caller, fewer tests needed, and the business rules are readable in one place.

The same math applies to configuration: 6 independent boolean options = 64 behaviours to test. Replacing them with 3 named modes = 3 behaviours.

---

## 6. Simplicity in system design

- **Fewer moving parts.** Every database, queue, cache, language and service needs deploys, monitoring, backups, upgrades, security patches and on-call knowledge. Before adding one, ask: can what we already have do it? (Postgres can handle job queues with `SKIP LOCKED`, full-text search, JSON documents, pub/sub with `LISTEN/NOTIFY` and scheduled jobs, up to a point.)
- **Choose boring technology** (Dan McKinley): a team has a few **innovation tokens**. Spend them on what differentiates your product, not on the plumbing.
- **Monolith first**; distribute only when there's a clear reason ([Modularity §8](../modularity/README.md#8-modules-vs-services)).
- **Managed services** remove whole categories of operational complexity (at a cost).
- **One source of truth** per piece of data; fewer copies = fewer consistency problems.
- **Synchronous and obvious** flows where performance allows; async where it's truly needed.
- **Golden paths**: one standard way to build, deploy and observe a service.

---

## 7. The cost of adding a component

### Worked example: a startup's architecture

A product with **3,000 users** and **4 engineers** has:

```
6 microservices · Kubernetes · PostgreSQL · MongoDB · Redis · Elasticsearch · Kafka · RabbitMQ
```

Each stateful component needs, at minimum: provisioning, backups and restore tests, upgrades, monitoring and alerts, security patching, failover understanding, local dev setup, and someone on call who understands it.

| | Components to operate | Datastores to back up and upgrade | Things a new engineer must learn |
|---|---|---|---|
| Current | 6 services + k8s + 6 data systems ≈ **13** | 6 | ~15 technologies |
| Simplified | 1 modular monolith + PostgreSQL + a managed queue + Redis ≈ **4** | 1–2 | ~5 technologies |

The simplified version easily handles 3,000 users (and, done well, 100× that: see [Scalability §21](../../runtime-operational/scalability/README.md#21-real-world-scaling-stories)). The 4 engineers get most of their time back for product work. Each removed component can come back later **when a measured need appears**.

**Before adding any component, write down:** the problem it solves, the measured need, who operates it, how it fails, how it's backed up, and what it would cost to remove later.

---

## 8. Measuring complexity

No single number captures it, but useful signals:
- **Time for a new engineer** to make a first meaningful change.
- **Time to implement a typical feature**, and its trend over months.
- **Files/modules touched per change** (change amplification).
- **Number of technologies / components** in production.
- **Code metrics** (use as hints, not targets): cyclomatic and cognitive complexity per function, file size, hotspots = churn × complexity ([Analyzability](../analyzability/README.md)).
- **Incident causes**: how many come from unexpected interactions between parts?
- **Developer surveys**: "how hard is it to make changes in X?"

---

## 9. Approaches by system type and stage

| Context | Simplicity strategy |
|---|---|
| **MVP / startup** | One app, one database, a PaaS, boring tech; no microservices, no Kubernetes unless the team already knows it well |
| **Growing product** | Modular monolith, consistent conventions, a short list of approved technologies, delete unused features |
| **Large organisation** | Platform team with golden paths; limit technology sprawl (tech radar); simplify interfaces between teams |
| **Libraries / APIs** | Small surface, consistent naming, sensible defaults, few options; easy things easy, hard things possible |
| **Data platforms** | Fewer pipelines and copies; one modelling layer; clear ownership |
| **Infrastructure** | Managed services; fewer custom scripts; IaC modules with few parameters |
| **Legacy systems** | Remove dead code and unused features first; simplify before migrating |

---

## 10. Trade-offs

| Simplicity vs | Tension |
|---|---|
| **Performance** | Optimisations (caches, denormalisation, async) add complexity. Do them when measured |
| **Scalability** | Distribution scales further but is much more complex. Delay until needed |
| **Flexibility** | Configurability and extension points add complexity. Add when a second real use case appears |
| **Availability** | Redundancy adds components. Usually worth it for critical paths |
| **Short-term speed** | The quick hack is often the complex option long-term |
| **Familiarity ("easy")** | The familiar tool may bring more hidden complexity than a simpler, less familiar one |

---

## 11. Anti-patterns

- **Resume-driven development**: choosing technologies to learn them, not because the problem needs them.
- **Premature microservices** and premature generalisation ("we might need plugins someday").
- **Lasagna architecture**: many thin layers that only pass calls through.
- **Special cases everywhere** instead of fixing the model.
- **Configuration as a programming language**.
- **Copies of the same data** in many stores with sync jobs between them.
- **Never deleting anything**: features, flags, endpoints, tables.
- **Clever code** that only its author understands.

---

## 12. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **The second-system effect** | Fred Brooks (*The Mythical Man-Month*, 1975) observed that a designer's second system tends to be over-designed with every idea held back from the first. | Rewrites and "v2" architectures attract accidental complexity. |
| **WhatsApp (2014)** | At its acquisition, WhatsApp served hundreds of millions of users with a few dozen engineers, using a small, focused stack (Erlang, FreeBSD) and a minimal feature set. | A simple, well-chosen stack scales a long way, and so does a small team. |
| **"Choose Boring Technology" (2015)** | Dan McKinley's essay, from his experience at Etsy, argued that each new technology carries a long-term operational cost and that teams should spend limited "innovation tokens" carefully. | Technology choices are system-wide complexity decisions. |
| **Stack Overflow** | Served a top-100 site for years from a monolith on a small number of servers, with careful performance work instead of distribution. | Simple architecture + good engineering beats premature distribution. |
| **Amazon Prime Video (2023)** | Consolidating a distributed serverless monitoring pipeline into one process cut cost by about 90% and simplified operations. | Distribution is a cost to justify, not a default. |
| **Unix philosophy** | Small tools that do one thing well, composed with simple text interfaces, have lasted 50+ years. | Simple interfaces enable composition and longevity. |

---

## 13. Checklist

- [ ] Every component and technology has a written reason and an owner
- [ ] No component added without a measured need
- [ ] Domain states modelled explicitly (enums/state machines), not combinations of flags
- [ ] One consistent way for logging, errors, config, naming
- [ ] Special cases and one-off flags reviewed and removed regularly
- [ ] Dead code, unused features and stale flags deleted
- [ ] New engineers can make a meaningful change in their first week

---

## 14. Exercises

1. **Inventory.** List every technology and component in a project you built. For each: why it exists, who operates it, and whether something you already have could replace it.
2. **States.** Find an entity in your code with 3+ boolean flags. Count the combinations and the valid ones. Redesign it as a state machine.
3. **Errors out of existence.** Take an API you wrote that returns many error cases. Which ones can you design away?
4. **Postgres for everything?** For a job queue, full-text search, a cache and pub/sub: how far can Postgres go for each, and at what point would you add a dedicated tool?
5. **Simple vs easy.** Name one tool you use that's easy but not simple, and one that's simple but not easy. What does each cost you over time?
6. **Delete.** Find 3 things you could delete from a codebase you work on (code, flags, endpoints, tables). What stops you? Make a plan to delete one.

---

## 15. Further reading

- John Ousterhout, *A Philosophy of Software Design*: the best book on managing complexity
- Fred Brooks, "No Silver Bullet: Essence and Accident in Software Engineering" (1986); *The Mythical Man-Month*
- Rich Hickey, "Simple Made Easy" (talk, 2011)
- Dan McKinley, "Choose Boring Technology" (boringtechnology.club)
- Kent Beck's four rules of simple design (passes tests, reveals intention, no duplication, fewest elements)
- Yaron Minsky, "Make illegal states unrepresentable"
- Sandi Metz, "The Wrong Abstraction"
- Kleppmann, *Designing Data-Intensive Applications* ch. 1 (simplicity as a maintainability principle)
