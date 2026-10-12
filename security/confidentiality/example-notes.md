# Confidentiality: hands-on practice notes

Planned labs for learning confidentiality by **stealing your own data, then making the theft worthless**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) with **Docker Compose**, **PostgreSQL**, **HashiCorp Vault** (dev mode) or **LocalStack KMS**, **mkcert**, **Wireshark/tcpdump** and **gitleaks**.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** secrets done right (configurability lab 3), structured logs (observability lab 3), webhook signing (integrity lab 1).

---

## Labs

### 1. Sniff it, then encrypt it
- **Break:** run messy-shop and Postgres over plain connections. Capture traffic with `tcpdump` and find a password and a customer record in the capture.
- **Fix:** TLS between client and app (mkcert), `sslmode=verify-full` between app and Postgres. Capture again.
- **Then:** set `sslmode=require` and point the app at a fake Postgres with a self-signed certificate. Does it connect? Why is `verify-full` different?
- **See:** README §2.

### 2. Field-level envelope encryption ⭐
- **Build:** add a `national_id` column to customers. Encrypt it with a per-record data key wrapped by a key in Vault Transit or LocalStack KMS, as in the README §3 worked example.
- **Check:** `pg_dump` the database and search the dump for a known ID. Copy one customer's ciphertext into another row and confirm decryption fails (AAD).
- **Then:** rotate the key-encryption key and confirm old rows still decrypt.

### 3. Leak hunt ⭐
- **Do:** place a fake "canary" card number and token in a request. Then search for them in app logs, traces, error-tracker payloads, the Nginx access log, browser history and a database backup.
- **Fix:** each leak you find: log allow-lists, collector redaction, tokens moved out of URLs, `Cache-Control: no-store`.
- **See:** README §4.

### 4. Secrets out of Git
- **Break:** commit a fake API key, then "remove" it in a later commit. Run `gitleaks` on the history.
- **Fix:** add a pre-commit hook and a CI check; move the app's DB password to Vault with a short-lived dynamic credential.
- **Check:** the app still starts after the credential expires and is reissued.

### 5. Masked data for staging
- **Build:** a script that copies production-like data to a staging database with names, emails, addresses and IDs replaced by realistic fake values (Faker, or `postgresql-anonymizer`), keeping foreign keys valid.
- **Check:** reports and tests still work on the masked copy; no real email address remains.
- **See:** README §5.

---

## Progress

- [ ] 1. Sniff it, then encrypt it
- [ ] 2. Field-level envelope encryption
- [ ] 3. Leak hunt
- [ ] 4. Secrets out of Git
- [ ] 5. Masked data for staging
