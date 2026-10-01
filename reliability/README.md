# Reliability

> **Reliability** = the system continues to **work correctly** — giving the right results, not losing or corrupting data — even when things go wrong (hardware faults, software bugs, network problems, human mistakes).
> "Can users *trust* what the system tells them and what it stores?"

Related: [Availability](../availability/README.md) (is it up?), [Scalability](../scalability/README.md) (distributing data creates consistency problems), [Maintainability](../maintainability/README.md) (testing, operability, and human error).

---

## Table of contents

1. [Core concepts and vocabulary](#1-core-concepts-and-vocabulary)
2. [Measuring reliability](#2-measuring-reliability)
3. [Kinds of faults](#3-kinds-of-faults)
4. [Durability: never lose acknowledged data](#4-durability-never-lose-acknowledged-data)
5. [Consistency models](#5-consistency-models)
6. [Transactions and isolation](#6-transactions-and-isolation)
7. [Distributed transactions and cross-service consistency](#7-distributed-transactions-and-cross-service-consistency)
8. [Consensus, leader election, and distributed locks](#8-consensus-leader-election-and-distributed-locks)
9. [Time, clocks, and ordering](#9-time-clocks-and-ordering)
10. [Reliable messaging](#10-reliable-messaging)
11. [Idempotency](#11-idempotency)
12. [Conflict resolution in replicated data](#12-conflict-resolution-in-replicated-data)
13. [Data integrity and correctness](#13-data-integrity-and-correctness)
14. [Software fault tolerance patterns](#14-software-fault-tolerance-patterns)
15. [Reducing human error](#15-reducing-human-error)
16. [Testing for reliability](#16-testing-for-reliability)
17. [Approaches by system type](#17-approaches-by-system-type)
18. [Trade-offs](#18-trade-offs)
19. [Anti-patterns](#19-anti-patterns)
20. [Checklist](#20-checklist)
21. [Further reading](#21-further-reading)

---

## 1. Core concepts and vocabulary

| Term | Meaning |
|---|---|
| **Fault** | A component deviates from its spec (a disk dies, a packet is lost, a bug triggers) |
| **Failure** | The *system as a whole* stops providing the required service to the user |
| **Fault tolerance** | Designing so that faults don't become failures |
| **Reliability** | Probability the system performs correctly over a period of time |
| **Availability** | Proportion of time the system is able to serve (see [Availability](../availability/README.md)) |
| **Durability** | Once data is acknowledged as written, it won't be lost |
| **Consistency** | Overloaded word: (1) ACID "C" — data obeys invariants; (2) replica consistency — all readers see the same data |
| **Correctness** | The system produces the right output for its inputs and keeps invariants |
| **Resilience** | Ability to absorb and recover from faults |

You cannot prevent all faults. Reliability engineering is about **tolerating** faults (redundancy, retries, replication) and **preventing** the ones you can (testing, validation, review), and **detecting** the rest quickly.

Availability vs reliability example:
- A system that returns a 503 during a DB failover is *less available* but still *reliable* (no wrong data).
- A system that keeps serving but returns another user's account balance is *available* but *unreliable* — usually much worse.

---

## 2. Measuring reliability

| Metric | What it tells you |
|---|---|
| **MTBF** (Mean Time Between Failures) | How often the system fails |
| **Failure rate** (λ = 1/MTBF) | Failures per unit time |
| **Error rate** | % of requests that return wrong/failed results |
| **Correctness SLIs** | % of records passing validation; reconciliation mismatches; data freshness |
| **Durability** (nines) | Probability an object survives a year (S3: 99.999999999% = 11 nines) |
| **RPO** | Max data loss tolerated on disaster (see [Availability §12](../availability/README.md#12-disaster-recovery)) |
| **Change failure rate** (DORA) | % of deployments causing incidents |
| **Defect escape rate** | Bugs found in production vs before |
| **Data loss incidents / corruption incidents** | Should be zero; track any |

**Bathtub curve**: hardware fails more often early (infant mortality) and late (wear-out). Disks have an annual failure rate (AFR) of ~0.5–2% → in a cluster of 10,000 disks, several die every day. **At scale, failure is normal**, not exceptional.

---

## 3. Kinds of faults

### 3.1 Hardware faults
- Disk failures, bad sectors, SSD wear, RAM bit flips (use ECC memory), CPU bugs, NIC failures, power loss.
- Usually **independent and random** → handled well by redundancy (RAID, replication, multi-AZ).

### 3.2 Software faults
- Bugs, memory leaks, crash loops, resource exhaustion, deadlocks, race conditions, leap-second/time bugs, integer overflow, poison-pill inputs, dependency failures.
- Often **correlated** — the same bug hits every replica at once (same input, same code). Redundancy doesn't help; isolation, gradual rollout, and testing do.

### 3.3 Network faults
In a distributed system, the network can:
- Lose, delay, duplicate, or reorder messages.
- **Partition** (some nodes can't talk to others).
- Be asymmetric (A can reach B, B can't reach A).

The crucial problem: when you send a request and get **no response**, you can't tell whether:
1. the request was lost,
2. the remote node crashed before processing,
3. the remote node processed it but the response was lost,
4. it's just slow.

This is why **idempotency** (§11) and **timeouts** are fundamental.

### 3.4 Process pauses
GC stop-the-world, VM live migration, OS swapping, `SIGSTOP`, CPU throttling. A node can be paused for seconds, then resume believing it's still the leader / still holds a lock → **fencing tokens** (§8.4).

### 3.5 Clock faults
Clocks drift, jump (NTP corrections), and differ between machines. Never assume clocks across machines agree (§9).

### 3.6 Human faults
Misconfiguration, wrong commands, bad deploys — the leading cause of outages (§15).

### 3.7 Fault models in distributed systems
| Model | Nodes can… |
|---|---|
| **Crash-stop** | Crash and never return |
| **Crash-recovery** | Crash and come back (with persisted state, maybe losing in-memory state) |
| **Byzantine** | Behave arbitrarily or maliciously (lie, send corrupted data) |

Most internal systems assume crash-recovery and non-Byzantine nodes. Byzantine fault tolerance (BFT) is needed in blockchains, aerospace, and systems with untrusted participants.

---

## 4. Durability: never lose acknowledged data

### 4.1 Single-node durability
- **Write-ahead log (WAL)**: append the change to a log and `fsync` before acknowledging; apply to data files later. On crash, replay the log. (Postgres WAL, MySQL redo log, LSM memtable + WAL.)
- **fsync semantics**: data in OS page cache is *not* durable. Only after `fsync`/`fdatasync` returns is it on stable storage (assuming disk honours flushes). Many "durable" systems lost data due to fsync mistakes (e.g., the 2018 "fsyncgate" in Postgres).
- **Group commit**: batch several transactions into one fsync — durability without fsync-per-transaction cost.
- **Atomic file writes**: write to temp file → fsync → rename → fsync directory.
- **Checksums** on every block/record to detect torn writes and corruption (see KingDB/LevelDB per-entry checksums).
- **Battery-backed / power-loss-protected** disk caches in enterprise hardware.

### 4.2 Multi-node durability
- **Replication**: data acknowledged only after written to N nodes (synchronous) or to a quorum. Survive node loss without data loss.
  - Kafka: `acks=all`, `replication.factor=3`, `min.insync.replicas=2`, `unclean.leader.election.enable=false`.
  - Postgres: synchronous standby; MySQL semi-sync; MongoDB `writeConcern: majority`.
  - Cassandra: `QUORUM` consistency level.
- **Erasure coding** (Reed–Solomon): split data into k data + m parity chunks; survive m losses with far less overhead than full replication (e.g., 1.5× instead of 3×). Used by S3, HDFS, Ceph, Backblaze.
- **Geographic replication**: survive data center/region loss.

### 4.3 Backups (protection against logical errors)
Replication copies mistakes instantly (`DELETE FROM users` replicates too). Backups protect against:
- Bugs that corrupt data
- Accidental deletions
- Ransomware/malicious deletes

Practices:
- Full + incremental backups; **point-in-time recovery** (base backup + WAL archive).
- Off-site, different account, **immutable** (object lock / WORM).
- **Regular restore tests**, ideally automated.
- **Soft deletes** and delayed hard deletion; delayed replicas (replica that lags by 1 hour on purpose).
- Versioning on object storage.

### 4.4 Durability of in-flight work
- Don't hold the only copy of a job in memory. Persist to a durable queue before acknowledging the client.
- Write-behind caches can lose data — use only when loss is acceptable.

---

## 5. Consistency models

When data is replicated, what can readers observe? From strongest to weakest:

### 5.1 Linearizability (strong consistency)
- The system behaves as if there is **a single copy** of the data and every operation takes effect atomically at some instant between its start and end.
- After a write completes, all subsequent reads (from anyone) see it.
- Required for: leader election, locks, uniqueness constraints (usernames), account balances, inventory with strict limits.
- Achieved by: single leader with synchronous reads from leader, consensus (Raft/Paxos), Spanner (TrueTime).
- Cost: latency (coordination), reduced availability under partitions (CAP).

### 5.2 Sequential consistency
- All operations appear in some total order consistent with each client's program order, but not necessarily real-time order.

### 5.3 Causal consistency
- Operations that are causally related (A happened-before B, e.g., a reply after a post) are seen in the same order by everyone. Concurrent operations may be seen in different orders.
- Strongest model that remains available under partitions. Implemented with version vectors / dependency tracking.

### 5.4 Session guarantees (client-centric)
| Guarantee | Meaning | Example violation |
|---|---|---|
| **Read-your-writes** | You always see your own writes | Update profile, refresh, old profile shown |
| **Monotonic reads** | You never see data go "back in time" | Comment appears, then disappears on refresh (different replica) |
| **Monotonic writes** | Your writes are applied in order | Second edit applied before first |
| **Writes-follow-reads** | A write after reading X is ordered after X | Reply visible before the message it replies to |
| **Consistent prefix** | Readers see writes in an order that makes sense | Answer seen before the question |

Implementation tricks: route a user's reads to the leader shortly after they write; pin a user to one replica; include a version/LSN token with the client and only read from replicas that have caught up.

### 5.5 Eventual consistency
- If no new writes occur, all replicas **eventually** converge. No guarantee *when*, and reads in between may be stale or inconsistent.
- Highly available and low latency (Dynamo-style stores, DNS, caches, CDNs).
- **Strong eventual consistency**: replicas that received the same updates are in the same state (CRDTs guarantee this).

### 5.6 CAP and PACELC
- **CAP**: in the presence of a network **P**artition, choose **C**onsistency (linearizability — refuse some requests) or **A**vailability (respond, maybe inconsistent). Partitions aren't optional, so the real choice is C vs A during partitions.
- **PACELC**: if Partition → A or C; Else (normal operation) → **L**atency or **C**onsistency.

| System | PACELC |
|---|---|
| Dynamo, Cassandra, Riak | PA/EL |
| MongoDB (default) | PA/EC |
| Spanner, CockroachDB, etcd, ZooKeeper | PC/EC |
| PostgreSQL single primary + sync standby | PC/EC |

### 5.7 Tunable consistency
Quorum systems: with `N` replicas, write quorum `W`, read quorum `R`:
- `W + R > N` → reads overlap with latest write (strong-ish, but edge cases with sloppy quorums and concurrent writes).
- `W = 1, R = 1` → fastest, eventual.
- Choose per operation (Cassandra `ONE`/`QUORUM`/`ALL`, `LOCAL_QUORUM` for multi-DC; DynamoDB strongly vs eventually consistent reads).

---

## 6. Transactions and isolation

### 6.1 ACID
- **Atomicity**: all or nothing — on error, everything rolls back. (Better called "abortability".)
- **Consistency**: invariants hold (largely the application's job + constraints).
- **Isolation**: concurrent transactions don't interfere (to the degree the isolation level promises).
- **Durability**: committed data survives crashes.

### 6.2 Concurrency anomalies

| Anomaly | Description | Example |
|---|---|---|
| **Dirty read** | Read uncommitted data of another transaction | See a payment that later rolls back |
| **Dirty write** | Overwrite uncommitted data | Two buyers' writes interleave |
| **Non-repeatable read / read skew** | Same query returns different data within a transaction | Sum of accounts seen mid-transfer |
| **Lost update** | Two read-modify-write cycles; one overwrites the other | Two counters increments, one lost |
| **Phantom read** | A query's result set changes due to inserts by others | Booking check sees no conflicting rows, another inserts one |
| **Write skew** | Two transactions read the same data, make disjoint writes that together violate an invariant | Two doctors both go off-call because each saw the other on call |

### 6.3 Isolation levels

| Level | Prevents | Notes |
|---|---|---|
| Read uncommitted | Dirty writes | Rarely used |
| **Read committed** | Dirty reads/writes | Default in Postgres, Oracle, SQL Server |
| **Repeatable read / snapshot isolation (MVCC)** | + read skew | MySQL InnoDB default; Postgres "repeatable read" = SI |
| **Serializable** | Everything (incl. write skew, phantoms) | Postgres SSI, CockroachDB default, Spanner |

Implementations of serializability: actual serial execution (VoltDB, Redis single thread), **two-phase locking (2PL)**, **serializable snapshot isolation (SSI)**.

### 6.4 Preventing lost updates and write skew in practice
- **Atomic operations**: `UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100`.
- **Pessimistic locking**: `SELECT ... FOR UPDATE`.
- **Optimistic concurrency control (OCC)**: version column / ETag; `UPDATE ... WHERE id = 1 AND version = 7`; retry on conflict. HTTP: `If-Match` header.
- **Constraints**: unique indexes, foreign keys, check constraints, exclusion constraints (no overlapping bookings).
- **Materialising conflicts**: create rows to lock for things that don't exist yet (e.g., time slots).
- **Serializable isolation** with retry loops for serialization failures.

---

## 7. Distributed transactions and cross-service consistency

When one business operation spans multiple databases/services, local ACID isn't enough.

### 7.1 Two-phase commit (2PC)
```
Coordinator: "prepare?" → all participants vote yes/no (and durably promise)
Coordinator: if all yes → "commit"; else → "abort"
```
- ✅ Atomic across participants.
- ❌ Blocking: if the coordinator dies after prepare, participants hold locks until it returns. Slow (multiple round trips + fsyncs). Reduces availability.
- Used in: XA transactions, inside distributed databases (Spanner combines 2PC with Paxos-replicated participants to avoid blocking).

### 7.2 Saga pattern
A long-running transaction split into local transactions, each with a **compensating action**.

```
Create order → Reserve inventory → Charge payment → Arrange shipping
      ↑ if payment fails:  release inventory ← cancel order
```
- **Choreography**: services react to each other's events (decentralised; harder to follow).
- **Orchestration**: a central orchestrator (state machine / workflow engine like Temporal, AWS Step Functions, Camunda) drives steps and compensations.
- Sagas give **eventual** consistency; intermediate states are visible (e.g., "order pending"). Design compensations carefully (you can't un-send an email — send a correction instead).
- Use **semantic locks** (status = PENDING) to prevent other operations from acting on in-progress data.

### 7.3 The dual-write problem and the outbox pattern
**Problem**: writing to the DB and publishing to a queue are two separate operations; a crash between them leaves them inconsistent.

```
❌  db.save(order); kafka.publish(OrderCreated)   // crash in between → event lost
```

**Transactional outbox**:
```
BEGIN;
  INSERT INTO orders ...;
  INSERT INTO outbox (event_type, payload) ...;
COMMIT;
-- separate relay process (or CDC with Debezium) reads outbox → publishes → marks sent
```
- Event published **at least once** → consumers must be idempotent.

**Inbox pattern** (consumer side): record processed message IDs in the same transaction as the side effect to deduplicate.

**Change Data Capture**: treat the DB's log as the source of events; avoids dual writes entirely.

### 7.4 TCC (Try–Confirm–Cancel)
Reserve resources in a "try" phase, then confirm or cancel. Similar to 2PC at the business level (e.g., hold a seat, then confirm booking).

### 7.5 Workflow engines / durable execution
Temporal, Cadence, AWS Step Functions, Azure Durable Functions: persist workflow state at each step, automatically retry and resume after crashes. Turns complex reliability logic (retries, timeouts, compensations) into ordinary-looking code.

### 7.6 Reconciliation
For anything involving money or external systems, periodically **compare** your records with the source of truth (payment provider, bank statements, partner systems) and fix discrepancies. Assume something will slip through.

---

## 8. Consensus, leader election, and distributed locks

### 8.1 Consensus
Getting multiple nodes to agree on a value (who is leader, the order of log entries) despite failures.
- **Raft** (etcd, Consul, CockroachDB, TiKV, Kafka KRaft) — designed for understandability: leader election + log replication.
- **Paxos / Multi-Paxos** (Spanner, Chubby), **Zab** (ZooKeeper), **Viewstamped Replication**.
- Requires a **majority** quorum: `2f + 1` nodes tolerate `f` failures. Use odd numbers (3, 5). More nodes = more fault tolerance but slower writes.
- FLP impossibility: no deterministic consensus algorithm can guarantee termination in a fully asynchronous system with even one crash — practical algorithms use timeouts to make progress.

You rarely implement consensus yourself — use etcd/ZooKeeper/Consul or a database built on it.

### 8.2 Leader election
- Use a consensus store: acquire a key with a **lease** (TTL), renew periodically. If renewal fails, step down.
- Kubernetes Lease objects, etcd election API, ZooKeeper ephemeral sequential nodes.

### 8.3 Distributed locks
- For **efficiency** (avoid doing duplicate work occasionally): Redis `SET key value NX PX 30000` is often fine.
- For **correctness** (must never have two holders): use a consensus-backed lock **and fencing tokens**. Redlock is controversial for correctness use cases (Kleppmann vs antirez debate).
- Always set a TTL; always handle the lock being lost mid-operation.

### 8.4 Fencing tokens
```
Client 1 gets lock (token 33) → long GC pause → lock expires
Client 2 gets lock (token 34) → writes to storage with token 34
Client 1 wakes up → tries to write with token 33 → storage rejects (33 < 34)
```
The protected resource must check tokens. This is how you make leases safe despite process pauses.

### 8.5 Uniqueness and constraints across nodes
Usernames, seat assignments, "only one active subscription" — need linearizable checks: single-leader DB with unique index, consensus store, or partitioning so the constraint is checked on a single partition.

---

## 9. Time, clocks, and ordering

### 9.1 Problems with physical clocks
- Clock drift (quartz ~ tens of ppm), NTP corrections can jump clocks **backwards**.
- Different machines disagree by milliseconds or more.
- Leap seconds, timezones, DST.

Rules:
- **Use monotonic clocks** for measuring durations/timeouts (`System.nanoTime`, `time.monotonic`, `CLOCK_MONOTONIC`).
- **Don't use wall-clock timestamps to order events across machines** (last-write-wins with skewed clocks silently drops data).
- Store timestamps in UTC with timezone info.

### 9.2 Logical clocks
- **Lamport timestamps**: counter incremented on each event and on receiving a message (`max(local, received) + 1`). Total order consistent with causality.
- **Vector clocks / version vectors**: one counter per node; can detect **concurrent** (conflicting) updates, not just order. Used by Dynamo, Riak.
- **Hybrid Logical Clocks (HLC)**: physical time + logical counter; used by CockroachDB, YugabyteDB.
- **TrueTime** (Spanner): GPS + atomic clocks give a bounded uncertainty interval; Spanner waits out the uncertainty to guarantee external consistency.

### 9.3 Ordering guarantees in practice
- **Total order** within a single partition of a log (Kafka partition, DB WAL). Put events that must be ordered relative to each other in the same partition (same key).
- **Sequence numbers** generated by a single leader.
- **Causal ordering** via dependencies/version vectors.

---

## 10. Reliable messaging

### 10.1 Delivery semantics

| Semantic | Meaning | How |
|---|---|---|
| **At-most-once** | May lose, never duplicate | Ack before processing; no retries |
| **At-least-once** | Never lose, may duplicate | Ack after processing; retry on failure — **the practical default** |
| **Exactly-once** | Effect happens exactly once | Really **at-least-once delivery + idempotent/transactional processing** |

"Exactly-once delivery" over an unreliable network is impossible; **exactly-once *processing*** is achievable:
- Kafka transactions (idempotent producer + atomic writes of output and consumer offsets) for Kafka-to-Kafka pipelines.
- Flink checkpoints + two-phase commit sinks.
- Idempotent consumers with deduplication (§11).

### 10.2 Ordering
- Ordering is only guaranteed per partition/queue, and only with one consumer per partition processing sequentially.
- Retries can reorder messages. If order matters, use sequence numbers and have consumers reject/park out-of-order events, or process per key sequentially.

### 10.3 Poison messages and dead letter queues
- A message that always fails will block the partition or be retried forever.
- After N attempts (with backoff), move it to a **DLQ**, alert, and allow inspection/replay.

### 10.4 Backpressure and durability in queues
- Persist messages (durable queues, replicated logs) before acking producers.
- Monitor consumer lag; define retention long enough to recover from consumer outages (e.g., 7 days).

### 10.5 Webhooks (sending to external systems)
- Retries with exponential backoff over hours/days; sign payloads (HMAC); include event ID for dedupe; allow receivers to replay; deliver via queue, not inline.

---

## 11. Idempotency

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. It's what makes retries safe — and retries are unavoidable (§3.3).

### 11.1 Naturally idempotent vs not
| Idempotent | Not idempotent |
|---|---|
| `SET balance = 100` | `balance = balance + 10` |
| `PUT /users/42 {...}` | `POST /orders` |
| `DELETE /items/7` | "Send email" |
| Upsert by natural key | Insert with auto-increment ID |

### 11.2 Making operations idempotent
- **Idempotency keys** (Stripe-style): client generates a unique key per logical operation and sends it in a header. Server stores `(key → result)`; on retry with the same key, returns the stored result instead of re-executing.
  ```
  POST /payments
  Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
  ```
  Store the key in the same transaction as the side effect; handle concurrent requests with the same key (lock or unique constraint); expire keys after a reasonable window (e.g., 24 h).
- **Client-generated IDs**: client creates the UUID for the new entity; a retry inserts the same ID → unique constraint makes it a no-op.
- **Deduplication table** for consumers: `processed_messages(message_id PRIMARY KEY)` written in the same transaction as the effect.
- **Conditional writes**: `UPDATE ... WHERE version = 7`, `INSERT ... ON CONFLICT DO NOTHING`, DynamoDB condition expressions.
- **State machines**: transitions only from valid states (`PENDING → PAID` can happen once; a second "pay" sees `PAID` and returns success without charging).
- **Convert deltas to absolutes** when possible, or record each delta with a unique ID.

---

## 12. Conflict resolution in replicated data

When multiple replicas accept writes concurrently (multi-leader, leaderless, offline clients), conflicts happen.

| Strategy | How | Trade-off |
|---|---|---|
| **Last-write-wins (LWW)** | Highest timestamp wins | Simple; **silently loses data**; vulnerable to clock skew |
| **Version vectors + siblings** | Keep all concurrent versions; app/user merges | No data loss; app complexity (Dynamo shopping cart) |
| **Application merge function** | Custom logic (union of sets, max, field-level merge) | Domain-specific |
| **CRDTs** (Conflict-free Replicated Data Types) | Data types whose merges are commutative, associative, idempotent → automatic convergence | Limited set of types; metadata overhead |
| **Operational Transformation (OT)** | Transform concurrent edits against each other | Collaborative text editing (Google Docs); complex |
| **Avoid conflicts** | Route all writes for a record to one leader/region ("home region") | Loses some availability/latency benefit |

Common CRDTs: G-Counter / PN-Counter (counters), G-Set / OR-Set (sets), LWW-Register, RGA / Yjs / Automerge (sequences/text, JSON documents).

---

## 13. Data integrity and correctness

### 13.1 Defend at the boundaries
- **Input validation** and schema validation (JSON Schema, Protobuf, OpenAPI) at every API boundary.
- **Database constraints** (NOT NULL, UNIQUE, FOREIGN KEY, CHECK) — the last line of defence; don't rely only on application code.
- **Strong types** for money (integers in minor units or decimal types — **never floats**), IDs, units.

### 13.2 Detect corruption
- **Checksums** at rest and in transit (TCP checksum is weak; use application-level CRC32C/xxHash/SHA).
- **End-to-end checks**: verify data at the final destination, not just per hop ("end-to-end argument").
- **Scrubbing**: periodically read all data and verify checksums (ZFS scrub, Ceph scrub, HDFS block scanner).
- **Auditing / invariant checks**: background jobs verifying business invariants (sum of ledger entries = account balance; no orphaned rows).
- **Reconciliation** with external systems (§7.6).

### 13.3 Immutable and auditable data
- **Append-only logs / event sourcing**: never update in place; you can always recompute derived state and audit history.
- **Double-entry bookkeeping** for financial systems: every transaction is balanced debits and credits; imbalances reveal bugs.
- **Audit logs**: who changed what, when, and why.
- **Soft deletes**, versioned records, temporal tables.

### 13.4 Schema evolution
- Backward- and forward-compatible changes so old and new code coexist during rollouts (add optional fields; never reuse field numbers in Protobuf; schema registry for Avro/Protobuf in Kafka).
- Expand–contract migrations (see [Availability §11](../availability/README.md#11-safe-deployments--change-management)).

### 13.5 Derived data
Caches, search indexes, materialised views, and analytics copies can drift from the source of truth. Make them **rebuildable** from the source (CDC + replay) and periodically verify.

---

## 14. Software fault tolerance patterns

| Pattern | Description |
|---|---|
| **Timeouts** | Bound every wait; treat timeout as "unknown outcome", not "failed" |
| **Retries with backoff + jitter** | Only with idempotency (§11) |
| **Circuit breakers / bulkheads** | Contain failures (see [Availability §8](../availability/README.md#8-resilience-patterns-containing-failures)) |
| **Fail fast** | Validate inputs and preconditions early; crash on impossible states rather than continue with corrupt state |
| **Crash-only software** | Design so the only way to stop is to crash and the only way to start is to recover — makes recovery path the normal path, well tested |
| **Supervision trees** (Erlang/OTP, Akka) | Supervisors restart failed child processes ("let it crash") |
| **Checkpointing** | Persist progress so long-running work resumes, not restarts |
| **Graceful degradation** | Reduced functionality rather than wrong results |
| **Defensive programming** | Assertions, invariant checks, explicit error handling |
| **Immutability** | Fewer shared-state race conditions |
| **Resource limits** | Bounded queues, memory limits, max request size, query timeouts |
| **Watchdogs** | Detect hung processes and restart them |
| **N-version / diversity** | Different implementations to avoid correlated bugs (aerospace; rarely in web systems) |
| **Safe defaults** | Fail closed for security/money (deny if unsure), fail open for non-critical features |

Error handling principles:
- Distinguish **retryable** (transient) vs **non-retryable** (validation, permanent) errors.
- Never swallow errors silently; log with context and propagate.
- Make "unknown outcome" explicit in APIs (e.g., payment status `PENDING`, then query or webhook to resolve).

---

## 15. Reducing human error

Humans cause most outages. Design systems that make mistakes hard and recovery easy:
- **Automation** over manual runbooks for routine tasks (deploys, failovers, scaling).
- **Infrastructure as code** (Terraform, Pulumi) with code review and plan/apply previews.
- **Guardrails**: confirmation for destructive operations, `--dry-run`, deletion protection on databases and buckets, policy-as-code (OPA), least-privilege IAM, separate prod credentials.
- **Gradual rollouts** and **fast rollback** for code *and* config.
- **Sandboxes/staging** environments that mirror production.
- **Good abstractions and APIs** that make the right thing easy.
- **Observability** to detect mistakes quickly.
- **Blameless postmortems**: fix the system that allowed the error, not the person.
- **Checklists** for risky manual operations (borrowed from aviation and surgery).
- **Two-person rule** for highly sensitive changes.

---

## 16. Testing for reliability

| Technique | What it catches |
|---|---|
| **Unit tests** | Logic bugs in isolation |
| **Integration tests** (with real DB/queue via Testcontainers) | Wiring, queries, serialisation |
| **Contract tests** (Pact) | API incompatibilities between services |
| **End-to-end tests** | Critical user journeys |
| **Property-based testing** (QuickCheck, Hypothesis, fast-check) | Edge cases you didn't think of; invariants hold for random inputs |
| **Fuzzing** (AFL, libFuzzer, go-fuzz) | Crashes and security bugs from malformed input |
| **Model-based / stateful testing** | Sequences of operations against a simple model |
| **Deterministic simulation testing** (FoundationDB, TigerBeetle, Antithesis) | Run the whole distributed system in a simulator with injected faults and reproducible seeds |
| **Jepsen-style testing** | Consistency violations under partitions, clock skew, crashes |
| **Fault injection / chaos engineering** | Behaviour under real failures (kill nodes, latency, packet loss) |
| **Formal methods** (TLA+, P, Alloy) | Design-level bugs in concurrent/distributed protocols (used at AWS, Azure, MongoDB) |
| **Disaster recovery drills** | Backups actually restore; failover actually works |
| **Canary analysis** | Bugs that only appear with production traffic |
| **Mutation testing** | Weak tests that don't actually assert behaviour |
| **Static analysis / type systems** | Null errors, races (Rust borrow checker, race detectors) |

The test pyramid: many fast unit tests, fewer integration tests, few slow end-to-end tests — plus targeted distributed-systems testing where correctness is critical.

---

## 17. Approaches by system type

### 17.1 Banking / payments / ledgers
- **Correctness above everything**; prefer unavailability to inconsistency (CP).
- Double-entry, append-only ledger; balances derived from or verified against entries.
- Serializable transactions or single-writer-per-account designs; strict constraints.
- Idempotency keys on every money-moving call; state machines for payment lifecycle; handle "unknown" outcomes from processors.
- Synchronous replication (RPO = 0); audited backups.
- Daily reconciliation with banks/processors; audit logs; regulatory requirements (PCI DSS, SOX).
- Money as integers in minor units or decimals; explicit currency.

### 17.2 E-commerce / booking / ticketing
- Inventory and seat allocation need strong consistency at the point of reservation (conditional decrements, unique constraints on seat IDs, reservations with expiry).
- Orders across services via **sagas** with compensation (release inventory, refund payment).
- Outbox pattern for order events; idempotent order creation (client-generated order ID).
- Cart can be eventually consistent (merge on conflict).

### 17.3 Social media / content platforms
- Eventual consistency is acceptable for counters, feeds, likes; but **read-your-writes** is important (users must see their own posts/comments).
- Strong consistency for: account security, privacy settings (a "make private" change must take effect everywhere quickly), blocks.
- Durable storage of user content (object storage with replication + versioning).

### 17.4 Messaging / chat
- Messages must not be lost or duplicated in the UI: client-generated message IDs, at-least-once delivery + dedupe, per-conversation sequence numbers for ordering.
- Store-and-forward for offline recipients; delivery/read receipts as separate idempotent events.
- E2E encryption systems must handle key changes reliably.

### 17.5 Collaborative editing (docs, whiteboards, design tools)
- CRDTs (Yjs, Automerge) or OT for convergence; offline edits merged on reconnect.
- Persist operation logs; periodic snapshots; version history for recovery.

### 17.6 Data pipelines / ETL / analytics
- **Idempotent, deterministic, re-runnable** jobs (overwrite partitions instead of appending).
- Keep immutable raw data so everything can be recomputed.
- Data quality checks (Great Expectations, dbt tests, Soda): row counts, nulls, uniqueness, freshness, distribution drift.
- Exactly-once processing in streams (Flink checkpoints, Kafka transactions) or idempotent sinks.
- Schema registry and contracts between producers and consumers (data contracts).
- Late and out-of-order events handled with event-time windows and watermarks.

### 17.7 Storage systems / databases / file systems
- WAL + fsync discipline, checksums on every block, replication or erasure coding, scrubbing, crash-consistency testing.
- Deterministic simulation testing and Jepsen tests.

### 17.8 Distributed coordination / infrastructure (config, discovery, schedulers)
- Consensus-based stores (etcd/ZooKeeper), fencing tokens, leases.
- Config changes validated, versioned, rolled out gradually — a bad global config push is a classic total outage.

### 17.9 IoT / edge / mobile offline
- Unreliable connectivity: local durable queue, at-least-once upload with message IDs, server-side dedupe.
- Clock skew on devices: use server receive time + device sequence numbers.
- Conflict resolution for offline edits (CRDTs, LWW with care, server-authoritative merge).
- Firmware updates: A/B partitions with automatic rollback.

### 17.10 Healthcare, aviation, automotive, industrial control (safety-critical)
- Formal verification, redundancy with diverse implementations, certification standards (DO-178C, ISO 26262, IEC 62304), extensive testing, fail-safe states, hard real-time guarantees.

### 17.11 ML systems
- Reliability includes **model correctness over time**: data validation on input features, training/serving skew detection, model monitoring for drift, versioned models and datasets, shadow deployments and rollback.
- LLM apps: output validation/guardrails, structured outputs with schema validation, fallbacks when the model or provider fails, evaluation suites as regression tests.

### 17.12 Internal CRUD / back-office tools
- Standard relational DB with transactions and constraints is usually enough; focus on backups, validation, audit logs, and access control.

### Summary table

| System | Consistency need | Key reliability techniques |
|---|---|---|
| Banking/payments | Strong (serializable) | Ledger, idempotency, reconciliation, sync replication |
| E-commerce/booking | Strong at reservation, eventual elsewhere | Sagas, outbox, conditional writes |
| Social | Eventual + read-your-writes | Session guarantees, durable content storage |
| Chat | Per-conversation order, no loss/dupes | Client IDs, sequence numbers, dedupe |
| Collaboration | Convergence | CRDTs/OT, op logs |
| Data pipelines | Exactly-once results | Idempotent jobs, immutable raw data, data quality checks |
| Storage engines | Durability | WAL, fsync, checksums, scrubbing |
| Infra/control plane | Linearizable | Consensus, leases, fencing |
| IoT/mobile | Eventual, offline | Local queues, dedupe, conflict resolution |
| Safety-critical | Verified correctness | Formal methods, diverse redundancy |

---

## 18. Trade-offs

| Reliability technique | What it costs |
|---|---|
| Strong consistency / linearizability | **Latency** (coordination), **availability** under partitions |
| Synchronous replication | Write **latency**; writes block if replicas unavailable |
| Serializable isolation | **Throughput** (aborts/locks), retry logic needed |
| fsync on every write | Write **latency/throughput** (mitigate with group commit) |
| 2PC | **Availability** (blocking), latency |
| Sagas | Complexity; intermediate states visible; compensations can be hard |
| Idempotency keys / dedupe tables | Storage, extra lookups |
| Checksums, scrubbing, validation | CPU and I/O overhead |
| Extensive testing / formal methods | Engineering **time** |
| Immutable / append-only data | Storage cost, compaction needs |
| CRDTs | Metadata overhead, limited operations |
| Conservative rollouts | Slower delivery of features |

The key judgement: **which data must be strongly correct, and which can be eventually correct?** Apply expensive guarantees only where they matter (money, inventory, permissions, uniqueness) and cheap ones elsewhere (likes, view counts, recommendations).

---

## 19. Anti-patterns

- Treating a timeout as "the operation failed" and retrying non-idempotent operations → double charges.
- **Dual writes** (DB + queue, DB + cache, DB + search) without outbox/CDC.
- Using wall-clock timestamps from different machines to order events or resolve conflicts.
- Floats for money.
- Relying on application checks without DB constraints (race conditions between check and insert).
- Distributed locks without fencing for correctness-critical work.
- Assuming "exactly-once delivery" from a message broker without idempotent consumers.
- Replication treated as a backup; backups never restore-tested.
- Swallowing exceptions; logging and continuing with corrupt state.
- Read-modify-write without locking or version checks (lost updates).
- Using the default isolation level without knowing what anomalies it allows.
- Global config changes pushed everywhere at once.
- No reconciliation with external systems of record.

---

## 20. Checklist

**Data durability**
- [ ] Writes acknowledged only after durable (fsync / replicated per RPO)
- [ ] Replication factor and write concern set deliberately
- [ ] Backups: automated, off-site, immutable, PITR, regularly restore-tested
- [ ] Checksums on stored data; corruption detection

**Consistency**
- [ ] Consistency requirement defined per data type (strong vs eventual)
- [ ] Read-your-writes for user-facing updates
- [ ] Isolation level chosen with known anomalies addressed
- [ ] DB constraints enforce invariants

**Distributed operations**
- [ ] All externally retried operations idempotent (idempotency keys / client IDs)
- [ ] Outbox or CDC instead of dual writes
- [ ] Sagas with compensations (or workflow engine) for cross-service operations
- [ ] Consumers idempotent; DLQs configured; ordering requirements handled
- [ ] Leases + fencing tokens for leader/lock-protected work
- [ ] No reliance on cross-machine wall-clock ordering

**Correctness**
- [ ] Input/schema validation at boundaries
- [ ] Invariant checks and reconciliation jobs
- [ ] Audit logs for important changes
- [ ] Derived data rebuildable from source of truth

**Process**
- [ ] Test strategy beyond unit tests (integration, property-based, fault injection)
- [ ] Automated, gradual, reversible deploys and config changes
- [ ] Guardrails against destructive human actions
- [ ] Postmortems with tracked action items

---

## 21. Further reading

- *Designing Data-Intensive Applications* (Kleppmann) — ch. 1 (reliability), 7 (transactions), 8 (distributed systems trouble), 9 (consistency & consensus), 11–12 (streams, correctness) — **the** book for this topic
- *Database Internals* (Alex Petrov) — storage engines, replication, consensus
- "Designing Data-Intensive Applications" companion: Kleppmann's "How to do distributed locking"
- Jepsen analyses — jepsen.io (real consistency bugs in real databases)
- Raft paper: "In Search of an Understandable Consensus Algorithm" + raft.github.io visualisation
- "Time, Clocks, and the Ordering of Events" (Lamport, 1978)
- Stripe engineering: "Designing robust and predictable APIs with idempotency"
- Microservices.io (Chris Richardson): Saga, Transactional Outbox patterns
- Pat Helland: "Life beyond Distributed Transactions", "Immutability Changes Everything"
- "Use of Formal Methods at Amazon Web Services" (Newcombe et al.)
- Crash-only software (Candea & Fox); Erlang/OTP "let it crash" philosophy
- FoundationDB / TigerBeetle talks on deterministic simulation testing
