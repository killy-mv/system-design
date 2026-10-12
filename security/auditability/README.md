# Auditability

> **Auditability** (or **accountability**) = every security-relevant action can be **traced to who did it, when, from where and with what result**, using records that can be trusted.
> "Someone changed this customer's bank details last Tuesday. Who, how, and can we prove the record wasn't edited afterwards?"

In **ISO/IEC 25010**, this is *accountability*: the degree to which the actions of an entity can be traced uniquely to that entity.

Related: [Non-repudiation](../non-repudiation/README.md) (proof strong enough that the actor can't deny it), [Authentication](../authentication/README.md) (without a verified identity there's nobody to attribute to), [Authorization](../authorization/README.md), [Compliance](../compliance/README.md) (most standards require audit logs), [Observability](../../operations-devops/observability/README.md) (a different kind of log), [Supportability](../../operations-devops/supportability/README.md) (entity timelines).

---

## 1. Audit logs vs application logs

| | **Application / debug logs** | **Audit logs** |
|---|---|---|
| Purpose | Find out why the system misbehaved | Find out who did what |
| Reader | Engineers | Security, auditors, customers, sometimes courts |
| Content | Technical detail, free-form | Fixed schema: actor, action, target, outcome |
| Loss | Sampling and dropping are acceptable | **Must not be lost**: no sampling |
| Change | Format changes freely | Treated as a stable record |
| Retention | Days to weeks | Months to years, set by policy or law |
| Access | Broad engineering access | Restricted, and access is itself logged |

Keep them in **separate streams**. An audit event buried in debug output gets sampled, rotated away or drowned.

---

## 2. What to record

| Category | Events |
|---|---|
| **Authentication** | Login success and failure, logout, MFA enrolled/removed, password reset, new device |
| **Authorization** | Denied requests, role and permission changes, sharing changes |
| **Data access** | Reads of sensitive records (health, financial, personal), exports, bulk downloads |
| **Data changes** | Create, update, delete of important entities, with before and after values |
| **Administration** | Impersonation, support actions, configuration and feature-flag changes, API key creation |
| **Infrastructure** | Cloud control-plane calls (AWS CloudTrail and equivalents), deploys, production shell access, database queries by humans |
| **The audit system itself** | Log access, export, retention changes, logging disabled |

Each event answers **who, what, on what, when, where, how, and with what result**:

```json
{
  "id": "01J8ZQ4K7N3V5X2M9T",
  "time": "2026-03-10T14:02:11.482Z",
  "actor": { "type": "user", "id": "u_812", "role": "support_agent",
             "on_behalf_of": null, "ip": "203.0.113.7", "session": "s_55f1" },
  "action": "customer.bank_details.update",
  "target": { "type": "customer", "id": "c_3391", "tenant": "t_acme" },
  "outcome": "success",
  "reason": "ticket #48213",
  "changes": { "iban": { "old": "…4471", "new": "…9902" } },
  "request_id": "4bf92f35",
  "prev_hash": "9c1e…", "hash": "a37f…"
}
```

Points to notice: a stable **action name**, the **real actor even under impersonation** (`on_behalf_of`), **masked** sensitive values, a **reason**, and a link to the trace (`request_id`).

**Never put secrets in the audit log**: no passwords, tokens, full card numbers or full personal identifiers. The log is long-lived and widely queried, and it becomes personal data in its own right ([Privacy](../privacy/README.md)).

---

## 3. Making the log trustworthy

An audit log an attacker (or an administrator) can edit proves nothing.

| Property | How |
|---|---|
| **Complete** | Write the audit event in the **same transaction** as the change (or via a transactional outbox), so one can't exist without the other. Fail the action if a required audit write fails |
| **Append-only** | The application role has `INSERT` only; no `UPDATE` or `DELETE` |
| **Out of reach** | Ship quickly to a separate system or account that application administrators can't modify |
| **Immutable storage** | Write-once storage (S3 Object Lock in compliance mode, WORM) with a retention period |
| **Tamper-evident** | Hash chains or Merkle trees; publish or store the latest hash elsewhere |
| **Accurate time** | Synchronised clocks (NTP), UTC timestamps, a server-assigned sequence |
| **Attributable** | Individual accounts only; no shared logins; service actions carry the originating user |

### Worked example: a hash chain

```
hash(n) = SHA-256( hash(n−1) || canonical_json(event n) )

#1  login            prev=0000…  hash=5a1c…
#2  bank.update      prev=5a1c…  hash=9c1e…
#3  payout.create    prev=9c1e…  hash=a37f…   ← publish/store this "head" externally every N minutes
```

If someone edits or deletes event #2, every later hash stops matching. A verifier that recomputes the chain finds the first broken link. The chain only detects tampering by someone who can't also rewrite the externally stored head, so anchor it somewhere the database administrator has no access to. For proof that holds against the operator too, events need **signatures** ([Non-repudiation](../non-repudiation/README.md)).

---

## 4. Using the log

A log nobody reads only helps after the damage is done.

- **Detection**: alerts for things like impossible travel, many failed logins, mass exports, a privilege grant followed by unusual access, audit logging turned off. Feed events to a SIEM.
- **Investigation**: search by actor, target, time and action. Practise it: can you list everything one account did in a week within minutes?
- **Customer-facing audit logs**: enterprise customers expect to see and export their own tenant's events (and often to stream them to their SIEM).
- **User-facing history**: "recent security activity" and "active sessions" pages let users spot misuse themselves.
- **Reviews**: periodic access reviews and sampling of privileged actions, with evidence kept for auditors ([Compliance](../compliance/README.md)).

---

## 5. Retention and privacy

- **Retention is a policy decision** driven by law, contracts and investigation needs. PCI DSS, for example, requires at least 12 months of audit log history with the most recent 3 months immediately available.
- **Minimise** what each event stores: IDs and masked values rather than full records.
- **Erasure requests vs immutable logs**: keep identifiers pseudonymous (user ID, not name and email) so the mapping can be deleted while the log stays intact, and document the legal basis for what you keep.
- **Access to audit logs is sensitive** and should itself be audited.

---

## 6. Anti-patterns

- Audit events mixed into debug logs, sampled or rotated after a week.
- Shared accounts (`admin`, `root`, `deploy`), so every action is anonymous.
- The audit table in the application database with full write access for the application and its administrators.
- Logging "record updated" with no actor, no target ID or no before/after values.
- Recording only successes; failures and denials are where attacks show.
- Impersonation that records the customer as the actor.
- Passwords, tokens or full card numbers in audit events.
- Best-effort audit writes that are silently dropped when the queue is full.
- Logs that are collected but never searched, alerted on or tested.

---

## 7. Checklist

- [ ] List of auditable events agreed and implemented, including denials and administrative actions
- [ ] Fixed schema: actor, action, target, time, origin, outcome, reason, request ID
- [ ] Audit writes are transactional with the change; no sampling
- [ ] Append-only, shipped to separate immutable storage; tamper evidence verified regularly
- [ ] Individual accounts only; impersonation records the real actor
- [ ] No secrets; sensitive values masked
- [ ] Retention period defined; access to logs restricted and logged
- [ ] Alerts on key events; investigations rehearsed
- [ ] Customers and users can see the activity relevant to them

---

## 8. Exercises

1. For an app you built, list the 15 events that most deserve an audit record. For each: who is the actor, what is the target, what would you need to know in an investigation?
2. An administrator with database access wants to hide one action. Walk through how they'd attempt it against your design and what would expose them.
3. A user asks for all their data to be erased. Your audit log has 3 years of their events. What do you delete, what do you keep, and why?
4. Design the customer-facing audit log for a multi-tenant SaaS: which events, which fields, how customers filter and export, and how you keep one tenant's events away from another.

---

## 9. Further reading

OWASP Logging Cheat Sheet and OWASP Top 10 ("Security Logging and Monitoring Failures"); NIST SP 800-92 (*Guide to Computer Security Log Management*); PCI DSS Requirement 10; AWS CloudTrail documentation (log file integrity validation is a real-world hash chain); Certificate Transparency (RFC 6962) and Russ Cox, "Transparent Logs for Skeptical Clients"; Crosby and Wallach, "Efficient Data Structures for Tamper-Evident Logging" (2009); the CloudEvents and OCSF schemas for event formats.
