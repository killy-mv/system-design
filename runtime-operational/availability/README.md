# Availability

> **Availability** = the proportion of time a system is able to correctly serve requests.
> "Can users use it *right now*?"

Related: [Reliability](../reliability/README.md) (does it give *correct* results / not lose data?), [Scalability](../scalability/README.md) (does it stay up *under load*?).

Availability ≠ reliability. A system can be *available* but *unreliable* (it answers, but with wrong/stale data), or *reliable* but *unavailable* (never wrong, but often down for maintenance).

---

## Table of contents

1. [Measuring availability](#1-measuring-availability)
2. [Why systems go down](#2-why-systems-go-down)
3. [Core principle: eliminate single points of failure](#3-core-principle-eliminate-single-points-of-failure)
4. [Redundancy at every layer](#4-redundancy-at-every-layer)
5. [Data replication](#5-data-replication)
6. [Failover](#6-failover)
7. [Failure detection: health checks & heartbeats](#7-failure-detection-health-checks--heartbeats)
8. [Resilience patterns (containing failures)](#8-resilience-patterns-containing-failures)
9. [Overload protection](#9-overload-protection)
10. [Geographic distribution](#10-geographic-distribution)
11. [Safe deployments & change management](#11-safe-deployments--change-management)
12. [Disaster recovery](#12-disaster-recovery)
13. [Operations: monitoring, alerting, incident response](#13-operations-monitoring-alerting-incident-response)
14. [Approaches by system type](#14-approaches-by-system-type)
15. [Trade-offs](#15-trade-offs)
16. [Anti-patterns](#16-anti-patterns)
17. [Checklist](#17-checklist)
18. [Further reading](#18-further-reading)

---

## 1. Measuring availability

### 1.1 The "nines"

```
Availability = uptime / (uptime + downtime)
```

| Availability | Name | Downtime / year | Downtime / month | Downtime / week |
|---|---|---|---|---|
| 90% | one nine | 36.5 days | 72 h | 16.8 h |
| 99% | two nines | 3.65 days | 7.2 h | 1.68 h |
| 99.9% | three nines | 8.77 h | 43.8 min | 10.1 min |
| 99.95% | three and a half | 4.38 h | 21.9 min | 5 min |
| 99.99% | four nines | 52.6 min | 4.38 min | 1 min |
| 99.999% | five nines | 5.26 min | 26.3 s | 6 s |

Rule of thumb: **each extra nine costs roughly 10× more** (engineering effort, infra, operational maturity). At four nines and above, *humans cannot be in the recovery loop* — failover must be automatic.

### 1.2 MTBF and MTTR

```
Availability ≈ MTBF / (MTBF + MTTR)
```

- **MTBF** – Mean Time Between Failures (how often things break).
- **MTTR** – Mean Time To Recovery (how fast you fix it). Often split into:
  - **MTTD** – time to *detect*
  - **MTTA** – time to *acknowledge* (a human picks it up)
  - **MTTM** – time to *mitigate* (users no longer affected)
  - **MTTR** – time to fully *resolve*

Two levers: **fail less often** (raise MTBF) or **recover faster** (lower MTTR). In practice, **lowering MTTR is usually cheaper and more effective** — you can't prevent every failure, but you can detect and recover from them in seconds.

### 1.3 Composing availability

**Serial dependencies (A needs B needs C) — multiply:**

```
A_total = A1 × A2 × A3
99.9% × 99.9% × 99.9% = 99.7%
```

Every hard dependency *lowers* your availability. A service with 10 dependencies at 99.9% each tops out at ~99.0%.

**Parallel redundancy (A or B) — failure probabilities multiply:**

```
A_total = 1 − (1 − A1) × (1 − A2)
1 − (0.001 × 0.001) = 99.9999%
```

This only holds if failures are **independent**. Two replicas in the same rack, on the same power supply, running the same buggy release, are *not* independent (correlated failure). This is why you spread across zones/regions and roll out changes gradually.

### 1.4 SLI, SLO, SLA, error budget

| Term | Meaning | Example |
|---|---|---|
| **SLI** (Indicator) | What you measure | % of HTTP requests returning non-5xx in < 500 ms |
| **SLO** (Objective) | Internal target | SLI ≥ 99.9% over 30 days |
| **SLA** (Agreement) | External contract with penalties | 99.5%, otherwise credits refunded |
| **Error budget** | 100% − SLO | 0.1% ≈ 43 min/month of allowed "badness" |

- SLA should be **looser** than SLO, so you get warned before you owe money.
- **Error budget policy**: while budget remains, ship features fast; when exhausted, freeze risky releases and focus on reliability. This turns "how available should we be?" into an explicit business decision.
- Measure availability **from the user's perspective** (request success rate at the edge / synthetic probes), not "the server process is running".

Request-based availability is usually better than time-based:

```
Availability = successful requests / total valid requests
```

---

## 2. Why systems go down

Knowing failure causes tells you where to invest:

| Cause | Examples | Main defence |
|---|---|---|
| **Changes** (most common, ~70% of outages in many studies) | Bad deploy, config push, schema migration, feature flag | Canary, gradual rollout, fast rollback |
| **Hardware** | Disk, NIC, server, power, rack switch failure | Redundancy, replication |
| **Software bugs** | Memory leak, deadlock, crash loop, poison message | Isolation, restarts, circuit breakers |
| **Dependencies** | Third-party API, DNS, cloud provider, certificate expiry | Timeouts, fallbacks, multi-provider |
| **Overload** | Traffic spike, retry storm, thundering herd, DDoS | Autoscaling, rate limiting, load shedding |
| **Network** | Partition, packet loss, BGP issue, latency spike | Multi-AZ, timeouts, retries |
| **Data** | Corruption, accidental deletion, full disk | Backups, PITR, quotas, alerts |
| **Human error** | Wrong command in prod, fat-finger | Automation, guardrails, least privilege |
| **Disasters** | Zone / region / data center loss | Multi-AZ, multi-region, DR plan |

---

## 3. Core principle: eliminate single points of failure

A **SPOF** is any component whose failure takes down the whole system.

How to find them: draw your architecture and, for every box and arrow, ask *"What happens if this disappears right now?"*

Common hidden SPOFs:
- The single primary database
- The load balancer itself
- DNS provider
- A single NAT gateway / firewall
- The config server / secret store / service discovery
- A shared cache that everything depends on
- Authentication service
- A cron job running on exactly one host
- A single region / single cloud account
- **One person** who knows how to fix it ("bus factor")
- Certificates and domain renewal

---

## 4. Redundancy at every layer

### 4.1 Redundancy types

| Type | Description | Failover time | Cost |
|---|---|---|---|
| **Cold standby** | Backup exists but is off; restore from backup | Hours | Low |
| **Warm standby** | Running, receiving replicated data, not serving traffic | Minutes | Medium |
| **Hot standby** | Fully synced, ready to take over instantly | Seconds | High |
| **Active–active** | All copies serve traffic simultaneously | ~0 (just stop routing to the dead one) | Highest complexity |

**N+1 / N+2 capacity**: if you need N servers to handle peak load, run N+1 (survive one failure) or N+2 (survive one failure *during* maintenance of another). With multi-AZ across 3 zones, each zone must handle 50% of peak so losing one zone is fine (i.e. over-provision by 50%).

### 4.2 Layer by layer

```
           ┌─────────────── DNS (multiple providers / anycast) ───────────────┐
           │                                                                   │
        CDN / Edge (many PoPs)                                                 │
           │                                                                   │
  Load balancers (pair, or managed LB spread across AZs)                       │
           │                                                                   │
  App servers (stateless, N+1, across ≥2–3 AZs, autoscaling group)             │
           │                                                                   │
  Cache cluster (replicated) ── Message queue (replicated) ── Search cluster   │
           │                                                                   │
  Database (primary + synchronous standby in another AZ + read replicas)       │
           │                                                                   │
  Object storage (11 nines durability, cross-region replication)               │
           └───────────────────────────────────────────────────────────────────┘
```

**DNS**
- Use a managed DNS with anycast (Route 53, Cloudflare, NS1); consider two providers for critical domains.
- Keep TTLs moderate (30–300 s) on records you might need to fail over.
- DNS-based failover is coarse: clients/resolvers cache longer than TTL.

**Load balancers**
- Never one LB box. Use managed cloud LBs (internally redundant) or an active–passive pair sharing a **Virtual IP** (VRRP / keepalived).
- L4 (TCP) vs L7 (HTTP) — L7 can do smarter health checks and routing.

**Application servers**
- **Stateless**: no local sessions, no local file uploads. Store session in Redis/DB/JWT, files in object storage. Then any instance is disposable.
- Run in an **autoscaling group** / Kubernetes Deployment with `replicas ≥ 2`, spread across zones (pod anti-affinity / topology spread constraints).
- PodDisruptionBudgets so maintenance never drains all replicas.

**Caches**
- Redis Sentinel / Redis Cluster / managed replication with automatic failover.
- Design so the system **survives cache loss** (degraded performance, not outage). Beware the cold-cache stampede (see [Performance](../performance/README.md)).

**Message queues**
- Kafka: replication factor 3, `min.insync.replicas=2`, `acks=all`.
- RabbitMQ: quorum queues. SQS/PubSub: managed, already redundant.

**Databases** — see next section.

**Background jobs / cron**
- Don't run cron on "that one server". Use a distributed scheduler, or run on multiple nodes with a **leader election / distributed lock** so only one executes.

---

## 5. Data replication

Stateless parts are easy; **state is the hard part of availability**.

### 5.1 Replication topologies

**Single-leader (primary–replica)**
```
        writes
Client ───────► Primary ──replicate──► Replica 1 (reads)
                        └─replicate──► Replica 2 (reads)
```
- Simple, most common (PostgreSQL, MySQL, MongoDB replica sets, Redis).
- On primary failure → promote a replica (failover).
- Writes are unavailable during failover (seconds to minutes).

**Multi-leader**
```
Region A Leader ◄──── async ────► Region B Leader
```
- Each region/data center accepts writes locally → survives region loss and gives low write latency.
- Requires **conflict resolution**: last-write-wins (loses data), merge functions, CRDTs, application-level resolution.
- Used for multi-region, offline-first clients (each device is a "leader"), collaborative editing.

**Leaderless (Dynamo-style)**
```
Client writes to N replicas, succeeds when W acknowledge
Client reads from R replicas, picks newest
If W + R > N → reads see the latest write (quorum overlap)
```
- Cassandra, ScyllaDB, DynamoDB (internally), Riak.
- Very high write availability — no failover needed, any node can accept.
- Tunable: `N=3, W=2, R=2` (balanced), `W=1` (fast writes, weaker), `R=1, W=N` (fast reads).
- Repair mechanisms: **read repair**, **hinted handoff**, **anti-entropy (Merkle trees)**.
- **Sloppy quorum**: accept writes on other nodes during partition → even more available, weaker consistency.

**Consensus-based (Raft / Paxos)**
- etcd, Consul, ZooKeeper, CockroachDB, TiDB, Spanner, YugabyteDB, Kafka KRaft.
- Majority (quorum) must agree: a cluster of `2f+1` nodes tolerates `f` failures (3 nodes → 1 failure, 5 nodes → 2).
- Strongly consistent *and* automatically fails over, but the **minority side of a partition becomes unavailable**.

### 5.2 Synchronous vs asynchronous

| | Synchronous | Asynchronous | Semi-sync |
|---|---|---|---|
| Write acknowledged when | Replica(s) confirmed | Primary only | At least one replica confirmed |
| Data loss on failover (RPO) | Zero | Some (replication lag) | Zero if that replica is promoted |
| Write latency | Higher | Lowest | Medium |
| Availability if replica down | Writes block (unless fallback) | Unaffected | Falls back / blocks |

Common pattern: **one synchronous standby in another AZ + async read replicas** (e.g., AWS RDS Multi-AZ, Postgres `synchronous_standby_names`).

### 5.3 Replication lag consequences

With async replicas, reads may be stale. Solutions (more in [Reliability](../reliability/README.md)):
- **Read-your-writes**: read from primary for a short time after a user writes, or track replication position.
- **Monotonic reads**: pin a user to one replica.
- **Consistent prefix reads**: causally related writes go to same partition.

---

## 6. Failover

### 6.1 Active–passive

```
Normal:   Clients → Primary          (Standby replicating)
Failure:  Clients → Standby (promoted)
```

Steps in an automated failover:
1. **Detect** the primary is dead (missed heartbeats / health checks).
2. **Elect** a new primary (consensus or orchestrator — Patroni, Orchestrator, Sentinel, cloud-managed).
3. **Fence** the old primary so it can't accept writes if it comes back (STONITH — "shoot the other node in the head", revoke its lease, block at network).
4. **Reconfigure** clients (update DNS / VIP / service discovery / proxy like PgBouncer, ProxySQL, HAProxy).
5. **Rebuild** redundancy (create a new standby).

### 6.2 Active–active

All nodes serve traffic. If one dies, the LB stops sending to it. Easy for stateless tiers; hard for stateful tiers (needs multi-leader or leaderless or sharded ownership).

Rule: in active–active with N nodes, each must run at ≤ (N−1)/N of capacity, otherwise losing one overloads the rest → **cascading failure**.

### 6.3 Failover pitfalls

- **Split brain**: network partition → both sides think they are primary → both accept writes → divergent data. Prevent with quorum (majority vote), fencing tokens, leases.
- **Flapping**: failing over back and forth. Use hysteresis (require N consecutive failures; don't auto-fail-back).
- **False positives**: a slow (GC pause) node looks dead → unnecessary failover, possibly split brain. Tune timeouts; use a witness/quorum node.
- **Data loss**: async replica promoted while behind. Know your RPO.
- **Untested failover doesn't work.** Practice it regularly (game days).

---

## 7. Failure detection: health checks & heartbeats

### 7.1 Types of health checks

| Check | Question | Used by |
|---|---|---|
| **Liveness** | Is the process alive / not deadlocked? | Orchestrator restarts it if failing |
| **Readiness** | Can it serve traffic right now? (warmed up, dependencies reachable) | LB adds/removes it from rotation |
| **Startup** | Has it finished booting? | Delays liveness checks for slow starters |
| **Deep / synthetic** | Can a real user complete a real flow? | External monitoring |

Pitfalls:
- **Liveness check that calls the database** → DB blips → all pods "dead" → restarted simultaneously → total outage. Liveness should check *only the process itself*.
- **Readiness check that depends on a shared dependency** → that dependency fails → every instance pulled from LB → zero capacity. Consider "fail open": if *all* instances are unhealthy, keep routing anyway (AWS ALB does this).
- Health endpoints must be cheap and not share a thread pool that can be exhausted.

### 7.2 Heartbeats and failure detectors

- Nodes send periodic heartbeats; missing K in a row → suspected dead.
- **Phi accrual failure detector** (Cassandra, Akka): outputs a suspicion level based on heartbeat arrival history instead of a fixed timeout — adapts to network jitter.
- **Gossip protocols** (SWIM, used by Consul/Serf, Cassandra): nodes randomly exchange membership info; scales to thousands of nodes with no central monitor.
- **Leases**: a node holds leadership only while it keeps renewing a time-limited lease from a consensus store.

---

## 8. Resilience patterns (containing failures)

Goal: **a failure in one component must not spread** (no cascading failures).

### 8.1 Timeouts
- **Every** network call needs a timeout (connect + read). Default library timeouts are often infinite.
- Set from measured latency: e.g. slightly above dependency's p99.9.
- **Deadline propagation**: pass the remaining time budget downstream (gRPC deadlines), so services don't keep working on requests the client has already given up on.

### 8.2 Retries
- Only retry **transient** errors (timeouts, 503, connection reset), not 400/404.
- Only retry **idempotent** operations — or make them idempotent with an **idempotency key**.
- **Exponential backoff + jitter**: `sleep = random(0, base × 2^attempt)`, capped.
- **Retry budget**: limit retries to e.g. 10% of requests; otherwise retries amplify load during outages (**retry storm**). With 3 layers each retrying 3×, one failure becomes 27 requests.
- Retry at **one** layer only, ideally closest to the user or closest to the failure.

### 8.3 Circuit breaker

```
CLOSED ──(failure rate > threshold)──► OPEN ──(after cooldown)──► HALF-OPEN
   ▲                                                                   │
   └──────────────(trial requests succeed)─────────────────────────────┘
                     (trial requests fail) → back to OPEN
```
- When a dependency is failing, stop calling it — fail fast instead of piling up threads waiting on timeouts.
- Gives the dependency time to recover.
- Libraries: Resilience4j, Polly, Hystrix (retired), Envoy/Istio outlier detection.

### 8.4 Bulkheads
Isolate resources so one failing part can't exhaust everything (named after ship compartments).
- Separate thread pools / connection pools per dependency.
- Separate clusters per customer tier or per feature (**cell-based architecture**: split the system into independent cells, each serving a subset of users; a bad cell affects only its users).
- **Shuffle sharding** (AWS): assign each customer to a random small combination of nodes, so a poison customer only affects the few others sharing exactly that combination.

### 8.5 Fallbacks & graceful degradation
When a dependency fails, return *something useful*:
- Cached / stale data ("last known good").
- Default values (generic recommendations instead of personalized).
- Hide the feature (no "reviews" section, page still loads).
- Queue the work for later ("your order is being processed").
- Read-only mode (writes disabled, reads still work).

Classify features into **critical path** (checkout, login) and **non-critical** (recommendations, analytics). Non-critical dependencies must never take down the critical path.

### 8.6 Asynchronous decoupling
- Put a **queue** between producers and consumers: if the consumer is down, messages wait; producer stays available.
- **Outbox pattern** to reliably publish events alongside DB writes.
- Accept the request, return `202 Accepted`, process in background.

### 8.7 Static stability
A system should keep working in its current state **even if the control plane (config, discovery, autoscaling) is down**. E.g., pre-provision capacity in each AZ instead of relying on launching new instances during an AZ failure (when everyone else is also trying to launch).

---

## 9. Overload protection

An overloaded system is an unavailable system. Often overload *causes* the outage (cascading failure).

| Technique | What it does |
|---|---|
| **Autoscaling** | Add capacity on CPU/RPS/queue depth. Slow (minutes) → not enough for sudden spikes. |
| **Rate limiting** | Cap requests per user/IP/API key (token bucket, leaky bucket, sliding window). Return 429. |
| **Load shedding** | When saturated, reject excess requests early (cheap 503) rather than serving everyone slowly. Prefer to shed low-priority traffic. |
| **Admission control / concurrency limits** | Cap in-flight requests per instance (adaptive limits like Netflix's concurrency-limits, based on latency). |
| **Backpressure** | Bounded queues; when full, signal upstream to slow down instead of buffering infinitely. |
| **Request prioritization** | Critical (checkout) > normal > background (batch, prefetch, analytics). |
| **Queue-based load leveling** | Absorb bursts in a queue; workers drain at a steady rate. |
| **Caching / CDN** | Absorb read traffic at the edge. |
| **DDoS protection** | Cloudflare, AWS Shield, anycast scrubbing. |
| **Waiting room** | Ticket sales / product launches: put users in a virtual queue. |

**Thundering herd / cache stampede**: many clients simultaneously retry or miss cache. Solutions: jittered retries, request coalescing (single-flight), cache locks, stale-while-revalidate.

**Metastable failures**: system stays down even after the trigger is gone (e.g., retries keep it overloaded, cold caches). Recovery may require shedding load aggressively then ramping traffic back up.

---

## 10. Geographic distribution

### 10.1 Failure domains

```
Process < Host < Rack < Availability Zone (data center) < Region < Cloud provider
```
Your redundancy must span the failure domain you want to survive.

| Strategy | Survives | Cost/complexity | Typical target |
|---|---|---|---|
| Single AZ, multiple hosts | Host failure | Low | 99%–99.9% |
| **Multi-AZ** (the standard default) | Data center failure | Moderate (cross-AZ latency ~1–2 ms, data transfer cost) | 99.9%–99.99% |
| Multi-region active–passive | Region failure (with minutes of downtime) | High | 99.99% |
| Multi-region active–active | Region failure with ~no downtime | Very high (data consistency problem) | 99.99%–99.999% |
| Multi-cloud | Provider-wide failure | Extreme | Rarely justified |

### 10.2 Routing users across regions
- **GeoDNS / latency-based DNS** (Route 53).
- **Anycast** (same IP announced from many locations; network routes to nearest; used by CDNs and DNS).
- **Global load balancers** (Google Cloud LB, AWS Global Accelerator, Cloudflare LB).

### 10.3 Data in multi-region
The real difficulty. Options:
- **Single write region, read replicas elsewhere** (writes fail over if the region dies; reads are local).
- **Partition by geography / user home region** — each user's data lives (and is written) in one region; good for data residency laws (GDPR).
- **Multi-leader with conflict resolution** (CRDTs, LWW).
- **Globally consistent DB** (Spanner, CockroachDB) — consensus across regions; writes pay cross-region latency (tens–hundreds ms).

---

## 11. Safe deployments & change management

Since most outages are caused by changes, deployment practice is an availability technique.

| Strategy | How | Rollback |
|---|---|---|
| **Rolling** | Replace instances a few at a time | Roll forward/back gradually |
| **Blue–green** | Stand up full new env (green), switch traffic at once | Switch back to blue instantly |
| **Canary** | Send 1% → 5% → 25% → 100% traffic to new version, watch metrics | Stop and route back |
| **Feature flags** | Ship code dark; enable per user/percentage | Flip flag off (no deploy) |
| **Shadow / dark launch** | Mirror real traffic to new version, discard responses | N/A — no user impact |
| **Zonal / regional waves** | Deploy to one AZ/region at a time, bake, then next | Limits blast radius |

Supporting practices:
- **Automated rollback** when error rate / latency SLO burns during rollout.
- **Backward/forward compatible changes**: old and new versions run simultaneously during rollout.
- **Expand–contract (parallel change) for schema migrations**: add new column → write both → backfill → read new → remove old. Never a breaking migration in one step.
- **Config changes are deploys too** — validate, canary, and roll out gradually.
- **Graceful shutdown**: on SIGTERM, stop accepting new requests, drain in-flight ones, then exit (connection draining).
- **Zero-downtime DB maintenance**: online schema change tools (gh-ost, pt-online-schema-change), `CREATE INDEX CONCURRENTLY`.

---

## 12. Disaster recovery

### 12.1 RPO and RTO

```
           RPO                         RTO
  ◄─────────────────────►  ◄─────────────────────►
──●──────────────────────✖──────────────────────●──►  time
 last good             disaster              service
 backup/replica                              restored
```

- **RPO (Recovery Point Objective)**: max acceptable **data loss** (time). RPO = 0 requires synchronous replication.
- **RTO (Recovery Time Objective)**: max acceptable **downtime**.

### 12.2 DR strategies (AWS terminology, widely used)

| Strategy | RTO | RPO | Cost | Description |
|---|---|---|---|---|
| **Backup & restore** | Hours–days | Hours | $ | Backups in another region; rebuild everything on disaster |
| **Pilot light** | 10s of minutes | Minutes | $$ | Core data replicated live; compute off, launched on demand |
| **Warm standby** | Minutes | Seconds–minutes | $$$ | Scaled-down full copy running; scale up on disaster |
| **Multi-site active–active** | ~0 | ~0 | $$$$ | Full capacity in multiple regions serving traffic |

### 12.3 Backups
- **3-2-1 rule**: 3 copies, 2 different media, 1 off-site. Add: **1 immutable/air-gapped** copy (ransomware) and **0 errors** on restore tests.
- **Point-in-time recovery (PITR)**: base backup + WAL/binlog archive.
- Replication is **not** a backup — `DROP TABLE` replicates instantly.
- **Test restores regularly.** An untested backup is a hope, not a backup.
- Use **infrastructure as code** so the environment itself can be recreated.

---

## 13. Operations: monitoring, alerting, incident response

This is how you lower MTTR.

- **Monitor the four golden signals**: latency, traffic, errors, saturation (see [Maintainability](../../development-evolution/maintainability/README.md) for observability in depth).
- **Synthetic monitoring**: external probes hitting real user flows from multiple locations.
- **SLO burn-rate alerts**: alert when you're consuming error budget too fast (e.g., 2% of monthly budget in 1 hour), not on every CPU spike.
- **Runbooks** for every alert: what it means, how to diagnose, how to mitigate.
- **On-call rotation** with escalation.
- **Incident process**: incident commander, communication channel, status page, mitigation first (rollback!) then root cause.
- **Blameless postmortems** with action items.
- **Chaos engineering**: deliberately inject failures (kill instances, add latency, drop AZ) to verify the system copes — Chaos Monkey, Gremlin, AWS FIS, Litmus. Start in staging, then controlled production experiments.
- **Game days**: rehearse failover and DR with the team.

---

## 14. Approaches by system type

Different systems need very different availability strategies. Always start from: *"What does downtime cost, and is it worse to be down or to be wrong?"*

### 14.1 Static website / blog / documentation
- **Target**: 99.9%+ is cheap here.
- Host on **object storage + CDN** (S3 + CloudFront, Netlify, Vercel, GitHub Pages, Cloudflare Pages). CDN serves cached copies even if origin is down.
- No servers to fail. Main risks: DNS, certificate expiry, CDN provider outage.

### 14.2 CMS / content site (e.g., WordPress, news site)
- Heavy reads, few writes → **full-page caching + CDN** with `stale-if-error` so cached pages are served when origin is down.
- Multiple stateless web nodes behind LB; uploads in shared/object storage (not local disk).
- DB: primary + standby (managed Multi-AZ). Read replicas for scaling.
- Admin/editing can tolerate brief downtime; public reads cannot.

### 14.3 Typical web application / SaaS (CRUD)
- **Target**: 99.9%–99.95%.
- Stateless app tier across 2–3 AZs, autoscaling.
- Managed relational DB with Multi-AZ synchronous standby + automated failover.
- Redis with replication; app degrades gracefully if cache lost.
- Queues for emails/notifications/exports.
- Canary/blue–green deploys, expand–contract migrations.
- Multi-tenant SaaS: **cell-based architecture** limits blast radius; noisy-neighbor protection via per-tenant rate limits.

### 14.4 E-commerce
- Downtime = directly lost revenue; peak events (Black Friday) are when failures are most likely.
- Separate **critical path** (browse → cart → checkout → payment) from non-critical (recommendations, reviews, analytics) with bulkheads and fallbacks.
- Pre-scale before known events; load test at 2–3× expected peak; waiting room for extreme spikes.
- **Cart**: highly available store (Amazon's Dynamo was literally built so "add to cart" never fails — accept writes, merge conflicts later).
- **Inventory/payments**: correctness matters more → strong consistency, idempotent payment calls, queue orders if payment provider is down.
- Multiple payment providers as fallback.

### 14.5 Banking / payments / financial ledger
- **Consistency > availability** (CP choice). Better to reject a transaction than to double-spend.
- Synchronous replication (RPO = 0), consensus-based databases, strict fencing.
- Availability achieved via: fast automated failover, multi-region active–passive with synchronous replication to a nearby region, maintenance windows avoided via online operations.
- **Idempotency keys** on every money-moving API so retries are safe.
- Degraded modes: "stand-in processing" (approve small card transactions offline with risk limits when the core system is down).
- Regulatory DR requirements (tested failover, defined RTO/RPO).

### 14.6 Social network / news feed / content platform
- **Availability > consistency** (AP choice). A stale feed or late like-count is fine; an error page is not.
- Leaderless/eventually consistent stores (Cassandra), heavy caching, async fan-out.
- Aggressive graceful degradation: if ranking service fails, show chronological; if media service is slow, show text.
- Multi-region active–active, users routed to nearest region.

### 14.7 Real-time messaging / chat
- Persistent connections (WebSocket) → when a server dies, all its clients reconnect at once → **reconnect storm**. Use jittered reconnect backoff.
- Connection/gateway tier is stateless-ish (session state in a store); message storage replicated.
- Clients buffer outgoing messages locally and retry with **client-generated message IDs** (dedupe).
- Delivery via queues/logs (Kafka) so messages aren't lost while a recipient's server is down.

### 14.8 Video / media streaming
- **CDN is the availability layer**: content replicated to hundreds of edge locations; multi-CDN with failover.
- Adaptive bitrate (HLS/DASH) is graceful degradation built-in: lower quality instead of stopping.
- Control plane (login, catalog, playback auth) is the critical path — keep it small and multi-region.
- Netflix-style: chaos engineering, regional evacuation (shift all traffic out of a failing region).

### 14.9 Real-time/live systems (gaming, live sports, trading)
- Low tolerance for failover pauses. Game servers: if a match server dies, the match usually ends — focus on **matchmaking/lobby/account** availability and quick reconnection.
- Trading: hot–hot redundant feeds/matching engines, deterministic replay from an event log to rebuild state instantly.

### 14.10 IoT / edge systems
- Devices must work when **offline** (network is unreliable by default): local buffering, store-and-forward, sync when connected.
- Ingestion endpoint must absorb reconnect bursts → queue-based (MQTT broker clusters, Kafka, Kinesis).
- Edge gateways provide local autonomy.

### 14.11 Mobile / offline-first apps
- The client is a replica: local database (SQLite/Realm), sync engine, conflict resolution (CRDTs / LWW / server-authoritative merge).
- App remains usable with no backend at all.

### 14.12 Batch / data pipelines / analytics
- Availability means "jobs complete before their deadline", not "100% uptime".
- Checkpointing, idempotent/retriable tasks, reprocessing from durable log/raw storage.
- Orchestrators (Airflow, Dagster) with retries; spot-instance interruptions handled by checkpoints.

### 14.13 Internal tools / admin dashboards
- 99%–99.5% is usually fine. Don't over-engineer; single-AZ with backups may be correct.

### 14.14 Infrastructure / control-plane systems (auth, config, service discovery, DNS)
- Everything depends on these → they need **higher** availability than the services using them.
- Consensus clusters (etcd/Consul/ZooKeeper) of 3 or 5 nodes across zones.
- Clients **cache last-known-good** config/credentials so they survive control-plane outages (static stability).
- Auth: long-lived signed tokens (JWT) can be validated locally without calling the auth service on every request.

### Summary table

| System | Priority | Typical target | Key techniques |
|---|---|---|---|
| Static site | A | 99.9%+ | CDN, object storage |
| Content/CMS | A | 99.9% | Page cache, `stale-if-error`, DB standby |
| SaaS/CRUD | A ≈ C | 99.9–99.95% | Multi-AZ, managed DB failover, canaries |
| E-commerce | A (browse/cart), C (payment) | 99.95–99.99% | Bulkheads, fallbacks, pre-scaling |
| Banking/payments | **C** first | 99.95–99.99% | Sync replication, consensus, idempotency |
| Social/feeds | **A** first | 99.95–99.99% | Eventual consistency, multi-region, degradation |
| Chat | A | 99.95% | Durable log, client retries, jittered reconnect |
| Streaming | A | 99.99% (playback) | Multi-CDN, ABR, regional evacuation |
| IoT / mobile | A (offline) | — | Local storage, sync, store-and-forward |
| Batch | Completion | Deadline-based | Checkpoints, idempotent retries |
| Control plane | A (highest) | 99.99%+ | Consensus, client-side caching |

---

## 15. Trade-offs

| Availability technique | What it costs |
|---|---|
| More replicas | **Cost**; more replicas to keep consistent |
| Async replication | **Consistency** (stale reads), **durability** (data loss on failover) |
| Sync replication | **Latency**; writes may block if replica unavailable |
| Multi-region | **Cost**, **complexity**, cross-region **latency**, **consistency** challenges |
| Active–active | **Consistency** (write conflicts), complexity |
| Retries | **Load amplification**; duplicate side effects unless idempotent |
| Serving stale cache on failure | **Correctness** (users see old data) |
| Graceful degradation | **Feature completeness**; more code paths to test |
| Over-provisioning (N+1, 50% headroom per AZ) | **Cost** (idle capacity) |
| Microservices with many dependencies | **Lower** availability (serial multiplication) unless isolated |
| Automatic failover | Risk of **split brain** / false failovers |

**CAP theorem**: during a network **P**artition you choose **C**onsistency (refuse some requests) or **A**vailability (answer, maybe stale/conflicting).
**PACELC**: if **P**artition → **A** or **C**; **E**lse → **L**atency or **C**onsistency.

---

## 16. Anti-patterns

- Measuring availability as "server is up" instead of "users succeed".
- Redundant components that share a hidden dependency (same rack, same DNS, same config push).
- Failover that has never been tested.
- No timeouts; infinite retries; retries at every layer.
- Liveness probes that check downstream dependencies.
- Big-bang deploys to all servers/regions at once.
- Storing state on app servers (sessions, uploads).
- Treating replication as backup.
- Running at 90% capacity with no headroom for losing a node/zone.
- Aiming for five nines on a system whose users would be fine with 99.5%.
- Non-critical features (analytics, ads, recommendations) able to break the critical path.
- Alert fatigue — so many alerts the important one is ignored.

---

## 17. Checklist

**Define**
- [ ] SLIs measured from user perspective
- [ ] SLOs and error budget agreed with the business
- [ ] RPO and RTO defined per data store
- [ ] Critical vs non-critical paths identified

**Architecture**
- [ ] No SPOF (walk every box and arrow)
- [ ] Stateless app tier, ≥2 instances, across ≥2 AZs
- [ ] Database replication with automated, tested failover
- [ ] Capacity to lose one node/AZ without overload
- [ ] Queues between components where async is acceptable
- [ ] Control-plane failure doesn't break the data plane

**Resilience**
- [ ] Timeouts on every network call
- [ ] Retries with backoff + jitter, only on idempotent ops, with a budget
- [ ] Circuit breakers on remote dependencies
- [ ] Bulkheads / separate pools for dependencies
- [ ] Fallbacks for non-critical features
- [ ] Rate limiting and load shedding

**Operations**
- [ ] Canary / gradual rollout with automated rollback
- [ ] Backward-compatible schema migrations
- [ ] Graceful shutdown & connection draining
- [ ] Backups tested via regular restores
- [ ] SLO-based alerts with runbooks
- [ ] On-call, incident process, postmortems
- [ ] Chaos experiments / game days

---

## 18. Further reading

- *Site Reliability Engineering* (Google) — chapters on SLOs, cascading failures, handling overload — free at sre.google
- *The Site Reliability Workbook* (Google) — practical SLO and alerting
- *Release It!* (Michael Nygard) — stability patterns: timeouts, circuit breakers, bulkheads
- *Designing Data-Intensive Applications* (Kleppmann) — ch. 5 (replication), ch. 8 (trouble with distributed systems), ch. 9 (consensus)
- Amazon Builders' Library — "Timeouts, retries, and backoff with jitter", "Static stability using Availability Zones", "Shuffle sharding", "Avoiding overload"
- AWS Well-Architected Framework — Reliability Pillar
- Dynamo paper (2007) — availability-first key-value store
- Netflix Tech Blog — chaos engineering, regional evacuation
