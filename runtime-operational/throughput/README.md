# Throughput

> **Throughput** = how much **work the system completes per unit of time**: requests/s, transactions/s, messages/s, MB/s, jobs/hour.
> "How many orders can we process per second? How long does it take to process the backlog?"

Related: [Performance](../performance/README.md) (latency, the other half), [Capacity](../capacity/README.md) (max throughput at SLO), [Scalability](../scalability/README.md) (raising throughput by adding resources), [Efficiency](../efficiency/README.md) (throughput per resource).

---

## 1. Throughput, latency, bandwidth

| Term | Meaning | Analogy (a highway) |
|---|---|---|
| **Latency** | Time for one operation | Travel time for one car |
| **Throughput** | Operations completed per second | Cars arriving per hour |
| **Bandwidth** | Maximum possible rate of a channel | Number of lanes × speed limit |
| **Goodput** | *Useful* throughput: excluding retries, errors, duplicates, overhead | Cars that reach the right destination |

They're linked by **Little's Law**: `concurrency = throughput × latency`. With 100 workers and 50 ms per item, max throughput = 100 / 0.05 = 2,000 items/s. To go higher: more workers (concurrency) or less time per item (latency).

**Goodput matters during overload**: a system can be "busy" at 10,000 RPS while 80% of that is retries of timed-out requests. Throughput looks high, useful work collapses (congestion collapse).

---

## 2. The bottleneck decides

A pipeline's throughput = the throughput of its **slowest stage**. Speeding up any other stage changes nothing.

### Worked example: an ingestion pipeline

```
Ingest API ──► Parser workers ──► Database writer
 50k/s          2 × 20k/s = 40k/s    1,000 transactions/s × 10 rows each = 10k rows/s
```

End-to-end: **10k/s**. The DB writer is the bottleneck. Adding parser workers does nothing.

Fix the bottleneck: batch **100 rows per transaction** (fewer commits/fsyncs). The DB now does ~600 transactions/s × 100 rows = **60k rows/s**.

```
Ingest 50k/s ──► Parsers 40k/s ──► DB writer 60k/s
```

End-to-end: **40k/s**. The bottleneck **moved** to the parsers. Add a third parser worker → 50k/s, and now ingest is the limit.

That's the loop: find the bottleneck → fix it → find the new one. Stop when you meet the target.

---

## 3. Techniques

| Technique | How it raises throughput |
|---|---|
| **Parallelism / partitioning** | More workers, threads, partitions, shards working at once |
| **Batching** | Pay fixed costs (round trip, commit, fsync, syscall) once per batch |
| **Pipelining** | Send the next request before the previous reply arrives (Redis pipelining, HTTP/2) |
| **Async / non-blocking I/O** | Threads don't sit idle waiting |
| **Less work per item** | Efficient code, fewer queries, smaller payloads, caching |
| **Sequential I/O** | Append-only logs, LSM trees; avoid random disk access |
| **Less coordination** | Fewer locks, no global counters, no cross-shard transactions |
| **Compression** | More items per MB on the network or disk (costs CPU) |
| **Zero-copy** | `sendfile`, memory mapping: skip copying buffers |

More detail: [Performance §10](../performance/README.md#10-batching-pipelining-precomputation-and-doing-less).

---

## 4. The throughput–latency trade-off

Many throughput techniques **add latency** to each item:
- Batching: an item waits for the batch to fill (Kafka `linger.ms`, Nagle's algorithm).
- High utilisation: more queueing delay per request.
- Big buffers: more items waiting.

Choose per workload:
- **Interactive** (APIs, UIs): latency first, small batches, keep utilisation moderate.
- **Bulk** (ETL, analytics, backups, log shipping): throughput first, big batches, high utilisation is fine.
- Use **size OR time** triggers for batches ("flush at 500 items or 10 ms") to bound the added latency.

---

## 5. Measuring throughput

- Measure **sustained** throughput (minutes to hours), not a 10-second burst.
- Measure it **at your latency SLO**: max throughput while p99 is still acceptable.
- Track **goodput** and error rate next to raw throughput.
- For queues/streams: **consumer lag** and **time to drain a backlog** (`backlog / (consume rate − produce rate)`).
- Watch for **degradation over time**: GC, compaction, growing tables, cache eviction.

---

## 6. Anti-patterns

- Adding workers to a stage that isn't the bottleneck.
- One item per transaction/round trip in bulk jobs.
- Reporting throughput while ignoring errors and retries (no goodput).
- Huge batches on interactive paths.
- Unbounded queues that hide that throughput is below the arrival rate (the backlog grows forever).

---

## 7. Checklist

- [ ] Throughput target defined (with latency SLO) for critical paths and pipelines
- [ ] Bottleneck identified by measurement for each pipeline
- [ ] Bulk paths use batching; interactive paths keep batches small
- [ ] Goodput and error rate tracked next to throughput
- [ ] Queue lag and backlog drain time monitored
- [ ] Sustained throughput tested (soak), not just bursts

---

## 8. Exercises

1. A consumer handles 3,000 msgs/s; producers send 2,500/s normally. After a 1-hour consumer outage, how long does it take to drain the backlog? What if producers send 2,900/s?
2. With 40 DB connections and 8 ms per query, what's the max query throughput? What two things can raise it?
3. Your ETL loads 50M rows one `INSERT` at a time at 2 ms each. How long does it take? Estimate with batches of 1,000 rows (assume ~20 ms per batch).
4. During an incident, RPS doubled but successful orders halved. Explain using goodput, and name the likely cause.

---

## 9. Further reading

*Designing Data-Intensive Applications* ch. 1 (describing load and performance) and ch. 10–11 (batch and stream processing); Eliyahu Goldratt, *The Goal* (theory of constraints, the classic bottleneck novel); Brendan Gregg, *Systems Performance*; Kafka documentation on producer batching (`batch.size`, `linger.ms`) and consumer lag.
