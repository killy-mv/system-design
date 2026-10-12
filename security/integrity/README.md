# Integrity

> **Integrity** = data and code are **changed only by authorized parties, in authorized ways**, and any other change is prevented or at least detected.
> "How do we know this order, this message or this binary is exactly what its author produced?"

Integrity is the "I" in the **CIA triad**. This page covers **deliberate tampering**. Accidental corruption (bit rot, bugs, lost updates) is covered in [Reliability §13](../../runtime-operational/reliability/README.md#13-data-integrity-and-correctness) and [Durability](../../runtime-operational/durability/README.md).

Related: [Confidentiality](../confidentiality/README.md), [Authorization](../authorization/README.md) (who may change what), [Non-repudiation](../non-repudiation/README.md) (proving *who* made a change), [Auditability](../auditability/README.md) (recording changes).

---

## 1. What needs integrity

| Thing | Tampering looks like |
|---|---|
| **Data in transit** | A request or response modified on the way; a replayed message |
| **Data at rest** | A balance, price, role or audit record edited directly in the database |
| **Input from clients** | A changed price, user ID or hidden form field; injected SQL |
| **Messages between systems** | A forged webhook; a forged or altered token |
| **Code and artifacts** | A malicious dependency, a modified build, a swapped container image |
| **Configuration and infrastructure** | A changed firewall rule, IAM policy or DNS record |

Encryption alone does **not** give integrity. An attacker can flip bits in unauthenticated ciphertext and change the plaintext in predictable ways. Use **authenticated encryption** (AES-GCM, ChaCha20-Poly1305).

---

## 2. The cryptographic tools

| Tool | Detects | Needs | Notes |
|---|---|---|---|
| **Hash** (SHA-256) | Accidental change; tampering *if the hash itself is trusted* | Nothing | An attacker who can change the data can usually change the hash next to it |
| **MAC** (HMAC-SHA-256) | Tampering by anyone without the key | A shared secret | Either side could have produced it, so no proof of *which* one |
| **Digital signature** (Ed25519, ECDSA, RSA-PSS) | Tampering by anyone without the private key | A key pair | Anyone can verify; only the holder of the private key can sign ([Non-repudiation](../non-repudiation/README.md)) |
| **AEAD** (AES-GCM) | Tampering with encrypted data | A shared key | Confidentiality and integrity together |
| **Hash chain / Merkle tree** | Removal, reordering or rewriting of history | A trusted latest hash | Git, Certificate Transparency, tamper-evident logs |

### Worked example: verifying a webhook

A payment provider calls your endpoint with `order paid`. Without verification, anyone who finds the URL can mark orders as paid.

```python
def verify(raw_body: bytes, header: str, secret: bytes, tolerance=300):
    ts, sig = parse(header)                       # e.g. "t=1718000000,v1=5257a8..."
    if abs(time.time() - int(ts)) > tolerance:    # reject old messages (replay)
        raise Invalid("stale")
    expected = hmac.new(secret, f"{ts}.".encode() + raw_body, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, sig):    # constant-time comparison
        raise Invalid("bad signature")
```

Four details that are easy to get wrong:

1. Verify the **raw bytes**, not re-serialised JSON (whitespace and key order change the hash).
2. Include a **timestamp** in the signed content and reject old messages.
3. Compare in **constant time**.
4. Still **deduplicate by event ID**, because a valid message can be delivered twice ([Reliability §11](../../runtime-operational/reliability/README.md#11-idempotency)).

---

## 3. Never trust the client

Anything that comes from a browser, app or partner can be changed by its sender.

| Attack | Example | Defence |
|---|---|---|
| **Parameter tampering** | `{"item": 42, "price": 0.01}` | Recompute prices, totals and discounts on the server from trusted data |
| **Injection** (SQL, command, template) | `'; DROP TABLE orders; --` | Parameterised queries, no string-built commands, output encoding |
| **Mass assignment** | `{"name": "x", "is_admin": true}` | Explicit allow-list of writable fields ([Authorization](../authorization/README.md)) |
| **CSRF** | Another site makes the user's browser submit a form | `SameSite` cookies, CSRF tokens, checking `Origin` |
| **Unsafe deserialisation** | A crafted object that runs code when loaded | Don't deserialise untrusted data into native objects; use plain data formats with schemas |
| **Token tampering** | Editing a JWT's claims; `alg: none` | Verify signatures with a pinned algorithm and key ([Authentication](../authentication/README.md)) |

Validate at the boundary: type, range, length, format, and **business rules** (quantity > 0, the coupon belongs to this user).

---

## 4. Integrity of stored data

- **Database constraints** as the last line: `NOT NULL`, `CHECK`, foreign keys, unique indexes. They hold even when the application has a bug.
- **Least-privilege database users**: the web app's account can't `ALTER`, `DROP` or touch tables it doesn't own.
- **Append-only records for money and history**: a ledger of immutable entries instead of an editable `balance` column. Corrections are new entries.
- **Separation of duties**: the person who requests a change isn't the one who approves it (payouts, production access, refunds above a limit).
- **Tamper evidence** for high-value records: hash chains, write-once storage ([Auditability](../auditability/README.md)).
- **No routine manual edits in production.** Changes go through code, migrations or audited admin tools.

---

## 5. Integrity of code: the supply chain

The software you run is mostly written by other people and built by machines.

| Risk | Defence |
|---|---|
| Malicious or hijacked dependency | Lockfiles with hashes, pinned versions, review of new dependencies, private registry/proxy |
| Dependency confusion / typosquatting | Scoped packages, registry rules, exact names in lockfiles |
| Tampered build or artifact | Isolated, reproducible builds; **signed artifacts** (Sigstore/cosign); **provenance** (SLSA); verify at deploy time ([Deployability §12](../../operations-devops/deployability/README.md#12-artifacts-and-supply-chain)) |
| Tampered third-party script in the browser | Subresource Integrity (`integrity="sha384-..."`), Content Security Policy |
| Unreviewed change to the main branch | Branch protection, required reviews, signed commits, no direct pushes |
| Mutable tags | Deploy images by digest (`@sha256:...`), not by `latest` |

The SolarWinds (2020) and `xz` backdoor (2024) incidents were both attacks on the build and release chain, not on the running product.

---

## 6. Anti-patterns

- Trusting prices, totals, roles or user IDs sent by the client.
- Building SQL or shell commands by concatenating strings.
- Storing a checksum next to the data and calling it tamper protection.
- Webhook endpoints that accept unsigned requests, or verify re-serialised JSON.
- Comparing signatures or tokens with `==`.
- Encryption without authentication (CBC without a MAC, home-made schemes).
- Deploying `latest`; installing dependencies without a lockfile.
- Shared database superuser for every service and every engineer.
- "Quick fixes" to production data with ad-hoc SQL.

---

## 7. Checklist

- [ ] All client input validated on the server; prices and permissions never taken from the client
- [ ] Parameterised queries everywhere; CSRF protection on state-changing requests
- [ ] Incoming webhooks and messages verified (signature, timestamp, deduplication)
- [ ] Authenticated encryption only
- [ ] Database constraints and least-privilege database accounts
- [ ] Money and other critical history stored as append-only records
- [ ] Lockfiles, dependency scanning, signed artifacts verified at deploy
- [ ] Protected branches with required review; production changes only through audited paths

---

## 8. Exercises

1. In an app you built, list every value the server accepts from the client and then trusts. Which ones could a user change to their advantage?
2. A partner sends you files by SFTP once a day. Design how you verify each file is complete, unmodified and really from them.
3. Explain why HMAC is enough for a payment provider's webhook to you, but not enough for a document that both of you may need to show to a court.
4. Trace one of your container images from commit to production. At which steps could someone swap it, and what would detect that?

---

## 9. Further reading

OWASP Top 10 (Injection; Software and Data Integrity Failures) and OWASP ASVS; OWASP Cheat Sheets (Input Validation, SQL Injection Prevention, CSRF Prevention); SLSA framework (slsa.dev) and Sigstore documentation; Stripe and GitHub docs on verifying webhook signatures; *Designing Data-Intensive Applications* (Martin Kleppmann), ch. 12 on integrity and end-to-end checks; RFC 2104 (HMAC) and RFC 8032 (Ed25519).
