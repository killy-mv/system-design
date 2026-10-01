# Scalability

> **Scalability** = the ability of a system to handle **growing load** (users, requests, data, complexity) by adding resources, while keeping performance acceptable and cost reasonable.
> "If traffic grows 10×, what breaks, and how do we fix it by adding resources rather than rewriting?"

Related: [Performance](../performance/README.md) (how fast is it *for one request*?), [Availability](../availability/README.md) (does it stay up?), [Reliability](../reliability/README.md) (is the data still correct when spread across many machines?).

**Performance vs scalability**:
- *Performance problem*: the system is slow for a single user.
- *Scalability problem*: the system is fast for one user but slow (or broken) under heavy load.

---

## Table of contents

1. [Describing load and measuring scalability](#1-describing-load-and-measuring-scalability)
2. [Back-of-the-envelope estimation](#2-back-of-the-envelope-estimation)
3. [Vertical vs horizontal scaling](#3-vertical-vs-horizontal-scaling)
4. [Stateless services](#4-stateless-services)
5. [Load balancing](#5-load-balancing)
6. [Caching for scale](#6-caching-for-scale)
7. [Content Delivery Networks (CDN)](#7-content-delivery-networks-cdn)
8. [Scaling the database: the usual path](#8-scaling-the-database-the-usual-path)
9. [Replication & read replicas](#9-replication--read-replicas)
10. [Partitioning / sharding](#10-partitioning--sharding)
11. [Choosing the right data store](#11-choosing-the-right-data-store)
12. [Asynchronous processing & message queues](#12-asynchronous-processing--message-queues)
13. [Event-driven architecture & stream processing](#13-event-driven-architecture--stream-processing)
14. [Service decomposition](#14-service-decomposition)
15. [Autoscaling & capacity planning](#15-autoscaling--capacity-planning)
16. [Scaling patterns for specific problems](#16-scaling-patterns-for-specific-problems)
17. [Scaling stages: from 1 user to millions](#17-scaling-stages-from-1-user-to-millions)
18. [Approaches by system type](#18-approaches-by-system-type)
19. [Trade-offs](#19-trade-offs)
20. [Anti-patterns](#20-anti-patterns)
21. [Checklist](#21-checklist)
22. [Further reading](#22-further-reading)

---

## 1. Describing load and measuring scalability

### 1.1 Load parameters
You can't talk about scaling without saying *what* grows. Pick the parameters that matter for your system:

| Parameter | Example |
|---|---|
| Requests per second (RPS / QPS) | 50k reads/s, 2k writes/s |
| Read/write ratio | 100:1 (feed), 1:1 (chat), 1:100 (logging) |
| Concurrent users / connections | 1M open WebSockets |
| Data volume | 50 TB, growing 2 TB/month |
| Data velocity | 1M events/s ingested |
| Fan-out | A celebrity post delivered to 100M followers |
| Object size | 1 KB tweets vs 4 GB videos |
| Hot-spot skew | 1% of keys get 90% of traffic |

Twitter's classic example: the challenge wasn't 4.6k tweets/s writes — it was **fan-out** to 300k timeline reads/s.

### 1.2 Scalability metrics
- **Throughput** vs resources: does doubling servers ~double throughput? (**linear scalability**)
- **Latency under load**: does p99 stay flat as RPS grows?
- **Cost per unit** (per request, per user, per GB): should stay flat or *decrease* with scale.

### 1.3 Laws that limit scaling

**Amdahl's law** — the serial part limits speedup:
```
Speedup(N) = 1 / ( S + (1 − S) / N )
S = serial fraction. If S = 5%, max speedup = 20× no matter how many machines.
```

**Universal Scalability Law (Gunther)** — adds **coherency cost** (nodes coordinating with each other):
```
C(N) = N / ( 1 + α(N − 1) + βN(N − 1) )
α = contention (queuing for shared resource), β = coherency (crosstalk)
```
With β > 0, adding nodes eventually makes throughput **go down**. Lesson: **minimise coordination** (shared locks, distributed transactions, chatty consensus) to scale.

**Little's Law** — concurrency in a system:
```
L = λ × W
in-flight requests = arrival rate × time in system
1,000 RPS × 0.2 s latency = 200 concurrent requests → size thread/connection pools accordingly
```

---

## 2. Back-of-the-envelope estimation

Used to decide *whether* you need a technique before applying it.

### 2.1 Useful numbers

| Quantity | Value |
|---|---|
| Seconds per day | ~86,400 ≈ **10⁵** |
| Seconds per month | ~2.6 × 10⁶ |
| 1M requests/day | ~**12 RPS** avg |
| 100M requests/day | ~1,200 RPS avg |
| Peak / average | typically **2–10×** |
| 1 KB × 1B | 1 TB |

Power-of-two sizes: 2¹⁰ ≈ 1 thousand (KB), 2²⁰ ≈ 1 million (MB), 2³⁰ ≈ 1 billion (GB), 2⁴⁰ ≈ 1 trillion (TB).

### 2.2 Rough capacity of a single component (order of magnitude, varies a lot)

| Component | Rough capacity |
|---|---|
| Web/app server (simple API) | 1k–10k+ RPS per instance |
| PostgreSQL/MySQL on a good box | 10k–50k simple queries/s; a few k writes/s |
| Redis (single instance) | 100k+ ops/s |
| Kafka broker | 100s of MB/s |
| Nginx serving static files | 10k–100k+ RPS |
| One server's open connections | 100k–1M (with tuning, event loop) |

### 2.3 Example estimation: photo-sharing app

```
DAU = 10M; each user views 50 photos/day, uploads 0.2/day
Reads:  10M × 50 / 10⁵  = 5,000 RPS avg → ~25,000 RPS peak
Writes: 10M × 0.2 / 10⁵ = 20 uploads/s avg
Storage: 2M photos/day × 2 MB = 4 TB/day → ~1.5 PB/year (×3 replication)
Bandwidth out: 5,000 × 2 MB = 10 GB/s → must use a CDN
Metadata: 2M rows/day × 1 KB = 2 GB/day → one DB is fine for years for metadata
```
Conclusions: read-heavy → cache + CDN; storage → object storage; metadata DB doesn't need sharding yet.

---

## 3. Vertical vs horizontal scaling

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| How | Bigger machine (CPU, RAM, faster disk) | More machines |
| Complexity | None in code | Distribution: LB, partitioning, consistency |
| Limit | Hardware ceiling (largest instance) | Practically unlimited (if designed for it) |
| Failure | Single point of failure | Redundancy comes naturally |
| Cost curve | Super-linear at the high end | Roughly linear, commodity hardware |
| Downtime to scale | Often requires restart | Add nodes live |
| Best for | Databases early on, stateful systems, quick wins | Stateless tiers, huge scale |

**Practical advice**: scale vertically first — modern machines are huge (hundreds of cores, TBs of RAM). Stack Overflow famously ran on a handful of big SQL Servers. Go horizontal when you hit the ceiling, need redundancy, or cost becomes unreasonable.

**Diagonal scaling**: scale up each node to a sweet spot, then scale out.

### The scale cube (AKF)

```
            Y-axis: split by function/service (microservices)
            ▲
            │
            │
            └──────────► X-axis: clone (identical copies behind LB)
           ╱
          ╱
         ▼ Z-axis: split by data/customer (sharding, cells)
```
- **X**: run N identical copies. Easy for stateless. Doesn't help data size.
- **Y**: split by function (users service, orders service). Scales teams and independent workloads.
- **Z**: split by data key (users A–M / N–Z, per tenant, per region). Scales data and writes.

---

## 4. Stateless services

The precondition for X-axis scaling: **any request can go to any instance**.

Move state out of the app server:

| State | Where to put it |
|---|---|
| User sessions | Redis/Memcached, database, or signed tokens (JWT) |
| Uploaded files | Object storage (S3/GCS/Azure Blob) |
| Caches | Shared cache cluster (or accept per-instance cache) |
| In-progress jobs | Queue / job store |
| WebSocket connections | Connection stays on one node, but routing info and messages go via pub/sub (Redis, Kafka, NATS) |
| Scheduled tasks | Distributed scheduler / leader-elected worker |

**Sticky sessions** (LB pins user to one server) are a workaround for stateful apps; they hurt load distribution and failover. Avoid unless needed (e.g., WebSockets, some legacy apps).

---

## 5. Load balancing

### 5.1 Where load balancing happens

```
DNS (GeoDNS / round robin)
  → Global LB / Anycast
    → Regional L4 LB (TCP/UDP)
      → L7 LB / API gateway / reverse proxy (HTTP)
        → Service-to-service: client-side LB or service mesh sidecar
```

| Layer | Sees | Can do | Examples |
|---|---|---|---|
| **L4** | IP, port, TCP | Very fast, connection-level balancing | AWS NLB, LVS, Maglev |
| **L7** | HTTP headers, path, cookies | Path routing, header routing, TLS termination, retries, rate limits | Nginx, HAProxy, Envoy, AWS ALB, Traefik |

### 5.2 Algorithms

| Algorithm | Good for |
|---|---|
| Round robin | Uniform requests, uniform servers |
| Weighted round robin | Servers of different sizes; canary traffic splits |
| Least connections | Long-lived/variable-duration requests |
| Least response time / EWMA | Heterogeneous latency |
| **Power of two random choices** | Pick 2 random servers, send to the less loaded — near-optimal and cheap; used by Envoy, Nginx |
| IP hash / consistent hash | Affinity: same client/key → same server (cache locality) |

### 5.3 Service discovery
How the LB/client knows which instances exist: DNS (Kubernetes Services), Consul, etcd, Eureka, cloud target groups. Instances register on start and deregister on shutdown/health failure.

### 5.4 Service mesh
Sidecar proxies (Envoy via Istio, Linkerd) handle LB, retries, mTLS, and telemetry for service-to-service traffic, removing that logic from application code.

---

## 6. Caching for scale

Caching is the single most effective scaling technique for read-heavy systems: it removes load from the bottleneck (usually the DB). Details on cache patterns, invalidation, and eviction are in [Performance §5](../performance/README.md#5-caching).

Scaling-specific points:
- **Cache hit ratio drives DB load**: at 1M RPS, going from 95% → 99% hit rate reduces DB load from 50k → 10k QPS (5×).
- **Distributed cache** (Redis Cluster, Memcached with consistent hashing) scales horizontally by key partitioning.
- **Hot keys**: one key (celebrity profile, viral product) can overload a single cache node. Solutions:
  - Local in-process cache (L1) in front of the distributed cache (L2).
  - Replicate the hot key under multiple names (`key#1..key#N`) and read a random one.
  - Read replicas for cache nodes.
- **Cache stampede** on expiry of popular keys: request coalescing, probabilistic early expiration, locks.
- Caches can **hide** capacity problems: if the cache cluster is lost, can the DB survive the full load? If not, you need cache redundancy and warm-up strategies.

---

## 7. Content Delivery Networks (CDN)

A CDN is a geographically distributed cache at the network edge.

- Serves static assets (JS, CSS, images, video) and increasingly dynamic content and APIs.
- **Offloads** origin bandwidth and requests (often 90%+).
- **Pull CDN**: CDN fetches from origin on first request, caches by TTL. Simple, most common.
- **Push CDN**: you upload content to the CDN proactively. Good for large, rarely changing files.
- **Cache keys** and `Cache-Control` headers determine what is shared.
- **Versioned / fingerprinted URLs** (`app.3f9a2c.js`) → cache forever, no invalidation needed.
- **Edge compute** (Cloudflare Workers, Lambda@Edge, Fastly Compute): run logic (auth, A/B, personalization, redirects) at the edge.
- **Origin shield**: a mid-tier cache so all edge PoPs don't hit the origin separately.

---

## 8. Scaling the database: the usual path

The database is almost always the hardest thing to scale because it holds state. Typical progression — go to the next step only when needed:

```
1. Optimise queries & indexes, fix N+1 queries          ← cheapest, often 10–100× win
2. Connection pooling (PgBouncer, ProxySQL, RDS Proxy)
3. Vertical scaling (bigger instance, faster SSD/NVMe)
4. Caching layer (Redis/Memcached)
5. Read replicas (scale reads)
6. Functional partitioning: separate DBs per domain/service
7. Move specific workloads to specialised stores
   (search → Elasticsearch, analytics → warehouse, sessions → Redis, files → S3)
8. Archive / tier cold data (partitioned tables, move old data to cheap storage)
9. Horizontal sharding (scale writes and data size)
10. Distributed SQL / NoSQL designed for scale-out (Spanner, CockroachDB, Vitess, Cassandra, DynamoDB)
```

Also:
- **Denormalisation**: duplicate data to avoid expensive joins at read time.
- **Materialised views / precomputed aggregates**: compute on write (or periodically) instead of on every read.
- **Table partitioning** (within one DB, e.g., Postgres declarative partitioning by date): faster queries on recent data, cheap deletion of old partitions.

---

## 9. Replication & read replicas

```
          writes                 reads
App ────────────► Primary   App ───────► Replica 1
                     │                   Replica 2
                     └── async replication ──┘
```

- Scale **reads** linearly by adding replicas. Does **not** scale writes (every replica applies every write).
- Replicas can be specialised: one for analytics queries, one for search indexing, one in another region for local reads.
- **Replication lag** → stale reads. Handle with read-your-writes routing (read from primary after a write, or wait for replica LSN), or tolerate staleness where acceptable.
- Routing: application-level (separate read/write connections), proxies (ProxySQL, Pgpool), or ORM support.

Replication topologies (single-leader, multi-leader, leaderless) are covered in [Availability §5](../availability/README.md#5-data-replication).

---

## 10. Partitioning / sharding

Split data across multiple nodes so each holds a subset. Scales **writes**, **storage**, and **reads**.

### 10.1 Partitioning strategies

**Range partitioning**
```
Shard 1: user_id 1–1M | Shard 2: 1M–2M | Shard 3: 2M–3M
```
- ✅ Efficient range scans (time ranges, alphabetical).
- ❌ Hot spots: sequential keys (timestamps, auto-increment IDs) all write to the last shard.
- Used by: HBase, Bigtable, MongoDB (range), CockroachDB/TiDB (auto-splitting ranges).

**Hash partitioning**
```
shard = hash(user_id) mod N
```
- ✅ Even distribution.
- ❌ No efficient range queries; `mod N` means changing N reshuffles almost all keys.

**Consistent hashing**
```
Keys and nodes placed on a ring (0 … 2³²). Key goes to the next node clockwise.
Adding a node moves only ~1/N of keys.
Virtual nodes (each physical node owns many points) smooth out imbalance.
```
- Used by: Dynamo, Cassandra, Riak, Memcached clients, Discord, CDNs.
- Variants: **rendezvous (HRW) hashing**, **jump consistent hash**.

**Fixed number of partitions (pre-split)**
- Create many more partitions than nodes (e.g., 1,024 partitions over 10 nodes). Rebalance by moving whole partitions. Kafka, Elasticsearch, Couchbase, Riak.

**Directory / lookup-based**
- A lookup service maps key → shard. Max flexibility (move individual tenants), but the directory is a dependency and must be highly available and cached.

**Geographic / tenant-based**
- Shard by region or by customer. Natural for multi-tenant SaaS and data residency. Big tenants can get dedicated shards.

### 10.2 Choosing a shard key
The most important sharding decision. A good shard key:
- Has **high cardinality** (many distinct values).
- Distributes **load** evenly (not just data — beware of celebrity users).
- Matches the **dominant access pattern** so most queries hit a single shard (e.g., shard chat messages by `conversation_id`, orders by `customer_id`).
- Rarely changes.

Compound keys help: `(tenant_id, user_id)`, or add a salt for hot keys: `hash(celebrity_id + random(0..9))` then read all 10.

### 10.3 Problems sharding introduces
| Problem | Mitigation |
|---|---|
| Cross-shard queries (scatter–gather) | Choose key to localise queries; secondary indexes; denormalise |
| Cross-shard joins | Denormalise, application-level joins, co-locate related data |
| Cross-shard transactions | Avoid by design; sagas; 2PC (slow); distributed SQL |
| Secondary indexes | **Local** (per-shard, scatter on read) vs **global** (partitioned by indexed term, async-updated) |
| Rebalancing | Pre-split partitions, consistent hashing, online migration tools |
| Hot shards | Split hot shards, salt keys, cache hot data |
| Global unique IDs | Snowflake IDs, UUIDv7/ULID, ID ranges per shard |
| Operational complexity | Use managed/sharded systems (Vitess, Citus, MongoDB, DynamoDB) |

### 10.4 Sharding middleware / systems
- **Vitess** (MySQL, YouTube/Slack/GitHub), **Citus** (Postgres), MongoDB sharded clusters, **DynamoDB/Cassandra** (sharded by design), **Spanner/CockroachDB/TiDB/YugabyteDB** (auto-sharded distributed SQL).

---

## 11. Choosing the right data store

Polyglot persistence: use stores that match the workload.

| Store type | Examples | Scales well for | Weak at |
|---|---|---|---|
| Relational (SQL) | PostgreSQL, MySQL | Complex queries, transactions, moderate scale (vertical + replicas) | Massive write scale-out (without sharding) |
| Distributed SQL | Spanner, CockroachDB, TiDB, YugabyteDB | SQL + horizontal scale + strong consistency | Latency of cross-region consensus; cost |
| Key-value | Redis, DynamoDB, Riak | Simple lookups at huge scale | Queries other than by key |
| Wide-column | Cassandra, ScyllaDB, HBase, Bigtable | Massive write throughput, time series, large datasets | Ad hoc queries, joins, transactions |
| Document | MongoDB, Couchbase, Firestore | Flexible schema, entity-centric access | Cross-document relationships |
| Search | Elasticsearch, OpenSearch, Solr | Full-text search, faceting | Being the source of truth |
| Time-series | TimescaleDB, InfluxDB, Prometheus, ClickHouse | Metrics, append-heavy time data | Updates, relational queries |
| Columnar / OLAP | ClickHouse, BigQuery, Snowflake, Redshift, Druid | Analytics over billions of rows | High-rate point writes/updates |
| Graph | Neo4j, Neptune | Relationship traversals | Sharding across many nodes |
| Object storage | S3, GCS, Azure Blob | Unlimited blob storage, cheap | Low-latency small random access |
| Vector | pgvector, Pinecone, Milvus, Qdrant | Similarity search (embeddings) | General queries |

Separate **OLTP** (transactions, many small reads/writes) from **OLAP** (analytics, big scans). Feed analytics via CDC/ETL into a warehouse rather than running reports on the production DB.

---

## 12. Asynchronous processing & message queues

Synchronous chains scale only as well as their slowest member. Async decouples them.

```
Client → API → [Queue] → Workers (scale independently)
          │
          └── 202 Accepted (job id) — client polls or gets a webhook/notification
```

Benefits:
- **Load leveling**: queue absorbs spikes; workers process at a steady rate.
- **Independent scaling**: scale workers on queue depth.
- **Decoupling**: producer doesn't need consumer to be up.
- **Retries** without blocking users.

Use for: emails, notifications, image/video processing, report generation, webhooks, search indexing, ML inference batches, payments settlement.

### Queue vs log

| | Message queue | Distributed log |
|---|---|---|
| Examples | RabbitMQ, SQS, ActiveMQ | Kafka, Pulsar, Kinesis, Redpanda |
| Consumption | Message deleted after ack; competing consumers | Messages retained; each consumer group tracks offset |
| Ordering | Usually per queue (weak with many consumers) | Per partition |
| Replay | No | Yes (rewind offsets) |
| Scale unit | Consumers | **Partitions** (max parallelism = number of partitions) |
| Best for | Task distribution, work queues | Event streaming, multiple consumers, high throughput |

Scaling concerns:
- **Partition count** sets max consumer parallelism in Kafka — plan ahead.
- Partition key determines ordering *and* balance (same hot-key problem as sharding).
- **Consumer lag** is the key metric for autoscaling consumers.
- **Delivery semantics**: at-least-once is the norm → consumers must be **idempotent** (see [Reliability](../reliability/README.md)).
- **Dead letter queues** for poison messages.

---

## 13. Event-driven architecture & stream processing

- **Pub/sub**: producers publish events (`OrderPlaced`), many consumers react independently (inventory, email, analytics). Adding a consumer doesn't touch the producer.
- **Event sourcing**: store the sequence of events as the source of truth; derive state by replay. Scales writes (append-only) and enables many read models.
- **CQRS** (Command Query Responsibility Segregation): separate write model from read models; each read model is optimised (and scaled) for its query.
- **Change Data Capture (CDC)**: stream DB changes (Debezium) to update caches, search indexes, warehouses without dual writes.
- **Stream processing**: Kafka Streams, Flink, Spark Structured Streaming — windowed aggregations, joins, real-time analytics at scale.
- **Lambda vs Kappa architecture**: batch + speed layers vs a single streaming pipeline with replay.

---

## 14. Service decomposition

### 14.1 Monolith → modular monolith → microservices

| | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Deploy unit | One | One | Many |
| Scaling | Whole app together (X-axis) | Same | Each service independently |
| Team scaling | Hard beyond ~20–50 engineers | Better (clear module boundaries) | Best (team owns service) |
| Operational overhead | Low | Low | High (network, observability, deployment, data consistency) |

Microservices are primarily an **organisational** scaling tool (Conway's law). Technical scaling can usually be done with a monolith + X/Z-axis scaling. Start with a well-modularised monolith; extract services when a part has distinct scaling needs, release cadence, or team ownership.

### 14.2 Service-level scaling concerns
- **Database per service** — avoid shared databases (coupling, contention).
- **API gateway / BFF** (Backend-for-Frontend) to aggregate calls for clients.
- Avoid **chatty** synchronous call chains — each hop adds latency and failure probability.
- **Serverless** (Lambda, Cloud Run, Cloud Functions): scales to zero and to thousands automatically; watch cold starts, concurrency limits, DB connection exhaustion (use proxies), and cost at sustained high load.

---

## 15. Autoscaling & capacity planning

### 15.1 Autoscaling types

| Type | How | Notes |
|---|---|---|
| **Reactive (target tracking)** | Keep metric at target, e.g., CPU 60% | Lag of minutes; leave headroom |
| **Step scaling** | Add N instances when metric crosses thresholds | More control |
| **Scheduled** | Scale up before known peaks (9am, sales) | Cheap, predictable |
| **Predictive** | ML forecasts from history | AWS predictive scaling |
| **Queue-based** | Scale workers on queue depth / consumer lag | KEDA for Kubernetes |
| **Horizontal pod (HPA)** | Kubernetes pods on CPU/memory/custom metrics | Combined with **cluster autoscaler / Karpenter** for nodes |
| **Vertical (VPA)** | Adjust resource requests per pod | Usually needs restart |

Tips:
- Scale on the metric closest to user experience (RPS per instance, latency, queue depth) rather than CPU alone.
- **Cooldowns** to avoid flapping; scale out fast, scale in slowly.
- Warm pools / pre-initialised instances for slow-starting apps.
- Autoscaling doesn't fix a bottleneck in the DB — adding app servers may make it *worse* (more DB connections).

### 15.2 Capacity planning
- Load test to find the **knee** of the latency curve and the first bottleneck (see [Performance §12](../performance/README.md#12-load-testing--benchmarking)).
- Plan for **peak × safety margin**, plus the capacity lost when a zone fails.
- Track growth trends; forecast 6–12 months ahead for things that take long to provision (DB clusters, reserved capacity, partitions).
- Know your **limits**: cloud quotas, max connections, file descriptors, partition counts, IP addresses, API rate limits of third parties.

---

## 16. Scaling patterns for specific problems

### 16.1 Fan-out (feeds, notifications)
- **Fan-out on write (push)**: on post, write to every follower's timeline cache. Fast reads, expensive writes for users with many followers.
- **Fan-out on read (pull)**: on timeline read, gather recent posts from all followees. Cheap writes, expensive reads.
- **Hybrid** (Twitter/Instagram): push for normal users, pull for celebrities, merge at read time.

### 16.2 Counters at scale (likes, views)
- Don't `UPDATE ... SET count = count + 1` on a single row at 100k/s (lock contention).
- **Sharded counters**: N sub-counters, sum on read.
- Buffer increments in memory/Redis (`INCR`) and flush periodically.
- **Approximate counting**: HyperLogLog for unique counts, Count-Min Sketch for frequencies.

### 16.3 Unique ID generation
- Auto-increment → single point of contention, leaks info, breaks with sharding.
- **UUIDv4** (random, bad index locality), **UUIDv7 / ULID** (time-ordered, good locality), **Snowflake** (timestamp + machine id + sequence, 64-bit), ID range allocation per node.

### 16.4 Rate limiting at scale
- Token bucket/sliding window counters in Redis (with Lua scripts for atomicity), or local limits with approximate global coordination.

### 16.5 Search
- Inverted index (Elasticsearch/OpenSearch) sharded by document; replicas for read throughput; feed via CDC/queue.

### 16.6 Large file uploads
- **Pre-signed URLs**: client uploads directly to object storage, bypassing app servers.
- Multipart/resumable uploads; async processing (thumbnails, transcoding) via queue.

### 16.7 Real-time connections
- Dedicated gateway tier holding WebSockets (event-loop servers: Node, Go, Elixir/Phoenix, Netty).
- Pub/sub backbone to route messages to the gateway holding the recipient's connection.
- Connection registry: `user_id → gateway_node` in Redis.

### 16.8 Geospatial queries
- Geohash, quadtrees, S2 cells, H3 hexagons to partition space; index nearby objects in Redis GEO / PostGIS / Elasticsearch.

### 16.9 Leaderboards / top-K
- Redis sorted sets; for global scale, approximate top-K with Count-Min Sketch + heap, or per-shard top-K merged.

### 16.10 Multi-tenancy
- **Pool** (shared tables with `tenant_id`), **bridge** (schema per tenant), **silo** (DB/stack per tenant). Large tenants → dedicated shards/cells.

---

## 17. Scaling stages: from 1 user to millions

A common evolution — don't skip ahead before you need it.

```
Stage 1 – Single server
  [App + DB on one box]

Stage 2 – Separate DB
  [App] → [DB]

Stage 3 – Horizontal app tier
  [LB] → [App ×N, stateless] → [DB]
  Sessions → Redis; files → S3

Stage 4 – Cache + CDN
  CDN for static assets; Redis cache for hot queries

Stage 5 – Read replicas
  [DB primary] → [replicas] for reads

Stage 6 – Async processing
  Queues + workers for slow tasks

Stage 7 – Split by function
  Separate DBs/services for distinct domains; search/analytics in specialised stores

Stage 8 – Shard / distributed databases
  Partition the largest/write-heavy data

Stage 9 – Multi-region
  Users served from nearest region; geo-partitioned data

Stage 10 – Cells
  Many independent copies of the full stack, each serving a subset of users
```

---

## 18. Approaches by system type

### 18.1 Read-heavy content (blogs, news, CMS, documentation, catalogs)
- **Read:write** often 1000:1.
- CDN + full-page caching first; it can handle almost unlimited readers.
- Stateless app servers; DB read replicas; object storage for media.
- Cache invalidation on publish (purge CDN by tag/surrogate key).
- Example (WordPress at scale): page cache (Varnish/CDN) → object cache (Redis) → PHP-FPM nodes behind LB → MySQL primary + replicas → media on S3 → search in Elasticsearch.

### 18.2 Write-heavy systems (logging, metrics, IoT telemetry, clickstream)
- Append-only, **log-structured** storage (LSM-tree stores: Cassandra, ClickHouse, InfluxDB, Kafka).
- Ingest via a distributed log (Kafka/Kinesis) partitioned by source; batch writes.
- Time-based partitioning; TTL / downsampling / tiered storage for old data.
- Pre-aggregate in streams (Flink) rather than querying raw data.

### 18.3 Social networks / feeds
- Graph of follows (sharded by user), posts (sharded by author), timelines (precomputed in cache).
- Hybrid fan-out; heavy caching; eventual consistency acceptable.
- Media via CDN; counters sharded/approximate.

### 18.4 E-commerce
- Catalog: read-heavy → cache + CDN + search index.
- Cart: key-value per user (DynamoDB/Redis), shard by user.
- Orders/inventory: relational, shard by customer/order; inventory hot items → reservations, sharded counters, queues.
- Flash sales: waiting rooms, pre-allocated inventory tokens, queue-based checkout, rate limits per user.

### 18.5 Chat / messaging
- Connection gateways scaled horizontally; pub/sub for routing.
- Messages sharded by `conversation_id` (wide-column stores like Cassandra/ScyllaDB — Discord, WhatsApp-style).
- Group chats with huge membership → fan-out on read for large groups.
- Presence: ephemeral data in Redis with TTL; gossip for large-scale presence.

### 18.6 Video / media streaming
- Upload → object storage → transcoding pipeline (queue + worker fleet / batch) → multiple bitrates/segments → CDN.
- Bandwidth is the scaling problem → multi-CDN, ISP-embedded caches (Netflix Open Connect).
- Metadata/recommendations scaled separately.

### 18.7 Search engines / web crawlers
- Crawler: URL frontier (distributed queue, partitioned by host for politeness), many stateless fetchers, dedupe via Bloom filters / content hashes.
- Index: sharded by document, replicated for query throughput; queries scatter–gather across shards.

### 18.8 Ride-sharing / location-based
- Location updates are write-heavy (every few seconds per driver) → in-memory geo-index sharded by geographic cell (S2/H3), not a relational DB.
- Matching service partitioned by city/region.
- Trips/payments in transactional stores.

### 18.9 Analytics / data platforms
- Separate OLAP from OLTP. CDC/ETL into a lakehouse (S3 + Iceberg/Delta/Hudi) or warehouse (BigQuery, Snowflake).
- Scale via massively parallel processing (Spark, Trino, warehouse engines), columnar formats (Parquet), partition pruning.
- Pre-aggregate with materialised views / OLAP cubes (Druid, Pinot, ClickHouse) for interactive dashboards.

### 18.10 Multi-tenant SaaS
- Shard/cell by tenant; tenant directory for routing.
- Noisy neighbours: per-tenant quotas and rate limits; move big tenants to dedicated cells.

### 18.11 APIs / platform (public API, payment gateway)
- API gateway with per-key rate limits and quotas.
- Idempotent endpoints, pagination (cursor-based, not offset), webhooks delivered via queues with retries.

### 18.12 ML / AI inference
- GPU capacity is the bottleneck → **batching** requests, model caching in memory, autoscale on queue depth/GPU utilisation.
- Async inference for long jobs; streaming responses for LLMs.
- Cache embeddings and repeated results; route to smaller models when acceptable.

### 18.13 Gaming
- Game servers: sharded by match/room/world zone, stateful, scaled by spinning up instances per match (Agones on Kubernetes).
- Matchmaking, accounts, leaderboards, inventories: classic scalable backend services.
- MMOs: spatial partitioning of the world, interest management (only send updates about nearby entities).

### 18.14 Event ticketing / flash sales
- Extreme spikes (100–1000× normal) for minutes.
- Virtual waiting room → admission control → queue-based purchase → inventory held in a fast store with atomic decrements → async confirmation.

### Summary table

| System | Dominant load | Key scaling techniques |
|---|---|---|
| Content/CMS | Reads | CDN, page cache, read replicas |
| Logging/metrics/IoT | Writes | Kafka, LSM stores, time partitioning, downsampling |
| Social feed | Fan-out reads | Hybrid fan-out, timeline cache, sharding by user |
| E-commerce | Reads + bursty writes | Cache/search for catalog, sharded orders, waiting rooms |
| Chat | Connections + writes | Gateway tier, pub/sub, shard by conversation |
| Streaming | Bandwidth | CDN, transcoding pipelines |
| Ride-sharing | Geo writes | In-memory geo-index sharded by cell |
| Analytics | Big scans | Columnar, MPP, lakehouse |
| SaaS | Tenants | Cells, tenant sharding, quotas |
| ML inference | GPU compute | Batching, queue-based autoscaling |

---

## 19. Trade-offs

| Scaling technique | What it costs |
|---|---|
| Horizontal scaling | Distributed systems complexity, network overhead |
| Caching | **Consistency** (stale data), invalidation complexity, memory cost |
| Read replicas | Replication lag → stale reads |
| Sharding | Cross-shard queries/transactions become hard; operational complexity |
| Denormalisation | Write amplification, risk of inconsistency between copies |
| Async processing | Eventual consistency, harder debugging, need for idempotency |
| Microservices | Network latency, operational overhead, distributed transactions |
| NoSQL / eventual consistency | Weaker guarantees, application handles conflicts |
| Autoscaling | Cost spikes, cold starts, lag |
| Multi-region | Cost, data consistency, compliance complexity |
| Approximate algorithms (HLL, sketches) | Exactness |

---

## 20. Anti-patterns

- **Premature sharding / microservices** before a single DB/monolith is exhausted.
- **Scaling the app tier when the DB is the bottleneck** (more connections → worse).
- **Stateful app servers** (local sessions/files) blocking horizontal scale.
- **Bad shard key** → hot shards; monotonically increasing keys on range partitioning.
- **Scatter–gather for every query** because the shard key doesn't match access patterns.
- **Distributed monolith**: microservices that must be deployed together and call each other synchronously in long chains.
- **Unbounded queries**: no pagination, `SELECT *` on huge tables, offset pagination on deep pages.
- **Running analytics on the OLTP database.**
- **Single global lock / counter / sequence** in the hot path.
- **Ignoring connection limits** (serverless + relational DB without a proxy).
- **Infinite retries** turning a slowdown into overload.
- Not load testing until launch day.

---

## 21. Checklist

**Understand**
- [ ] Load parameters identified (RPS, read/write ratio, data size, fan-out, skew)
- [ ] Back-of-the-envelope estimates for peak traffic, storage, bandwidth
- [ ] Current bottleneck identified by measurement, not guessing

**App tier**
- [ ] Stateless; sessions/files externalised
- [ ] Behind a load balancer; autoscaling configured on meaningful metrics
- [ ] Connection pools sized (Little's Law), DB connection proxy if many instances

**Data tier**
- [ ] Queries indexed and optimised; no N+1
- [ ] Caching layer with hot-key and stampede protection
- [ ] Read replicas with lag handling
- [ ] OLTP and OLAP separated
- [ ] Sharding strategy and shard key chosen (or a documented plan for when needed)
- [ ] Unique ID strategy that works across shards

**Async**
- [ ] Slow work moved to queues; workers autoscale on queue depth
- [ ] Idempotent consumers, DLQs

**Edge**
- [ ] CDN for static and cacheable content; versioned asset URLs
- [ ] Rate limiting per client

**Planning**
- [ ] Load tests at ≥2× expected peak
- [ ] Capacity headroom for zone failure
- [ ] Known limits/quotas documented and monitored

---

## 22. Further reading

- *Designing Data-Intensive Applications* (Kleppmann) — ch. 1 (scalability), 5 (replication), 6 (partitioning), 11 (stream processing)
- *The Art of Scalability* (Abbott & Fisher) — the AKF scale cube
- *System Design Interview* vol. 1 & 2 (Alex Xu) — worked examples (feeds, chat, rate limiter, ID generator)
- `donnemartin/system-design-primer` (GitHub)
- Papers: Dynamo, Bigtable, Spanner, Kafka, Cassandra, "Scaling Memcache at Facebook", "TAO: Facebook's Distributed Data Store"
- Engineering blogs: Discord (storing trillions of messages), Instagram (sharding & IDs), Slack (Vitess), Uber (H3, schemaless), Netflix, Shopify (pods/cells)
- highscalability.com — architecture case studies
