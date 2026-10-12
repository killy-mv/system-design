# Elasticity

> **Elasticity** = the system **automatically adds and removes** capacity to follow demand, quickly enough that users don't notice and you don't pay for idle resources.
> "When traffic doubles at 9am, does capacity follow? When it drops at night, do we stop paying for it?"

Related: [Scalability](../scalability/README.md) (whether capacity *can* grow), [Capacity](../capacity/README.md) (how much is needed), [Efficiency](../efficiency/README.md) (the cost side), [Scalability §15](../scalability/README.md#15-autoscaling--capacity-planning) (autoscaling types).

---

## 1. Elasticity vs scalability

| | Scalability | Elasticity |
|---|---|---|
| Question | Can we grow by adding resources? | Do resources follow demand automatically, both ways? |
| Direction | Mostly up | Up **and down** |
| Time scale | Months (growth) | Minutes (daily cycles, spikes) |
| Benefit | Handle growth | Handle spikes **and** save money |

A system can be scalable but not elastic: it *can* run on 50 servers, but someone has to add them by hand.

---

## 2. Prerequisites

Elasticity only works if adding and removing instances is **safe and fast**:
- **Stateless** instances (sessions, files, jobs stored elsewhere).
- **Fast startup**: small images, quick boot, no long warm-up. Seconds, not minutes.
- **Graceful shutdown**: on scale-in, stop taking new work, finish in-flight requests, then exit.
- **Downstream can absorb it**: databases, caches and third-party APIs must handle the extra instances ([Scalability §15.3](../scalability/README.md#153-worked-example-autoscaling-into-the-database-wall)).
- **Decoupling via queues** for background work, so workers can scale on queue depth.

---

## 3. Mechanisms

| Layer | Examples |
|---|---|
| VMs | AWS Auto Scaling groups, GCP managed instance groups, Azure VM Scale Sets |
| Containers | Kubernetes HPA (pods), VPA (pod size), Cluster Autoscaler / Karpenter (nodes), KEDA (event-driven, scale to zero) |
| Serverless | Lambda, Cloud Run, Cloud Functions, Azure Functions: per-request scaling |
| Data | DynamoDB on-demand, Aurora Serverless, serverless Kafka/queues, read replica autoscaling |

**Policies:** target tracking (keep CPU at 60%), step scaling, **scheduled** (known daily peaks), **predictive** (forecast from history), queue-based (scale on lag/backlog).

---

## 4. The reaction time problem

Scaling out is not instant:

```
Spike starts
 ├─ metrics collected and aggregated            30–60 s
 ├─ autoscaler decides (stabilisation window)   15–60 s
 ├─ new VM / node provisioned                    1–3 min   (a pod on an existing node: seconds)
 ├─ app starts + warms up (JIT, caches, pools)  10 s – 2 min
 └─ passes health checks, joins load balancer   10–30 s
≈ 2–7 minutes until new capacity serves traffic
```

### Worked example

You run at a 60% CPU target. Traffic **doubles within 1 minute** (a push notification, a TV ad).

```
Demand jumps to 120% of current capacity → overloaded for the 2–7 minutes until scale-out finishes
```

Options:
- **More headroom**: target 40–50% instead of 60% (costs more all the time).
- **Scheduled / predictive scaling** before known events (the marketing team knows when the push goes out).
- **Faster scale-out**: warm pools of pre-booted instances, smaller images, pods on spare nodes, serverless for burst.
- **Absorb the gap**: queues, rate limiting and load shedding protect you during those minutes.

---

## 5. Scaling in safely

- **Scale out fast, scale in slowly**: long cooldowns before removing capacity avoid flapping.
- Drain connections and finish jobs before termination (handle `SIGTERM`, `preStop` hooks, termination grace period).
- Keep a **minimum** that survives a zone failure.
- Long-lived connections (WebSockets) and stateful work make scale-in harder: reconnect logic and checkpoints help.

---

## 6. Pitfalls

- **Wrong metric**: scaling on CPU for an I/O-bound service; scale on RPS per instance, latency or queue depth instead.
- **Flapping**: thresholds too close, no cooldown.
- **Cost runaway**: a bug, retry storm or attack scales you to 500 instances. Always set a **max** and a budget alert.
- **Cold starts** in scale-to-zero / serverless: the first requests are slow.
- **Quotas**: the cloud refuses new instances at the worst moment.
- **Everyone scales at once**: during a zone outage, every customer requests capacity in the remaining zones (pre-provision instead: "static stability", [Availability §8.7](../availability/README.md#87-static-stability)).

---

## 7. Checklist

- [ ] Instances stateless, start in seconds, shut down gracefully
- [ ] Scaling metric reflects real load (RPS, latency, queue depth)
- [ ] Min sized for zone failure; max sized for downstream limits and budget
- [ ] Scale-out fast, scale-in slow; cooldowns tuned
- [ ] Scheduled/predictive scaling for known peaks
- [ ] Reaction time measured; headroom or buffering covers it
- [ ] Budget alerts for unexpected scale-out

---

## 8. Exercises

1. Measure (or estimate) the scale-out time of a service you run, step by step as in §4. Which step is the longest, and how could you shorten it?
2. A queue-processing service gets 10 jobs/s at night and 2,000 jobs/s in bursts at noon. Each worker handles 50 jobs/s. Design the scaling policy (metric, target, min, max, cooldowns).
3. What can go wrong if an autoscaler scales in a worker in the middle of a 10-minute video transcoding job? Two fixes.
4. When is serverless a better elasticity choice than Kubernetes HPA, and when is it worse (think cold starts, cost at sustained load, connection limits)?

---

## 9. Further reading

Kubernetes docs: Horizontal Pod Autoscaling, Cluster Autoscaler, Karpenter; KEDA docs; AWS Auto Scaling (target tracking, predictive scaling, warm pools); Amazon Builders' Library "Static stability using Availability Zones"; Google SRE workbook ch. "Managing Load".
