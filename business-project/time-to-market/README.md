# Time to market

> **Time to market** = how long it takes to get an idea **from decision to working software in users' hands**, for the first release and for every change after it.
> "A competitor shipped this feature last week. How long until ours is live, and what's the slowest step?"

Related: [Deployability](../../operations-devops/deployability/README.md) (the release part of the path), [Modifiability](../../development-evolution/modifiability/README.md) and [Simplicity](../../development-evolution/simplicity/README.md) (how fast code can change), [Testability](../../development-evolution/testability/README.md) (fast feedback), [Cost efficiency](../cost-efficiency/README.md) (speed is often bought with money), [Projected lifetime](../projected-lifetime/README.md) (how much speed to trade for longevity).

---

## 1. Why it matters

- **Learning speed.** Most product ideas are partly wrong. The sooner real users see something, the sooner you find out which part.
- **Cost of delay.** A feature earns nothing until it ships. A month's delay costs a month of its value, plus whatever a competitor gains.
- **Windows close.** Regulations, seasons, contracts and launches have dates.
- **Small and fast is also safer.** Small frequent changes are easier to review, test and roll back than big rare ones.

Two different things get called time to market:

| | First release | Every release after |
|---|---|---|
| Question | How fast can we launch a new product? | How fast can we change a running one? |
| Dominated by | Scope, build-vs-buy choices, team setup | Lead time, code health, deployment pipeline, coordination |
| Risk of optimising it alone | Shortcuts that slow everything later | Polishing the process while building the wrong thing |

---

## 2. Measuring it

| Metric | Meaning |
|---|---|
| **Lead time** | Request (or decision) → in production |
| **Cycle time** | Work started → in production |
| **Flow efficiency** | Active work time ÷ total lead time |
| **Lead time for changes** (DORA) | Commit → running in production |
| **Deployment frequency** (DORA) | How often you release to production |
| **Change fail rate / failed deployment recovery time** (DORA) | How often releases go wrong, and how fast you recover |
| **Work in progress (WIP)** | Items started and not finished |
| **Time to first value** | Sign-up or project start → first real user outcome |

The DORA research (*Accelerate*) found that speed and stability go together: teams that deploy often also tend to have lower failure rates, because both come from small changes and good automation. Measure them together so speed isn't bought by breaking things.

### Worked example: where the time actually goes

A feature took **20 working days** from "approved" to "live". The team maps each step:

| Step | Working | Waiting |
|---|---|---|
| Waiting in the backlog to be picked up | – | 6 days |
| Design and clarification | 1 day | 2 days (waiting for answers) |
| Coding | 3 days | – |
| Code review | 0.5 day | 2 days (waiting for a reviewer) |
| QA | 1 day | 1.5 days (waiting for the test environment) |
| Release | 0.5 day | 2.5 days (waiting for the fortnightly release train) |
| **Total** | **6 days** | **14 days** |

```
Flow efficiency = 6 / 20 = 30%
```

70% of the lead time is **waiting**. Making coding twice as fast saves 1.5 days. Removing the release train and the review queue saves 4.5 days, with no one typing faster. The bottleneck is almost never typing speed.

---

## 3. What makes delivery slow

| Cause | Symptom |
|---|---|
| **Queues and hand-offs** | Work waits for another team, an approval, an environment, a release slot |
| **Big batches** | Large features and large releases: long reviews, risky deploys, late feedback |
| **Too much work in progress** | Everyone is busy, nothing finishes; constant context switching |
| **Coupling** | A change needs several teams or services to release together ([Modularity](../../development-evolution/modularity/README.md)) |
| **Slow or unreliable feedback** | 40-minute CI, flaky tests, manual regression testing |
| **Hard-to-change code** | Every change touches many files and breaks something far away |
| **Manual, risky releases** | Releases are rare because they hurt, and they hurt because they're rare |
| **Unclear requirements** | Rework after building the wrong thing |
| **Building what you could buy** | Months spent on auth, billing or search instead of the product |

---

## 4. Techniques

**Reduce scope**
- **MVP**: the smallest thing that tests the riskiest assumption. Cut features, not quality on the part you ship.
- **Thin vertical slices**: one narrow path through UI, API and database that works end to end, then widen it.
- Say **no** or "later" explicitly; a written non-goals list is a speed tool.

**Buy and reuse**
- Buy what doesn't make your product different: authentication, payments, email, search, observability ([Cost efficiency](../cost-efficiency/README.md) covers the money side; [Vendor independence](../vendor-independence/README.md) the exit side).
- **Boring technology** the team already knows; frameworks with conventions; a PaaS before Kubernetes.
- **Golden paths / internal platforms**: a new service gets CI, deploy, logging and dashboards from a template in an hour.

**Shorten the path to production**
- Trunk-based development, small pull requests, fast reviews.
- CI in minutes; automated tests you trust ([Testability](../../development-evolution/testability/README.md)).
- Continuous delivery: any commit can be released ([Deployability §3](../../operations-devops/deployability/README.md#3-the-deployment-pipeline)).
- **Separate deploy from release** with feature flags: ship code dark, turn it on when the business is ready ([Deployability §6](../../operations-devops/deployability/README.md#6-feature-flags)).

**Reduce coordination**
- Teams aligned to parts of the product they can change and release alone (*Team Topologies*: stream-aligned teams).
- Stable interfaces and backward compatibility, so teams don't release together.
- Limit WIP; finish before starting.

**Keep speed over time**
- Pay down the debt that slows the changes you make most often.
- Keep the architecture simple enough for new people to be productive in days.

---

## 5. Prioritising: cost of delay

When several things compete, compare what each week of delay costs, relative to how long each takes (**CD3**, cost of delay divided by duration; SAFe's WSJF is a variant).

### Worked example

Illustrative numbers:

| Feature | Value once live | Duration | Cost of delay ÷ duration |
|---|---|---|---|
| A. Annual-billing option | $8,000 / week | 4 weeks | 2,000 |
| B. Checkout fix | $5,000 / week | 1 week | 5,000 |
| C. Enterprise SSO | $12,000 / week | 8 weeks | 1,500 |

Doing them biggest-value-first (C, A, B) costs this much in delay while waiting:

```
C done at week 8, A at week 12, B at week 13
delay cost = 12,000×8 + 8,000×12 + 5,000×13 = $257,000
```

Doing them by cost of delay ÷ duration (B, A, C):

```
B done at week 1, A at week 5, C at week 13
delay cost = 5,000×1 + 8,000×5 + 12,000×13 = $201,000
```

Same work, same team, **$56,000 less delay cost**, just from ordering. Short, valuable items go first.

---

## 6. Trade-offs

| Speed vs | Tension |
|---|---|
| **Quality / technical debt** | Shortcuts are a loan: fine if deliberate, recorded and repaid; ruinous if they pile up in code you change every week |
| **Longevity** | Fast choices (a vendor's proprietary service, a quick schema) may be expensive to change later. Go fast on reversible decisions, slow on irreversible ones ([Evolvability §4](../../development-evolution/evolvability/README.md#4-reversible-and-irreversible-decisions)) |
| **Cost** | Managed services, more environments and bigger teams buy speed |
| **Team size** | Adding people to a late project makes it later (Brooks's law): onboarding and communication cost more than the new hands add |
| **Security / compliance** | Reviews take time. Build them into the pipeline instead of skipping or queueing them |
| **Scalability** | Designing for millions of users before you have hundreds delays learning whether anyone wants it |

---

## 7. Anti-patterns

- **Measuring activity** (story points, lines of code, utilisation) instead of lead time and outcomes.
- **100% utilisation**: with no slack, every item queues behind another.
- **Big-bang launches** after months without user feedback.
- **"We'll fix it after launch"** with no record of what "it" is.
- **Release trains and approval boards** as the only way to feel safe.
- **Building commodity components** because it's more interesting than the product.
- **Premature scale and generality**: microservices, plugins and multi-region for a product with no users yet.
- **Starting everything**, finishing nothing.
- **Deadline by decree**: fixing scope, date and team size at once, leaving quality as the only variable.

---

## 8. Checklist

- [ ] Lead time measured from request to production, not just coding time
- [ ] Waiting time between steps visible; the largest queue has an owner
- [ ] DORA metrics tracked, speed and stability together
- [ ] Changes are small; releases are routine and can happen any day
- [ ] Deploy separated from release with feature flags
- [ ] Commodity capabilities bought or reused; the decision recorded
- [ ] Work in progress limited
- [ ] Deliberate shortcuts recorded with a plan to repay them
- [ ] A new engineer can ship a change in their first week

---

## 9. Exercises

1. Take a recent feature you shipped. Write the timeline of each step with working and waiting time. What's the flow efficiency? Which single queue would you remove first?
2. For a new SaaS product, list ten capabilities it needs (login, billing, email, search, admin, analytics…). For each: build, buy or open source, and how many weeks that decision saves or costs.
3. Three features with different values and durations compete for one team. Compute the delay cost of two different orders, as in §5.
4. A project is two months late and management offers five more engineers. Explain what will happen in the first month and propose alternatives.

---

## 10. Further reading

Nicole Forsgren, Jez Humble & Gene Kim, *Accelerate*, and the yearly DORA "State of DevOps" reports (dora.dev); Donald Reinertsen, *The Principles of Product Development Flow* (cost of delay, queues, batch size); Eric Ries, *The Lean Startup*; Fred Brooks, *The Mythical Man-Month*; Matthew Skelton & Manuel Pais, *Team Topologies*; Jez Humble & David Farley, *Continuous Delivery*; Dan McKinley, "Choose Boring Technology"; Martin Fowler, "Technical Debt Quadrant".
