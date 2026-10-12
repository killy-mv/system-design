# Scalability: hands-on practice notes

Planned labs for learning scalability by **load testing, finding the bottleneck, fixing only that, and measuring again**. Theory is in [README.md](README.md).

Every lab uses the same loop:

```
1. Load test → 2. Find the bottleneck (metrics, EXPLAIN, profiler) → 3. Fix one thing → 4. Load test again → compare
```

Keep a small results table in each lab's README (RPS, p50/p99 latency, error rate, CPU of each component). The comparison between runs is the lesson.

They mostly need **Docker Compose** locally, with **k6** for load and **PostgreSQL** as the main database.

**Note:** Docker on one laptop shares one CPU, so absolute numbers mean little. Watch **trends and which component saturates first**. Use `--cpus` limits on containers to make bottlenecks show up clearly.

**Suggested order:** 1 → 3 → 2 → 4 → 5 → 9 → 7, then the rest. Labs 1–5 follow README §8 and §17 (the usual scaling path) in order.

**Planned folder layout:**

```
runtime-operational/scalability/labs/
  shared/
    k6/                    ← reusable load test scripts
    monitoring/            ← Prometheus + Grafana (optional)
  01-baseline-and-knee/
    docker-compose.yml
    app/
    README.md              ← steps + results table
  02-scale-out-app-tier/
  ...
```

---

## Labs

### 1. Baseline: find the knee ⭐
- **Build:** a small API (for example a product catalog: list, get by ID, search, create) + Postgres with ~1M rows of seed data. A k6 script that **ramps** RPS up step by step.
- **See:** the latency curve stays flat, then hits a **knee** and shoots up. Record the max RPS before p99 crosses your target.
- **Check Little's Law** (§1.3): in-flight requests ≈ RPS × latency. Compare with your thread/connection pool sizes.
- **Optional:** add Prometheus + Grafana (or just `docker stats`) to see which container saturates first.

### 2. Scale out the app tier
- **Build:** make the app stateless, put **Nginx** in front, and run 1 → 2 → 4 → 8 replicas (limit each with `--cpus=0.5`).
- **See:** throughput grows... then stops, because the **DB** becomes the bottleneck. Plot throughput vs N and compare with the USL table in §1.4.
- **Bonus:** store sessions in memory first, so you see why stateless matters (§4).

### 3. Fix the database first ⭐
- **Build:** add classic problems on purpose: a missing index, an N+1 query (list products + one query per product's category), `OFFSET` pagination on deep pages, `SELECT *`.
- **Fix:** use `EXPLAIN ANALYZE` and `pg_stat_statements` to find them. Add the index, use a join/batch query, switch to cursor (keyset) pagination.
- **See:** often a **10–100×** win with zero new servers. This is step 1 of the §8 path, and it's the cheapest.

### 4. Connection pool exhaustion
- **Build:** 10+ app replicas each with a pool of 20, against Postgres with `max_connections = 100`.
- **Break:** load test → "too many connections" errors and slower queries.
- **Fix:** add **PgBouncer** in transaction mode. Compare the server-side connection count and latency.
- **See:** the §15.3 worked example happening for real.

### 5. Caching: hit ratio, hot keys, stampedes ⭐
- **Build:** cache-aside with **Redis** for "get product by ID".
- **Measure:** DB QPS at different hit ratios (change the key distribution in k6: uniform vs Zipf/skewed).
- **Break 1, stampede:** give a popular key a short TTL and make the DB query slow (`pg_sleep`). When it expires, hundreds of requests hit the DB at once.
- **Fix:** request coalescing (single-flight), a lock, or early probabilistic refresh.
- **Break 2, cold cache:** `FLUSHALL` under load. Can the DB survive the full load?
- **See:** §6.

### 6. Read replicas
- **Build:** Postgres primary + 2 streaming replicas. Route reads to replicas, writes to the primary.
- **See:** read throughput scales with replicas, and write throughput **doesn't** (every replica applies every write).
- **Note:** replication lag and read-your-writes are covered in reliability lab 6, so you can reuse that setup.
- **See:** §9.

### 7. Sharding by hand ⭐
- **Build:** 3 Postgres containers as shards. A small router in the app: `shard = hash(user_id) mod 3`.
- **Break:** add a 4th shard and write a script that counts how many keys **would move**.
- **Fix:** switch to **consistent hashing** with virtual nodes (or many logical shards mapped to physical ones, like Instagram/Notion). Count again.
- **Then:** try a query by something other than the shard key (scatter–gather) and measure the cost.
- **Optional:** redo it with **Citus** (sharded Postgres) to see what a real system does for you.
- **See:** §10.

### 8. Hot shard / bad shard key
- **Build:** range-partition events by timestamp across 3 shards.
- **Break:** all new writes land on the last shard. Then simulate one "celebrity" user getting 50% of the traffic with a hash shard key.
- **Fix:** hash a better key; salt the hot key (`user_id + random(0..9)`) and read all salted keys.
- **See:** §10.2.

### 9. Async processing with a queue ⭐
- **Build:** "upload image → generate thumbnails" done synchronously in the request (make it CPU-heavy). Then move it to a queue (**RabbitMQ** or **Redis Streams**) + a worker pool, returning `202 Accepted`.
- **Load test with a spike:** compare API latency, and watch the queue absorb the spike and drain afterwards (load leveling).
- **Scale:** add workers and watch queue depth fall.
- **See:** §12.

### 10. Kafka partitions and consumer parallelism
- **Build:** **Redpanda** (Kafka API) with a topic of 3 partitions and a consumer group.
- **See:** run 6 consumers: only 3 do work. Increase partitions and watch keys move between partitions, breaking per-key ordering assumptions.
- **See:** §12 (max parallelism = number of partitions).

### 11. Counters at scale
- **Build:** a "like" endpoint as `UPDATE posts SET likes = likes + 1 WHERE id = 1` on one hot row, under heavy concurrent load.
- **See:** throughput collapses from row lock contention.
- **Fix:** compare (a) sharded counters (N rows, sum on read), (b) Redis `INCR` + periodic flush to the DB. Then try **HyperLogLog** (`PFADD`/`PFCOUNT`) for unique viewers and measure its error vs exact counting.
- **See:** §16.2.

### 12. Fan-out: build a mini feed
- **Build:** users, follows, posts. Implement timelines with **pull** (query followees' posts on read) and **push** (write to each follower's Redis list on post).
- **Measure:** read latency vs write cost. Add a celebrity with 100k followers and watch the push model's write cost explode.
- **Fix:** hybrid: push for normal users, pull for celebrities, merge on read.
- **See:** §16.1, the Twitter story in §21.

### 13. CDN / edge caching
- **Build:** put **Nginx** (`proxy_cache`) or **Varnish** in front of the app as a mini-CDN. Set `Cache-Control` headers, and use fingerprinted asset URLs.
- **Measure:** origin requests with and without caching under the same load. Then purge on "publish".
- **See:** §7, §18.1 (connects to availability lab 9, stale-if-error).

### 14. Autoscaling in Kubernetes (optional, later)
- **Build:** with **kind** or **k3d**: **HPA** on CPU for the API, **KEDA** on queue depth for the workers from lab 9.
- **See:** the lag between a spike and new capacity, scale-in cooldowns, and (if you skipped lab 4) the database wall from §15.3.
- **See:** §15.

---

## Bonus: back-of-the-envelope practice

No code. Estimate traffic, storage and bandwidth for well-known systems (YouTube uploads, WhatsApp messages, Uber location updates) using §2. Then search for the numbers they've published and compare. Being off by 10× is normal at first. Being off by 1000× means a wrong assumption: find it.

Also: take the throughput-vs-N data from lab 2, fit the USL (a spreadsheet is enough, or Python `scipy.optimize.curve_fit`), and predict the peak N.

---

## Tools

| Tool | Used for |
|---|---|
| Docker Compose | Running every lab locally (`--cpus` to create bottlenecks) |
| k6 | Load testing, ramping, skewed key distributions |
| Prometheus + Grafana | Seeing which component saturates (optional) |
| Nginx / Varnish | Load balancing, caching proxy / mini-CDN |
| PostgreSQL + `pg_stat_statements` | Query analysis, replicas, manual shards |
| PgBouncer | Connection pooling |
| Redis | Cache, counters, HyperLogLog, timelines |
| Citus | Sharded Postgres (lab 7 optional) |
| RabbitMQ / Redis Streams / Redpanda | Queues, partitions, consumer groups |
| kind / k3d + HPA + KEDA | Autoscaling |

## Progress

- [ ] 1. Baseline: find the knee
- [ ] 2. Scale out the app tier
- [ ] 3. Fix the database first
- [ ] 4. Connection pool exhaustion
- [ ] 5. Caching: hit ratio, hot keys, stampedes
- [ ] 6. Read replicas
- [ ] 7. Sharding by hand
- [ ] 8. Hot shard / bad shard key
- [ ] 9. Async processing with a queue
- [ ] 10. Kafka partitions and consumer parallelism
- [ ] 11. Counters at scale
- [ ] 12. Fan-out: build a mini feed
- [ ] 13. CDN / edge caching
- [ ] 14. Autoscaling in Kubernetes
- [ ] Bonus: estimation practice + USL fit
