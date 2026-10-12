# Recoverability

> **Recoverability** = how **quickly** and **completely** the system gets back to a correct, working state after a failure, including its data.
> "Something broke badly. How long until we're back, and how much did we lose?"

Related: [Fault tolerance](../fault-tolerance/README.md) (avoid failing in the first place), [Durability](../durability/README.md) (data survives), [Availability §12](../availability/README.md#12-disaster-recovery) (disaster recovery strategies), [Availability §1.2](../availability/README.md#12-mtbf-and-mttr) (MTTR).

---

## 1. Metrics

| Metric | Question | Example target |
|---|---|---|
| **RTO** (Recovery Time Objective) | Max acceptable downtime | 1 hour |
| **RPO** (Recovery Point Objective) | Max acceptable data loss, as time | 5 minutes |
| **MTTD** | How long until we notice? | < 5 min |
| **MTTR** | How long until it's fixed? | < 30 min |
| **Restore success rate** | Do restore tests actually work? | 100% |

```
         RPO                     RTO
◄──────────────────►◄──────────────────────►
──●─────────────────✖───────────────────────●──► time
last good copy    failure               back to normal
```

Fault tolerance tries to make failures invisible. Recoverability accepts that some will get through and makes them **short and cheap**.

---

## 2. What you might need to recover from

| Scope | Typical recovery | Typical time |
|---|---|---|
| Process crash | Supervisor restarts it | Seconds |
| Bad deploy / config | Rollback, flip a feature flag | Minutes |
| Node loss | Replace instance (autoscaling, Kubernetes) | Minutes |
| Primary DB loss | Failover to replica | Seconds–minutes |
| Data corruption / bad migration / accidental delete | Point-in-time restore, replay from log, fix-up script | Hours |
| Zone / region loss | Fail over to another zone/region (DR plan) | Minutes–hours |
| Ransomware / compromised account | Restore from immutable backups into a clean environment | Hours–days |

The scary ones are at the bottom: **logical** damage (bugs, humans, attackers) that replication copies everywhere.

---

## 3. Techniques

- **Automate the common recoveries**: restarts, instance replacement, failover, rollback. A human in the loop adds 10–30 minutes.
- **Immutable infrastructure + infrastructure as code**: rebuild any environment from code instead of repairing it by hand.
- **Fast rollback**: keep the previous version deployable; feature flags as kill switches; backward-compatible schema changes so you *can* roll back.
- **Backups + PITR**, and **delayed replicas** (a replica 1 hour behind lets you recover from a mistake quickly).
- **Event logs**: if state is derived from an append-only log (event sourcing, Kafka), you can rebuild it by replay.
- **Soft deletes and versioning**: undo at the record level instead of restoring a whole DB.
- **Checkpoints**: long jobs resume from the last checkpoint instead of starting over.
- **Runbooks** for recoveries that can't be automated: exact commands, owners, verification steps.

### Worked example: can a backup meet the RTO?

A 2 TB Postgres database is lost. The RTO is **1 hour**. Recovery from backup:

```
Download + restore base backup: 2,000,000 MB / 200 MB/s  ≈ 10,000 s ≈ 2.8 h
Replay WAL since the backup (a day of changes)               ≈ 45 min
Validate data, reconnect apps, warm caches                   ≈ 30 min
Total                                                        ≈ 4 h
```

**4 hours vs a 1-hour RTO: the plan fails.** And this is the optimistic version: nobody has practised it.

Options:
- A **hot standby replica** for hardware/zone loss (minutes), keeping backup restore only for logical damage.
- **Smaller pieces**: sharding or per-tenant databases mean you restore 50 GB, not 2 TB.
- More frequent base backups (less WAL to replay), faster storage, parallel restore.
- Or agree with the business that 4 hours is the real RTO for this scenario.

---

## 4. Recovery is also a process

- **Detect** fast: alerting on symptoms (error rate, latency), not causes.
- **Mitigate first, diagnose later**: rollback, fail over, shed load, then find the root cause.
- **Communicate**: incident channel, an incident commander, a status page.
- **Verify** after recovery: data integrity checks, reconciliation, consumer lag back to normal.
- **Learn**: blameless postmortem, action items that make the next recovery faster.

---

## 5. Test it

An untested recovery plan is a hope.
- **Restore drills**: restore backups regularly (automate it: restore nightly into a test environment and run checks).
- **Failover drills / game days**: actually fail over, time it, compare with the RTO.
- **DR tests**: bring up the service in another region from scratch.
- Measure each drill's real RTO/RPO and track them over time.

---

## 6. Anti-patterns

- RTO/RPO numbers that were never measured.
- Backups that cover the database but not configs, secrets, object storage or infrastructure.
- Recovery that depends on the thing that failed (runbooks stored in the wiki that's down; restore credentials in the compromised account).
- Rollback impossible because the schema migration wasn't backward-compatible.
- Only full restores possible, when the real need is "restore one customer" or "one table".

---

## 7. Checklist

- [ ] RTO and RPO defined per service/data store and agreed with the business
- [ ] Common recoveries automated (restart, replace, failover, rollback)
- [ ] Backups + PITR + delayed replica / versioning for logical errors
- [ ] Partial restores possible (one table, one tenant, one object)
- [ ] Runbooks stored somewhere available during an outage
- [ ] Restore and failover drills on a schedule, with measured times

---

## 8. Exercises

1. For a project you built, list 5 failure scenarios from §2. For each, write the recovery steps and estimate RTO and RPO honestly.
2. Redo the §3 worked example for your own database size. What restore speed would you need to meet a 30-minute RTO?
3. A migration corrupted 2% of rows in `orders` 6 hours ago, and new orders keep arriving. Why is "restore the whole DB to 6 hours ago" a bad idea? Design a better recovery.
4. Write a one-page runbook for "primary database is down", including how you verify the recovery worked.

---

## 9. Further reading

Google SRE book ch. 26 "Data Integrity: What You Read Is What You Wrote" and ch. 14 "Managing Incidents"; AWS whitepaper "Disaster Recovery of Workloads on AWS"; GitLab's 2017 database incident postmortem; PagerDuty Incident Response documentation (response.pagerduty.com); *The Field Guide to Understanding 'Human Error'* (Sidney Dekker).
