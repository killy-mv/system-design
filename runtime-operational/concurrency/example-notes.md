# Concurrency: hands-on practice notes

Planned labs for learning concurrency by **reproducing races, deadlocks and contention, then fixing them**. Theory is in [README.md](README.md).

Every lab needs a **correctness check** (an invariant like "stock ≥ 0" or "sum of balances unchanged") and runs the operation **many times concurrently**, because race bugs only appear under the right timing.

They need a language with threads or async (Go is great here because of `-race`; Python, Node or Java also work), plus **PostgreSQL** in Docker.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** lost updates and write skew (reliability labs 1–2), distributed lock with fencing (reliability lab 9), counters at scale (scalability lab 11), blocking the event loop (performance lab 6).

---

## Labs

### 1. In-memory race ⭐
- **Build:** a counter incremented by 100 threads/goroutines × 10,000 times. Expected result: 1,000,000.
- **Break:** no synchronisation → a smaller number, different every run. In Go, run with `-race` and read the report.
- **Fix:** mutex, then atomic, then a single goroutine owning the counter with a channel. Benchmark all three.
- **See:** README §2, §3.

### 2. Async isn't race-free
- **Build:** Node.js or Python asyncio: `if (!cache[k]) cache[k] = await load(k)` hit by 100 concurrent requests, where `load` takes 100 ms.
- **Break:** count how many times `load` runs (it's many, not one).
- **Fix:** store the **promise/future** in the cache (request coalescing / single-flight).
- **See:** README §7, "awaits still interleave".

### 3. Overselling and the DB fixes ⭐
- **Build:** the README §2 worked example: `stock = 10`, 200 concurrent "buy" requests doing SELECT then UPDATE.
- **Break:** count orders vs stock (more than 10 orders).
- **Fix:** atomic conditional update, `FOR UPDATE`, version column + retry. Compare correctness, throughput and p99 for each.
- **See:** README §2, §4.

### 4. Deadlock
- **Build:** a transfer endpoint that locks the two account rows in request order. Run concurrent A→B and B→A transfers.
- **Break:** watch Postgres report `deadlock detected` and abort one transaction.
- **Fix:** lock in ID order; add retry on deadlock and serialization errors.
- **See:** README §5.

### 5. Job queue with SKIP LOCKED
- **Build:** a `jobs` table and 20 workers.
- **Compare:**
  1. `SELECT ... LIMIT 1` then `UPDATE` → duplicate processing.
  2. `FOR UPDATE` → no duplicates, but workers wait on each other.
  3. `FOR UPDATE SKIP LOCKED` → no duplicates and no waiting.
- **Measure:** jobs/s and duplicates for each.
- **See:** README §4.

---

## Progress

- [ ] 1. In-memory race
- [ ] 2. Async isn't race-free
- [ ] 3. Overselling and the DB fixes
- [ ] 4. Deadlock
- [ ] 5. Job queue with SKIP LOCKED
