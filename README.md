# System Design — Quality Attributes

> **Quality attributes** (a.k.a. **non-functional requirements / NFRs**, or "-ilities") describe *how well* a system does its job, as opposed to *what* it does.
> Main reference standard: **ISO/IEC 25010** (2023 revision).

Detailed notes:

- [Availability](runtime-operational/availability/README.md)
- [Reliability](runtime-operational/reliability/README.md)
- [Scalability](runtime-operational/scalability/README.md)
- [Performance](runtime-operational/performance/README.md)
- [Maintainability](development-evolution/maintainability/README.md)

---

## Table of contents

1. [Runtime / Operational](#1-runtime--operational)
2. [Security](#2-security)
3. [Development / Evolution](#3-development--evolution)
4. [Operations / DevOps](#4-operations--devops)
5. [Portability / Compatibility](#5-portability--compatibility)
6. [UI / UX](#6-ui--ux)
7. [Business / Project](#7-business--project)
8. [Safety (critical systems)](#8-safety-critical-systems)
9. [Common trade-offs](#9-common-trade-offs)
10. [Priorities for large-scale products](#10-priorities-for-large-scale-products)

---

## 1. Runtime / Operational

How the system behaves while running.

| Attribute | Meaning | Typical metric |
|---|---|---|
| [**Availability**](runtime-operational/availability/README.md) | System is up and reachable | Uptime % ("nines"), SLA/SLO |
| [**Reliability**](runtime-operational/reliability/README.md) | Works correctly over time without failure | MTBF, error rate |
| **Fault tolerance / Resilience** | Keeps working when parts fail | Graceful degradation, failover time |
| **Recoverability** | Recovers after failure | RTO, RPO |
| **Durability** | Data isn't lost once written | "11 nines" (S3), replication factor |
| [**Performance (Latency)**](runtime-operational/performance/README.md) | How fast a single request is | p50 / p95 / p99 latency |
| **Throughput** | How much work per unit of time | QPS/RPS, TPS |
| [**Scalability**](runtime-operational/scalability/README.md) | Handles growth by adding resources | Horizontal vs vertical, linear scaling |
| **Elasticity** | Scales up *and down* automatically with load | Auto-scaling reaction time |
| **Efficiency / Resource utilization** | Work done per CPU / memory / network unit | CPU %, memory footprint |
| **Capacity** | Max load before degradation | Peak concurrent users |
| **Consistency** | All nodes see the same data | Strong vs eventual (CAP / PACELC) |
| **Concurrency** | Handles simultaneous operations safely | Lock contention, race freedom |

## 2. Security

| Attribute | Meaning |
|---|---|
| **Confidentiality** | Only authorized parties can read data |
| **Integrity** | Data isn't tampered with |
| **Authenticity / Authentication** | Identities are verified |
| **Authorization** | Users can only do what they're allowed to do |
| **Accountability / Auditability** | Actions are traceable to who did them (audit logs) |
| **Non-repudiation** | Users can't deny actions they performed |
| **Privacy** | Personal data handled properly (GDPR, etc.) |
| **Compliance** | Meets legal / regulatory rules (HIPAA, PCI-DSS, SOC 2) |

## 3. Development / Evolution

How easy the system is to change.

| Attribute | Meaning |
|---|---|
| [**Maintainability**](development-evolution/maintainability/README.md) | Easy to fix and change (umbrella term for the rows below) |
| **Modularity** | Split into independent components |
| **Reusability** | Components can be reused elsewhere |
| **Analyzability / Readability** | Easy to understand and diagnose |
| **Modifiability** | Changes don't cause ripple effects |
| **Testability** | Easy to write and run tests: isolation, mocking, determinism |
| **Extensibility** | New features can be added without rewriting |
| **Simplicity** | Avoids unnecessary complexity |
| **Evolvability** | Architecture can adapt to new requirements over years |

## 4. Operations / DevOps

How easy the system is to run.

| Attribute | Meaning |
|---|---|
| **Observability** | Internal state is visible through logs, metrics, traces |
| **Monitorability** | Health can be watched and alerted on |
| **Debuggability** | Production problems can be investigated |
| **Deployability** | Easy and safe to release (CI/CD, rollback, blue-green) |
| **Configurability** | Behavior can change without code changes |
| **Manageability / Administrability** | Easy for ops teams to run |
| **Supportability / Serviceability** | Easy to support users and fix issues |

## 5. Portability / Compatibility

| Attribute | Meaning |
|---|---|
| **Portability** | Runs on different platforms / clouds |
| **Installability** | Easy to install and set up |
| **Adaptability** | Adjusts to different environments |
| **Replaceability** | Components can be swapped out (e.g., changing DB vendor) |
| **Interoperability** | Works with other systems via APIs / standards |
| **Compatibility / Co-existence** | Shares an environment with other software without conflict |
| **Backward compatibility** | New versions don't break old clients |

## 6. UI / UX

Qualities from the user's perspective.

| Attribute | Meaning |
|---|---|
| **Usability** | Easy to use (umbrella term for the rows below) |
| **Learnability** | New users get productive quickly |
| **Operability** | Easy to control and operate |
| **Efficiency of use** | Experienced users finish tasks quickly |
| **Memorability** | Users remember how to use it after a break |
| **User error protection** | Prevents mistakes (confirmations, undo) |
| **Accessibility (a11y)** | Usable by people with disabilities (WCAG) |
| **Aesthetics / UI appeal** | Visually pleasing and consistent |
| **Responsiveness (perceived performance)** | Feels fast (loading states, optimistic UI) |
| **Internationalization / Localization (i18n / l10n)** | Supports multiple languages and locales |
| **Consistency** | Uniform patterns across the app |
| **Satisfaction** | Users enjoy it (NPS, CSAT) |

## 7. Business / Project

| Attribute | Meaning |
|---|---|
| **Cost efficiency** | Low infrastructure and operating cost (TCO) |
| **Time to market** | How fast features can ship |
| **Projected lifetime** | How long the system stays viable |
| **Vendor independence** | Avoids lock-in |
| **Legal / Licensing** | Open-source license compliance |
| **Sustainability** | Energy usage, green computing |

## 8. Safety (critical systems)

| Attribute | Meaning |
|---|---|
| **Safety** | Doesn't cause harm to people or the environment (medical, automotive) |
| **Fail-safe** | Fails into a safe state |
| **Hazard warning** | Alerts users to dangerous conditions |

---

## 9. Common trade-offs

Quality attributes often conflict with each other:

- **Consistency vs Availability**: the CAP theorem
- **Security vs Usability**: more checks mean more friction
- **Performance vs Maintainability**: heavy optimization makes code harder to read
- **Cost vs Availability**: redundancy costs money

---

## 10. Priorities for large-scale products

All of these attributes matter, but at large scale some matter more than others. Failures there hit millions of users at once and cost the most money.

### Tier 1: Must-have (failure here means a direct business loss)

| # | Attribute | Why it's critical at scale |
|---|---|---|
| 1 | **Availability** | Downtime means lost revenue right away. Amazon has estimated losing millions per minute of outage. |
| 2 | **Scalability** | Traffic grows and spikes (Black Friday, viral events). If the system can't scale, it collapses. |
| 3 | **Reliability / Fault tolerance** | At scale, failure is normal: with thousands of servers, some disk, node or network is always broken. Design for failure. |
| 4 | **Performance (latency)** | Users leave slow products. Google found an extra 500 ms cut traffic by about 20%, and Amazon found 100 ms cost about 1% of sales. p99 matters more than the average. |
| 5 | **Security** | One breach can expose millions of users, bring legal penalties and destroy trust. |
| 6 | **Durability** | Losing user data (photos, payments, messages) is often unforgivable and can't be recovered. |

### Tier 2: Make scale possible (without these, Tier 1 can't be maintained)

| # | Attribute | Why |
|---|---|---|
| 7 | **Observability** | You can't fix what you can't see. With hundreds of microservices, logs, metrics and traces are how you find problems at all. |
| 8 | **Maintainability / Modularity** | Hundreds of engineers work on the same system, so code has to be changeable without breaking other teams. |
| 9 | **Deployability** | Big companies deploy thousands of times a day, so safe, automated releases with fast rollback are essential. |
| 10 | **Consistency (chosen per use case)** | Not "always strong". The point is choosing correctly: payments need strong consistency, while likes and view counts can be eventually consistent. |
| 11 | **Cost efficiency** | At scale, a 10% saving can be worth millions per year, and infrastructure is a major expense. |

### Tier 3: Important, but they depend on the domain

- **Usability / Accessibility**: critical for consumer apps, less so for internal platforms. Accessibility is often legally required.
- **Testability**: supports maintainability and deployability.
- **Compliance / Privacy**: critical in finance, healthcare and the EU (GDPR).
- **Interoperability / Backward compatibility**: critical for API and platform products such as Stripe and AWS.
- **Localization (i18n)**: critical for global products.

### The priority shifts with the product type

| Product type | Top priorities |
|---|---|
| **Banking / Payments** | Consistency, Security, Durability, Compliance |
| **Social media** | Availability, Scalability, Latency (eventual consistency is fine) |
| **Streaming (Netflix, YouTube)** | Availability, Throughput, Latency (CDN) |
| **E-commerce** | Availability, Latency, Consistency (inventory and orders) |
| **Messaging (WhatsApp)** | Reliability (no lost messages), Latency, Security (E2E encryption) |
| **Healthcare** | Safety, Privacy, Compliance, Reliability |
| **SaaS / Developer platforms** | Availability, Backward compatibility, Security |

### Key takeaway

If you remember only five: **Availability, Scalability, Reliability, Performance, Security**. Almost every large-scale system design discussion revolves around these. Teams usually trade away **strong consistency** first, accepting eventual consistency to gain availability and scalability.
