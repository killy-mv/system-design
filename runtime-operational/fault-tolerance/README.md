# Fault tolerance

> **Fault tolerance** = the system **keeps working correctly**, possibly in a degraded mode, when some of its components fail.
> "A disk, a server, a dependency or a whole zone just died. Did users notice?"

Related: [Availability §3–§8](../availability/README.md#3-core-principle-eliminate-single-points-of-failure) (redundancy, failover, resilience patterns), [Reliability §3](../reliability/README.md#3-kinds-of-faults) (kinds of faults) and [§14](../reliability/README.md#14-software-fault-tolerance-patterns) (software patterns), [Recoverability](../recoverability/README.md) (what happens after a failure gets through).

---

## 1. Fault → error → failure

```
Fault (a component breaks) → Error (wrong internal state) → Failure (users see wrong or no service)
   disk dies                   write to that disk fails      "upload failed" for users
```

Fault tolerance **stops the chain** before it reaches users.

| Term | Focus |
|---|---|
| **Fault tolerance** | Keep working *through* faults (masking them) |
| **High availability** | Keep downtime low (may include short failover gaps) |
| **Resilience** | Absorb faults *and* recover/adapt afterwards |
| **Graceful degradation** | Keep the core working with reduced features |

---

## 2. Decide what you tolerate (the fault model)

You can't tolerate everything. Write down:
- **Which faults**: crash, slow response (often worse than crash), wrong response, network partition, Byzantine (malicious/arbitrary).
- **How many at once**: survive 1 node? 1 zone? 1 region?
- **Which failure domains**: process < host < rack < zone < region < provider.

Useful numbers:
- Crash faults with consensus: **2f + 1** nodes to survive **f** failures (3 nodes → 1, 5 nodes → 2).
- Byzantine faults: **3f + 1** nodes to survive **f**.

---

## 3. The four building blocks

| Block | Forms | Examples |
|---|---|---|
| **Redundancy** | Spatial (more copies), temporal (try again), information (extra bits) | Replicas, N+1 servers; retries; checksums, ECC RAM, RAID, erasure coding |
| **Isolation** | Contain the fault so it can't spread | Bulkheads, separate pools, cells, processes instead of threads, timeouts |
| **Detection** | Notice quickly | Health checks, heartbeats, watchdogs, checksums, invariant checks |
| **Recovery** | Get back to a good state | Failover, restart ("let it crash"), retry, rollback, compensation, fallback |

Common patterns built from these: timeouts, retries with backoff, circuit breakers, bulkheads, fallbacks, supervision trees, checkpoints, quorum writes, idempotency. Details in [Availability §8](../availability/README.md#8-resilience-patterns-containing-failures).

---

## 4. Correlated failures: the real enemy

Redundancy assumes copies fail **independently**. Often they don't.

### Worked example

3 replicas, each with a 1% chance of being down at any moment, independently:

```
P(all 3 down) = 0.01³ = 0.000001  (one in a million)
```

Now add a **common-mode** fault: a bad config push that hits all replicas at once, 0.5% of the time.

```
P(all down) ≈ 0.000001 + 0.005 ≈ 0.5%
```

The shared fault is **5,000× more likely** than the independent one. More replicas don't help at all. What helps:
- **Staged rollouts** (one replica / zone / cell at a time) for code *and* config.
- **Diversity**: different zones, power, network paths, sometimes different software versions or providers.
- **Cells and shuffle sharding** to limit the blast radius.
- Avoid shared hidden dependencies (same DNS, same auth service, same certificate).

---

## 5. Prove it: fault injection

Fault tolerance that hasn't been tested is a guess.
- **Chaos engineering**: kill instances, add latency, drop packets, fill disks, block a dependency, in staging first, then carefully in production.
- **Game days**: the team rehearses a zone failure or a failover.
- Tools: Chaos Monkey, Chaos Mesh, Litmus, Gremlin, AWS FIS, Toxiproxy (for dependencies).

---

## 6. Anti-patterns

- Replicas that share a failure domain (same host, rack, zone, or config push).
- No timeouts: a slow dependency is not tolerated, it spreads.
- Retries without backoff or budget, turning one fault into overload.
- Failover that has never been triggered.
- Fallbacks that are never exercised and break when finally needed.

---

## 7. Checklist

- [ ] Fault model written down (which faults, how many, which domains)
- [ ] Redundancy spans the failure domain you must survive
- [ ] Timeouts, retries with budget, circuit breakers, bulkheads on dependencies
- [ ] Degraded modes defined for non-critical features
- [ ] Changes rolled out gradually to avoid common-mode faults
- [ ] Faults injected regularly to prove tolerance

---

## 8. Exercises

1. Write the fault model for a project you built: which faults does it survive today? Which single fault takes it down?
2. You need to tolerate 2 simultaneous node failures in an etcd cluster. How many nodes? What happens to writes when 3 fail?
3. List 5 common-mode faults that could take down all replicas of a service at once, and one defence for each.
4. Design a degraded mode for an online store when (a) search is down, (b) recommendations are down, (c) payments are down.

---

## 9. Further reading

*Release It!* (Michael Nygard), stability patterns; *Designing Data-Intensive Applications* ch. 8; Jim Gray, "Why Do Computers Stop and What Can Be Done About It?" (1985); *Principles of Chaos Engineering* (principlesofchaos.org); Amazon Builders' Library "Workload isolation using shuffle-sharding"; Joe Armstrong's thesis on Erlang, "Making reliable distributed systems in the presence of software errors".
