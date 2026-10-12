# Privacy: hands-on practice notes

Planned labs for learning privacy by **finding every copy of one person's data, then building the machinery to export, expire and erase it**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) with **Docker Compose**, **PostgreSQL**, **Redis**, a search index (**OpenSearch** or **Meilisearch**), a small analytics store (**ClickHouse** or **DuckDB**), and a nightly `pg_dump` backup. Seed it with fake customers (Faker) and some activity.

Never use real personal data for these labs.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** leak hunt and masked data for staging (confidentiality labs 3 and 5), customer-facing audit log (auditability lab 5), each module owns its data (modularity lab 5).

---

## Labs

### 1. Map the data ⭐
- **Do:** pick one seeded customer. Find every place their email, name, address and IP appear: tables, cache, search, analytics, logs, backups, outgoing calls to "vendors" (fake email and support services).
- **Write:** a data inventory table: field, location, purpose, retention, deletion method. Tag personal columns in the schema (comments or a small YAML catalogue).
- **See:** README §3 worked example.

### 2. Minimise and pseudonymise
- **Fix:** remove fields the signup form doesn't need. Replace `user_email` in analytics events and logs with a pseudonymous ID; keep the mapping in one table.
- **Check:** analytics queries still work. `grep` logs and the analytics store for any seeded email address.
- **Then:** store IP addresses truncated, and birth dates as a year. What breaks?

### 3. Export and erase ⭐
- **Build:** `GET /me/export` returning everything held about the user as JSON, and an erasure job driven by the lab 1 inventory: delete or anonymise rows, purge cache and search, call vendor deletion APIs, keep invoices with a recorded reason.
- **Check:** after erasure, repeat lab 1's search and list what's left and why. Restore last night's backup into a scratch database: the user is back. Add a "deletions to re-apply after restore" list.
- **See:** README §5.

### 4. Retention and crypto-shredding
- **Build:** retention jobs: logs 30 days, abandoned carts 90 days, inactive accounts notified then removed. Use time-partitioned tables and drop partitions.
- **Then:** encrypt each customer's personal fields in the order event history with a per-customer key. Delete one key and confirm the events remain but that customer's fields are unreadable, in backups too.
- **See:** README §4.

### 5. How anonymous is "anonymised"?
- **Do:** export a dataset of orders with names and emails removed but postcode, birth year and gender kept. Count how many rows are unique on those three columns.
- **Fix:** generalise values until every combination has at least *k* = 5 rows. Add a minimum-group-size rule to the admin reports.
- **Optional:** add Laplace noise to a published count and see how the answer varies between runs (differential privacy in miniature).
- **See:** README §4.

---

## Progress

- [ ] 1. Map the data
- [ ] 2. Minimise and pseudonymise
- [ ] 3. Export and erase
- [ ] 4. Retention and crypto-shredding
- [ ] 5. How anonymous is "anonymised"?
