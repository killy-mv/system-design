# Throughput: hands-on practice notes

Planned labs for learning throughput by **building pipelines, finding the slowest stage, and moving the bottleneck**. Theory is in [README.md](README.md).

They need **Docker Compose**, **PostgreSQL**, **Redpanda** (Kafka API) or **RabbitMQ**, and **k6**.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** batching and pipelining with Redis/Postgres (performance lab 7), async processing with a queue (scalability lab 9), Kafka partitions and consumer parallelism (scalability lab 10).

---

## Labs

### 1. Move the bottleneck ⭐
- **Build:** the README §2 pipeline: an ingest API → a queue → parser workers → a DB writer. Make each stage's speed configurable (workers, batch size).
- **Measure:** per-stage throughput and queue lag between stages. Find the bottleneck from the lag (the queue *in front of* the slowest stage grows).
- **Fix it, find the next one:** batch DB writes, add parsers, and so on. Keep a table of end-to-end throughput after each change.

### 2. Batch size vs latency
- **Build:** a Kafka/Redpanda producer sending 1 KB messages, with `linger.ms` = 0, 5, 20, 100 and `batch.size` varied.
- **Measure:** messages/s, CPU, and per-message end-to-end latency (timestamp in the message, compared at the consumer).
- **See:** where the curve gives most of the throughput for little added latency.
- **See:** README §4.

### 3. Goodput under overload ⭐
- **Build:** a service with a 100 ms timeout, and clients that retry 3 times immediately on timeout.
- **Break:** push traffic to 1.5× capacity. Plot raw RPS vs **successful** RPS over time: raw goes up, goodput collapses.
- **Fix:** retries with backoff + jitter and a retry budget, plus load shedding. Plot again.
- **See:** README §1 goodput, [Availability §8.2](../availability/README.md#82-retries).

### 4. Backlog drain time
- **Build:** a consumer group processing a topic. Stop the consumers for 10 minutes while producers keep sending.
- **Measure:** lag growth, then drain time after restart. Compare with the formula `backlog / (consume rate − produce rate)`.
- **Then:** add consumers (up to the partition count) and measure again.
- **See:** README §5, exercise 1.

---

## Progress

- [ ] 1. Move the bottleneck
- [ ] 2. Batch size vs latency
- [ ] 3. Goodput under overload
- [ ] 4. Backlog drain time
