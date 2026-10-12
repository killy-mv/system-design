# Privacy

> **Privacy** = personal data is collected and used **only for purposes people would expect and have a legal basis for**, kept no longer than needed, and stays under the control of the person it describes.
> "Do we need this data at all, do people know we have it, and can we find and delete all of it when they ask?"

Privacy is **not** the same as [confidentiality](../confidentiality/README.md). Confidentiality keeps outsiders away from data. Privacy also limits what *you*, the legitimate holder, may do with it. A perfectly encrypted database of data you should never have collected is a privacy failure.

Related: [Confidentiality](../confidentiality/README.md) (the technical protections), [Compliance](../compliance/README.md) (GDPR and other laws as audited obligations), [Authorization](../authorization/README.md) (who inside may see personal data), [Auditability](../auditability/README.md) (recording access).

---

## 1. Vocabulary

| Term | Meaning |
|---|---|
| **Personal data** (PII) | Any information relating to an identifiable person: name, email, IP address, device ID, location, order history |
| **Special categories / sensitive data** | Health, biometrics, religion, political views, sexual orientation, and similar. Stricter rules |
| **Data subject** | The person the data is about |
| **Controller** | Decides why and how data is processed (usually your company) |
| **Processor** | Processes data on the controller's behalf (your cloud and SaaS vendors, or you, for your B2B customers) |
| **Pseudonymisation** | Replacing identifiers with tokens; still personal data, because it can be linked back |
| **Anonymisation** | Irreversibly unlinkable to a person; no longer personal data. Much harder than it looks |

---

## 2. Principles

The **GDPR** (Article 5) principles are a good design checklist anywhere in the world:

| Principle | Design question |
|---|---|
| **Lawfulness, fairness, transparency** | Which legal basis covers this processing (consent, contract, legal obligation, legitimate interests, ...)? Is it explained in plain language? |
| **Purpose limitation** | Was this data collected for *this* use? A phone number given for 2FA must not be used for marketing |
| **Data minimisation** | Do we need this field at all? At this precision? |
| **Accuracy** | Can people see and correct their data? |
| **Storage limitation** | When is each kind of data deleted, and what does the deleting? |
| **Integrity and confidentiality** | Is it protected ([Confidentiality](../confidentiality/README.md))? |
| **Accountability** | Can we demonstrate all of the above? |

**Privacy by design and by default** (GDPR Article 25): the protective option is built in and switched on without the user having to ask.

---

## 3. Know your data

You can't protect, export or delete data you can't find.

- **Data inventory / map**: for each kind of personal data, record where it's collected, why, the legal basis, where it's stored (every system), who it's shared with, and the retention period. GDPR calls this a record of processing activities.
- **Tag personal data in schemas** (column annotations, a data catalogue) so tooling can find it.
- **Follow the copies**: replicas, caches, search indexes, analytics warehouses, logs, backups, exports, third-party tools, ML training sets.

### Worked example: where one email address lives

A user signs up for messy-shop. Six months later, their email address is in:

| Location | Why it's there | Deletion plan |
|---|---|---|
| `users` table | Account | Delete or anonymise the row |
| `orders` table (copied at checkout) | Invoices | Keep for the legal invoice-retention period, then anonymise |
| Search index | Admin customer search | Reindex on delete |
| Redis session and cache entries | Performance | TTL; explicit purge |
| Application logs | Someone logged the request body | Stop logging it; logs expire in 30 days |
| Analytics warehouse | Event tracking with `user_email` | Should have been a pseudonymous ID from the start |
| Email provider, support desk, error tracker | Sending mail, tickets, error context | Call each vendor's deletion API |
| Database backups | Disaster recovery | Expire within the backup retention window; re-apply deletions after any restore |
| A CSV on an analyst's laptop | "Quick export" | No plan. This is the real problem |

Nine places, and only the first is obvious. An erasure request must reach all of them, which is why it has to be **automated and driven by the inventory**.

---

## 4. Engineering techniques

| Goal | Techniques |
|---|---|
| **Collect less** | Optional fields, coarse values (age range instead of birth date, city instead of GPS), processing on the device |
| **Separate identity from behaviour** | Pseudonymous IDs in analytics and logs; the mapping kept in one protected place |
| **Limit internal access** | Field-level access control, masking by role, purpose-based access, audit of reads ([Authorization](../authorization/README.md)) |
| **Limit lifetime** | TTLs, scheduled deletion jobs, partition-by-time tables that are dropped, short log retention |
| **Make deletion reach everywhere** | A deletion pipeline fed by the inventory; **crypto-shredding** (per-user encryption key, delete the key) for backups and event logs |
| **Honour consent** | Store consent per purpose with timestamp and version; check it at the point of use; withdrawal as easy as giving |
| **Safer analytics** | Aggregation, k-anonymity thresholds (don't show groups smaller than *k*), differential privacy for published statistics |
| **Safer non-production data** | Synthetic or masked data in staging and development |
| **Control sharing** | Data-processing agreements, vendor inventory, scrub payloads before they go to third parties (including LLM APIs) |

**Anonymisation is fragile.** Removing names is not enough: combinations of ordinary attributes identify people. AOL's 2006 release of "anonymised" search logs and the Netflix Prize dataset were both re-identified by researchers and journalists. Treat "anonymised" row-level data as personal data unless proven otherwise.

---

## 5. People's rights as system features

Under GDPR, requests must normally be answered within **one month**. Similar rights exist under California's CCPA/CPRA, Brazil's LGPD and others.

| Right | What the system needs |
|---|---|
| **Access** | Export of all data held about a person, in readable form |
| **Portability** | The same in a machine-readable format (JSON, CSV) |
| **Rectification** | Edit screens or a support process that reaches every copy |
| **Erasure** | A deletion pipeline; a list of what must legally be kept (invoices, fraud records) and why |
| **Restriction / objection** | Flags that stop specific processing (marketing, profiling) |
| **Withdraw consent** | Per-purpose consent records checked at use |
| **Not to be subject to solely automated decisions** with significant effects | Human review path, explanation of the logic |

Verify the requester's identity first: an access request is also a way to steal someone's data.

---

## 6. Anti-patterns

- Collecting everything "in case it's useful later".
- No inventory: nobody can say where a user's data is.
- Personal data in logs, URLs, analytics events and error reports.
- Using data for a new purpose without asking (security phone numbers used for ads).
- "Delete account" that only sets `deleted = true`, forever.
- No retention limits; backups and warehouses that keep everything.
- Calling pseudonymised or lightly scrubbed data "anonymous".
- Dark patterns: pre-ticked boxes, consent walls, "reject" hidden three screens deep.
- Production data copied to staging, laptops and spreadsheets.
- Sending full user records to third parties and LLM APIs without review.

---

## 7. Checklist

- [ ] Data inventory: what, why, legal basis, where, shared with, retention
- [ ] Each field justified; sensitive categories identified
- [ ] Analytics and logs use pseudonymous IDs; no personal data in URLs
- [ ] Retention periods enforced by automation, including backups and warehouses
- [ ] Access, export and erasure automated across all stores and vendors, with identity verification
- [ ] Consent stored per purpose and checked at use; withdrawal is easy
- [ ] Internal access to personal data restricted, masked and audited
- [ ] Privacy review (DPIA where required) for new features that use personal data in new ways
- [ ] Vendors listed, with data-processing agreements and known data locations
- [ ] Breach response plan that includes notification duties ([Compliance](../compliance/README.md))

---

## 8. Exercises

1. Draw the data map for one field (email, phone or location) in a system you built, like the worked example in §3. How many copies did you find?
2. Review a signup form you know. Which fields could be removed, made optional or made coarser without hurting the product?
3. Design account deletion for a shop that must keep invoices for a legally required period and has event-sourced order history. What's deleted, what's anonymised, what stays, and how do backups fit in?
4. A product manager wants to use support-chat transcripts to train a model. List the privacy questions to answer first and the technical measures you'd propose.

---

## 9. Further reading

GDPR text (especially Articles 5, 6, 15–22, 25, 32–35) and the European Data Protection Board guidelines; UK ICO guidance (clear, practical explanations); NIST Privacy Framework; ISO/IEC 27701; Ann Cavoukian, "Privacy by Design: The 7 Foundational Principles"; Narayanan and Shmatikov, "Robust De-anonymization of Large Sparse Datasets" (2008); Latanya Sweeney, "k-Anonymity: A Model for Protecting Privacy" (2002); Dwork and Roth, *The Algorithmic Foundations of Differential Privacy*; *Designing Data-Intensive Applications* (Martin Kleppmann), final chapter; OWASP Top 10 Privacy Risks.
