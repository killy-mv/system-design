# Availability: hands-on practice notes

Planned labs for learning availability by breaking things on purpose and then fixing them. Theory is in [README.md](README.md).

Each lab builds on the previous one. They mostly need **Docker Compose** locally.

**Suggested order:** 1 → 2 → 6 → 4 → 7, then the rest. Lab 6 gives the biggest payoff for the effort.

**Planned folder layout:**

```
runtime-operational/availability/labs/
  shared/probe/            ← the probe script from lab 1, reused everywhere
  01-measure/
    docker-compose.yml
    app/
    README.md              ← steps + expected results
  02-kill-the-server/
  ...
```

---

## Labs

### 1. Measure it
- **Build:** a tiny HTTP app, plus a **probe** script that hits it every 200 ms and logs success rate, latency and calculated "nines".
- **See:** availability measured *from the user's side* (README §1.4).
- **Note:** every other lab uses this probe to measure the impact.

### 2. Kill the single server
- **Build:** 1 app container → `docker kill` it while the probe runs. Then put **Nginx/HAProxy** in front of 3 replicas with health checks and kill one again.
- **See:** a single point of failure versus redundancy. The short burst of errors shows how long health-check detection takes (§3, §4, §7).

### 3. Stateful vs stateless
- **Build:** store sessions in app memory → kill a replica → users get logged out. Move sessions to **Redis**.
- **See:** why the app tier must be stateless (§4.2).

### 4. Database failover
- **Build:** Postgres primary + replica with **Patroni + etcd** (or **Redis + Sentinel**, which is easier). Kill the primary and time the failover in the probe log. Keep writing during the failover and check what was lost.
- **See:** failover time, RPO with async replication, how clients reconnect (§5, §6).

### 5. Split brain
- **Build:** `docker network disconnect` to cut the primary off from the others.
- **See:** who becomes primary, and whether the old primary keeps accepting writes. Quorum, fencing, CAP in practice (§6.3, §15).

### 6. Slow dependency ⭐
- **Build:** use **Toxiproxy** to add 5 s latency to a downstream service.
  1. No timeout → threads pile up → everything dies.
  2. Add a timeout.
  3. Add retries with backoff + jitter.
  4. Add a circuit breaker.
  5. Add a fallback.
- **See:** how failures cascade, and how resilience patterns contain them (§8). Teaches the most.

### 7. Overload
- **Build:** load test with **k6** past capacity. Then add rate limiting (Nginx `limit_req`) and load shedding.
- **See:** overload causing an outage, versus clean 429/503 responses that protect the system (§9).

### 8. Zero-downtime deploy
- **Build:** rolling update while k6 runs, first **without** and then **with** graceful shutdown (handle SIGTERM, drain connections).
- **See:** deploys cause outages, and draining fixes it (§11).

### 9. Stale-if-error
- **Build:** Nginx cache in front of WordPress with `proxy_cache_use_stale error timeout`, then stop MySQL.
- **See:** the site keeps serving pages with the database down: graceful degradation (§8.5, §14.2).

### 10. Kubernetes (optional, later)
- **Build:** with **kind** or **k3d**: liveness/readiness probes, PodDisruptionBudget, `kubectl drain` a node, **Chaos Mesh**. Then deliberately write a liveness probe that checks the DB.
- **See:** the probe anti-pattern from §7.1 happening for real (the DB blips → every pod restarts).

---

## Bonus: mini game day

Have someone else break a container without telling you. Find and fix it using **only your probe and logs**, and time yourself. That's the MTTR part of availability (§1.2, §13).

---

## Tools

| Tool | Used for |
|---|---|
| Docker Compose | Running every lab locally |
| Nginx / HAProxy | Load balancing, health checks, rate limiting, caching |
| Redis (+ Sentinel) | Shared sessions, simple failover |
| Postgres + Patroni + etcd | Database replication and automated failover |
| Toxiproxy | Injecting latency, timeouts, dropped connections |
| k6 | Load testing |
| kind / k3d + Chaos Mesh | Kubernetes probes, disruption, chaos experiments |

## Progress

- [ ] 1. Measure it
- [ ] 2. Kill the single server
- [ ] 3. Stateful vs stateless
- [ ] 4. Database failover
- [ ] 5. Split brain
- [ ] 6. Slow dependency
- [ ] 7. Overload
- [ ] 8. Zero-downtime deploy
- [ ] 9. Stale-if-error
- [ ] 10. Kubernetes
- [ ] Bonus: game day
