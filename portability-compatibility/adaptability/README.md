# Adaptability

> **Adaptability** = how well a system **adjusts to different or changing environments** (hardware size, network quality, available services, deployment model, region) without code changes or a separate fork.
> "The same build runs on a 512 MB edge box and a 64-core server, online and offline, in our cloud and in the customer's. Does it behave sensibly in each?"

Related: [Portability](../portability/README.md) (can it run there at all?), [Configurability](../../operations-devops/configurability/README.md) (the main tool), [Elasticity](../../runtime-operational/elasticity/README.md) (adapting to *load*), [Evolvability](../../development-evolution/evolvability/README.md) (adapting to *requirements* over years), [Internationalization](../../UI-UX/internationalization/README.md) (adapting to languages and locales).

---

## 1. What the environment can vary in

| Dimension | Range | What should adapt |
|---|---|---|
| **Resources** | 0.5 CPU / 512 MB → many cores / hundreds of GB | Thread and connection pool sizes, cache sizes, heap, batch sizes |
| **Network** | Fast LAN → slow mobile → offline | Timeouts, payload sizes, sync strategy, offline queue |
| **Backing services** | Postgres or SQLite; S3 or local disk; Redis present or absent | Which adapter is used; features that depend on an optional service |
| **Deployment model** | Managed cloud, on-premises, air-gapped, edge | Update mechanism, telemetry, licence checks, external calls |
| **Topology** | Single node → cluster → multi-region | Coordination, leader election, where state lives |
| **Platform versions** | OS, kernel, browser, database versions | Use newer features when present, fall back when not |
| **Tenant / region rules** | Data residency, retention, enabled features | Storage location, retention jobs, feature set |
| **Client device** | Phone, tablet, desktop, screen reader | Layout and input ([Responsiveness](../../UI-UX/responsiveness/README.md), [Accessibility](../../UI-UX/accessibility/README.md)) |

---

## 2. Techniques

| Technique | Idea |
|---|---|
| **Configuration, not forks** | One codebase and one artifact; differences live in config. A per-customer branch is the opposite of adaptability |
| **Derive defaults from the environment** | Read the CPU and memory actually available (container limits, not the host) and size pools from them; let config override |
| **Capability detection** | Ask "is this feature available?" rather than "which environment/version is this?" |
| **Ports and adapters** | A storage, queue or mail interface with several implementations chosen at start-up |
| **Graceful degradation** | An optional dependency is missing → the feature turns off cleanly, the rest works |
| **Profiles / presets** | Named bundles of settings (`small`, `standard`, `ha`) instead of fifty separate knobs |
| **Adaptive behaviour at run time** | Adaptive concurrency limits, adaptive bitrate video, backoff based on observed latency |

### Capability detection vs environment detection

```js
// Fragile: guesses from the environment's name
if (navigator.userAgent.includes("Chrome")) useWebShare();

// Robust: asks for the capability
if ("share" in navigator) useWebShare(); else showCopyLinkButton();
```

The same applies on servers: check that the database supports a feature (or try it and fall back) instead of branching on `if cloud == "aws"`.

---

## 3. Worked example: the same image on a small and a large machine

An API ships as one container image. In the cloud it gets 8 CPUs and 16 GB. A customer runs it on-premises with a container limit of 1 CPU and 1 GB, and it is OOM-killed every few minutes.

Cause: fixed settings tuned for the big machine.

```
workers        = 16          # 16 processes × ~150 MB = 2.4 GB  > 1 GB limit
db_pool_size   = 20 per worker → 320 connections
cache_size_mb  = 4096
```

Fix: derive defaults from what the container is actually given, and allow overrides.

```
cpu_limit    = read cgroup CPU quota   (falls back to CPU count)
mem_limit    = read cgroup memory.max  (falls back to system memory)

workers       = clamp(cpu_limit × 2, 1, 32)
db_pool_size  = clamp(DB_MAX_CONNECTIONS / (workers × replicas), 2, 20)
cache_size_mb = mem_limit × 0.25
```

| | Cloud (8 CPU / 16 GB) | On-prem (1 CPU / 1 GB) |
|---|---|---|
| workers | 16 | 2 |
| cache | 4 GB | 256 MB |
| Result | As before | Stable; lower throughput, as expected |

Runtimes differ in how much of this they do for you. Modern JVMs size the heap from the container's memory limit (`-XX:MaxRAMPercentage`); older ones read the host's memory. Go only made `GOMAXPROCS` follow the container CPU limit by default in recent releases; before that, teams set it themselves or used a library. Check what your runtime actually does, and log the values chosen at start-up.

---

## 4. The cost: every option multiplies the test matrix

Three databases × two storage backends × two deployment models × three sizes = 36 combinations. Nobody tests 36.

- Support **few, named configurations** and say which are tested.
- Prefer **automatic adaptation** (reading limits, detecting capabilities) to more knobs.
- Test the **extremes**: the smallest supported size, the missing optional dependency, the offline case.
- Remove adaptability nobody uses ([Simplicity](../../development-evolution/simplicity/README.md)).

---

## 5. Anti-patterns

- A fork or long-lived branch per customer or environment.
- `if env == "production"` scattered through business logic.
- Defaults tuned for the developer's laptop or the largest production machine.
- Reading host CPU and memory inside a container with limits.
- Optional dependencies that crash the whole app when absent.
- So many settings that no two installations behave alike, and none matches what is tested.
- Detecting versions or vendors instead of capabilities.

---

## 6. Checklist

- [ ] One artifact for all environments; differences are configuration
- [ ] Resource-related defaults derive from the limits actually available
- [ ] Chosen values are logged at start-up
- [ ] Optional dependencies degrade cleanly when missing
- [ ] Environment-specific code sits behind adapters, not in business logic
- [ ] Supported configurations are named, documented and tested, including the smallest
- [ ] Capability checks used instead of environment or version sniffing

---

## 7. Exercises

1. Find every setting in a project you built that assumes a machine size. What would each be on a machine a quarter the size?
2. Your product must run with Redis (cloud) and without it (small on-premises installs). Which features degrade, and how does the user find out?
3. A mobile app must work on a train with a connection that drops every few minutes. List what adapts: requests, sync, UI.
4. You support 3 databases and 2 deployment models. Propose a test strategy that doesn't need all combinations on every commit.

---

## 8. Further reading

*The Twelve-Factor App* (factors III "Config" and IV "Backing services"); Alistair Cockburn, "Hexagonal Architecture" (ports and adapters); MDN, "Implementing feature detection"; Kubernetes docs, "Resource Management for Pods and Containers"; Netflix Tech Blog on adaptive concurrency limits; *Release It!* (Michael Nygard) on stability under varying conditions; ISO/IEC 25010 (adaptability).
