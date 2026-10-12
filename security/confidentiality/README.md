# Confidentiality

> **Confidentiality** = data can be read **only by the people and systems that are supposed to read it**, wherever it is: on the wire, on disk, in backups, in logs, in memory.
> "If someone steals a disk, sniffs the network or reads our logs, what do they actually learn?"

Confidentiality is the "C" in the **CIA triad** (confidentiality, [integrity](../integrity/README.md), availability).

Related: [Authentication](../authentication/README.md) and [Authorization](../authorization/README.md) (deciding *who* may read), [Privacy](../privacy/README.md) (confidentiality of *personal* data plus rules about its use), [Compliance](../compliance/README.md), [Configurability](../../operations-devops/configurability/README.md) (where secrets live).

---

## 1. Classify the data first

You can't protect everything equally, and you don't need to. Label data, then attach rules to the labels.

| Class | Examples | Typical rules |
|---|---|---|
| **Public** | Marketing pages, open API docs | None |
| **Internal** | Wiki pages, non-sensitive metrics | Staff only |
| **Confidential** | Customer records, contracts, source code | Access on need, encrypted, access logged |
| **Restricted** | Passwords, keys, card numbers, health data, government IDs | Field-level encryption or tokenisation, tight access, never in logs |

Then ask the cheapest question of all: **do we need to store this at all?** Data you don't collect can't leak ([Privacy](../privacy/README.md)).

---

## 2. The three states of data

| State | Threat | Main defences |
|---|---|---|
| **In transit** | Eavesdropping, man-in-the-middle | TLS 1.2+ (prefer 1.3) everywhere, including *inside* the network; mTLS between services; HSTS; certificate validation never disabled |
| **At rest** | Stolen disk, snapshot, backup or database dump | Disk/volume encryption, database encryption (TDE), **application-level (field) encryption** for the most sensitive fields, encrypted backups |
| **In use** | Memory dumps, a compromised host, a curious operator | Least privilege, short-lived plaintext, confidential computing (AMD SEV-SNP, Intel TDX, enclaves) where justified |

**Disk encryption is necessary but weak on its own.** It protects against a stolen disk. It does nothing against SQL injection, a leaked database password or an over-privileged employee, because to all of them the data arrives already decrypted. For restricted data, encrypt in the application so that the database only ever sees ciphertext.

---

## 3. Keys and secrets

Encryption turns a data-protection problem into a **key-management** problem.

- **Use a KMS or HSM** (AWS KMS, Google Cloud KMS, Azure Key Vault, HashiCorp Vault). Keys never leave it; you send it things to encrypt or decrypt, and every use is logged.
- **Envelope encryption**: encrypt data with a per-record or per-tenant **data key (DEK)**; encrypt the DEK with a **key-encryption key (KEK)** held in the KMS. Rotating the KEK only means re-wrapping DEKs.
- **Rotate** keys on a schedule and immediately after any suspected exposure. Design the ciphertext format with a key ID so old data stays readable.
- **Secrets** (DB passwords, API keys, signing keys): in a secrets manager, injected at runtime, never in Git, images or CI logs. Prefer **short-lived, dynamically issued** credentials over long-lived ones.
- **Use standard algorithms and libraries**: AES-256-GCM or ChaCha20-Poly1305 (authenticated encryption), libsodium / Tink / your platform's crypto library. Never invent your own scheme.

### Worked example: envelope encryption of one field

```python
# Write path
dek_plain, dek_wrapped = kms.generate_data_key(key_id="alias/customers")  # KEK stays in KMS
nonce = os.urandom(12)
ciphertext = AESGCM(dek_plain).encrypt(nonce, national_id.encode(), aad=customer_id.encode())
db.save(customer_id, national_id_enc=ciphertext, nonce=nonce, dek=dek_wrapped, key_ver="v3")
del dek_plain                                    # keep plaintext keys in memory briefly

# Read path (only the service role allowed to call kms.decrypt on this key can do this)
dek_plain = kms.decrypt(row.dek)
national_id = AESGCM(dek_plain).decrypt(row.nonce, row.national_id_enc, aad=customer_id.encode())
```

What this buys you:

| Attacker gets… | Without field encryption | With it |
|---|---|---|
| A database dump or backup | Every national ID | Ciphertext and wrapped keys only |
| Read access via SQL injection | Every national ID | Ciphertext |
| The app's KMS permission *and* the database | Everything | Everything, but each decrypt call is logged and rate-limitable |

The `aad` (additional authenticated data) binds the ciphertext to its customer, so an attacker can't copy one customer's encrypted value into another row.

---

## 4. Where confidential data leaks

Most leaks don't break encryption. The data simply ends up somewhere unprotected.

| Leak | Example | Fix |
|---|---|---|
| **Logs and traces** | Request bodies, tokens, `Authorization` headers in logs | Structured logging with allow-lists; redaction in the collector ([Observability §4](../../operations-devops/observability/README.md#4-logs)) |
| **URLs** | Tokens or emails in query strings (stored in history, proxies, `Referer`) | Put secrets in headers or bodies |
| **Error messages** | Stack traces, SQL, internal hostnames returned to clients | Generic message + reference ID ([Supportability](../../operations-devops/supportability/README.md)) |
| **Backups, snapshots, exports** | Unencrypted dump in a public bucket | Encrypt, restrict, expire; block public access at account level |
| **Non-production environments** | Production data copied to staging or laptops | Synthetic or masked data |
| **Caches and CDNs** | A personalised page cached and served to another user | `Cache-Control: private` / `no-store`; cache keys that include the user |
| **Third parties** | Analytics, error trackers, LLM APIs receiving full payloads | Scrub before sending; contracts and data-processing agreements |
| **Source control** | Keys committed to Git | Secret scanning in pre-commit and CI; rotate anything that was ever pushed |
| **Side channels** | Response time or size reveals whether a record exists | Constant-time comparison for secrets; uniform responses |

---

## 5. Reducing what can leak

- **Tokenisation**: replace a value (card number) with a random token; the real value lives in a small, heavily protected vault. Most systems then never touch the real data.
- **Masking**: show `**** **** **** 4242` or `m***@example.com` to people who don't need the whole value.
- **Hashing** for values you only need to compare (with a keyed hash/HMAC if the input space is small, such as phone numbers).
- **End-to-end encryption**: the server only relays ciphertext (Signal, WhatsApp). The strongest option, but the server can't search, moderate or recover the data.
- **Retention limits**: delete what you no longer need. Deleting a per-user key ("crypto-shredding") makes that user's data unreadable everywhere, including backups.

---

## 6. Anti-patterns

- "It's encrypted at rest", meaning disk encryption only, for data that deserves field-level protection.
- Plain HTTP or unverified TLS "because it's the internal network".
- Encryption keys stored next to the data they protect (same database, same bucket, same repo).
- Secrets in Git, environment dumps, container images or CI output.
- Home-made cryptography, ECB mode, reused nonces, encryption without authentication.
- Production data in staging, demos and developers' laptops.
- Logging whole request and response bodies.
- No plan for key rotation, so keys are never rotated.

---

## 7. Checklist

- [ ] Data classified; restricted fields identified and listed
- [ ] TLS on every connection, internal ones included; certificates validated
- [ ] Storage and backups encrypted; restricted fields encrypted or tokenised in the application
- [ ] Keys in a KMS/HSM, separated from data, with rotation and usage logs
- [ ] Secrets in a secrets manager; secret scanning in CI; no long-lived credentials where avoidable
- [ ] Logs, traces, errors and analytics checked for sensitive data
- [ ] Non-production environments use masked or synthetic data
- [ ] Retention limits defined and enforced by automation

---

## 8. Exercises

1. Pick an app you built. List every place a user's email address ends up (tables, logs, caches, backups, third parties). Which copies are unprotected?
2. An attacker gets a full database dump. For each sensitive column, what do they learn? Which columns should move to field-level encryption or tokenisation?
3. Design key rotation for the worked example in §3: how do you rotate the KEK, how do you rotate DEKs, and what happens to reads during rotation?
4. A support agent needs to verify a customer's identity by national ID. Design this so the agent never sees the full number.

---

## 9. Further reading

OWASP Cryptographic Storage, Transport Layer Security and Secrets Management Cheat Sheets; OWASP ASVS (the data protection and cryptography chapters); NIST SP 800-57 (key management) and SP 800-52 (TLS); *Serious Cryptography* (Jean-Philippe Aumasson); *Cryptography Engineering* (Ferguson, Schneier, Kohno); Google Tink and libsodium documentation; AWS KMS docs, "Envelope encryption".
