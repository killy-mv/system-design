# Recoverability: hands-on practice notes

Planned labs for learning recoverability by **breaking things, recovering, and timing every step**. Theory is in [README.md](README.md).

The point of every lab is a **measured** RTO and RPO: start a stopwatch when the failure happens, stop it when users are served correctly again, and count what data was lost.

They need **Docker Compose**, **PostgreSQL**, **pgBackRest** or **WAL-G**, **MinIO**, and optionally **Kafka/Redpanda**.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** DB failover (availability lab 4), backup and PITR drill (reliability lab 12), mini game day (availability bonus).

---

## Labs

### 1. Measure restore time ⭐
- **Build:** Postgres with ~10–20 GB of generated data and WAL archiving to MinIO.
- **Run:** a full restore into a fresh container, timing each phase (download, restore, WAL replay, start, verify).
- **Then:** extrapolate to your production size like the README §3 worked example. Does it meet your RTO?

### 2. Delayed replica vs backup
- **Build:** a normal replica plus a **delayed replica** (`recovery_min_apply_delay = '30min'`).
- **Break:** an accidental `DELETE` on an important table.
- **Recover twice:** from the delayed replica (pause replay, copy the rows back) and from PITR. Time both.
- **See:** README §3 techniques.

### 3. Roll back a bad deploy ⭐
- **Build:** an app + migration pipeline. Ship a release with a **non-backward-compatible** migration (rename a column).
- **Break:** the release has a bug and you try to roll back: the old version crashes on the new schema.
- **Fix:** redo it with expand–contract ([Availability §11](../availability/README.md#11-safe-deployments--change-management)) and a feature flag as kill switch. Roll back again and time it.

### 4. Rebuild state from a log
- **Build:** a service that keeps account balances as a projection of events in Kafka/Redpanda (or an `events` table).
- **Break:** corrupt or delete the projection table.
- **Recover:** replay events from the start (or from a snapshot) to rebuild it. Time it, then add periodic snapshots and time it again.

### 5. Restore one tenant
- **Build:** a multi-tenant schema (`tenant_id` on every table).
- **Break:** delete all data for one tenant (the Atlassian 2022 scenario, [Reliability §20](../reliability/README.md#20-real-world-incidents)).
- **Recover:** restore a backup into a **separate** database and copy back only that tenant's rows, without touching the other tenants. Write the runbook as you go.

---

## Progress

- [ ] 1. Measure restore time
- [ ] 2. Delayed replica vs backup
- [ ] 3. Roll back a bad deploy
- [ ] 4. Rebuild state from a log
- [ ] 5. Restore one tenant
