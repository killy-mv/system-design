# Durability: hands-on practice notes

Planned labs for learning durability by **counting lost writes under different settings**. Theory is in [README.md](README.md).

Every lab uses the same **writer + checker**: a writer sends numbered records (1, 2, 3, ...) and logs every one that was acknowledged; after the failure, a checker lists acknowledged records that are missing. **Acknowledged but missing = a durability bug.**

They need **Docker Compose**, **PostgreSQL**, **Redis**, and **Redpanda** (Kafka API).

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** fsync and checksums on a single node (reliability lab 8), Kafka durability settings (reliability lab 7), backup and PITR drill (reliability lab 12).

---

## Labs

### 1. Redis persistence modes ⭐
- **Build:** the writer sending `SET` commands to Redis, under each config: no persistence, RDB snapshots only, AOF `everysec`, AOF `always`.
- **Break:** `docker kill` Redis (and, if you can, a real VM power-off) mid-stream, restart, run the checker.
- **Measure:** lost writes and write throughput for each mode.
- **See:** README §3 worked example.

### 2. Postgres synchronous_commit
- **Build:** the writer inserting rows, with `synchronous_commit = on` vs `off`, then a primary + sync standby with `remote_apply`.
- **Break:** kill the primary mid-stream (and promote the standby in the replicated setup).
- **Measure:** lost acknowledged rows and commit latency in each mode.
- **Note:** like in reliability lab 8, a process kill keeps the OS page cache, so loss with `off` may be rare. A VM power-off or LazyFS shows the real behaviour.

### 3. Replication is not backup
- **Build:** primary + replica, with object storage (MinIO) holding versioned uploads.
- **Break:** `DELETE FROM documents` and overwrite/delete files in MinIO.
- **See:** the replica deletes everything instantly; MinIO versioning lets you restore files; only a backup/PITR brings the rows back.
- **Then:** enable **object lock** on the backup bucket and try to delete a backup.
- **See:** README §2, §4.

### 4. Detect corruption
- **Build:** store files with a SHA-256 checksum in a metadata table.
- **Break:** flip bytes in a few stored files.
- **Fix:** a scrubbing job that re-reads everything, compares checksums, and restores bad files from a second copy.
- **See:** README §4 "Verification".

---

## Progress

- [ ] 1. Redis persistence modes
- [ ] 2. Postgres synchronous_commit
- [ ] 3. Replication is not backup
- [ ] 4. Detect corruption
