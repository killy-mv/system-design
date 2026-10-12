# Reliability: hands-on practice notes

Planned labs for learning reliability by reproducing wrong results, lost data and duplicates on purpose, then fixing them. Theory is in [README.md](README.md).

Availability labs break things so the system goes **down**. Reliability labs break things so the system stays up but gets the **wrong answer**. So every lab needs a way to **check correctness**: an invariant (stock never negative, ledger sums to zero), a count (orders sent = orders received), or a comparison against expected data.

They mostly need **Docker Compose** locally, with **PostgreSQL** as the main database.

**Suggested order:** 1 → 3 → 4 → 7 → 9 → 12, then the rest. Labs 3 (idempotency) and 4 (outbox) teach the patterns you'll use most in real jobs.

**Planned folder layout:**

```
runtime-operational/reliability/labs/
  shared/checker/          ← invariant/count checker reused across labs
  01-lost-updates/
    docker-compose.yml
    app/
    README.md              ← steps + expected results
  02-write-skew/
  ...
```

---

## Labs

### 1. Lost updates and isolation levels
- **Build:** Postgres in Docker + a script that runs 100 concurrent "buy 1 item" requests as read → modify → write, against `stock = 50`.
- **Break:** under `READ COMMITTED`, stock ends above 0 while 100 orders succeed (overselling), or updates get lost.
- **Fix:** try each in turn, then compare correctness and throughput:
  1. Atomic update: `UPDATE ... SET stock = stock - 1 WHERE stock > 0`
  2. `SELECT ... FOR UPDATE`
  3. Version column (optimistic locking) + retry
  4. `SERIALIZABLE` + retry on serialization failure
- **See:** README §6.2–§6.4.

### 2. Write skew
- **Build:** the "doctors on call" example: two concurrent transactions each check "is someone else on call?" and then go off call.
- **Break:** under `REPEATABLE READ` (snapshot isolation) both succeed and nobody is on call.
- **Fix:** `SELECT ... FOR UPDATE` on the rows read, or `SERIALIZABLE`.
- **See:** an anomaly that repeatable read does *not* prevent (§6.2, §6.3).

### 3. Idempotent payments API ⭐
- **Build:** a fake payment provider that **charges, then randomly delays or drops the response**. A checkout service calls it with a timeout and retries.
- **Break:** count customers charged twice.
- **Fix:** an `Idempotency-Key` header + a keys table stored in the same transaction as the result. Then test the edge cases:
  - a retry after success
  - two concurrent requests with the same key
  - the same key with a different amount
  - a crash after charging but before saving (recover with a `PENDING` state + reconciliation)
- **See:** §11, especially the §11.3 timeline.

### 4. Dual write → transactional outbox ⭐
- **Build:** a service that saves an order to Postgres and then publishes `OrderCreated` to a queue (**Redis Streams** or **RabbitMQ**, or **Redpanda** if you want a Kafka API).
- **Break:** kill the app between the DB write and the publish (add a `sleep` + `docker kill`). Compare orders in the DB with events received.
- **Fix:** an outbox table written in the same transaction, plus a relay process that publishes and marks rows as sent. Then kill the relay after publishing but before marking: you get **duplicates**.
- **Fix the duplicates:** an idempotent consumer with an **inbox** (processed message IDs) table.
- **See:** §7.3, §10.1. Optional extension: replace the relay with **Debezium** (CDC).

### 5. Poison messages and the DLQ
- **Build:** a consumer that crashes on one malformed message.
- **Break:** watch it retry forever and block everything behind it.
- **Fix:** a retry limit with backoff → dead letter queue → alert → replay tool.
- **See:** §10.3.

### 6. Replication lag and read-your-writes
- **Build:** Postgres primary + async replica. Add lag on purpose with `recovery_min_apply_delay = '5s'` on the replica. The app writes to the primary and reads from the replica.
- **Break:** update a profile, refresh, see the old value. Refresh again and the value may jump around.
- **Fix:** read from the primary for N seconds after a user writes, or track the WAL position (LSN) and only read from a replica that has caught up.
- **See:** §5.4 session guarantees.

### 7. Message durability settings
- **Build:** Redpanda/Kafka with 3 brokers. A producer sends numbered messages; a checker finds gaps.
- **Break:** `acks=1` + kill the partition leader during writes → messages lost.
- **Fix:** `acks=all`, `replication.factor=3`, `min.insync.replicas=2`. Repeat and count again.
- **See:** §4.2. "We acknowledged it" only means something if it's actually replicated.

### 8. Durability on a single node (fsync)
- **Build:** a tiny append-only log in your language of choice: write records, with and without `fsync`, plus a **CRC per record**.
- **Break:**
  - Process crash (`kill -9`) **doesn't** lose unsynced data, because the OS page cache survives. Note this, it surprises people.
  - To simulate power loss you need **LazyFS** (from the Jepsen team, a FUSE file system that can drop unsynced data) or a throwaway VM.
  - Flip a byte in the file with a hex editor and make sure the CRC detects it.
- **Fix:** write → fsync → rename → fsync the directory; checksums; truncate a torn last record on recovery.
- **See:** §4.1, §13.2.

### 9. Distributed lock without fencing ⭐
- **Build:** two workers take a Redis lock (`SET NX PX 5000`) and then write to a shared resource.
- **Break:** `docker pause` worker 1 for 10 s while it holds the lock. Worker 2 gets the lock. Unpause worker 1: **both write**, and the data is corrupted.
- **Fix:** a fencing token (incrementing number) issued with the lock. The resource rejects writes with an older token.
- **See:** §3.4, §8.3, §8.4. A process pause is a real fault.

### 10. Clock skew and last-write-wins
- **Build:** two app containers writing to a store with LWW by wall-clock timestamp. Give one container a skewed clock with **libfaketime** (for example `FAKETIME="-2s"`).
- **Break:** a newer write loses to an older one, and the data disappears silently.
- **Fix:** version numbers / compare-and-set, or a Lamport/hybrid logical clock.
- **See:** §9, §12.

### 11. Saga with compensation
- **Build:** order → reserve inventory → charge payment → ship, as separate small services. Use either choreography (events) or an orchestrator. **Temporal** is great for this.
- **Break:** fail payment after inventory is reserved, and kill the orchestrator mid-saga.
- **Fix:** compensations (release inventory, cancel order), a `PENDING` semantic lock, and resuming after a crash.
- **See:** §7.2, §7.5.

### 12. Backup and point-in-time restore drill ⭐
- **Build:** Postgres with WAL archiving (**pgBackRest** or **WAL-G**) to a MinIO bucket.
- **Break:** run an "accidental" `DELETE FROM orders` without a `WHERE`. Note that the replica (if you have one from lab 6) deletes them too.
- **Fix:** restore to the second before the delete. Time it: that's your real RTO. Then restore **only one table / one customer**, which is the Atlassian lesson.
- **See:** §4.3. Replication isn't backup.

### 13. Property-based testing of a ledger
- **Build:** a small double-entry ledger (accounts, transfers). Write a property test with **Hypothesis** (Python) or **fast-check** (JS) that generates random transfer sequences, including concurrent ones.
- **Check invariants:** total sum = 0, no negative balances (if that's a rule), every balance = the sum of its entries.
- **See:** the test finds bugs you didn't think to write cases for (§13.3, §16).

### 14. Reconciliation job
- **Build:** a nightly job comparing your `payments` table with the fake provider's records (from lab 3).
- **Break:** inject mismatches: missing on your side, missing on theirs, different amounts.
- **Fix:** report and categorize each mismatch, auto-fix the safe ones, and alert on the rest.
- **See:** §7.6. You assume something will slip through, and you catch it.

---

## Bonus: distributed systems challenges

**Fly.io "Gossip Glomers"** challenges, run on Jepsen's **Maelstrom**. You implement echo, unique IDs, broadcast, a grow-only counter, a Kafka-style log and transactions, and Maelstrom tests them under partitions and checks correctness for you. It's the closest thing to Jepsen testing your own code, and it maps directly to README §5, §8, §10 and §12.

---

## Tools

| Tool | Used for |
|---|---|
| Docker Compose | Running every lab locally |
| PostgreSQL | Isolation levels, replication, outbox, PITR |
| Redis | Locks, Redis Streams |
| RabbitMQ / Redpanda (Kafka API) | Messaging, durability settings, DLQs |
| Debezium | Change data capture (lab 4 extension) |
| Temporal | Durable workflows / sagas |
| pgBackRest or WAL-G + MinIO | Backups and point-in-time recovery |
| libfaketime | Clock skew |
| LazyFS | Simulating loss of unsynced data |
| Hypothesis / fast-check | Property-based testing |
| Maelstrom | Distributed systems correctness checking |

## Progress

- [ ] 1. Lost updates and isolation levels
- [ ] 2. Write skew
- [ ] 3. Idempotent payments API
- [ ] 4. Dual write → transactional outbox
- [ ] 5. Poison messages and the DLQ
- [ ] 6. Replication lag and read-your-writes
- [ ] 7. Message durability settings
- [ ] 8. Durability on a single node (fsync)
- [ ] 9. Distributed lock without fencing
- [ ] 10. Clock skew and last-write-wins
- [ ] 11. Saga with compensation
- [ ] 12. Backup and point-in-time restore drill
- [ ] 13. Property-based testing of a ledger
- [ ] 14. Reconciliation job
- [ ] Bonus: Gossip Glomers
