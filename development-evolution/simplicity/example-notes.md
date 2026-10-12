# Simplicity: hands-on practice notes

Planned labs for learning simplicity by **finding accidental complexity in a real codebase and removing it, measuring what you saved**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)), plus your other lab projects.

**Suggested order:** 1 → 2 → 3 → 4 → 5 → 6.

**Related labs elsewhere:** job queue with `SKIP LOCKED` (concurrency lab 5), package by feature (modularity lab 3).

---

## Labs

### 1. Complexity inventory ⭐
- **Do:** for messy-shop *and* all the lab environments you've built so far (availability, reliability, observability...), list every component and technology. For each: why it's there, what it costs to operate (backups, upgrades, monitoring), and what would replace it.
- **Then:** pick one to remove and remove it.
- **See:** README §7 worked example.

### 2. Booleans → state machine ⭐
- **Do:** replace messy-shop's `is_paid / is_shipped / is_cancelled / is_refunded` with a status enum and an explicit transition table, plus a DB check constraint on allowed values.
- **Measure:** impossible combinations removed (16 → 5), defensive `if` checks deleted, tests needed before vs after.
- **See:** README §5 worked example.

### 3. Postgres for (almost) everything
- **Do:** in a branch, replace Redis/RabbitMQ pieces with Postgres: a job queue (`FOR UPDATE SKIP LOCKED`), search (`tsvector` + GIN index), notifications (`LISTEN/NOTIFY`), scheduled jobs (`pg_cron`).
- **Measure:** throughput and latency of each vs the dedicated tool with k6. Find where Postgres stops being good enough. Write down the threshold.
- **See:** README §6.

### 4. Define errors out of existence
- **Do:** take messy-shop's cart API (errors for "item not in cart", "cart empty", "quantity zero", "already removed"...). Redesign so most cases can't happen or don't need to be errors (idempotent removes, setting quantity 0 = remove, empty cart = empty list).
- **Measure:** error codes and branches before vs after; client code simplified.

### 5. Delete things
- **Do:** find dead code with **vulture** (Python) or **knip** / **ts-prune** (TS), unused endpoints from access logs, stale feature flags, unused tables. Delete them.
- **Measure:** lines and files removed; CI time before/after.

### 6. The "new engineer" test
- **Do:** write 3 small change tasks for messy-shop. Ask a friend (or yourself after 2 weeks away) to do them before and after the simplification labs. Time each and note where they got confused.
- **See:** README §8. The most honest measure of simplicity.

---

## Progress

- [ ] 1. Complexity inventory
- [ ] 2. Booleans → state machine
- [ ] 3. Postgres for (almost) everything
- [ ] 4. Define errors out of existence
- [ ] 5. Delete things
- [ ] 6. The "new engineer" test
