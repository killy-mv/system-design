# Consistency: hands-on practice notes

Planned labs for learning consistency by **making stale and conflicting reads happen, then choosing the right fix for each**. Theory is in [README.md](README.md).

They need **Docker Compose**, **PostgreSQL** (primary + replica), **Redis**, and **Nginx**.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** replication lag and read-your-writes (reliability lab 6), outbox (reliability lab 4), clock skew and last-write-wins (reliability lab 10), split brain (availability lab 5).

---

## Labs

### 1. The profile photo bug, layer by layer ⭐
- **Build:** recreate the README §3 worked example: Postgres primary + replica with `recovery_min_apply_delay = '2s'`, a Redis cache (TTL 5 min), Nginx caching `/avatars/*` for 1 day, and a page showing the profile.
- **Break:** update the photo and reload. Note which layer serves stale data at each moment.
- **Fix one layer at a time:** read-your-writes routing, delete-on-write, versioned URLs. Re-test after each.
- **See:** README §3.

### 2. Cache update race
- **Build:** two concurrent writers updating the same product price, each doing "write DB, then **set** cache".
- **Break:** add a random delay between the two steps and run many times: sometimes the cache keeps the **older** price permanently (until TTL).
- **Fix:** "write DB, then **delete** cache", and compare. Then try CDC (Debezium) driving invalidation.
- **See:** README §6.

### 3. Quorums by hand
- **Build:** a tiny key-value store with 3 replicas (3 containers, an HTTP server each) and a client that writes to `W` nodes and reads from `R` nodes, picking the highest version.
- **Break:** `W = 1, R = 1` with one replica paused (`docker pause`) during the write → stale reads.
- **Fix:** `W = 2, R = 2`. Then pause 2 replicas and see that requests now **fail** instead of returning stale data: the CAP trade-off.
- **See:** README §2, §5.

### 4. Measure freshness
- **Build:** a search index (OpenSearch or just a second table updated by a worker) fed from the main DB via a queue.
- **Measure:** the time from write to "visible in search" (p50/p99). Define a freshness SLO ("99% visible within 5 s") and add a reconciliation job that finds documents missing from the index.
- **See:** README §5 "Measure it".

---

## Progress

- [ ] 1. The profile photo bug, layer by layer
- [ ] 2. Cache update race
- [ ] 3. Quorums by hand
- [ ] 4. Measure freshness
