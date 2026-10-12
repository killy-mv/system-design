# Co-existence

> **Co-existence** = a system does its job **while sharing an environment and resources with other software**, without harming the others or being harmed by them.
> "Our service shares a node, a database server and a network with twenty others. When one of them misbehaves, who else suffers?"

Related: [Interoperability](../interoperability/README.md) (the other half of ISO/IEC 25010 *compatibility*: working *with* other systems, not just beside them), [Fault tolerance](../../runtime-operational/fault-tolerance/README.md) (bulkheads), [Capacity](../../runtime-operational/capacity/README.md), [Efficiency](../../runtime-operational/efficiency/README.md), [Backward compatibility](../backward-compatibility/README.md) (two *versions* of the same thing running side by side).

---

## 1. What gets shared, and how it goes wrong

| Shared thing | Conflict |
|---|---|
| **CPU, memory, disk I/O, network** | **Noisy neighbour**: one workload starves the rest |
| **Disk space, inodes** | One service's logs fill the disk; everything on the host fails |
| **Ports and sockets** | Two processes want `:8080`; ephemeral ports exhausted by one client |
| **Files and directories** | Same temp file names, same config path, same lock file |
| **Shared libraries and runtimes** | **Dependency hell**: app A needs `libfoo` 1.x, app B needs 2.x, one global install |
| **Environment variables, global config** | A system-wide setting changed for one app breaks another |
| **A database server** | One tenant's slow query or connection flood affects all; table name clashes in one schema |
| **A message broker or cache** | Key collisions (`user:42` from two apps), one consumer's backlog fills the broker |
| **A cluster** | No quotas: one team's deployment takes all the capacity |
| **A web page** | Global JavaScript variables, CSS leaking between widgets, cookies on a shared domain, third-party scripts blocking the main thread |
| **A phone** | Battery, background execution, storage, notification overload |
| **An upstream API** | A shared rate limit used up by one caller |

---

## 2. Isolation: limiting what a neighbour can do to you

| Level | Mechanism |
|---|---|
| **Dependencies** | Per-application environments (virtualenv, `node_modules`, vendoring), static linking, containers |
| **Process** | Linux **namespaces** (own view of PIDs, network, mounts) and **cgroups** (limits on CPU, memory, I/O): the basis of containers |
| **Stronger boundaries** | Virtual machines, microVMs (Firecracker), sandboxed runtimes (gVisor), WebAssembly, for untrusted or very noisy neighbours |
| **Cluster** | Kubernetes namespaces, requests and limits, `ResourceQuota`, `LimitRange`, priority classes, dedicated node pools |
| **Data stores** | Separate databases or schemas and credentials per application; per-role connection limits and statement timeouts; key prefixes |
| **Application** | **Bulkheads**: separate pools per dependency or tenant; per-tenant rate limits and quotas |
| **Browser** | Modules instead of globals, scoped CSS or Shadow DOM, iframes with `sandbox`, a Content Security Policy |

Isolation is never complete. Containers on one node still share the kernel, the page cache, the disk and the network card.

---

## 3. Being a good neighbour

- **Bound your own usage**: memory limits, pool sizes, log rotation, cache size caps, temp-file clean-up.
- **Make names configurable and prefixed**: ports, paths, queue names, cache keys, metric names.
- **Don't assume you're alone**: no "kill all processes named `python`", no writing to global locations, no changing system-wide settings.
- **Back off**: honour `429` and `Retry-After`; retry with jitter so you don't amplify someone else's incident.
- **Run heavy work politely**: batch jobs at lower priority, throttled, outside peak hours.
- **Follow platform conventions**: XDG directories on Linux, app sandbox rules on mobile, the standard signals for shutdown.

---

## 4. Worked example: a noisy neighbour on a shared node

A reporting job and the checkout API run on the same Kubernetes node. Neither declares resources. Every night when reports start, checkout p99 goes from 120 ms to 2 s, and some checkout pods are killed.

Why: with no requests or limits both pods are **BestEffort**. The report job takes every CPU it can and grows its memory until the node is under pressure; the kubelet then evicts pods, BestEffort first, in no useful order.

```yaml
# checkout: guaranteed what it needs
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { memory: "512Mi" }
---
# reports: allowed to use spare capacity, but capped and first to go
resources:
  requests: { cpu: "200m", memory: "1Gi" }
  limits:   { cpu: "2",    memory: "2Gi" }
priorityClassName: batch-low
```

```yaml
# per-namespace ceiling so one team can't take the cluster
apiVersion: v1
kind: ResourceQuota
metadata: { name: reports-quota, namespace: reports }
spec:
  hard: { requests.cpu: "8", requests.memory: 16Gi, limits.memory: 32Gi }
```

| Change | Effect |
|---|---|
| CPU **requests** | The scheduler places pods only where that CPU is free; under contention CPU is shared in proportion to requests, so checkout keeps its share |
| Memory **limit** on reports | The report job is OOM-killed at 2 Gi instead of pushing the node into memory pressure |
| Low priority class for reports | If something must be evicted, it's the batch job |
| `ResourceQuota` | A mistake in one namespace can't consume the cluster |

Still shared after this: disk I/O, network and the kernel. If reports hammer the same database as checkout, the conflict has only moved there; it needs its own fix (a read replica for reports, a statement timeout, a connection limit for the reporting role).

---

## 5. Anti-patterns

- Workloads with no resource requests or limits on shared nodes.
- One global dependency install for several applications.
- Hard-coded ports, temp paths and cache keys.
- Many applications sharing one database user and schema.
- Analytics queries on the primary that serves user traffic.
- Unbounded logs, caches or queues on a shared disk or broker.
- Retrying without backoff against a shared dependency.
- A third-party script given full access to the page.

---

## 6. Checklist

- [ ] Every shared resource the system uses is listed, with its limit
- [ ] Resource requests and limits set; quotas per team or tenant
- [ ] Dependencies isolated per application
- [ ] Separate credentials, connection limits and timeouts per application on shared data stores
- [ ] Names, ports, paths and key prefixes configurable
- [ ] Own usage bounded (memory, disk, connections, retries)
- [ ] Batch and background work throttled and lower priority than user traffic

---

## 7. Exercises

1. List everything your service shares with other software (node, database, broker, DNS, rate limits). For each, what happens to you if a neighbour uses all of it?
2. Three teams share one Postgres server. Design the isolation: roles, schemas, connection limits, timeouts. What can still go wrong?
3. An embeddable chat widget is loaded on customers' websites. How do you make sure it neither breaks their page nor is broken by it?
4. Explain why setting CPU *limits* can add latency even when the node has idle CPU, and when you'd still set them.

---

## 8. Further reading

Kubernetes docs: "Resource Management for Pods and Containers", "Pod Quality of Service Classes", "Resource Quotas"; Linux kernel documentation on cgroups v2 and namespaces; *Release It!* (Michael Nygard), bulkheads; AWS Well-Architected SaaS Lens on tenant isolation and noisy neighbours; Google's Borg paper (Verma et al., 2015) on sharing machines between production and batch workloads; MDN on Shadow DOM and Content Security Policy; ISO/IEC 25010 (co-existence).
