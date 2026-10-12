# Durability

> **Durability** = once the system says data is saved, it **stays saved**: through crashes, power loss, disk failures, data center loss, bugs and human mistakes.
> "If we told the user 'saved', can we lose it?"

Related: [Reliability §4](../reliability/README.md#4-durability-never-lose-acknowledged-data) (durability mechanisms in depth), [Recoverability](../recoverability/README.md) (getting data back after loss), [Availability §12](../availability/README.md#12-disaster-recovery) (disaster recovery, backups).

---

## 1. Durability vs availability

They're independent:
- **Available, not durable**: a cache serving fast responses; if it restarts, data is gone.
- **Durable, not available**: data safe in a backup vault, but the service is down for hours.

Amazon S3 is designed for **99.999999999% (11 nines) durability** but has a **99.9–99.99% availability** SLA. Losing data is far worse than a few minutes of downtime, so durability targets are much higher.

What 11 nines means (AWS's own example): store **10 million objects**, and on average expect to lose **one object every 10,000 years**.

---

## 2. What threatens data

| Threat | Defence |
|---|---|
| Process crash | Write-ahead log (WAL) |
| OS crash / power loss | `fsync` before acknowledging; power-loss-protected disks |
| Disk failure | RAID, replication, erasure coding |
| Silent corruption (bit rot, bad firmware) | Checksums + scrubbing |
| Node / zone / region loss | Replication across zones/regions |
| Software bug corrupting data | Backups, point-in-time recovery, versioning |
| Human error (`DROP TABLE`, wrong script) | Backups, soft delete, deletion protection |
| Ransomware / malicious deletion | **Immutable**, off-account backups (object lock) |
| Cloud account loss / provider issue | Backups in a separate account or provider |

**Replication is not a backup**: it copies mistakes instantly. You need both.

---

## 3. What does "acknowledged" actually mean?

Every data store lets you choose when "saved" is reported. The defaults are not always the safest.

| System | Setting | Trade-off |
|---|---|---|
| PostgreSQL | `synchronous_commit` (`on` / `off` / `remote_apply`), `synchronous_standby_names` | `off` = faster commits, may lose the last few hundred ms of commits on crash |
| MySQL InnoDB | `innodb_flush_log_at_trx_commit` (1 / 2 / 0), `sync_binlog` | `1` = fsync per commit (safest) |
| MongoDB | `writeConcern` (`w: 1` / `w: "majority"`, `j: true`) | `majority` survives primary loss |
| Kafka | `acks`, `min.insync.replicas`, `replication.factor` | `acks=1` can lose data on leader failure |
| Redis | `appendonly`, `appendfsync` (`always` / `everysec` / `no`) | `everysec` may lose ~1 s of writes |
| Cassandra | Consistency level (`ONE` / `QUORUM`), `commitlog_sync` | `ONE` = one replica has it |

### Worked example: "only one second"

Redis with `appendfsync everysec` taking **5,000 writes/s**. The server loses power:

```
Up to 1 s of writes not yet fsynced → up to 5,000 lost writes
```

Fine for a cache or rate-limit counters. Not fine if Redis is the only place a shopping cart or job queue lives. **Match the setting to what the data is**, and know your RPO for every store.

---

## 4. Layers of defence

```
Single node:   WAL + fsync + checksums
Many nodes:    synchronous replication / quorum writes (survive node loss)
Many places:   multi-AZ / multi-region copies (survive data center loss)
Time:          backups + point-in-time recovery + versioning (survive bugs and humans)
Isolation:     immutable, separate-account copies (survive attackers and account loss)
Verification:  checksums, scrubbing, regular restore tests (prove it works)
```

Backup rule of thumb, **3-2-1-1-0**: 3 copies, 2 different media/services, 1 off-site, 1 immutable/offline, 0 errors in restore tests.

---

## 5. Anti-patterns

- Acknowledging writes before they're fsynced or replicated, without knowing it.
- Treating replicas as backups.
- Backups in the same account/region as production.
- Backups never restored ("Schrödinger's backup").
- The only copy of in-flight work in memory (in-process queues, write-behind caches).
- No checksums, so corruption spreads into backups unnoticed.

---

## 6. Checklist

- [ ] RPO defined per data store
- [ ] Write durability settings chosen deliberately for each store
- [ ] Replication across zones for important data
- [ ] Automated backups + PITR, in a separate account, immutable
- [ ] Object storage versioning and deletion protection
- [ ] Checksums and corruption detection
- [ ] Restore tested regularly (automated if possible)

---

## 7. Exercises

1. For each store in a project you built, write down: what "saved" means today, and what you'd lose on (a) a process crash, (b) power loss, (c) disk loss, (d) `DELETE` without `WHERE`.
2. Kafka with `replication.factor=3`, `min.insync.replicas=1`, `acks=all`. Is it safe? What happens when 2 brokers are slow?
3. Design backups for a SaaS DB of 500 GB with RPO 5 minutes and protection against ransomware.
4. Why does the 11-nines durability figure assume failures are independent? What correlated events could break it?

---

## 8. Further reading

*Designing Data-Intensive Applications* ch. 3, 5, 7; AWS S3 durability documentation and "Building and operating a pretty big storage system called S3" (Andy Warfield, 2023); the PostgreSQL "fsyncgate" discussion (2018); "Files are hard" (Dan Luu); Backblaze drive failure statistics (published quarterly).
