# System Design — Quality Attributes

> **Quality attributes** (a.k.a. **non-functional requirements / NFRs**, or "-ilities") describe *how well* a system does its job, as opposed to *what* it does.
> Main reference standard: **ISO/IEC 25010** (2023 revision).

Detailed notes:

- [Availability](runtime-operational/availability/README.md)
- [Reliability](runtime-operational/reliability/README.md)
- [Scalability](runtime-operational/scalability/README.md)
- [Performance](runtime-operational/performance/README.md)
- [Capacity](runtime-operational/capacity/README.md)
- [Concurrency](runtime-operational/concurrency/README.md)
- [Consistency (data)](runtime-operational/consistency/README.md)
- [Durability](runtime-operational/durability/README.md)
- [Efficiency](runtime-operational/efficiency/README.md)
- [Elasticity](runtime-operational/elasticity/README.md)
- [Fault tolerance](runtime-operational/fault-tolerance/README.md)
- [Recoverability](runtime-operational/recoverability/README.md)
- [Throughput](runtime-operational/throughput/README.md)
- [Observability](operations-devops/observability/README.md)
- [Deployability](operations-devops/deployability/README.md)
- [Monitorability](operations-devops/monitorability/README.md)
- [Configurability](operations-devops/configurability/README.md)
- [Debuggability](operations-devops/debuggability/README.md)
- [Manageability](operations-devops/manageability/README.md)
- [Supportability](operations-devops/supportability/README.md)
- [Maintainability](development-evolution/maintainability/README.md)
- [Modularity](development-evolution/modularity/README.md)
- [Simplicity](development-evolution/simplicity/README.md)
- [Testability](development-evolution/testability/README.md)
- [Evolvability](development-evolution/evolvability/README.md)
- [Modifiability](development-evolution/modifiability/README.md)
- [Analyzability](development-evolution/analyzability/README.md)
- [Extensibility](development-evolution/extensibility/README.md)
- [Reusability](development-evolution/reusability/README.md)
- [Usability](UI-UX/usability/README.md)
- [Accessibility](UI-UX/accessibility/README.md)
- [Responsiveness](UI-UX/responsiveness/README.md)
- [Internationalization](UI-UX/internationalization/README.md)
- [User error protection](UI-UX/user-error-protection/README.md)
- [Consistency](UI-UX/consistency/README.md)
- [Learnability](UI-UX/learnability/README.md)
- [Memorability](UI-UX/memorability/README.md)
- [Efficiency of use](UI-UX/efficiency-of-use/README.md)
- [Satisfaction](UI-UX/satisfaction/README.md)
- [Operability](UI-UX/operability/README.md)
- [Aesthetics](UI-UX/aesthetics/README.md)
- [Confidentiality](security/confidentiality/README.md)
- [Integrity](security/integrity/README.md)
- [Authentication](security/authentication/README.md)
- [Authorization](security/authorization/README.md)
- [Auditability](security/auditability/README.md)
- [Non-repudiation](security/non-repudiation/README.md)
- [Privacy](security/privacy/README.md)
- [Compliance](security/compliance/README.md)
- [Portability](portability-compatibility/portability/README.md)
- [Installability](portability-compatibility/installability/README.md)
- [Adaptability](portability-compatibility/adaptability/README.md)
- [Replaceability](portability-compatibility/replaceability/README.md)
- [Interoperability](portability-compatibility/interoperability/README.md)
- [Co-existence](portability-compatibility/co-existence/README.md)
- [Backward compatibility](portability-compatibility/backward-compatibility/README.md)
- [Cost efficiency](business-project/cost-efficiency/README.md)
- [Time to market](business-project/time-to-market/README.md)
- [Projected lifetime](business-project/projected-lifetime/README.md)
- [Vendor independence](business-project/vendor-independence/README.md)
- [Licensing](business-project/licensing/README.md)
- [Sustainability](business-project/sustainability/README.md)
- [Safety](safety/safety/README.md)
- [Fail-safe](safety/fail-safe/README.md)
- [Hazard warning](safety/hazard-warning/README.md)
- [How search works](SEO-optimize/how-search-works/README.md)
- [Crawlability](SEO-optimize/crawlability/README.md)
- [Indexability](SEO-optimize/indexability/README.md)
- [Site architecture](SEO-optimize/site-architecture/README.md)
- [Page experience](SEO-optimize/page-experience/README.md)
- [On-page relevance](SEO-optimize/on-page-relevance/README.md)
- [Structured data](SEO-optimize/structured-data/README.md)
- [Authority (links)](SEO-optimize/authority/README.md)
- [International SEO](SEO-optimize/international-seo/README.md)
- [Local SEO](SEO-optimize/local-seo/README.md)
- [SEO monitoring](SEO-optimize/seo-monitoring/README.md)

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
9. [SEO (search engine optimization)](#9-seo-search-engine-optimization)
10. [Common trade-offs](#10-common-trade-offs)
11. [Priorities for large-scale products](#11-priorities-for-large-scale-products)

---

## 1. Runtime / Operational

How the system behaves while running.

| Attribute | Meaning | Typical metric |
|---|---|---|
| [**Availability**](runtime-operational/availability/README.md) | System is up and reachable | Uptime % ("nines"), SLA/SLO |
| [**Reliability**](runtime-operational/reliability/README.md) | Works correctly over time without failure | MTBF, error rate |
| [**Fault tolerance / Resilience**](runtime-operational/fault-tolerance/README.md) | Keeps working when parts fail | Graceful degradation, failover time |
| [**Recoverability**](runtime-operational/recoverability/README.md) | Recovers after failure | RTO, RPO |
| [**Durability**](runtime-operational/durability/README.md) | Data isn't lost once written | "11 nines" (S3), replication factor |
| [**Performance (Latency)**](runtime-operational/performance/README.md) | How fast a single request is | p50 / p95 / p99 latency |
| [**Throughput**](runtime-operational/throughput/README.md) | How much work per unit of time | QPS/RPS, TPS |
| [**Scalability**](runtime-operational/scalability/README.md) | Handles growth by adding resources | Horizontal vs vertical, linear scaling |
| [**Elasticity**](runtime-operational/elasticity/README.md) | Scales up *and down* automatically with load | Auto-scaling reaction time |
| [**Efficiency / Resource utilization**](runtime-operational/efficiency/README.md) | Work done per CPU / memory / network unit | CPU %, memory footprint |
| [**Capacity**](runtime-operational/capacity/README.md) | Max load before degradation | Peak concurrent users |
| [**Consistency**](runtime-operational/consistency/README.md) | All nodes see the same data | Strong vs eventual (CAP / PACELC) |
| [**Concurrency**](runtime-operational/concurrency/README.md) | Handles simultaneous operations safely | Lock contention, race freedom |

## 2. Security

| Attribute | Meaning |
|---|---|
| [**Confidentiality**](security/confidentiality/README.md) | Only authorized parties can read data |
| [**Integrity**](security/integrity/README.md) | Data isn't tampered with |
| [**Authenticity / Authentication**](security/authentication/README.md) | Identities are verified |
| [**Authorization**](security/authorization/README.md) | Users can only do what they're allowed to do |
| [**Accountability / Auditability**](security/auditability/README.md) | Actions are traceable to who did them (audit logs) |
| [**Non-repudiation**](security/non-repudiation/README.md) | Users can't deny actions they performed |
| [**Privacy**](security/privacy/README.md) | Personal data handled properly (GDPR, etc.) |
| [**Compliance**](security/compliance/README.md) | Meets legal / regulatory rules (HIPAA, PCI-DSS, SOC 2) |

## 3. Development / Evolution

How easy the system is to change.

| Attribute | Meaning |
|---|---|
| [**Maintainability**](development-evolution/maintainability/README.md) | Easy to fix and change (umbrella term for the rows below) |
| [**Modularity**](development-evolution/modularity/README.md) | Split into independent components |
| [**Reusability**](development-evolution/reusability/README.md) | Components can be reused elsewhere |
| [**Analyzability / Readability**](development-evolution/analyzability/README.md) | Easy to understand and diagnose |
| [**Modifiability**](development-evolution/modifiability/README.md) | Changes don't cause ripple effects |
| [**Testability**](development-evolution/testability/README.md) | Easy to write and run tests: isolation, mocking, determinism |
| [**Extensibility**](development-evolution/extensibility/README.md) | New features can be added without rewriting |
| [**Simplicity**](development-evolution/simplicity/README.md) | Avoids unnecessary complexity |
| [**Evolvability**](development-evolution/evolvability/README.md) | Architecture can adapt to new requirements over years |

## 4. Operations / DevOps

How easy the system is to run.

| Attribute | Meaning |
|---|---|
| [**Observability**](operations-devops/observability/README.md) | Internal state is visible through logs, metrics, traces |
| [**Monitorability**](operations-devops/monitorability/README.md) | Health can be watched and alerted on |
| [**Debuggability**](operations-devops/debuggability/README.md) | Production problems can be investigated |
| [**Deployability**](operations-devops/deployability/README.md) | Easy and safe to release (CI/CD, rollback, blue-green) |
| [**Configurability**](operations-devops/configurability/README.md) | Behavior can change without code changes |
| [**Manageability / Administrability**](operations-devops/manageability/README.md) | Easy for ops teams to run |
| [**Supportability / Serviceability**](operations-devops/supportability/README.md) | Easy to support users and fix issues |

## 5. Portability / Compatibility

| Attribute | Meaning |
|---|---|
| [**Portability**](portability-compatibility/portability/README.md) | Runs on different platforms / clouds |
| [**Installability**](portability-compatibility/installability/README.md) | Easy to install and set up |
| [**Adaptability**](portability-compatibility/adaptability/README.md) | Adjusts to different environments |
| [**Replaceability**](portability-compatibility/replaceability/README.md) | Components can be swapped out (e.g., changing DB vendor) |
| [**Interoperability**](portability-compatibility/interoperability/README.md) | Works with other systems via APIs / standards |
| [**Compatibility / Co-existence**](portability-compatibility/co-existence/README.md) | Shares an environment with other software without conflict |
| [**Backward compatibility**](portability-compatibility/backward-compatibility/README.md) | New versions don't break old clients |

## 6. UI / UX

Qualities from the user's perspective.

| Attribute | Meaning |
|---|---|
| [**Usability**](UI-UX/usability/README.md) | Easy to use (umbrella term for the rows below) |
| [**Learnability**](UI-UX/learnability/README.md) | New users get productive quickly |
| [**Operability**](UI-UX/operability/README.md) | Easy to control and operate |
| [**Efficiency of use**](UI-UX/efficiency-of-use/README.md) | Experienced users finish tasks quickly |
| [**Memorability**](UI-UX/memorability/README.md) | Users remember how to use it after a break |
| [**User error protection**](UI-UX/user-error-protection/README.md) | Prevents mistakes (confirmations, undo) |
| [**Accessibility (a11y)**](UI-UX/accessibility/README.md) | Usable by people with disabilities (WCAG) |
| [**Aesthetics / UI appeal**](UI-UX/aesthetics/README.md) | Visually pleasing and consistent |
| [**Responsiveness (perceived performance)**](UI-UX/responsiveness/README.md) | Feels fast (loading states, optimistic UI) |
| [**Internationalization / Localization (i18n / l10n)**](UI-UX/internationalization/README.md) | Supports multiple languages and locales |
| [**Consistency**](UI-UX/consistency/README.md) | Uniform patterns across the app |
| [**Satisfaction**](UI-UX/satisfaction/README.md) | Users enjoy it (NPS, CSAT) |

## 7. Business / Project

| Attribute | Meaning |
|---|---|
| [**Cost efficiency**](business-project/cost-efficiency/README.md) | Low infrastructure and operating cost (TCO) |
| [**Time to market**](business-project/time-to-market/README.md) | How fast features can ship |
| [**Projected lifetime**](business-project/projected-lifetime/README.md) | How long the system stays viable |
| [**Vendor independence**](business-project/vendor-independence/README.md) | Avoids lock-in |
| [**Legal / Licensing**](business-project/licensing/README.md) | Open-source license compliance |
| [**Sustainability**](business-project/sustainability/README.md) | Energy usage, green computing |

## 8. Safety (critical systems)

| Attribute | Meaning |
|---|---|
| [**Safety**](safety/safety/README.md) | Doesn't cause harm to people or the environment (medical, automotive) |
| [**Fail-safe**](safety/fail-safe/README.md) | Fails into a safe state |
| [**Hazard warning**](safety/hazard-warning/README.md) | Alerts users to dangerous conditions |

---

## 9. SEO (search engine optimization)

How easily people find the system through search engines. Most of it is engineering (crawling, rendering, speed, URLs), not marketing.

| Attribute | Meaning |
|---|---|
| [**How search works**](SEO-optimize/how-search-works/README.md) | The crawl → index → rank pipeline that every other row feeds |
| [**Crawlability**](SEO-optimize/crawlability/README.md) | Bots can discover and fetch the pages (robots.txt, sitemaps, server health) |
| [**Indexability**](SEO-optimize/indexability/README.md) | Fetched pages are rendered and stored, one canonical version each |
| [**Site architecture**](SEO-optimize/site-architecture/README.md) | Clean URLs, internal links, safe redirects and migrations |
| [**Page experience**](SEO-optimize/page-experience/README.md) | Fast, stable, mobile-friendly pages (Core Web Vitals, CDN, server location) |
| [**On-page relevance**](SEO-optimize/on-page-relevance/README.md) | Content matches what searchers want (intent, titles, quality) |
| [**Structured data**](SEO-optimize/structured-data/README.md) | Machine-readable facts (schema.org) for rich results |
| [**Authority (links)**](SEO-optimize/authority/README.md) | Other sites vouch for this one (backlinks, PageRank) |
| [**International SEO**](SEO-optimize/international-seo/README.md) | The right language/country version reaches each user (hreflang) |
| [**Local SEO**](SEO-optimize/local-seo/README.md) | Shows up for "near me" and map searches |
| [**SEO monitoring**](SEO-optimize/seo-monitoring/README.md) | Search Console, crawl logs and CI checks catch regressions |

---

## 10. Common trade-offs

Quality attributes often conflict with each other:

- **Consistency vs Availability**: the CAP theorem
- **Security vs Usability**: more checks mean more friction
- **Performance vs Maintainability**: heavy optimization makes code harder to read
- **Cost vs Availability**: redundancy costs money
- **Interactivity vs SEO**: client-side-rendered apps feel rich but are harder for crawlers; server rendering costs more server work

---

## 11. Priorities for large-scale products

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
