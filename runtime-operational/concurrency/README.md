# Concurrency

> **Concurrency** = the ability to handle many operations **at overlapping times**, while keeping results **correct**.
> "When 1,000 requests touch the same data at once, does it still give the right answer, without slowing to a crawl?"

Related: [Performance §9](../performance/README.md#9-concurrency-models--io) (concurrency models and I/O), [Reliability §6](../reliability/README.md#6-transactions-and-isolation) (transactions and isolation), [Consistency](../consistency/README.md), [Throughput](../throughput/README.md).

---

## 1. Concurrency vs parallelism

| | Concurrency | Parallelism |
|---|---|---|
| Meaning | *Dealing with* many things at once (structure) | *Doing* many things at once (execution) |
| Needs multiple cores? | No: one core can interleave tasks | Yes |
| Example | One event loop handling 10k connections | 8 cores each resizing a different image |

"Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once." (Rob Pike)

Web systems are concurrent everywhere: many threads in one process, many processes on one machine, many machines sharing one database. The same problems show up at every level.

---

## 2. What goes wrong

| Problem | What happens | Example |
|---|---|---|
| **Race condition** | The result depends on timing | Two threads do `count = count + 1`; one increment is lost |
| **Check-then-act (TOCTOU)** | The condition changes between check and action | "Is the username free?" → yes → insert → duplicate username |
| **Lost update** | Two read-modify-write cycles overwrite each other | Two admins edit the same record; the first edit vanishes |
| **Deadlock** | Two tasks each wait for a lock the other holds | Transfer A→B locks A then B; transfer B→A locks B then A |
| **Livelock** | Tasks keep reacting to each other, no progress | Two retries backing off in sync forever |
| **Starvation** | Some task never gets the resource | A writer never gets a lock because readers keep arriving |
| **Visibility / memory ordering** | One thread doesn't see another's write | Missing `volatile`/atomic/memory barrier in Java, C++, Go |
| **Contention** | Correct, but everyone queues on one lock or row | A global counter row updated by every request |

### Worked example: overselling the last item

```
stock = 1

Request A: SELECT stock → 1          Request B: SELECT stock → 1
Request A: 1 > 0, OK                 Request B: 1 > 0, OK
Request A: UPDATE stock = 0          Request B: UPDATE stock = 0
Request A: create order              Request B: create order
→ 2 orders, 1 item.
```

Each request is correct alone. Together they're wrong. Fixes (pick one):

```sql
-- 1. Atomic conditional update: the check and the write are one operation
UPDATE items SET stock = stock - 1 WHERE id = 42 AND stock > 0;   -- 0 rows → sold out

-- 2. Pessimistic lock: the second request waits
SELECT stock FROM items WHERE id = 42 FOR UPDATE;

-- 3. Optimistic: fail and retry if someone changed it
UPDATE items SET stock = 0, version = 8 WHERE id = 42 AND version = 7;
```

---

## 3. Tools inside one process

| Tool | Use when |
|---|---|
| **Mutex / lock** | Protecting shared mutable state; keep critical sections short |
| **Read-write lock** | Many readers, few writers |
| **Atomics / compare-and-swap (CAS)** | Counters, flags, lock-free structures |
| **Immutable data** | Nothing to protect if nothing changes |
| **Message passing (channels, actors, queues)** | "Don't communicate by sharing memory; share memory by communicating" (Go) |
| **Confinement** | Only one thread/task owns the data (event loop, actor, single-writer) |
| **Concurrent collections** | `ConcurrentHashMap`, `sync.Map`, etc. instead of locking a plain map |
| **Semaphores / bounded pools** | Limit how many tasks do something at once |

---

## 4. Tools across processes and machines

The database is usually where concurrency gets decided:
- **Atomic statements** and **constraints** (unique indexes make "check-then-insert" safe).
- **Transactions** with a conscious **isolation level** ([Reliability §6](../reliability/README.md#6-transactions-and-isolation)).
- **Row locks** (`SELECT ... FOR UPDATE`), and `FOR UPDATE SKIP LOCKED` for job queues (many workers, each takes different rows).
- **Optimistic concurrency**: version columns, HTTP `ETag` + `If-Match`.
- **Single writer per key**: partition so only one worker/consumer handles a given key (Kafka partition by key, actor per entity). No locking needed.
- **Distributed locks** with leases and **fencing tokens** ([Reliability §8](../reliability/README.md#8-consensus-leader-election-and-distributed-locks)).
- **Idempotency** so duplicates from concurrent retries are harmless.

---

## 5. Deadlocks

A deadlock needs all four (Coffman conditions): mutual exclusion, hold-and-wait, no preemption, circular wait. Break any one:
- **Lock ordering**: always lock in the same global order (e.g. lower account ID first).
- **Timeouts / try-lock**, then back off and retry.
- **Lock less**: shorter transactions, fewer rows, no user interaction or network calls while holding a lock.
- Databases detect deadlocks and abort one transaction, so **your code must retry** on deadlock errors.

---

## 6. Testing concurrency

Concurrency bugs hide in normal tests because the bad timing is rare.
- **Race detectors**: Go `-race`, ThreadSanitizer (C/C++/Rust), Java `jcstress`.
- **Stress tests**: run the operation from 50 threads × 10,000 times and check invariants.
- **Deterministic testing**: `loom` (Rust), deterministic simulation (FoundationDB style).
- **Jepsen-style** tests for distributed systems.

---

## 7. Anti-patterns

- Check-then-act in application code without a constraint or lock.
- Holding a lock (or a DB transaction) during a network call.
- One global lock/counter in the hot path.
- Ignoring deadlock/serialization errors instead of retrying.
- Shared mutable state in async code "because it's single-threaded" (awaits still interleave).
- Assuming tests passing means race-free.

---

## 8. Checklist

- [ ] Every read-modify-write on shared data is atomic, locked, or versioned
- [ ] Uniqueness enforced by constraints, not just application checks
- [ ] Locks acquired in a consistent order; no I/O while holding locks
- [ ] Deadlock / serialization failures retried
- [ ] Hot rows/keys identified and split (sharded counters, partitioning)
- [ ] Race detector / stress tests in CI for concurrent code

---

## 9. Exercises

1. Write the timeline for two concurrent "transfer $100" requests that cause a deadlock. Fix it with lock ordering.
2. A signup form checks `SELECT ... WHERE email = ?` then inserts. Show the race. Which single DB change fixes it?
3. In Node.js, `if (!cache[k]) { cache[k] = await load(k) }` is called by 100 concurrent requests. How many times does `load` run? Fix it (request coalescing).
4. Design a job queue in Postgres for 20 workers so no job runs twice and workers don't block each other.

---

## 10. Further reading

*Designing Data-Intensive Applications* ch. 7 (transactions, race conditions); *Java Concurrency in Practice* (Goetz); *The Little Book of Semaphores* (Downey, free); Go blog "Share Memory By Communicating"; Rob Pike, "Concurrency is not Parallelism" (talk); Postgres docs "Explicit Locking".
