# Vendor independence

> **Vendor independence** = the business can **change or leave a supplier** (cloud provider, database, SaaS, library, AI model provider) at a cost and in a time it can afford, so the supplier's prices, outages and decisions don't become its own.
> "If this vendor doubled its price, changed its licence, or shut the product down next year, what would it cost us to leave, and how long would it take?"

Related: [Portability](../../portability-compatibility/portability/README.md) (the technical ability to run elsewhere), [Replaceability](../../portability-compatibility/replaceability/README.md) (swapping one component), [Interoperability](../../portability-compatibility/interoperability/README.md) (standards), [Licensing](../licensing/README.md) (licence changes are a vendor risk), [Cost efficiency](../cost-efficiency/README.md) (pricing power), [Projected lifetime](../projected-lifetime/README.md).

Portability and replaceability are the engineering means. Vendor independence is the business outcome: **negotiating power and survivable exits**. This page covers the risk and the decision; see those pages for the techniques.

---

## 1. Kinds of lock-in

| Kind | What holds you | Examples |
|---|---|---|
| **Data** | Getting data out is slow, expensive or lossy | Proprietary formats, egress fees, no bulk export, huge volumes |
| **API / feature** | Code written against proprietary interfaces | A cloud's own queue, serverless runtime or NoSQL API; database-specific SQL and stored procedures |
| **Architecture** | The design assumes one provider's building blocks | Workflows built from a provider's functions, event bus and identity system |
| **Operational** | Tooling, monitoring, deployment and security built around one platform | IAM policies, IaC modules, dashboards, runbooks |
| **Skills** | The team knows one stack | Certifications, hiring, habits |
| **Contractual** | Commitments and terms | Multi-year spend agreements, minimum terms, auto-renewal, per-seat licences |
| **Ecosystem** | Integrations, marketplaces, customers' expectations | Plugins that exist only for one platform |

Lock-in isn't only about clouds. A framework, a programming language, an identity provider, a payment processor, an AI model API and an open-source project with a single company behind it all create switching costs.

---

## 2. What can go wrong

- **Price increases** or pricing model changes, once you can't easily leave.
- **Product shutdown or deprecation** on the vendor's timetable.
- **Licence or terms change** ([Licensing](../licensing/README.md)).
- **Outages** you can only wait out ([Availability](../../runtime-operational/availability/README.md)).
- **Acquisition or bankruptcy** of the vendor.
- **Roadmap mismatch**: the feature you need never arrives.
- **Legal and geopolitical change**: data residency rules, sanctions, regulatory requirements for an exit plan (for example, the EU's DORA regulation for financial firms).
- **Account suspension**, by mistake or policy.

---

## 3. Lock-in is a trade, not a sin

Using a vendor's proprietary features buys **speed, lower operating effort and often better products**. Avoiding all lock-in means building on the lowest common denominator and running more yourself, which is its own heavy cost. The goal is to **know the price of leaving and keep it affordable**, not to make it zero.

A simple way to decide per dependency:

```
exposure = probability you'll need to leave × cost of leaving
```

compared with what independence costs you every year in extra work and missed features.

### Worked example: is the abstraction worth it?

A team uses a cloud provider's proprietary queue and NoSQL database. Illustrative estimates:

```
Cost to migrate later (rewrite + data move + dual running):  6 engineer-months
Chance of needing to in the next 5 years:                    10%
Expected cost of lock-in  = 0.10 × 6  = 0.6 engineer-months

Cost of staying portable now (own abstraction layer, self-run
open-source equivalents, fewer features):                     1.5 engineer-months up front
                                                            + 1 engineer-month per year
Over 5 years              = 1.5 + 5 × 1 = 6.5 engineer-months
```

Paying 6.5 to avoid an expected 0.6 is a bad deal. Use the proprietary services, and spend a few days on the cheap protections in §4 instead.

The answer changes when the numbers do. For the **primary database holding 50 TB of customer data**, migration might be 30 engineer-months plus a risky cutover, and the vendor's pricing power grows every year. There, choosing a database with several independent providers of the same engine (e.g. PostgreSQL) costs almost nothing extra and removes most of the exposure.

**Rule of thumb**: be strict about lock-in for **data and core domain logic**; be relaxed about it for stateless glue that's cheap to rewrite.

---

## 4. Keeping the exit affordable

| Technique | What it protects |
|---|---|
| **Open standards and protocols** | SQL / PostgreSQL wire protocol, S3-compatible object API, Kubernetes, OCI images, OpenTelemetry, OAuth 2.0 / OIDC, AMQP / Kafka protocol, OpenAPI: several vendors implement each |
| **Open-source engines with several providers** | You can move between managed offerings or self-host. Check the licence and who controls the project |
| **Own your data** | Regular exports in open formats, tested; data and backups in an account you control; know the egress cost |
| **Thin adapters at the boundary** | Wrap each vendor behind an interface in your own terms (ports and adapters). Keep vendor SDK types out of the domain code ([Replaceability](../../portability-compatibility/replaceability/README.md)) |
| **Infrastructure as code** | The whole setup is written down and can be re-targeted, instead of living in a console |
| **Own the identifiers** | Your own domain names, customer IDs, API URLs. Don't expose vendor IDs or hostnames to customers |
| **Second source for critical paths** | Two payment processors, two SMS providers, two model providers behind one interface, with real traffic on both |
| **Contract terms** | Data export and deletion rights, price caps on renewal, notice periods, source-code escrow for critical niche vendors |
| **A written exit plan** | For each critical vendor: the alternative, the steps, the estimate, the trigger. Review it yearly |

**Don't build a generic abstraction over everything.** A home-made "cloud-neutral" layer usually exposes only the features all providers share, has to be maintained forever, and still leaks. Abstract at the level of *your* use case ("store invoice", "send notification"), not the vendor's API.

**Multi-cloud** is mostly a different goal. Running the same workload actively on two clouds is expensive (two sets of skills, tooling, networking, and data transfer between them) and is justified by specific needs: regulation, customer demand, or surviving a provider-wide outage. For vendor independence alone, **being able to move** is nearly always enough; you don't have to be on both at once.

---

## 5. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Parse (2016–2017)** | Facebook announced it would shut down the Parse mobile backend service, giving developers a year to migrate, and open-sourced a server implementation. | Hosted platforms can disappear. An export path and an open implementation made the exit possible. |
| **Terraform → OpenTofu (2023)** | HashiCorp moved Terraform from an open-source licence to the Business Source License. The community forked the last open version as OpenTofu under the Linux Foundation. | Open source controlled by one company is still a vendor. Forks are the escape hatch that an open licence provides. |
| **Redis → Valkey (2024)** | Redis changed to source-available licences; a fork, Valkey, was created under the Linux Foundation with backing from several cloud providers. | A widely implemented protocol plus an open code base keeps switching cheap. |
| **Cloud exit fees (2024)** | Around the time the EU Data Act came into force, the major cloud providers announced they would waive data transfer fees for customers leaving their platforms. | Regulation and competition change switching costs. Recheck your exit estimate from time to time. |

---

## 6. Anti-patterns

- **Lock-in by accident**: nobody decided; proprietary features crept in one convenience at a time.
- **Abstracting everything "just in case"**, paying a permanent tax to avoid an unlikely move.
- **Lowest-common-denominator design**: using a cloud as if it were only virtual machines, getting the cost without the benefit.
- **Multi-cloud for its own sake**, with half the reliability and twice the complexity.
- **Data hostage**: no tested export; backups stored only inside the vendor being backed up.
- **Vendor identifiers in public contracts**: customers integrate with the vendor's URLs and IDs.
- **A "second source" that's never used**, so it doesn't work on the day it's needed.
- **Believing self-hosting is independence**: you've swapped a vendor for dependence on your own staff and on the open-source project.
- **Signing multi-year commitments** without an exit clause or an exit estimate.

---

## 7. Checklist

- [ ] Inventory of vendors and critical dependencies, each with an owner and a criticality rating
- [ ] For each critical one: the lock-in type, an exit estimate (time and cost) and a named alternative
- [ ] Use of proprietary features is a recorded decision, not an accident
- [ ] Data exportable in open formats; export and restore tested; egress cost known
- [ ] Vendor SDKs and types confined to adapters at the edge
- [ ] Infrastructure defined in code
- [ ] Customer-facing names and identifiers owned by you
- [ ] Contract terms cover data return, notice periods and renewal pricing
- [ ] Vendor health and licence changes watched; exit plans reviewed yearly

---

## 8. Exercises

1. List every external vendor and platform a system you know depends on. For each, name the lock-in type from §1 and estimate the time to leave. Which is the most dangerous?
2. Redo the §3 calculation for your own primary database and for your email-sending provider. Do the two answers differ? Why?
3. Your payment provider has a 6-hour outage. Design a second-source setup: what's shared, what's per provider, how traffic is split, and what can't be made portable (stored cards, subscriptions)?
4. Write a one-page exit plan for one critical vendor: trigger conditions, target alternative, migration steps, data transfer, estimated cost, and what you'd do this quarter to make it cheaper.

---

## 9. Further reading

Gregor Hohpe, "Don't get locked up into avoiding lock-in" (martinfowler.com) and *Cloud Strategy*; Alistair Cockburn, "Hexagonal Architecture" (ports and adapters); the Twelve-Factor App ("Backing services"); Cloud Native Computing Foundation landscape (open, multi-vendor projects); OpenTofu and Valkey project announcements (Linux Foundation, 2023–2024); the EU Data Act (switching between data processing services) and the EU Digital Operational Resilience Act (exit strategies for critical ICT providers); Dan McKinley, "Choose Boring Technology".
