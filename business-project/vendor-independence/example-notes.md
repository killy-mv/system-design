# Vendor independence: hands-on practice notes

Planned labs for learning vendor independence by **measuring the cost of leaving, then actually swapping a vendor**. Theory is in [README.md](README.md).

They need **Docker Compose**, a **spreadsheet**, and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)). Local stand-ins replace real vendors: **MinIO** (S3-compatible storage), **Mailpit** (SMTP), **PostgreSQL**, and the fake payment provider messy-shop already has.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** branch by abstraction (evolvability lab 4), shotgun surgery → registry (modifiability lab 1), build vs buy TCO (cost-efficiency lab 5). The portability and replaceability notes cover the techniques in more depth.

---

## Labs

### 1. Lock-in inventory ⭐
- **Do:** list every external dependency of a system you know (or of messy-shop as you'd deploy it): cloud services, database, SaaS tools, auth, email, payments, CI, monitoring, key libraries.
- **Build:** a table with the lock-in type, what data lives there, a named alternative, an exit estimate in engineer-weeks, and the probability you'd need to leave in 5 years.
- **Measure:** exposure = probability × cost for each. Rank them. Is the top item the one you expected?
- **See:** README §1, §3.

### 2. Swap the object store ⭐
- **Build:** messy-shop stores product images and invoices in MinIO through the S3 API.
- **Break:** search the code for places that use the storage SDK directly. How many files would change to use a different, non-S3 store (local disk, or another provider's SDK)?
- **Fix:** put storage behind a small interface in your own terms (`save_invoice`, `get_product_image`). Add a second adapter (filesystem). Switch with one config value, and run the same contract tests against both.
- **See:** README §4.

### 3. Second source for a critical path
- **Build:** a second fake payment provider with a different API shape and different error codes. One internal `PaymentGateway` interface, two adapters, a routing rule (90/10 split) and failover when one provider is down.
- **Check:** kill provider A during a k6 run: do payments continue? Write down what can't be made portable (stored card tokens, refunds for a payment made on the other provider).
- **See:** README §4.

### 4. Data export drill
- **Do:** export everything a "vendor" holds (the Postgres database and the MinIO bucket) to open formats, then restore it into a fresh, different setup (another Postgres version or another bucket) and run the app against it.
- **Measure:** time taken, data size, and what was lost or needed manual fixing (sequences, permissions, extensions, object metadata).
- **Then:** estimate the same for 100× the data, including an illustrative per-GB egress price.
- **See:** README §4.

### 5. Write the exit plan
- **No new code.** For the top item from lab 1, write a one-page exit plan: triggers, the alternative, migration steps (use a pattern from [Evolvability §7](../../development-evolution/evolvability/README.md#7-migration-patterns)), estimate, and the cheapest thing you could do now to halve that estimate.
- **Check:** does the plan depend on something only the current vendor provides, such as its export tool or its support?
- **See:** README §3, §4.

---

## Progress

- [ ] 1. Lock-in inventory
- [ ] 2. Swap the object store
- [ ] 3. Second source for a critical path
- [ ] 4. Data export drill
- [ ] 5. Write the exit plan
