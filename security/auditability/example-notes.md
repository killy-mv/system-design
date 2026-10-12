# Auditability: hands-on practice notes

Planned labs for learning auditability by **recording who did what, trying to erase the evidence, and running an investigation from the log alone**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) with **Docker Compose**, **PostgreSQL**, **MinIO** (S3-compatible, supports Object Lock) and **Grafana + Loki** or **OpenSearch** for searching events.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** entity timeline and safe support actions (supportability labs 2–3), structured logs (observability lab 3), domain events inside the monolith (modularity lab 6), signed approvals (non-repudiation lab 1).

---

## Labs

### 1. Audit events with a real schema ⭐
- **Build:** an `audit_events` table and a small `audit.record(actor, action, target, outcome, reason, changes)` helper using the README §2 schema. Instrument logins, role changes, refunds, price changes and customer data edits.
- **Check:** the audit row is written in the same transaction as the change. Kill the app between the two writes in a non-transactional version and see the gap.
- **Check:** failed and denied attempts appear too; impersonated actions show the real actor.

### 2. Tamper-evident log ⭐
- **Build:** add `prev_hash` and `hash` columns as a hash chain, revoke `UPDATE`/`DELETE` from the app role, and write a verifier that walks the chain.
- **Break:** as a database superuser, edit one old event, then delete another. Run the verifier.
- **Then:** rewrite the whole chain after your edit so it verifies again. Stop that by storing the chain head every minute in a separate place.
- **See:** README §3 worked example.

### 3. Ship it out of reach
- **Build:** export audit events in batches to a MinIO bucket with Object Lock (compliance mode, 30-day retention), using credentials that can write but not delete.
- **Break:** try to delete or overwrite an object as the app, then as the bucket admin.
- **Check:** restore the full audit history from the bucket after dropping the table.

### 4. Investigate an incident
- **Do:** have a friend (or a script with randomised steps) act as a rogue support agent: look up several customers, change one payout account, issue a refund, then try to cover it up.
- **Investigate:** using only the audit log, reconstruct the timeline, the accounts affected and the first suspicious action. Time yourself.
- **Then:** write two alert rules that would have fired during the incident.
- **See:** README §4.

### 5. Customer-facing audit log
- **Build:** an "Activity" page and export endpoint that show a tenant its own events, with filters for actor, action and date.
- **Check:** with two tenants, confirm neither can see the other's events; confirm masked values stay masked in the export.
- **Then:** handle an erasure request: remove the user's profile but keep pseudonymous audit events (README §5).

---

## Progress

- [ ] 1. Audit events with a real schema
- [ ] 2. Tamper-evident log
- [ ] 3. Ship it out of reach
- [ ] 4. Investigate an incident
- [ ] 5. Customer-facing audit log
