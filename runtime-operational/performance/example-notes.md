# Performance: hands-on practice notes

Planned labs for learning performance by **measuring correctly, finding where time goes, and making one request faster**. Theory is in [README.md](README.md).

How these differ from the scalability labs: scalability asks "what breaks when load grows 10×?". Performance asks "why is **this** request slow, and how do I make it faster with the **same** resources?". The tools here are profilers, traces, query plans and percentiles. Some overlap is intentional (N+1 queries, caching); these labs go deeper into the *why*.

Every lab uses the loop from README §3.1:

```
Define the goal → Measure → Find the bottleneck → Fix ONE thing → Re-measure → keep or revert
```

Record p50 / p95 / p99 (never just the average) before and after each change.

They mostly need **Docker Compose** locally, plus **k6** or **wrk2** for load and **PostgreSQL** as the database. Language is up to you. Most labs work in Python, Node or Go. The profiling tools are listed per language.

**Suggested order:** 1 → 3 → 4 → 5 → 7 → 2 → 11, then the rest.

**Planned folder layout:**

```
runtime-operational/performance/labs/
  shared/
    k6/                    ← reusable load scripts (open model, constant arrival rate)
    tracing/               ← OpenTelemetry collector + Jaeger compose file
  01-measure-correctly/
    docker-compose.yml
    app/
    README.md              ← steps + before/after table
  02-queueing-knee/
  ...
```

---

## Labs

### 1. Measure correctly: percentiles and coordinated omission ⭐
- **Build:** a server that answers in 5 ms, but **freezes for 1 s** every 10 s (simulating a GC pause or a stuck lock).
- **Measure it 3 ways:**
  1. Average latency only.
  2. A closed-model tool (`wrk`, or k6 with fixed virtual users): requests wait for the previous one, so the freeze hides most of its impact.
  3. An open-model tool (`wrk2 -R`, or k6 `constant-arrival-rate`): requests keep arriving during the freeze, like real users.
- **See:** the average looks fine, the closed model under-reports, and the open model shows the real p99/p99.9.
- **See:** README §1.2, §12.2. Every later lab depends on measuring this way.

### 2. Queueing: find the knee
- **Build:** a server with **one worker** and a fixed 10 ms service time (`sleep` or a CPU loop).
- **Run:** ramp arrival rate from 10% to 98% of capacity (10 → 98 RPS) and record latency at each step.
- **See:** plot the measured latency against `S / (1 − ρ)` from §1.1. Then add randomness to the service time and watch the knee move earlier.
- **Lesson:** why you keep critical resources at 60–75% utilisation.

### 3. Distributed tracing: find where time goes ⭐
- **Build:** recreate the README §4 worked example: an API that calls 3 small downstream services one after another, and makes N+1 DB queries. Instrument with **OpenTelemetry**, view traces in **Jaeger**.
- **Fix in order:** the N+1, then parallelise the downstream calls, then add a timeout + fallback on the slowest one.
- **See:** the trace waterfall change from a staircase to parallel bars. Compare p99 after each fix, like the table in §4.

### 4. CPU profiling and flame graphs ⭐
- **Build:** an endpoint with hidden hot spots: a regex compiled inside a loop, an O(n²) duplicate check over a list, heavy JSON encoding of a big object.
- **Profile under load:**
  - Python: **py-spy** (`py-spy record -o flame.svg`)
  - Node: **0x** or **clinic flame**
  - Go: **pprof** (`go tool pprof -http`)
- **See:** learn to read a flame graph (width = time). Fix the widest bar, re-profile, repeat.
- **Bonus:** reproduce the Stack Overflow regex incident (§18) with `^[\s‌]+|[\s‌]+$` and a 20,000-space string, then fix it.

### 5. Query plans and indexes ⭐
- **Build:** Postgres with an `orders` table of ~5M rows (generate with `generate_series`).
- **Experiment with `EXPLAIN (ANALYZE, BUFFERS)`**, timing each:
  - no index → index on `customer_id` → composite `(customer_id, created_at)` → covering index with `INCLUDE`
  - a partial index (`WHERE status = 'pending'`)
  - `WHERE DATE(created_at) = ...` (can't use the index) vs a range (can)
  - `OFFSET 100000` vs keyset pagination
- **Also:** enable `pg_stat_statements`, run a mixed workload, and find the top queries by total time.
- **See:** §6.1, §6.2. Then measure the **write cost** of each index you added (§16).

### 6. Blocking the event loop
- **Build:** a Node.js (or Python asyncio) server with a fast `/health` endpoint and a CPU-heavy `/report` endpoint (for example hashing in a loop).
- **Break:** call `/report` while load testing `/health`. Every request stalls, even the trivial ones.
- **Fix:** move CPU work to **worker threads** (Node) or a process pool (Python). Compare `/health` p99 before and after.
- **See:** §9.1, "never block an event loop with CPU work".

### 7. Batching and pipelining ⭐
- **Build:** a script that writes 100,000 items:
  - Redis: one `SET` per round trip → **pipeline** → `MSET`
  - Postgres: one `INSERT` per row (autocommit) → one transaction → multi-row `INSERT` → `COPY`
- **See:** differences of **10–100×**, almost all from fewer round trips and fewer commits (fsyncs). Then add 5 ms of network latency with `tc netem` and watch the gap grow.
- **See:** §2 (round trips dominate), §10.

### 8. Connections and protocols
- **Build:** an Nginx server with TLS. Add realistic latency with `tc qdisc add dev eth0 root netem delay 100ms` inside the client container.
- **Compare:**
  - keep-alive off vs on (new TCP + TLS handshake every request)
  - HTTP/1.1 vs HTTP/2 for a page with 50 small assets
  - TLS 1.2 vs TLS 1.3 handshake time
- **See:** with 100 ms of latency, the **number of round trips** dominates everything (§7.1).

### 9. Serialisation and compression
- **Build:** an endpoint returning a large list (for example 5,000 products).
- **Compare** response size and encode/decode CPU time for: JSON, JSON with only needed fields, gzip, Brotli, MessagePack, Protobuf.
- **See:** that "send fewer fields" often beats "use a faster format". Find where compression stops paying off for small responses (§7.2, §8.3).

### 10. Garbage collection and memory
- **Build:** a service that allocates heavily per request and keeps a large in-memory cache (like the Discord story in §18). Go or Java show this best; Node also works.
- **See:** p99.9 spikes that line up with GC (Go: `GODEBUG=gctrace=1`; Java: GC logs).
- **Fix:** reduce allocations (reuse buffers), tune `GOGC`/`GOMEMLIMIT` (or the JVM collector), move the cache off-heap or shrink it. Compare tail latency.
- **Soak test:** run for 1 hour with a deliberate leak (an unbounded map) and watch memory and latency drift.
- **See:** §8.2, §11.

### 11. Tail latency and hedged requests ⭐
- **Build:** a backend where 1% of requests take 500 ms (the rest 10 ms). An aggregator calls it **20 times in parallel** per user request.
- **See:** the aggregator's **median** is slow, which matches `1 − 0.99²⁰ ≈ 18%` of requests hitting the slow path. Increase to 100 calls (≈ 63%).
- **Fix:** hedged requests: if no reply by the backend's p95, send a second copy to another replica and use whichever answers first. Measure the p99 improvement and the extra load (should be ~5%).
- **See:** §1.2, §11, "The Tail at Scale" in §18.

### 12. HTTP caching in the browser
- **Build:** a small site served by Nginx with different headers per asset: `max-age` + `immutable` on fingerprinted files, `no-cache` + `ETag` on HTML, `stale-while-revalidate` on an API.
- **See:** in DevTools → Network, find "(memory cache)", "(disk cache)", `304 Not Modified` and normal `200` responses. Explain which requests saved a round trip and which only saved bytes.
- **See:** §5.7.

### 13. Frontend: Core Web Vitals
- **Build:** take a page (your WordPress site works well) and make it deliberately slow: a huge hero image, images without dimensions, a blocking script, a long task on click.
- **Measure:** Lighthouse + the DevTools Performance panel with "Slow 4G" and CPU 4× slowdown.
- **Fix:** LCP (image format/size, preload), CLS (`width`/`height` on images, reserved space), INP (split the long task, `requestIdleCallback`, Web Worker). Re-measure each.
- **See:** §1.3, §13. This also connects to the UI-UX responsiveness notes.

### 14. Performance regression tests in CI
- **Build:** a benchmark for one hot function (`pytest-benchmark`, Go `testing.B` + **benchstat**, or **tinybench** for JS) and a k6 script with **thresholds** (`p(95)<200`).
- **Break:** commit a deliberate slowdown and make the CI check fail.
- **See:** §12.2. Performance needs guarding like correctness, or it slowly gets worse.

---

## Bonus: the 60-second Linux analysis

No app needed. Learn the USE method (§3.2) on your own machine with the checklist from Netflix's "Linux Performance Analysis in 60,000 Milliseconds" blog post (Brendan Gregg):

```
uptime · dmesg | tail · vmstat 1 · mpstat -P ALL 1 · pidstat 1 · iostat -xz 1 · free -m · sar -n DEV 1 · sar -n TCP,ETCP 1 · top
```

Run a CPU-heavy, a disk-heavy (`fio` or `dd`) and a network-heavy load in turn. For each, find which command reveals the bottleneck first.

---

## Tools

| Tool | Used for |
|---|---|
| Docker Compose | Running every lab locally |
| k6 / wrk2 | Load generation (open model, correct percentiles) |
| OpenTelemetry + Jaeger | Distributed tracing |
| py-spy / 0x / clinic / pprof | CPU profiling and flame graphs |
| PostgreSQL + `EXPLAIN ANALYZE` + `pg_stat_statements` | Query analysis |
| Redis | Pipelining, batching |
| `tc netem` | Adding network latency and packet loss |
| Nginx | TLS, HTTP/2, HTTP caching headers |
| Lighthouse + Chrome DevTools | Frontend performance |
| pytest-benchmark / Go bench + benchstat / tinybench | Microbenchmarks and regression checks |
| vmstat, iostat, pidstat, sar, perf | System-level analysis (USE method) |

## Progress

- [ ] 1. Measure correctly: percentiles and coordinated omission
- [ ] 2. Queueing: find the knee
- [ ] 3. Distributed tracing: find where time goes
- [ ] 4. CPU profiling and flame graphs
- [ ] 5. Query plans and indexes
- [ ] 6. Blocking the event loop
- [ ] 7. Batching and pipelining
- [ ] 8. Connections and protocols
- [ ] 9. Serialisation and compression
- [ ] 10. Garbage collection and memory
- [ ] 11. Tail latency and hedged requests
- [ ] 12. HTTP caching in the browser
- [ ] 13. Frontend: Core Web Vitals
- [ ] 14. Performance regression tests in CI
- [ ] Bonus: the 60-second Linux analysis
