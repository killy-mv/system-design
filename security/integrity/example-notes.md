# Integrity: hands-on practice notes

Planned labs for learning integrity by **tampering with requests, data and artifacts, then making each tampering attempt fail or show up**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) with **Docker Compose**, **PostgreSQL**, **curl** or **Burp Suite Community / OWASP ZAP**, **sqlmap**, and **cosign**.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** outgoing webhooks (extensibility lab 4), property-based testing of a ledger (reliability lab 13), tamper-evident audit log (auditability lab 2).

---

## Labs

### 1. Forge a webhook, then verify it ⭐
- **Break:** messy-shop's `/webhooks/payment` marks orders as paid. Call it with `curl` and get a free order.
- **Fix:** HMAC verification over the raw body with a timestamp, constant-time comparison and event-ID deduplication (README §2 worked example).
- **Check:** a modified body, a replayed old request and a duplicate delivery are each handled correctly. Then break your own fix by verifying parsed-and-re-serialised JSON and see when it fails.

### 2. Tamper with the cart ⭐
- **Break:** intercept the checkout request and change the price, a negative quantity, another user's coupon and an `is_admin` field on the profile update.
- **Fix:** server-side recalculation, validation with a schema, and an allow-list of writable fields.
- **See:** README §3.

### 3. SQL injection
- **Break:** find the string-built query in messy-shop's product search. Exploit it by hand, then with `sqlmap`.
- **Fix:** parameterised queries; a database user for the app that can't read tables it doesn't need.
- **Check:** `sqlmap` finds nothing; the injection that worked now returns no extra rows even if you reintroduce the bug in a table the app user can't read.

### 4. A ledger you can't quietly edit
- **Build:** replace the `loyalty_points` balance column with an append-only entries table. Add a `CHECK` constraint, revoke `UPDATE`/`DELETE` from the app role, and compute the balance from entries.
- **Check:** try to edit history as the app user. Write a reconciliation query that proves the balance equals the sum of entries.
- **See:** README §4.

### 5. Sign and verify an image
- **Build:** in CI, build the messy-shop image, sign it with `cosign` (keyless or a local key pair), and deploy by digest.
- **Break:** push a different image under the same tag.
- **Check:** a verification step (`cosign verify`, or an admission policy in a local Kubernetes cluster) rejects the unsigned image.
- **See:** README §5.

---

## Progress

- [ ] 1. Forge a webhook, then verify it
- [ ] 2. Tamper with the cart
- [ ] 3. SQL injection
- [ ] 4. A ledger you can't quietly edit
- [ ] 5. Sign and verify an image
