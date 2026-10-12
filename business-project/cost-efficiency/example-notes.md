# Cost efficiency: hands-on practice notes

Planned labs for learning cost efficiency by **estimating before building, allocating every dollar to an owner, and doing the arithmetic before optimising**. Theory is in [README.md](README.md).

Most labs need only a **spreadsheet** and a provider's **pricing calculator**. The others use **Docker**, **kind** or **k3d**, **OpenCost**, **Terraform** + **Infracost** (free API key) and **k6**. None of them need a paid cloud account; if you do use one, set a budget alert first (lab 7).

All prices you write down are a snapshot: note the date and region.

**Suggested order:** 1 → 2 → 3 → 4, then the rest in any order.

**Related labs elsewhere:** right-sizing and observability cost (efficiency labs 2 and 4), scaling to zero (elasticity lab 3), cardinality explosion (observability lab 5), building a capacity plan (capacity lab 2), complexity inventory (simplicity lab 1).

**Planned folder layout:**

```
business-project/cost-efficiency/labs/
  01-unit-economics/
    README.md              ← steps + results
    model.xlsx / model.csv
  02-estimate-first/
  ...
```

---

## Labs

### 1. Unit economics model ⭐
- **Build:** a spreadsheet for `messy-shop` (or the observability demo system) with one row per component: app tier, database, cache, queue, logs/metrics/traces, data transfer, CI, non-production. Price each from the pricing calculator at 100k orders a month.
- **Measure:** cost per order, and each component's share. Mark each row as fixed or variable.
- **Then:** model 1M and 10M orders. Does cost per order fall, stay flat or rise? Which row grows fastest?
- **See:** README §3.

### 2. Estimate before building ⭐
- **Do:** write a one-page cost estimate for a new feature: "store an invoice PDF per order, keep it 7 years, let customers download it". Requests × price, GB × price, egress × price, for years 1, 3 and 7.
- **Then:** change the design (compression, storage tiering after 90 days, CDN, generate on demand instead of storing) and re-estimate each variant.
- **Check:** state a cost target ("under $X per order") and say which design meets it.
- **See:** README §4, §6.

### 3. Cost allocation on Kubernetes ⭐
- **Build:** a kind cluster with 3 namespaces (`shop`, `reports`, `platform`), a few deployments with different resource requests, and **OpenCost** installed (it works with default prices on a local cluster).
- **Measure:** cost per namespace and per workload. Find the pod whose requests are far above its usage and compute the idle cost.
- **Break:** deploy something with no `team` label. Add an admission policy (Kyverno or OPA Gatekeeper) that rejects unlabelled workloads.
- **See:** README §5.

### 4. Cost in the pull request
- **Build:** a small Terraform configuration (it doesn't need to be applied) with a few instances, a database and a load balancer. Run **Infracost** locally, then in a GitHub Action that comments the monthly cost difference on pull requests.
- **Try:** a PR that doubles the database size and one that adds a NAT gateway. Is the cost change obvious to a reviewer?
- **See:** README §5.

### 5. Build vs buy TCO
- **No code.** Pick one component (PostgreSQL, Kafka, search, or observability). Build a 3-year TCO comparison of a managed service against self-hosting: infrastructure, engineer time for setup and for running it (upgrades, backups, on-call), training, and exit cost.
- **Then:** find the volume or team size at which the answer flips, and write it down as the trigger for revisiting the decision (an ADR is a good home for it).
- **See:** README §2, §7.

### 6. Commitment planner
- **Build:** a spreadsheet with an hourly instance count for a typical week (use autoscaling results from the elasticity labs if you have them, or invent a day/night curve). Model on-demand only, commit to the minimum, commit to the average, and commit to the peak, at a 30% discount.
- **Measure:** total cost, coverage and utilisation for each. Find the commitment level with the lowest cost.
- **Then:** cut the fleet by 30% (as if you'd right-sized after committing) and see what the over-commitment costs.
- **See:** README §8.

### 7. Budgets and a cost incident drill ⭐
- **Build:** in a cloud free tier or sandbox account: a monthly budget with alerts at 50/80/100% and on forecast, plus anomaly detection if available. Without a cloud account: a script that reads the OpenCost API from lab 3 and alerts when daily cost for a namespace exceeds a threshold.
- **Break:** simulate a runaway: a job in a retry loop that scales a deployment to its maximum. How long until someone would know?
- **Fix:** an autoscaling maximum, a quota on the namespace, and a short runbook: who's alerted, how to stop the spend.
- **See:** README §10.

### 8. Is it worth it?
- **Do:** list ten possible optimisations for the system you modelled in lab 1 (delete idle environments, shorter log retention, cache a hot endpoint, move batch to spot, rewrite a service…). Estimate monthly saving and effort for each.
- **Measure:** payback period for each. Sort by saving ÷ effort.
- **Check:** mark any that reduce redundancy, backups or observability, and decide what they would really cost.
- **See:** README §9.

---

## Progress

- [ ] 1. Unit economics model
- [ ] 2. Estimate before building
- [ ] 3. Cost allocation on Kubernetes
- [ ] 4. Cost in the pull request
- [ ] 5. Build vs buy TCO
- [ ] 6. Commitment planner
- [ ] 7. Budgets and a cost incident drill
- [ ] 8. Is it worth it?
