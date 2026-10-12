# Efficiency

> **Efficiency** = how much useful work the system does **per unit of resource**: CPU, memory, storage, network, energy, and money.
> "Are we doing this work with the least waste? What does each request, user or GB cost us?"

In **ISO/IEC 25010** this is *performance efficiency*: time behaviour, **resource utilisation** and capacity. This page focuses on resource utilisation and cost. For speed see [Performance](../performance/README.md); for maximum load see [Capacity](../capacity/README.md).

Related: [Elasticity](../elasticity/README.md) (don't pay for idle capacity), [Scalability](../scalability/README.md) (cost per unit should stay flat or fall as you grow).

---

## 1. Metrics

| Metric | Example |
|---|---|
| **Utilisation** | Average CPU 15% across the fleet |
| **Work per resource** | Requests/s per core, events/s per GB of RAM |
| **Unit cost** | Cost per 1M requests, per active user, per order, per GB stored |
| **Bytes per request** | 40 KB response, 3 DB round trips per request |
| **Waste** | Idle instances, unattached volumes, unused reserved capacity |
| **Energy / carbon** | kWh or gCO₂e per request (green software) |

**Unit cost is the key one**: total cloud bill alone says nothing; "cost per order went from $0.04 to $0.07" does.

---

## 2. Where waste comes from

- **Over-provisioning**: sized for a peak that happens 2 hours a day, running 24/7.
- **Wrong instance types**: memory-heavy app on CPU-optimised machines, or old generations.
- **Low bin-packing**: Kubernetes pods with requests far above actual use.
- **Inefficient code**: N+1 queries, repeated work, huge JSON payloads, chatty services.
- **Data transfer**: cross-AZ / cross-region traffic and internet egress are often surprisingly expensive.
- **Data kept forever**: logs, metrics with high cardinality, old backups, unused tables.
- **Zombie resources**: forgotten test environments, unattached disks, old snapshots, idle load balancers.
- **Retries and duplicates**: doing the same work several times.

---

## 3. Techniques

| Area | Techniques |
|---|---|
| **Compute** | Right-size; autoscale and scale to zero; spot/preemptible instances for interruptible work; ARM instances (often better price/performance); newer generations |
| **Commitments** | Reserved instances / savings plans for the steady baseline; on-demand only for the variable part |
| **Kubernetes** | Set requests from real usage (VPA recommendations), bin-pack nodes, cluster autoscaler / Karpenter |
| **Code** | Fix the hot paths found by profiling; caching; batching; efficient serialisation; compression |
| **Storage** | Lifecycle rules (hot → infrequent → archive), retention policies, compression, delete unused data |
| **Network** | Keep chatty components in one AZ, CDN for egress, compress responses |
| **Observability** | Sample traces, drop debug logs in prod, limit metric label cardinality |
| **Process (FinOps)** | Tag every resource with owner/team; show each team its cost; budgets and anomaly alerts |

### Worked example: right-sizing a fleet

A service runs **20 instances** at **15% average CPU** (35% at peak). Each instance costs **$140/month** (made-up price).

```
Current cost: 20 × $140 = $2,800/month
Peak demand ≈ 20 × 35% = 7 instances' worth of CPU
```

Target 60% at peak, plus surviving one zone loss across 3 zones:

```
7 / 0.60 ≈ 12 instances for peak
Zone loss: the remaining 2 zones must hold 12 → 6 per zone → 18 total
```

18 × $140 = $2,520/month: only a ~10% saving, because zone redundancy dominates. Better options:
- **Autoscale**: run 18 at peak (2 hours/day) and ~9 off-peak → average ~10 instances → **~$1,400/month**.
- Combine with a **savings plan** on the always-on 9 → another 20–40% off that part.

Lesson: efficiency gains come from **matching capacity to demand over time**, not just from smaller static fleets. And never cut the redundancy that availability depends on.

---

## 4. Trade-offs

| Efficiency vs | Tension |
|---|---|
| Availability | Headroom and redundancy *look* like waste but are insurance |
| Latency | Batching, compression and high utilisation save money but add latency |
| Developer time | Below a certain scale, engineer hours cost more than servers. Optimise when the bill (or energy use) justifies it |
| Simplicity | Spot instances, complex autoscaling and tiering add operational complexity |

---

## 5. Anti-patterns

- Optimising the cloud bill without unit metrics (cost goes up because the business grew, which is fine).
- Cutting redundancy to save money.
- Nobody owns cost; no tags, so nobody knows which team runs what.
- Keeping all logs and metrics forever "just in case".
- Optimising code that isn't on the hot path.

---

## 6. Checklist

- [ ] Unit cost metric defined (per request / user / order) and tracked over time
- [ ] Resources tagged by owner; cost visible per team/service
- [ ] Instances and pod requests right-sized from real usage
- [ ] Autoscaling for variable load; commitments for the steady baseline
- [ ] Storage lifecycle and retention policies
- [ ] Cross-AZ/egress traffic reviewed
- [ ] Budget and anomaly alerts

---

## 7. Exercises

1. Define a unit cost metric for a product you know (food delivery, SaaS, a blog). What would make it go up?
2. A Kubernetes cluster has pods requesting 2 CPU but using 0.3 on average. What's the waste per 100 pods? What's the risk of lowering requests too far?
3. Logs cost more than compute in some companies. Propose a policy: what to keep, at what level, for how long, sampled how?
4. Which workloads in a typical web company can run on spot instances, and which must not?

---

## 8. Further reading

FinOps Foundation (finops.org) framework; AWS Well-Architected **Cost Optimization** and **Sustainability** pillars; Green Software Foundation, "Principles of Green Software Engineering" and the Software Carbon Intensity (SCI) spec; Corey Quinn / Last Week in AWS on cloud cost; Brendan Gregg on CPU utilisation as a misleading metric.
