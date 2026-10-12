# Cost efficiency

> **Cost efficiency** = the system delivers its business value at the **lowest total cost that still meets its other requirements**: infrastructure, licences, and the people who build and run it.
> "What does one order / user / request cost us, is that number going down as we grow, and would the business still work at 10× the size?"

Related: [Efficiency](../../runtime-operational/efficiency/README.md) (the technical side: work per CPU, right-sizing, waste), [Maintainability §15](../../development-evolution/maintainability/README.md#15-cost-efficiency-finops) (short version: FinOps basics), [Elasticity](../../runtime-operational/elasticity/README.md) (don't pay for idle capacity), [Simplicity](../../development-evolution/simplicity/README.md) (every component has a running cost), [Sustainability](../sustainability/README.md) (less waste usually means less carbon), [Vendor independence](../vendor-independence/README.md) (pricing power).

Efficiency asks "how much work per resource?". Cost efficiency asks the business question on top: "**how much money per unit of value**, and who decides?"

---

## Table of contents

1. [Why cost is a design dimension](#1-why-cost-is-a-design-dimension)
2. [Total cost of ownership](#2-total-cost-of-ownership)
3. [Unit economics](#3-unit-economics)
4. [How cloud pricing works](#4-how-cloud-pricing-works)
5. [FinOps: making cost visible and owned](#5-finops-making-cost-visible-and-owned)
6. [Cost-aware architecture](#6-cost-aware-architecture)
7. [Build vs buy, managed vs self-hosted](#7-build-vs-buy-managed-vs-self-hosted)
8. [Commitments and purchasing](#8-commitments-and-purchasing)
9. [Deciding whether an optimisation is worth it](#9-deciding-whether-an-optimisation-is-worth-it)
10. [Guardrails: budgets, alerts, limits](#10-guardrails-budgets-alerts-limits)
11. [Approaches by system type](#11-approaches-by-system-type)
12. [Trade-offs](#12-trade-offs)
13. [Anti-patterns](#13-anti-patterns)
14. [Real-world stories](#14-real-world-stories)
15. [Checklist](#15-checklist)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Why cost is a design dimension

- **Cost decides whether the business works.** For a software company, infrastructure is part of the cost of goods sold: it comes straight out of gross margin.
- **Architecture fixes most of the cost early.** Choosing a chatty microservice design, a per-request-priced service on a hot path, or cross-region replication sets the bill for years. Tuning later only moves it by a fraction.
- **Cloud made spending a per-engineer decision.** Anyone who can merge a Terraform change can commit the company to a monthly cost. Without visibility, nobody notices until the invoice.
- **Cost is a signal of waste.** An unexpectedly expensive component is usually doing unnecessary work, which also hurts latency and energy use.

Cost efficiency is **not** the same as spending little. Spending more to earn more is fine. The goal is to know what you're buying.

---

## 2. Total cost of ownership

**TCO** = everything a system costs over its life, not just the monthly invoice.

| Cost | Examples | Often forgotten? |
|---|---|---|
| **Infrastructure** | Compute, storage, databases, network, CDN | No |
| **Data transfer** | Internet egress, cross-AZ, cross-region, NAT gateways | Yes |
| **Third-party services** | Observability, auth, email/SMS, payments fees, search, AI APIs | Partly |
| **Licences and support** | Database licences, enterprise support plans, per-seat tools | Partly |
| **People: build** | Engineering time to design and implement | Yes |
| **People: run** | On-call, upgrades, patching, backups, incident response | Yes |
| **Non-production** | Dev, staging, CI runners, preview environments, load tests | Yes |
| **Compliance and security** | Audits, penetration tests, tooling | Yes |
| **Opportunity cost** | What the team didn't build while doing this | Almost always |
| **Exit cost** | Migrating away, data export, contract penalties | Almost always |

People are usually the largest item. A design that saves $1,000 a month in servers but needs a day a week of an engineer's attention is more expensive, not cheaper.

**Capex vs opex**: buying hardware is a capital expense (paid up front, depreciated over years); cloud is an operating expense (paid as you go). Cloud trades a higher unit price for no up-front spend, no capacity guessing, and the ability to stop paying. That trade is best when load is uncertain or variable, and weakest for large, steady, predictable workloads.

---

## 3. Unit economics

The total bill says nothing on its own. Divide it by a **unit of business value**:

| Business | Useful unit |
|---|---|
| E-commerce | Cost per order |
| SaaS | Cost per active customer / per seat / per tenant |
| API product | Cost per 1M API calls |
| Streaming | Cost per hour streamed |
| Storage product | Cost per GB stored per month |
| AI feature | Cost per conversation / per generated document |

Then compare unit cost with **unit revenue**. That gives the margin per unit, which is what the business actually cares about.

### Worked example: bill up, unit cost down

Made-up numbers for a shop:

```
January:  bill $42,000   orders 1,200,000   → $0.035 per order
July:     bill $60,000   orders 2,000,000   → $0.030 per order
```

The bill grew **43%** and finance is worried. But orders grew 67% and cost per order **fell 14%**. The system is getting more efficient with scale. This is good news, and without the unit metric nobody could tell.

Now break the July unit cost down:

| Component | Monthly | Per order |
|---|---|---|
| Compute (app tier) | $21,000 | $0.0105 |
| Databases | $15,000 | $0.0075 |
| Observability (logs, metrics, traces) | $12,000 | $0.0060 |
| Data transfer | $7,000 | $0.0035 |
| Storage, queues, other | $5,000 | $0.0025 |
| **Total** | **$60,000** | **$0.0300** |

Observability is 20% of the bill, as much as data transfer, storage and queues together. That's where to look first ([Observability §12](../../operations-devops/observability/README.md#12-cost-and-cardinality)), not at the app tier everyone assumed.

**Per-tenant cost** matters as much as the average: in most SaaS products a few customers cause most of the cost. If a customer on a $500/month plan costs $900/month to serve, that's a pricing or limits problem, not an infrastructure one.

**Fixed vs variable**: a unit cost that falls as you grow means part of the bill is fixed (baseline clusters, minimum replicas). A unit cost that *rises* with growth means something scales worse than linearly: find it early.

---

## 4. How cloud pricing works

You can't design for cost without knowing what's metered.

| Pricing dimension | How it's charged | Design consequence |
|---|---|---|
| **Compute time** | Per second/hour of a running instance, used or not | Idle capacity costs the same as busy capacity |
| **Per request / invocation** | Serverless functions, API gateways, managed queues | Cheap at low volume, expensive on hot paths at high volume |
| **Storage** | Per GB-month, by tier; plus per-request and retrieval fees on colder tiers | Archive tiers are cheap to keep and expensive to read |
| **Provisioned performance** | IOPS, throughput, capacity units | You pay for what you reserved, not what you used |
| **Data transfer** | Per GB leaving a region or cloud; often per GB between zones; inbound usually free | Architecture topology is a cost decision |
| **Managed service premium** | Higher unit price than raw compute | You're buying operations ([§7](#7-build-vs-buy-managed-vs-self-hosted)) |
| **Per seat / per host / per GB ingested** | SaaS tooling, observability, security agents | Scales with team size or data volume, not with revenue |

### Worked example: the data transfer surprise

A media service sends **50 TB** a month from object storage directly to users. Using an illustrative **$0.09 per GB** egress price:

```
50 TB = 50,000 GB × $0.09 = $4,500 / month for egress alone
```

Storing those files might cost a fraction of that. Two design changes: serve through a **CDN** (usually a lower per-GB price, and cache hits never reach the origin), and **compress or transcode** to cut bytes. The same arithmetic applies inside the system: two services exchanging 10 TB a month across zones pay a per-GB fee in both directions that a same-zone placement avoids.

Always check current prices in the provider's pricing calculator: they change, and they differ by region.

---

## 5. FinOps: making cost visible and owned

**FinOps** is the practice of bringing engineering, finance and product together so that the people who create cost can see it and are responsible for it. The FinOps Foundation describes three repeating phases:

| Phase | What happens |
|---|---|
| **Inform** | Allocate every cost to an owner; dashboards, unit costs, forecasts |
| **Optimise** | Right-size, remove waste, buy commitments, improve architecture |
| **Operate** | Budgets, policies, reviews, automation; make it routine |

Practices:
- **Allocation**: tag/label every resource with `team`, `service`, `environment`, `cost-centre`. Enforce tags at creation time (policy as code), because retro-tagging never finishes.
- **Shared costs**: decide how to split clusters, networking and support plans (by usage, evenly, or kept central). An imperfect rule everyone understands beats a perfect one nobody trusts.
- **Showback → chargeback**: first show each team its cost; later, if the organisation wants, charge it to their budget.
- **Kubernetes** hides cost inside shared nodes. Tools such as **OpenCost** or Kubecost allocate node cost to namespaces and workloads from resource requests and usage.
- **Standard billing data**: the **FOCUS** specification defines a common schema for cost and usage data across providers.
- **Cost in the engineering loop**: cost estimates on infrastructure pull requests (e.g. **Infracost** for Terraform), cost per service on the same dashboards as latency and errors, cost as a section in design documents.
- **Regular review**: monthly per team, with the top changes explained.

The principle behind all of it: **engineers make better cost decisions than any central team, once they can see the numbers.**

---

## 6. Cost-aware architecture

The low-level techniques (right-sizing, spot, storage lifecycle, bin-packing) are in [Efficiency §3](../../runtime-operational/efficiency/README.md#3-techniques). This section is about the design choices that set the size of the bill before any tuning.

| Decision | Cheaper choice (usually) | Why |
|---|---|---|
| **Do less work** | Cache, precompute, deduplicate, drop unused features | The cheapest request is the one never executed |
| **Fewer components** | Modular monolith + one database before many services | Each component has a fixed baseline cost and an operations cost ([Simplicity §7](../../development-evolution/simplicity/README.md#7-the-cost-of-adding-a-component)) |
| **Match the pricing model to the load shape** | Serverless for spiky/low volume; containers or VMs for steady high volume | Per-request pricing has no idle cost but a high unit price |
| **Keep data where it's used** | Same zone/region for chatty components; CDN at the edge | Data transfer is metered |
| **Tier data by temperature** | Hot / warm / cold / delete | Most data is rarely read after a few weeks |
| **Async and batch what can wait** | Queues, off-peak batch on spot capacity | Smooths peaks, uses cheaper capacity |
| **Multi-tenancy** | Shared (pooled) infrastructure with isolation controls | Dedicated stacks per customer multiply the baseline |
| **Bound every resource** | Quotas, rate limits, retention limits, max sizes | Unbounded usage is an unbounded bill |
| **Right level of redundancy** | Multi-AZ by default; multi-region only where the business needs it | Each extra "nine" costs much more than the previous one ([Availability](../../runtime-operational/availability/README.md)) |

**Cost as a non-functional requirement**: write a target into the design, like a latency budget. "This feature must cost under $0.002 per order at 5M orders a month." Then estimate before building: requests × price, GB × price, instances × price. A spreadsheet estimate that's wrong by 2× is still far better than none.

**AI and other usage-priced APIs** make this urgent: when each call costs real money per token, prompt size, caching, model choice and per-user limits are architecture decisions, and one abusive user or a retry loop can dominate the bill.

---

## 7. Build vs buy, managed vs self-hosted

### Worked example: "self-hosting is cheaper"

A team considers running its own Kafka instead of a managed service. Illustrative numbers:

```
Managed service:           $3,000 / month

Self-hosted:
  instances + disks:       $1,200 / month
  engineer time: 0.4 of one engineer × $12,500 loaded cost per month
                         = $5,000 / month
  total:                   $6,200 / month
```

The invoice says self-hosting saves $1,800. The TCO says it costs **$3,200 more**, before counting the outage risk of a small team running a system it rarely touches. The answer flips at larger scale: at 20× the volume the managed bill might be $60,000 while self-hosting needs perhaps two engineers and $24,000 of infrastructure. Then a dedicated team pays for itself.

| Choose managed / buy when | Choose self-hosted / build when |
|---|---|
| It's not what makes your product different | It's core to the product or a real advantage |
| The team is small; operations skills are scarce | You already have a team that runs it well |
| Volume is low or uncertain | Volume is large and steady, so the premium is big in absolute terms |
| You need it this month | You have requirements no vendor meets |

Recheck the decision as you grow. It's one of the few cost decisions that reverses with scale. Keep the exit possible ([Vendor independence](../vendor-independence/README.md)).

---

## 8. Commitments and purchasing

Providers sell discounts in exchange for commitment or flexibility:

| Model | Discount | You give up | Use for |
|---|---|---|---|
| **On-demand** | None | Nothing | Variable and unknown load |
| **Commitments** (reserved instances, savings plans, committed-use discounts) | Significant | Flexibility: you pay for 1–3 years whether you use it or not | The steady baseline |
| **Spot / preemptible** | Large | Reliability: capacity can be taken back at short notice | Stateless, batch, CI, fault-tolerant work |
| **Negotiated agreements** | Varies | A minimum total spend over several years | Large, predictable organisations |

### Worked example: commitment break-even

An instance costs **$100/month** on demand, or **$70/month** with a one-year commitment (illustrative 30% discount), paid whether it runs or not.

```
Break-even utilisation = committed price / on-demand price = 70 / 100 = 70%
```

If that capacity would be running more than 70% of the time over the year, commit. If less, stay on demand. So:

- A fleet that runs **9 instances at night and 18 at peak**: commit to the 9 that are always on, keep the other 9 on demand or spot.
- **Commit after optimising**, not before: committing to an over-provisioned fleet locks in the waste.
- Track **coverage** (share of usage under commitment) and **utilisation** (share of commitment actually used). Low utilisation means you're paying for nothing.

Commitments are also a form of lock-in: a multi-year spend agreement makes leaving expensive even if the technology is portable.

---

## 9. Deciding whether an optimisation is worth it

Engineering time is a cost too. Before optimising, do the arithmetic:

```
payback period = cost of the work / monthly saving
```

Illustrative: rewriting a batch job saves **$800/month** and takes **3 engineer-weeks** at about **$3,000 per week**:

```
$9,000 / $800 ≈ 11 months to pay back
```

Probably not worth it unless growth will multiply the saving. Compare with deleting forgotten staging environments: **$2,500/month**, half a day of work, pays back in days.

Rules of thumb:
- **Sort by saving ÷ effort.** Waste removal (idle resources, retention, non-prod schedules) nearly always comes first; rewrites last.
- **Fix the biggest line items.** A 10% saving on the largest component beats 50% on a small one.
- **Consider the growth curve.** A cost that scales with users deserves attention earlier than a fixed one.
- **Below some scale, don't.** A startup with a $2,000 bill should spend its time finding customers.
- **Count risk.** A saving that reduces redundancy or headroom isn't a saving; it's an unpriced incident.

---

## 10. Guardrails: budgets, alerts, limits

Cost incidents are real incidents: a retry loop, a leaked credential used for crypto-mining, a recursive function trigger or a debug log level left on can spend a month's budget in a day.

- **Budgets** per account/team with alerts at 50%, 80%, 100%, and on the **forecast**, not only the actual.
- **Anomaly detection** on daily spend per service.
- **Hard limits** where the platform allows: autoscaling maximums, concurrency limits on functions, spending caps on third-party APIs, quotas per tenant.
- **Policy as code**: required tags, allowed instance types and regions, no public buckets, expiry on non-production resources (e.g. Cloud Custodian, OPA).
- **Automatic expiry** for preview environments and experiments.
- **Separate accounts/projects** per environment and team: clean cost boundaries and a limited blast radius.
- **A runbook** for a cost incident: who's paged, how to stop the spend, how to ask the provider for help.

---

## 11. Approaches by system type

| System | Main cost drivers | Typical levers |
|---|---|---|
| **Startup / MVP** | People, not infrastructure | PaaS and managed services, one region, use free tiers and credits carefully, set a budget alert on day one |
| **SaaS (multi-tenant)** | Compute, databases, per-tenant overhead | Pooled tenancy, per-tenant cost visibility, plan limits tied to cost, scale-to-zero for idle tenants |
| **E-commerce / consumer web** | Peak capacity, databases, CDN | Autoscaling, caching, CDN, commitments for the baseline, pre-scale for known events |
| **Data / analytics platforms** | Storage, query compute, duplicate pipelines | Tiered storage, partitioning and columnar formats, query cost limits, delete unused datasets, spot for batch |
| **Media / streaming** | Egress, storage, transcoding | CDN contracts, efficient codecs, tiering by popularity, transcode on spot |
| **AI / ML** | GPUs, per-token API calls | Smaller models where good enough, caching, batching, quotas per user, spot for training, shut down idle GPUs |
| **Internal tools** | Idle time | Scale to zero, shared clusters, office-hours schedules |
| **Observability stack** | Ingest volume, cardinality, retention | Sampling, drop rules, retention tiers ([Observability §12](../../operations-devops/observability/README.md#12-cost-and-cardinality)) |

---

## 12. Trade-offs

| Cost vs | Tension |
|---|---|
| **Availability** | Redundancy, multi-region and headroom cost money; they are insurance, priced against the cost of downtime |
| **Performance** | Lower latency often means more replicas, more cache memory, more regions |
| **Time to market** | Managed services and over-provisioning buy speed; optimising early delays shipping ([Time to market](../time-to-market/README.md)) |
| **Vendor independence** | The cheapest option is often the most provider-specific; portability and multi-cloud cost extra |
| **Simplicity** | Spot, tiering, complex autoscaling and many purchase options save money and add operational complexity |
| **Security / compliance** | Isolation, encryption, audit logging and dedicated tenancy add cost |
| **Developer experience** | Rich non-production environments and fast CI cost money and save engineer time |

---

## 13. Anti-patterns

- **No owner**: untagged resources; the bill is "the platform team's problem".
- **Watching the total instead of the unit cost**, then cutting spend that was growing for good reasons.
- **Cloud bill ≠ TCO**: "saving" on infrastructure by spending far more in engineer time.
- **Premature optimisation**: weeks of work to save a few hundred dollars a month.
- **Cutting redundancy, backups or observability** to hit a number.
- **Committing before right-sizing**, or buying commitments nobody tracks.
- **Lift and shift** to the cloud without changing the architecture: paying cloud prices for a design that can't use elasticity.
- **Unbounded things**: no retention limits, no autoscaling maximum, no per-tenant quota, no API spending cap.
- **Zombie environments**: every experiment and demo kept forever.
- **Cost reviews as blame**: teams learn to hide spend instead of explaining it.
- **Ignoring data transfer** in architecture diagrams.

---

## 14. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Dropbox "Magic Pocket" (2015–2016)** | Dropbox moved most of its file storage from a public cloud to its own storage system. Its 2018 IPO filing reported a large reduction in infrastructure cost over the following two years. | At very large, steady, predictable scale, owning infrastructure can pay off. It took a dedicated team and years. |
| **37signals leaves the cloud (2022–2023)** | The company behind Basecamp and HEY publicly described moving its applications from cloud services to its own hardware, and published its bills and expected multi-year savings. | Steady workloads and an experienced operations team change the build-vs-buy answer. The published numbers also show how much a managed premium can add up to. |
| **Amazon Prime Video monitoring (2023)** | A team wrote that moving a video quality monitoring tool from a distributed serverless design to a single process cut its infrastructure cost by about 90%. | Per-invocation pricing and data passing between steps can dominate on a high-volume path. Match the pricing model to the load. |
| **Snap's IPO filing (2017)** | Snap disclosed multi-year commitments to spend billions of dollars with cloud providers. | At scale, infrastructure purchasing is a board-level business decision, and commitments are a form of lock-in. |
| **"The Cost of Cloud, a Trillion Dollar Paradox" (2021)** | An Andreessen Horowitz essay argued that cloud spend weighs heavily on the margins of large software companies, and that the trade-off shifts as companies mature. | Cloud is usually right early and should be re-examined later. The essay is an argument, not a measurement; read the critiques too. |

---

## 15. Checklist

**Visibility**
- [ ] Every resource tagged with owner, service and environment; untagged spend near zero
- [ ] Cost per team and per service visible to the engineers who own them
- [ ] A unit cost metric defined and tracked next to revenue per unit
- [ ] Cost per tenant known for the largest customers

**Design**
- [ ] New designs include a cost estimate and a cost target
- [ ] Pricing model of each service matches its load shape
- [ ] Data transfer paths reviewed (cross-zone, cross-region, egress)
- [ ] Every resource bounded: retention, quotas, autoscaling maximums
- [ ] Managed-vs-self-hosted decisions include people cost and are revisited as scale changes

**Operations**
- [ ] Budgets with forecast alerts; anomaly detection on daily spend
- [ ] Waste removal is routine (idle resources, non-prod schedules, expiry)
- [ ] Commitments cover the steady baseline only; coverage and utilisation tracked
- [ ] Regular cost review per team; optimisations ranked by saving ÷ effort
- [ ] Runbook for a cost incident

---

## 16. Exercises

1. **Unit cost.** Define a unit cost metric for a product you know. List the three components you expect to dominate it, then check against a real or estimated bill.
2. **Estimate before building.** A feature stores a 2 MB image per order and serves it 20 times, for 1M orders a month, kept for 3 years. Estimate storage and egress per month in year 1 and year 3 using a provider's pricing calculator. What would you change in the design?
3. **TCO.** Compare a managed PostgreSQL service with self-hosting on VMs for a 5-person team. Include backups, upgrades, failover and on-call. At what team size or database count does the answer change?
4. **Commitments.** A fleet runs 30 instances for 6 hours a day and 12 the rest of the time. With a 35% commitment discount, how many instances would you commit to? What's the break-even utilisation?
5. **Serverless vs containers.** A function runs 200 ms at 512 MB. Using current list prices, at how many requests per month does a pair of small always-on containers become cheaper? What does that leave out?
6. **Payback.** List five cost optimisations for a system you know, estimate saving and effort for each, and rank them. Which would you refuse to do?
7. **Cost incident.** Design the guardrails that would have stopped a recursive function trigger from running all weekend.

---

## 17. Further reading

- **FinOps Foundation** (finops.org): the FinOps Framework, and the **FOCUS** billing data specification.
- J.R. Storment & Mike Fuller, *Cloud FinOps* (O'Reilly).
- **AWS Well-Architected Framework, Cost Optimization pillar**; Google Cloud Architecture Framework (cost optimisation); Azure Well-Architected Framework (cost optimisation).
- Sarah Wang & Martin Casado (a16z), "The Cost of Cloud, a Trillion Dollar Paradox" (2021).
- Dropbox engineering blog on Magic Pocket; 37signals / David Heinemeier Hansson, "Why we're leaving the cloud" and follow-ups.
- Prime Video Tech blog, "Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%" (2023).
- Corey Quinn, *Last Week in AWS* (cloud billing in practice).
- OpenCost (opencost.io) and Infracost documentation.
- Werner Vogels, "The Frugal Architect" (thefrugalarchitect.com): laws for cost-aware architecture.
