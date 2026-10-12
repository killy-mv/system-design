# Capacity: hands-on practice notes

Planned labs for learning capacity by **measuring it, planning it, and hitting limits on purpose**. Theory is in [README.md](README.md).

They need **Docker Compose**, **k6**, and **PostgreSQL**. Use `--cpus` / `--memory` limits on containers so limits show up clearly on a laptop.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** finding the knee (scalability lab 1), connection pool exhaustion (scalability lab 4), queueing (performance lab 2). Do those first if you haven't.

---

## Labs

### 1. Measure capacity at the SLO ⭐
- **Build:** a simple API + Postgres, the app limited to `--cpus=1`. A k6 script that ramps arrival rate step by step (open model).
- **Measure:** RPS at which p99 crosses 200 ms (capacity) vs RPS at which errors start (crash point). Record both.
- **Repeat** with `--cpus=2`: does capacity double? If not, which resource saturated? (USE method: check CPU, memory, DB connections, disk.)
- **See:** README §1, §2.

### 2. Build a capacity plan
- **No new code.** Use your lab 1 numbers to write a one-page plan in a spreadsheet: growth forecast, per-instance capacity, zone redundancy, safety margin, and the date you'll run out at today's size.
- **Then:** change growth from 8% to 15% per month and see how the "run out" date moves.
- **See:** README §3 worked example.

### 3. Hit hard limits on purpose
- **Break one limit at a time** and note the error message and symptom of each:
  - file descriptors: `ulimit -n 256` in the app container, then open many connections
  - Postgres `max_connections = 20` with a large pool
  - ephemeral ports: a client making short-lived connections to one destination without keep-alive
  - container memory limit: an endpoint that loads a big result into memory (OOM kill)
- **Write a "limits inventory"** with an alert threshold for each.
- **See:** README §2.

### 4. Overload behaviour
- **Build:** push traffic to 2× capacity with and without protection: Nginx `limit_req`, a concurrency limit in the app returning `503`, and a priority rule (health checks and "checkout" always served).
- **Compare:** without protection everything slows down for everyone; with it, most users get fast responses and some get a clean rejection.
- **See:** README §5, [Availability §9](../availability/README.md#9-overload-protection).

---

## Progress

- [ ] 1. Measure capacity at the SLO
- [ ] 2. Build a capacity plan
- [ ] 3. Hit hard limits on purpose
- [ ] 4. Overload behaviour
