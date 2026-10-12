# Elasticity: hands-on practice notes

Planned labs for learning elasticity by **watching capacity follow (or fail to follow) traffic**. Theory is in [README.md](README.md).

They need **kind** or **k3d** (local Kubernetes), **metrics-server**, **KEDA**, and **k6**. Give the cluster a few nodes so scheduling is visible.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** autoscaling basics (scalability lab 14), zero-downtime deploy and graceful shutdown (availability lab 8), connection pool exhaustion (scalability lab 4).

---

## Labs

### 1. Measure the reaction time ⭐
- **Build:** an API Deployment with an **HPA** on CPU (target 60%, min 2, max 20).
- **Run:** k6 traffic that jumps from 100 to 600 RPS instantly.
- **Measure:** a timeline of metric change → HPA decision → pod scheduled → pod ready → latency back to normal. Watch with `kubectl get hpa -w` and the k6 output.
- **Then:** make startup slow (`sleep 30` before ready) and compare. Add a readiness probe and watch errors during the gap.
- **See:** README §4.

### 2. Right metric, wrong metric
- **Build:** an I/O-bound endpoint (mostly waiting on a slow downstream with `sleep`).
- **Break:** HPA on CPU never scales because CPU stays low while latency climbs.
- **Fix:** scale on a custom metric (in-flight requests or RPS per pod) via Prometheus Adapter or KEDA's Prometheus scaler.
- **See:** README §6 "Wrong metric".

### 3. Queue-based scaling to zero ⭐
- **Build:** workers consuming a queue (RabbitMQ or Redis Streams) with **KEDA** scaling on queue length, `minReplicaCount: 0`.
- **Run:** push 5,000 jobs, then nothing for 10 minutes, then a burst.
- **See:** scale up from 0, drain, scale back to 0, and the cold start delay on the next burst.
- **See:** README §3, §6 "Cold starts".

### 4. Safe scale-in
- **Build:** workers that take 60 s per job and an HPA that scales in quickly.
- **Break:** watch jobs get killed mid-way when pods are removed.
- **Fix:** handle `SIGTERM` (stop taking jobs, finish the current one), set `terminationGracePeriodSeconds`, lengthen the scale-down stabilisation window. Re-test.
- **See:** README §5.

---

## Progress

- [ ] 1. Measure the reaction time
- [ ] 2. Right metric, wrong metric
- [ ] 3. Queue-based scaling to zero
- [ ] 4. Safe scale-in
