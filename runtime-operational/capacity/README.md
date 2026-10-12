# Capacity

> **Capacity** = the maximum load a system can handle **while still meeting its SLOs** (latency, error rate).
> "How much traffic can we take before it gets too slow or starts failing, and when do we need more?"

Related: [Throughput](../throughput/README.md) (the rate of work actually completed), [Scalability](../scalability/README.md) (how to *raise* capacity), [Elasticity](../elasticity/README.md) (adjusting capacity automatically), [Performance](../performance/README.md) (latency at a given load).

---

## 1. Capacity vs related terms

| Term | Question |
|---|---|
| **Throughput** | How much work are we doing per second *right now*? |
| **Capacity** | What's the *most* work we can do per second and still meet the SLO? |
| **Headroom** | Capacity − current peak load |
| **Scalability** | Can we raise capacity by adding resources? |
| **Elasticity** | Does capacity follow demand automatically, up and down? |

**Usable capacity ≠ maximum throughput.** A server might reach 1,000 RPS before falling over, but its p99 crosses your 300 ms SLO at 700 RPS. Its capacity is **700**, not 1,000. Find it by load testing to the knee of the latency curve ([Performance §1.1](../performance/README.md#11-latency-vs-throughput-vs-response-time)).

---

## 2. What limits capacity

Capacity is set by the **first resource to saturate**. Making anything else bigger changes nothing.

| Resource | Typical limit |
|---|---|
| CPU | Cores × utilisation target |
| Memory | Working set, cache size, per-connection memory |
| Disk | IOPS, MB/s, space |
| Network | Bandwidth, packets/s, cross-AZ transfer |
| Connections | DB `max_connections`, file descriptors (`ulimit -n`), ephemeral ports (~28k per destination IP:port by default on Linux) |
| Pools | Thread pools, worker processes, connection pools |
| Locks / hot rows | One row or lock everyone needs |
| Quotas | Cloud quotas (vCPUs, IPs, instances), third-party API rate limits, licence limits |
| Partitions | Kafka partitions (max consumer parallelism), shard count |

---

## 3. Capacity planning

```
required capacity = forecast peak × safety margin
instances needed  = required capacity / capacity per instance (at SLO)
                    + redundancy (N+1, or a whole zone)
```

Steps:
1. **Measure capacity per unit** (per instance, per DB, per partition) with a load test.
2. **Forecast demand**: growth trend, seasonality (daily/weekly), known events (launches, sales, holidays).
3. **Add redundancy**: survive losing one node, or one zone, at peak.
4. **Add a safety margin** for forecast error (20–50%).
5. **Check lead times**: things that take weeks (DB migrations, reserved capacity, quota increases, hardware) must be planned months ahead.
6. **Review regularly** (monthly/quarterly) against real usage.

### Worked example

Today's peak is **4,000 RPS**, growing **8% per month**. Plan for 12 months.

```
Peak in 12 months = 4,000 × 1.08¹² ≈ 10,000 RPS
```

Load testing shows one instance hits the knee at 700 RPS. At a 70% target that's **~500 RPS per instance**.

```
Instances for load:  10,000 / 500 = 20
```

Run across **3 zones** and survive losing one at peak: the remaining 2 zones must carry everything.

```
20 instances in 2 zones → 10 per zone → 3 zones × 10 = 30 instances
```

So: **30 instances** in a year, vs 20 if you forgot zone failure. Also check that the database and every downstream system can take 10,000 RPS. The app tier is rarely the real limit ([Scalability §15.3](../scalability/README.md#153-worked-example-autoscaling-into-the-database-wall)).

---

## 4. Utilisation targets and headroom

- Keep critical resources at **60–75%** at peak. Above that, queueing delay grows fast.
- Headroom absorbs: traffic spikes, node or zone loss, autoscaling lag, retries during incidents, deploys (rolling updates temporarily remove capacity).
- Different resources need different targets: CPU can run hotter than a DB connection pool or a disk queue.

---

## 5. When demand exceeds capacity

You **will** exceed capacity sometimes. Decide in advance what happens instead of falling over:
- **Rate limiting** per client and **load shedding** of low-priority traffic (clean `429`/`503`).
- **Queues** to absorb bursts and process later.
- **Graceful degradation**: turn off expensive features (recommendations, search suggestions).
- **Waiting rooms** for known extreme events.

Details in [Availability §9](../availability/README.md#9-overload-protection).

---

## 6. Anti-patterns

- Defining capacity as "max RPS before crashing" instead of "max RPS at SLO".
- Capacity plans that ignore zone failure or deploy-time capacity loss.
- Only planning the app tier; forgetting DB connections, quotas and third-party limits.
- Discovering a cloud quota during an incident.
- Running near 100% "because it's efficient".
- One-time planning with no regular review.

---

## 7. Checklist

- [ ] Capacity per instance/component measured at the SLO, not at the crash point
- [ ] Demand forecast includes growth, seasonality and known events
- [ ] Plan survives losing one node and one zone at peak
- [ ] Inventory of hard limits (quotas, max connections, partitions, ports, API rate limits) with alerts at 70–80%
- [ ] Long-lead-time items planned months ahead
- [ ] Overload behaviour defined (rate limits, shedding, degradation)
- [ ] Capacity reviewed regularly against real usage

---

## 8. Exercises

1. Peak is 2,500 RPS, growing 5% per month. One instance handles 400 RPS at SLO. How many instances do you need in 18 months across 3 zones, surviving one zone loss?
2. List every hard limit in a project you built (DB connections, file descriptors, cloud quotas, API rate limits). Which one would you hit first at 10× traffic?
3. Your Black Friday peak is 6× normal and lasts 4 hours. Compare: permanent over-provisioning, scheduled pre-scaling, autoscaling only. What fails with each?
4. Why does a rolling deploy of 20% of instances at a time need capacity headroom? How much?

---

## 9. Further reading

Google SRE book, ch. 18 "Software Engineering in SRE" (Auxon, intent-based capacity planning) and ch. 21–22 (handling overload, cascading failures); *The Art of Capacity Planning* (John Allspaw); Brendan Gregg, *Systems Performance* (USE method for finding saturated resources); AWS Well-Architected Reliability Pillar ("Manage service quotas and constraints").
