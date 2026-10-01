# Performance

> **Performance** = how quickly and efficiently a system does its work: **latency** (how long one operation takes) and **throughput** (how many operations per unit time), for a given amount of resources.
> "How fast is it for a user, and how much work does each machine do?"

Related: [Scalability](../scalability/README.md) (keeping performance as load grows), [Availability](../availability/README.md) (a system that's too slow is effectively down), [Maintainability](../maintainability/README.md) (observability is how you find performance problems).

---

## Table of contents

1. [Performance metrics](#1-performance-metrics)
2. [Latency numbers every engineer should know](#2-latency-numbers-every-engineer-should-know)
3. [Methodology: measure, find the bottleneck, fix, repeat](#3-methodology-measure-find-the-bottleneck-fix-repeat)
4. [Anatomy of a request: where time goes](#4-anatomy-of-a-request-where-time-goes)
5. [Caching](#5-caching)
6. [Database performance](#6-database-performance)
7. [Network & protocol performance](#7-network--protocol-performance)
8. [Application & code-level performance](#8-application--code-level-performance)
9. [Concurrency models & I/O](#9-concurrency-models--io)
10. [Batching, pipelining, precomputation, and doing less](#10-batching-pipelining-precomputation-and-doing-less)
11. [Tail latency](#11-tail-latency)
12. [Load testing & benchmarking](#12-load-testing--benchmarking)
13. [Frontend / client performance](#13-frontend--client-performance)
14. [Infrastructure, OS, and hardware](#14-infrastructure-os-and-hardware)
15. [Approaches by system type](#15-approaches-by-system-type)
16. [Trade-offs](#16-trade-offs)
17. [Anti-patterns](#17-anti-patterns)
18. [Checklist](#18-checklist)
19. [Further reading](#19-further-reading)

---

## 1. Performance metrics

### 1.1 Latency vs throughput vs response time
- **Latency**: time an operation spends *waiting to be handled* plus being handled. Often used loosely as "response time".
- **Response time**: what the client sees = network + queueing + service time.
- **Throughput**: operations completed per second (RPS, QPS, TPS, MB/s).
- **Bandwidth**: maximum capacity of a channel; throughput is what you actually achieve.
- **Utilisation**: % of time a resource is busy.
- **Saturation**: amount of queued work a resource can't service yet.

Latency and throughput interact: as utilisation approaches 100%, **queueing delay explodes**.

```
Latency
  │                                   ╱
  │                                 ╱
  │                              ╱
  │                         _ ─
  │ ___________________ ─ ─
  └──────────────────────────────────────── Utilisation
  0%                 ~70%            100%
```
For an M/M/1 queue: `wait ∝ ρ / (1 − ρ)`. At 50% utilisation, waiting ≈ 1× service time; at 90% → 9×; at 99% → 99×. **Keep critical resources well below 100%** (often 60–75% target).

### 1.2 Use percentiles, not averages
Averages hide outliers. Use:
- **p50** (median): typical experience.
- **p95, p99**: the slow experiences — these are often your most valuable users (more data, bigger carts).
- **p99.9, max**: tail latency.

**Tail latency amplification**: if a page calls 100 backends in parallel, each with 1% chance of being slow, then `1 − 0.99¹⁰⁰ ≈ 63%` of page loads hit at least one slow call. The backend's p99 becomes the user's median.

Don't average percentiles across servers — aggregate the raw histograms (HdrHistogram, t-digest, Prometheus histograms).

**Coordinated omission**: load testers that wait for a slow response before sending the next request under-report latency. Use tools that correct for it (wrk2, k6 with constant-arrival-rate, Gatling).

### 1.3 User-centric metrics
- Web: **Core Web Vitals** — LCP (Largest Contentful Paint), INP (Interaction to Next Paint), CLS (Cumulative Layout Shift); also TTFB, FCP.
- Mobile: app start time, frame rate (jank), time to interactive.
- APIs: latency per endpoint at p50/p95/p99; error rate.
- Batch: job duration, records/s, time-to-freshness.
- Perception thresholds: **< 100 ms** feels instant; **< 1 s** keeps flow of thought; **> 10 s** users leave.

### 1.4 Efficiency metrics
- Requests per CPU-core, cost per 1M requests, bytes per request, memory per connection. Performance is also about **doing the same work with fewer resources**.

---

## 2. Latency numbers every engineer should know

Approximate (modern hardware; orders of magnitude matter, not exact values):

| Operation | Time | Relative |
|---|---|---|
| L1 cache reference | ~1 ns | |
| Branch mispredict | ~3 ns | |
| L2 cache reference | ~4 ns | |
| Mutex lock/unlock (uncontended) | ~17 ns | |
| Main memory reference | ~100 ns | 100× L1 |
| Compress 1 KB (Snappy/zstd) | ~2 µs | |
| Read 1 MB sequentially from memory | ~3 µs | |
| Send 1 KB over 10 Gbps network | ~1 µs | |
| Read 4 KB random from NVMe SSD | ~20–100 µs | |
| Round trip within same data center | ~0.5 ms | |
| Read 1 MB sequentially from SSD | ~50–200 µs | |
| Redis GET (same AZ, incl. network) | ~0.2–1 ms | |
| Simple indexed DB query (same AZ) | ~1–5 ms | |
| Disk seek (HDD) | ~2–10 ms | |
| Read 1 MB sequentially from HDD | ~1–5 ms | |
| Round trip cross-AZ | ~1–2 ms | |
| TLS handshake (new connection) | 1–2 RTTs | |
| Round trip US East ↔ US West | ~60–70 ms | |
| Round trip US ↔ Europe | ~80–150 ms | |
| Round trip US ↔ Asia/Australia | ~150–250 ms | |

Takeaways:
- Memory is ~1000× faster than SSD random access; SSD is ~100× faster than HDD seek.
- **Network round trips dominate** most web request latency — minimise the *number* of round trips.
- Sequential I/O is much faster than random I/O.
- You can't beat the speed of light: for global users, move data/compute closer (CDN, edge, regional replicas).

---

## 3. Methodology: measure, find the bottleneck, fix, repeat

> "Premature optimisation is the root of all evil" — but *measured* optimisation of the bottleneck is engineering.

### 3.1 The loop
1. **Define the goal**: e.g., "p99 checkout API < 300 ms at 2,000 RPS".
2. **Measure** the current state under realistic load.
3. **Find the bottleneck** (the one resource limiting performance). Optimising anything else does nothing (Theory of Constraints).
4. **Hypothesise and fix** one thing.
5. **Re-measure**. Keep the change only if it helped.
6. Repeat until the goal is met. Then stop.

### 3.2 Analysis methods

**USE method** (for resources — Brendan Gregg): for every resource (CPU, memory, disk, network, connection pools, thread pools, locks) check
- **U**tilisation, **S**aturation, **E**rrors.

**RED method** (for services):
- **R**ate (requests/s), **E**rrors, **D**uration (latency distribution).

**Four golden signals** (Google SRE): latency, traffic, errors, saturation.

### 3.3 Tools

| Need | Tools |
|---|---|
| Distributed tracing (where does time go across services?) | OpenTelemetry, Jaeger, Zipkin, Tempo, Datadog APM, Honeycomb |
| CPU profiling | perf, async-profiler (JVM), pprof (Go), py-spy (Python), Node `--prof`/clinic, dotnet-trace |
| Flame graphs | Visualise where CPU time is spent (Brendan Gregg) |
| Continuous profiling in prod | Pyroscope, Parca, Datadog/Google Cloud Profiler |
| Memory / GC analysis | heap dumps, GC logs, `jcmd`, pprof heap, Valgrind/massif |
| System-level | `top`/`htop`, `vmstat`, `iostat`, `sar`, `ss`, `netstat`, eBPF tools (bcc, bpftrace) |
| Database | `EXPLAIN ANALYZE`, slow query log, `pg_stat_statements`, Performance Schema |
| Web | Lighthouse, WebPageTest, Chrome DevTools, RUM (real user monitoring) |
| Load generation | k6, Gatling, JMeter, Locust, wrk/wrk2, vegeta |

### 3.4 Common bottlenecks (in rough order of frequency in web systems)
1. Database queries (missing indexes, N+1, lock contention, too much data)
2. Network round trips (chatty APIs, serial calls, cross-region calls)
3. External/third-party APIs
4. Lack of caching / poor cache hit ratio
5. Serialisation (big JSON payloads)
6. Inefficient algorithms / data structures in hot paths
7. Thread/connection pool exhaustion → queueing
8. GC pauses / memory pressure
9. Lock contention
10. Disk I/O, CPU saturation

---

## 4. Anatomy of a request: where time goes

```
User ─► DNS lookup ─► TCP handshake ─► TLS handshake ─► CDN/edge
     ─► Load balancer ─► API gateway (auth, rate limit)
     ─► Service A ─► (cache) ─► (DB query) ─► Service B ─► (external API)
     ─► serialise response ─► compress ─► network ─► browser parse/render
```

Latency budget example for a 300 ms p99 target:

| Step | Budget |
|---|---|
| Network to edge + TLS (with connection reuse) | 30 ms |
| Gateway + auth | 10 ms |
| Service logic | 30 ms |
| Cache lookups | 5 ms |
| DB queries | 80 ms |
| Downstream service | 100 ms |
| Serialisation + response | 20 ms |
| Slack | 25 ms |

Set budgets per component, track them, and make each team own its slice.

**Critical path analysis**: only the longest *sequential* chain matters. Parallelising independent calls shortens it:
```
Serial:   A(50) → B(80) → C(60) = 190 ms
Parallel: A, B, C at once     =  80 ms (slowest)
```

---

## 5. Caching

Caching = keep a copy of data somewhere faster/closer to avoid repeated expensive work.

### 5.1 Cache layers

```
Browser cache → CDN/edge → Reverse proxy cache (Varnish/Nginx) → API gateway cache
  → Application in-process cache (L1) → Distributed cache (Redis/Memcached, L2)
    → Database buffer pool / query cache → OS page cache → Disk controller cache
```
Each layer closer to the user saves more latency but is harder to invalidate.

### 5.2 What to cache
- Results of expensive queries/computations
- Rendered HTML pages/fragments
- API responses from slow/rate-limited third parties
- Session/user profile data
- Configuration, feature flags, permission lookups
- Static assets
- Precomputed aggregates (counts, leaderboards, feeds)

### 5.3 Caching patterns

**Cache-aside (lazy loading)** — most common
```
read:  v = cache.get(k); if miss: v = db.read(k); cache.set(k, v, ttl)
write: db.write(k, v); cache.delete(k)
```
- ✅ Only caches what's used; cache failure → falls back to DB.
- ❌ First request is a miss; stale data possible between write and delete.
- Prefer **delete on write** over update-on-write (avoids race conditions where an old value overwrites a new one).

**Read-through**: the cache itself loads from the DB on miss (library/provider handles it).

**Write-through**: writes go to cache and DB synchronously.
- ✅ Cache always fresh. ❌ Higher write latency; caches data that may never be read.

**Write-behind (write-back)**: write to cache, flush to DB asynchronously.
- ✅ Very fast writes, batching. ❌ Risk of data loss if cache fails before flush.

**Write-around**: write only to DB; cache populated on read. Good for data rarely re-read soon after writing.

**Refresh-ahead**: proactively refresh popular entries before they expire.

### 5.4 Invalidation strategies
> "There are only two hard things in Computer Science: cache invalidation and naming things."

| Strategy | Notes |
|---|---|
| **TTL (time-based)** | Simplest; bounded staleness. Add jitter to TTLs to avoid synchronised expiry. |
| **Explicit invalidation** | Delete on write. Must cover every write path. |
| **Event-driven (CDC)** | Stream DB changes (Debezium) → invalidate/update cache. Catches all writers. |
| **Versioned keys** | `user:123:v7` — bump version on change; old entries expire naturally. |
| **Content-addressed / fingerprinted URLs** | `app.3f9a2c.js` — never invalidate, just reference a new URL. |
| **Tag-based purge** | CDN surrogate keys: purge everything tagged `product-42`. |

### 5.5 Eviction policies
- **LRU** (least recently used) — default for most.
- **LFU** (least frequently used) — better for stable popularity skew.
- **TinyLFU / W-TinyLFU** (Caffeine) — high hit ratio, scan-resistant.
- **FIFO, random** — simple, sometimes fine.
- **ARC** — adapts between recency and frequency.
- Redis: `allkeys-lru`, `allkeys-lfu`, `volatile-ttl`, etc.

### 5.6 Cache problems and fixes

| Problem | Description | Fix |
|---|---|---|
| **Cache stampede / dogpile** | Popular key expires → thousands of requests hit DB at once | Request coalescing (single-flight), locks/leases, probabilistic early expiration (XFetch), stale-while-revalidate |
| **Cache penetration** | Requests for keys that don't exist always miss → hit DB | Cache negative results (short TTL), Bloom filter of existing keys |
| **Cache avalanche** | Many keys expire at the same time, or cache cluster restarts | TTL jitter, warm-up, multi-level cache, cache HA |
| **Hot key** | One key overwhelms one cache node | Local L1 cache, key replication, read replicas |
| **Stale data** | Cache not invalidated correctly | Shorter TTL, CDC-based invalidation, versioned keys |
| **Big keys/values** | Large values block single-threaded Redis, waste network | Split, compress, store in object storage with pointer |
| **Cold start** | Empty cache after deploy/restart | Pre-warming, gradual traffic ramp |

### 5.7 HTTP caching
- `Cache-Control: public, max-age=31536000, immutable` — fingerprinted static assets.
- `Cache-Control: private, no-cache` + `ETag` / `Last-Modified` — revalidate with `304 Not Modified` (saves bandwidth, not round trip).
- `stale-while-revalidate=60` — serve stale while fetching fresh in background.
- `stale-if-error=86400` — serve stale if origin fails (availability bonus).
- `Vary` header — be careful; varying on `Cookie` or `User-Agent` destroys hit rate.

### 5.8 Measuring cache effectiveness
- Hit ratio (overall and per key-pattern), latency of hits vs misses, eviction rate, memory usage, backend load reduction.

---

## 6. Database performance

### 6.1 Indexing
- Index columns used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`.
- **B-tree** (default): equality and range queries, sorting.
- **Hash**: equality only.
- **GIN / inverted**: full-text, JSONB, arrays.
- **GiST / R-tree / SP-GiST**: geospatial, ranges.
- **BRIN**: huge, naturally ordered tables (timestamps) — tiny index.
- **Composite indexes**: column order matters (leftmost prefix rule). Put equality columns first, then range.
- **Covering indexes** (`INCLUDE` columns): query answered from index alone (index-only scan).
- **Partial indexes**: `WHERE status = 'active'` — smaller and faster.
- **Expression indexes**: `LOWER(email)`.
- Costs: every index slows writes and uses storage/memory. Remove unused indexes.

### 6.2 Query optimisation
- Read the plan: `EXPLAIN (ANALYZE, BUFFERS)`. Look for sequential scans on big tables, bad row estimates, nested loops on large sets, sorts spilling to disk.
- **N+1 queries**: one query for a list, then one per item. Fix with joins, `IN (...)` batching, ORM eager loading, DataLoader (GraphQL).
- Select only needed columns (no `SELECT *` on wide tables).
- **Pagination**: keyset/cursor pagination (`WHERE id > last_id LIMIT 50`) instead of `OFFSET 100000` (which scans and discards).
- Avoid functions on indexed columns in `WHERE` (`WHERE DATE(created_at) = ...` can't use index; use a range).
- Keep statistics up to date (`ANALYZE`); watch for parameter-sniffing / generic plan issues.
- Avoid long transactions (hold locks, block vacuum).
- Batch inserts/updates (`INSERT ... VALUES (...), (...)`, `COPY`).

### 6.3 Schema design
- **Normalise** for write integrity; **denormalise** selectively for read performance.
- Appropriate data types (int vs bigint vs UUID, `timestamptz`, avoid giant text in hot tables).
- **Vertical partitioning**: move rarely used/large columns to a separate table.
- **Table partitioning** by time/range for very large tables.
- **Materialised views / summary tables** for expensive aggregations.

### 6.4 Connections and concurrency
- Opening a connection is expensive (TCP + TLS + auth + backend process in Postgres). **Use connection pools**.
- Pool size: more is not better. A common start: `connections ≈ (CPU cores × 2) + effective_spindles` for the DB; queue the rest in the pool. External poolers: PgBouncer, ProxySQL, RDS Proxy.
- **Lock contention**: hot rows (counters), long transactions, `SELECT ... FOR UPDATE`. Use optimistic concurrency, shorter transactions, sharded counters, `SKIP LOCKED` for job queues.
- Choose isolation level consciously (higher = more locking/aborts).

### 6.5 Storage engine characteristics
| | B-tree (InnoDB, Postgres) | LSM-tree (RocksDB, Cassandra, LevelDB) |
|---|---|---|
| Reads | Fast, predictable | Slower (check multiple levels; Bloom filters help) |
| Writes | Random I/O, in-place updates | Sequential, very fast |
| Space amplification | Fragmentation | Compaction overhead |
| Best for | OLTP read-heavy/mixed | Write-heavy, time series |

### 6.6 Database tuning knobs (examples)
- Memory for buffer pool / shared buffers (`innodb_buffer_pool_size`, `shared_buffers` + OS cache) — working set should fit in RAM.
- WAL/redo settings, checkpoint frequency, `fsync`/commit durability trade-offs (`synchronous_commit=off` trades durability for latency).
- Autovacuum tuning (Postgres) to avoid bloat.

### 6.7 OLAP performance
- Columnar storage (Parquet, ClickHouse, warehouses): read only needed columns; great compression.
- Partition pruning, clustering/sort keys, pre-aggregation, approximate queries.

---

## 7. Network & protocol performance

### 7.1 Reduce round trips
- **Connection reuse / keep-alive**: avoid repeated TCP + TLS handshakes.
- **HTTP/2**: multiplexing many requests over one connection, header compression.
- **HTTP/3 (QUIC)**: over UDP, 0-RTT/1-RTT handshakes, no TCP head-of-line blocking — great for mobile/lossy networks.
- **TLS 1.3**: 1-RTT handshake (0-RTT resumption).
- **Batch API calls**; avoid chatty interfaces. GraphQL or BFF aggregation to fetch everything in one request.
- **Co-locate** services that talk a lot (same AZ/region); avoid cross-region calls in the request path.

### 7.2 Reduce bytes
- **Compression**: gzip, **Brotli** (better for text), zstd. Don't compress already compressed media.
- **Efficient serialisation**: Protobuf, FlatBuffers, MessagePack, Avro vs JSON (smaller + faster parse). JSON is fine for most public APIs.
- Send only what's needed: field selection, pagination, sparse fieldsets.
- Image optimisation: WebP/AVIF, responsive sizes, lazy loading.

### 7.3 Get closer to the user
- **CDN** for static and cacheable dynamic content.
- **Edge computing** for personalisation/auth near the user.
- **Multi-region deployment** with geo-routing.
- **Anycast** for DNS and edge.

### 7.4 Protocol choices

| Protocol | Use for |
|---|---|
| REST/HTTP+JSON | Public APIs, simplicity, cacheability |
| gRPC (HTTP/2 + Protobuf) | Internal service-to-service, low latency, streaming |
| GraphQL | Flexible client-driven fetching, fewer round trips (watch for expensive queries) |
| WebSocket | Bidirectional real-time (chat, games, collaboration) |
| Server-Sent Events | Server → client streaming (notifications, LLM token streaming) |
| Long polling | Fallback for real-time |
| WebRTC | Peer-to-peer audio/video/data |
| UDP / custom | Games, real-time media where latency > reliability |

### 7.5 TCP tuning (for high-throughput/long-distance)
- Congestion control (BBR), larger socket buffers, `TCP_NODELAY` (disable Nagle for latency-sensitive small messages), connection pooling.

---

## 8. Application & code-level performance

### 8.1 Algorithms and data structures
- Hot-path complexity matters: O(n²) over 10k items = 100M operations.
- Use the right structure: hash map lookups vs list scans; sets for membership; heaps for top-K; tries for prefix search; Bloom filters for "definitely not present".
- Avoid repeated work inside loops (regex compilation, allocations, I/O).

### 8.2 Memory and garbage collection
- Reduce allocations in hot paths (object pooling, reuse buffers, avoid boxing).
- **GC pauses** cause latency spikes: tune GC (G1/ZGC/Shenandoah for JVM, `GOGC`/`GOMEMLIMIT` for Go), right-size heap, avoid huge heaps when low latency matters.
- Watch for memory leaks (unbounded caches/maps, listeners not removed).
- **Data locality**: contiguous arrays are cache-friendly; pointer-chasing (linked lists, object graphs) is not.

### 8.3 Serialisation and parsing
- JSON encode/decode is often a top CPU consumer in APIs. Use fast libraries (simdjson, orjson, jsoniter), or binary formats internally.
- Stream large responses instead of building them fully in memory.

### 8.4 Lazy and incremental work
- **Lazy loading**: compute/fetch only when needed.
- **Pagination / infinite scroll** rather than loading everything.
- **Incremental computation**: update aggregates on change instead of recomputing from scratch.

### 8.5 Language/runtime choice
- Interpreted languages (Python, Ruby, PHP) are usually fine for I/O-bound web apps; CPU-bound hot paths can be moved to native extensions, compiled services (Go, Rust, Java, C++), or optimised libraries (NumPy).
- JIT warm-up (JVM, .NET, V8) affects first requests after deploy — warm up before taking traffic.
- Cold starts in serverless: keep functions small, provisioned concurrency, lighter runtimes.

### 8.6 Logging and instrumentation overhead
- Excessive synchronous logging in hot paths hurts. Use async logging, sampling, appropriate log levels.

---

## 9. Concurrency models & I/O

### 9.1 Models

| Model | How | Strengths | Weaknesses | Examples |
|---|---|---|---|---|
| **Process per request** | Fork/prefork workers | Isolation | Memory heavy, limited concurrency | PHP-FPM, Gunicorn sync |
| **Thread per request** | Blocking I/O, thread pool | Simple code | Thousands of threads = memory + context switches | Classic Java Servlet, Rails Puma |
| **Event loop (async I/O)** | Single thread, non-blocking I/O, callbacks/promises | 10k–1M connections, low memory | CPU work blocks everything | Node.js, Nginx, Redis, Python asyncio, Netty |
| **Green threads / goroutines / virtual threads** | Lightweight user-space threads scheduled on few OS threads | Simple blocking-style code + high concurrency | Runtime-specific | Go, Erlang/Elixir, Java 21 virtual threads, Kotlin coroutines |
| **Actor model** | Isolated actors exchanging messages | No shared state locks, distribution | Mailbox backpressure | Erlang/Akka/Orleans |

Rule: **I/O-bound → async/lightweight threads; CPU-bound → parallelism across cores/processes** (never block an event loop with CPU work — offload to a worker pool).

### 9.2 Parallelism
- **Parallelise independent I/O calls** (fan-out with `Promise.all`, `asyncio.gather`, goroutines + WaitGroup, CompletableFuture).
- **Data parallelism** for CPU-bound work (split across cores, SIMD, GPU).
- Watch shared state: locks, contention, false sharing. Prefer immutable data, partitioning, lock-free structures where appropriate.

### 9.3 Pools and queues
- Size thread/connection pools using **Little's Law**: `concurrency = throughput × latency`.
- Unbounded queues hide overload and increase latency; prefer bounded queues with backpressure.

### 9.4 Zero-copy and efficient I/O
- `sendfile`, memory-mapped files, `io_uring`, direct I/O for databases, kernel bypass (DPDK) for extreme cases.

---

## 10. Batching, pipelining, precomputation, and doing less

The fastest work is the work you don't do.

| Technique | Example |
|---|---|
| **Batching** | Insert 1,000 rows in one statement; Kafka producer batches; GPU inference batches; DataLoader batching GraphQL resolvers |
| **Pipelining** | Redis pipelining (send many commands without waiting for each reply); HTTP/2 multiplexing |
| **Precomputation** | Precompute feeds, recommendations, search indexes, aggregates, thumbnails at write time or offline |
| **Materialised views** | Store query results; refresh on schedule or change |
| **Denormalisation** | Store author name with post to avoid a join |
| **Async / deferred work** | Send email after response is returned; update analytics via queue |
| **Request coalescing** | Collapse identical concurrent requests into one backend call |
| **Debouncing / throttling** | Search-as-you-type sends a request after 200 ms pause, not on every keystroke |
| **Approximation** | HyperLogLog counts, sampling, cached "about 1.2M results" |
| **Early termination** | Stop scanning when enough results found; `LIMIT` |
| **Compression** | Less data to move |
| **Delta sync** | Send only changes, not full state |

Batching trades latency for throughput: a batch waits to fill. Use **size OR time** triggers (e.g., flush at 500 items or 10 ms, whichever first).

---

## 11. Tail latency

Large-scale systems care about p99/p99.9 because of fan-out amplification (§1.2).

Causes: GC pauses, noisy neighbours, background jobs (compaction, backups), queueing bursts, cache misses, retries, slow disks, network hiccups, lock contention, cold JIT/caches.

Techniques (see "The Tail at Scale", Dean & Barroso):
- **Hedged requests**: send to one replica; if no response within p95 time, send to another; use first reply. Small extra load (~5%) for big tail reductions.
- **Tied requests**: send to two replicas, each tells the other to cancel when it starts processing.
- **Backup / speculative tasks** in batch systems (MapReduce speculative execution).
- **Micro-partitioning**: many small partitions for fine-grained load balancing.
- **Selective replication** of hot data.
- **Latency-aware load balancing** (least outstanding requests, EWMA, power of two choices).
- **Separate background work** from serving (throttle compactions, schedule batch jobs off-peak, separate pools).
- **Timeouts with good budgets**; return partial results ("good enough") instead of waiting for stragglers (search engines do this).
- Reduce GC impact (low-pause collectors, off-heap memory, smaller heaps).

---

## 12. Load testing & benchmarking

### 12.1 Types of tests

| Test | Purpose |
|---|---|
| **Load test** | Behaviour at expected peak |
| **Stress test** | Push beyond capacity to find breaking point and failure mode |
| **Soak / endurance test** | Hours/days at normal load — finds leaks, degradation |
| **Spike test** | Sudden jump in traffic — autoscaling, cold caches |
| **Capacity test** | Max throughput while meeting SLO |
| **Microbenchmark** | Specific function/component (JMH, Go `testing.B`, Criterion, pytest-benchmark) |

### 12.2 Doing it right
- **Production-like environment** (same instance types, data volume, config). Small test DBs give misleadingly fast queries.
- **Realistic traffic mix** (endpoint ratios, payload sizes, user think time, cache hit/miss distribution); replay production traffic if possible.
- **Open model** (constant arrival rate, like real users) rather than closed model (fixed number of users waiting for each response) to avoid coordinated omission.
- **Warm up** before measuring (JIT, caches, connection pools).
- **Measure percentiles** and resource metrics on every tier simultaneously.
- **Increase load gradually** to find the knee of the curve.
- Run in CI for key paths to catch **performance regressions** (compare against baseline).
- Be careful load testing in production (use synthetic accounts, rate-limit, coordinate with on-call).

---

## 13. Frontend / client performance

For user-facing systems, most of the perceived latency is often in the client.

- **Critical rendering path**: inline critical CSS, defer non-critical JS (`defer`, `async`), avoid render-blocking resources.
- **Code splitting & tree shaking**: ship only the JS needed for the current page.
- **Bundle size budgets**; avoid heavy dependencies.
- **Images**: modern formats (AVIF/WebP), responsive `srcset`, correct dimensions, lazy loading, CDN image resizing.
- **Fonts**: subset, `font-display: swap`, preload.
- **Resource hints**: `preconnect`, `dns-prefetch`, `preload`, `prefetch`.
- **Rendering strategies**:
  - **SSR** (server-side rendering) — fast first paint, SEO.
  - **SSG** (static generation) — fastest; pages prebuilt and on CDN.
  - **ISR** (incremental static regeneration) — static with periodic rebuild.
  - **CSR** (client-side rendering) — fast subsequent navigation, slow first load.
  - **Streaming SSR / partial hydration / islands / React Server Components** — reduce JS shipped.
- **Optimistic UI**: update the UI immediately, reconcile with server later.
- **Skeleton screens / progressive loading** improve perceived performance.
- **Service workers** for offline caching; **HTTP caching** of assets.
- **Virtualised lists** for long lists; avoid layout thrashing; use `requestAnimationFrame`; Web Workers for heavy computation.
- **Mobile**: reduce requests on high-latency networks, prefetch on Wi-Fi, local database, background sync, minimise app startup work.

---

## 14. Infrastructure, OS, and hardware

- **Right-size instances**: CPU-optimised, memory-optimised, storage-optimised, network-optimised; ARM (Graviton) often better price/performance.
- **Storage**: NVMe local SSD for latency-critical DBs; provisioned IOPS volumes; separate WAL/log disks.
- **Network**: placement groups, enhanced networking, same-AZ placement for chatty components.
- **Kernel tuning**: file descriptor limits, `somaxconn`, TCP buffers, ephemeral port range, `vm.swappiness`, transparent huge pages (often disable for DBs).
- **CPU**: pinning/affinity, NUMA awareness for large boxes, avoid CPU throttling in containers (CFS quotas cause latency spikes — set requests/limits carefully).
- **Containers/Kubernetes**: resource requests/limits, avoid noisy neighbours, topology-aware routing to keep traffic in-zone.
- **Hardware acceleration**: GPUs/TPUs for ML, SmartNICs, AES-NI for TLS, FPGAs for specialised workloads.

---

## 15. Approaches by system type

### 15.1 Content websites / blogs / CMS (e.g., WordPress)
- Goal: low TTFB and fast LCP for anonymous readers.
- **Full-page cache** (CDN or Varnish/Nginx FastCGI cache) — serves most requests without touching PHP/DB.
- **Object cache** (Redis) for DB query results; **opcode cache** (OPcache).
- Optimise images, minify/defer assets, limit plugins that add queries on every request.
- DB: index meta tables, clean up autoloaded options, avoid expensive `meta_query`.
- SSG/headless for maximum speed.

### 15.2 REST/GraphQL APIs and SaaS backends
- Target p99 per endpoint; trace slow endpoints.
- Fix N+1 (DataLoader for GraphQL), add indexes, cache reads (cache-aside + Redis), paginate.
- Parallelise downstream calls; set timeouts; move slow work async.
- Response compression; avoid over-fetching; ETags for conditional requests.
- GraphQL: query complexity limits, persisted queries (cacheable).

### 15.3 E-commerce
- Product/category pages: CDN + edge caching, SSR/SSG, image optimisation (directly impacts conversion — every 100 ms matters).
- Search: dedicated search engine, autocomplete with prefix indexes and caching.
- Cart/checkout: low-latency key-value stores, minimal third-party scripts on checkout, parallel calls to pricing/inventory/shipping.

### 15.4 Real-time (chat, collaboration, multiplayer)
- Persistent connections (WebSocket), binary protocols, small messages, `TCP_NODELAY`.
- Keep hot state in memory (actor per room/document), avoid DB in the message hot path (write async/batched).
- Collaboration: CRDTs/OT computed locally for instant feedback.
- Games: UDP, client-side prediction, server reconciliation, interpolation, tick rate tuning, regional servers to minimise RTT.

### 15.5 Low-latency trading / ad bidding (µs–ms budgets)
- Everything in memory; avoid GC (or use Rust/C++ or GC-free Java techniques); lock-free ring buffers (LMAX Disruptor); kernel bypass; CPU pinning; co-location with exchanges.
- Ad bidding: strict deadlines (~100 ms total), precomputed models/features in memory, return default bid on timeout.

### 15.6 Search & recommendation
- Precompute candidate sets and embeddings offline; online stage only ranks a small set.
- Inverted indexes and ANN indexes (HNSW, IVF) in memory; caching of popular queries.
- Partial results on timeout; tiered indexes (hot/cold).

### 15.7 Data pipelines / batch / analytics
- Throughput over latency: big batches, columnar formats (Parquet/ORC), predicate pushdown, partition pruning.
- Avoid shuffles (data skew is the classic Spark bottleneck — salt skewed keys), broadcast small tables for joins.
- Incremental processing instead of full recomputation; caching intermediate results.
- Right-size clusters; spot instances for cost.

### 15.8 Streaming systems (Kafka/Flink)
- Throughput: producer batching (`linger.ms`, `batch.size`), compression, partition count, consumer parallelism.
- Latency: smaller batches, fewer hops, tuned checkpoints, state backends (RocksDB vs in-memory).
- Watch consumer lag and backpressure.

### 15.9 Video / media
- Startup time: CDN proximity, small initial segments, prefetch manifest, adaptive bitrate starting low.
- Rebuffering: ABR algorithms, multi-CDN steering.
- Encoding: hardware encoders, parallel chunked transcoding, per-title encoding.

### 15.10 Mobile apps
- Reduce network calls (aggregate APIs/BFF), cache aggressively locally, optimistic UI, background prefetch.
- Small payloads (Protobuf), image sizing per device, HTTP/3 for lossy networks.
- App start time: lazy init, avoid main-thread I/O.

### 15.11 ML / LLM inference
- **Batching** (dynamic/continuous batching), **quantisation** (INT8/FP8/4-bit), model distillation, smaller models where acceptable.
- KV-cache reuse / prompt caching, speculative decoding, streaming tokens (time-to-first-token is the perceived latency).
- GPU utilisation monitoring; keep models warm; place inference near data.
- Cache embeddings and repeated responses (semantic caching).

### 15.12 Databases / storage engines themselves
- Log-structured writes, Bloom filters, block cache, compression, group commit (batch fsyncs), zero-copy reads (mmap), compaction throttling (see KingDB/LevelDB notes).

### Summary table

| System | What "fast" means | Key techniques |
|---|---|---|
| Content/CMS | TTFB, LCP | Page cache, CDN, image optimisation |
| APIs/SaaS | p99 per endpoint | Indexes, no N+1, cache-aside, parallel calls |
| E-commerce | Page load, checkout latency | Edge caching, SSR/SSG, search engine |
| Real-time | Message RTT | WebSockets/UDP, in-memory state, regional servers |
| Trading/ad tech | µs–ms deadlines | In-memory, lock-free, no GC, co-location |
| Search/recs | Query latency | Precompute, in-memory indexes, partial results |
| Batch/analytics | Throughput, job time | Columnar, partition pruning, skew handling |
| Streaming | Lag/end-to-end latency | Batching tuning, partitions, backpressure |
| Video | Startup, rebuffering | CDN, ABR, prefetch |
| Mobile | Perceived speed | Local cache, optimistic UI, fewer requests |
| ML inference | TTFT, tokens/s, cost | Batching, quantisation, caching |

---

## 16. Trade-offs

| Performance technique | What it costs |
|---|---|
| Caching | Staleness (consistency), invalidation complexity, memory |
| Denormalisation / precomputation | Write cost, storage, risk of inconsistent copies |
| Async processing | Eventual consistency; user may not see result immediately |
| Batching | Added latency per item (waiting for batch) |
| Write-behind cache / `synchronous_commit=off` | Durability (data loss on crash) |
| More indexes | Slower writes, more storage |
| Hedged requests | Extra load on backends |
| Binary protocols | Debuggability, tooling, human readability |
| Low-level optimisation (lock-free, manual memory) | Maintainability, bug risk |
| Over-provisioning to keep utilisation low | Cost |
| Edge computing / multi-region | Complexity, consistency, cost |
| Approximate algorithms | Exactness |
| Returning partial results on timeout | Completeness of results |

---

## 17. Anti-patterns

- **Optimising without measuring** (guessing the bottleneck).
- Looking at **averages** instead of percentiles.
- **N+1 queries** hidden by an ORM.
- **Chatty APIs** / serial calls that could be parallel or batched.
- **No timeouts**, leading to thread pool exhaustion.
- `OFFSET` pagination on large tables; `SELECT *`; missing indexes on foreign keys.
- **Caching everything** without an invalidation strategy.
- Running at **~100% utilisation** and wondering why latency is terrible.
- Blocking the **event loop** with CPU work or sync I/O.
- **Unbounded** queues, caches, result sets.
- Load testing with tiny datasets or a closed-loop tool that hides tail latency.
- Huge frontend bundles and unoptimised images.
- Logging every request synchronously at debug level in production.
- Micro-optimising code while the request makes 40 DB queries.

---

## 18. Checklist

**Goals & measurement**
- [ ] Latency SLOs (p50/p95/p99) and throughput targets per critical endpoint/flow
- [ ] Latency histograms (not averages); distributed tracing; continuous profiling
- [ ] Real user monitoring for frontend (Core Web Vitals)

**Data**
- [ ] Indexes support all frequent queries; slow query log reviewed
- [ ] No N+1; cursor pagination; only needed columns
- [ ] Connection pooling sized correctly
- [ ] Working set fits in memory (DB buffer pool / cache)

**Caching**
- [ ] Cache layers identified (browser, CDN, app, distributed)
- [ ] Invalidation strategy per cached item
- [ ] Stampede, penetration, hot-key protections
- [ ] Hit ratio monitored

**Request path**
- [ ] Latency budget per component
- [ ] Independent calls parallelised; round trips minimised
- [ ] Timeouts everywhere; slow work async
- [ ] Compression and efficient serialisation

**Runtime**
- [ ] Appropriate concurrency model; event loop not blocked
- [ ] GC/memory tuned; no leaks under soak test
- [ ] Utilisation kept below the knee (headroom)

**Testing**
- [ ] Load, stress, soak, spike tests in production-like environment
- [ ] Performance regression tests in CI

---

## 19. Further reading

- *Systems Performance* (Brendan Gregg) — the reference on methodology and OS-level analysis; brendangregg.com (USE method, flame graphs)
- *Designing Data-Intensive Applications* (Kleppmann) — ch. 1 (percentiles), ch. 3 (storage engines)
- "The Tail at Scale" (Dean & Barroso, 2013)
- "Latency Numbers Every Programmer Should Know" (Jeff Dean / Peter Norvig, updated versions online)
- *High Performance Browser Networking* (Ilya Grigorik) — free at hpbn.co
- *Use The Index, Luke!* — free at use-the-index-luke.com (SQL indexing)
- web.dev — Core Web Vitals and frontend performance
- Gil Tene, "How NOT to Measure Latency" (coordinated omission)
- Mechanical Sympathy / LMAX Disruptor papers — low-latency design
