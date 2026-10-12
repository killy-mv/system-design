# Fault tolerance: hands-on practice notes

Planned labs for learning fault tolerance by **injecting faults and checking that users don't notice**. Theory is in [README.md](README.md).

Most of the core fault tolerance labs are already in the availability notes. This page lists what's **new** here and points to the rest.

**Already covered elsewhere (do these first):**
- killing servers behind a load balancer: availability lab 2
- database failover: availability lab 4
- split brain: availability lab 5
- slow dependency with timeouts, retries, circuit breaker and fallback: **availability lab 6** (the most important one)

They need **Docker Compose**, **Toxiproxy**, and optionally **kind/k3d** + **Chaos Mesh**.

**Suggested order:** 1 → 2 → 3 → 4.

---

## Labs

### 1. Bulkheads ⭐
- **Build:** an API with one shared thread/connection pool calling two dependencies: "payments" (critical) and "recommendations" (not critical).
- **Break:** make recommendations hang with Toxiproxy. Watch the shared pool fill up and **payments fail too**.
- **Fix:** separate pools (or semaphores) per dependency. Re-run: recommendations fail, payments keep working.
- **See:** README §3 "Isolation".

### 2. Let it crash: supervision
- **Build:** a worker process with a bug that crashes on 1 in 1,000 messages. Run it under a supervisor (Docker `restart: always`, systemd, Kubernetes, or an Erlang/Elixir supervisor if you want to try OTP).
- **See:** the service keeps going despite crashes. Then make it crash on startup (a crash loop) and see how the supervisor backs off.
- **Fix:** poison message handling (DLQ) so a single bad input doesn't crash-loop forever.
- **See:** README §3 "Recovery".

### 3. Common-mode fault
- **Build:** 3 replicas behind a load balancer, all reading the same config file.
- **Break:** push a bad config to all 3 at once: total outage despite redundancy.
- **Fix:** a staged rollout script (one replica, health check, wait, next) with automatic rollback on failure. Push the bad config again.
- **See:** README §4 worked example.

### 4. A chaos experiment
- **Build:** a small app on kind/k3d with **Chaos Mesh**.
- **Write a proper experiment:** steady-state hypothesis ("p99 < 300 ms and error rate < 0.1%"), the fault (kill a pod, add 200 ms network delay, partition from the DB), the blast radius, and the abort condition.
- **Run it** and write a short report: did the hypothesis hold? What broke?
- **See:** README §5.

---

## Progress

- [ ] 1. Bulkheads
- [ ] 2. Let it crash: supervision
- [ ] 3. Common-mode fault
- [ ] 4. A chaos experiment
