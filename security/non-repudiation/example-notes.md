# Non-repudiation: hands-on practice notes

Planned labs for learning non-repudiation by **staging disputes and seeing which evidence survives them**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) with **OpenSSL**, a crypto library (**libsodium**/PyNaCl, `cryptography`, or Node's `crypto`), a browser with **WebAuthn** support, and **cosign**.

For each lab, write a short "dispute note": what the other party claims, and what you can show a neutral third person.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** forge a webhook, then verify it (integrity lab 1), tamper-evident log (auditability lab 2), passkeys (authentication lab 5), sign and verify an image (integrity lab 5).

---

## Labs

### 1. Signed approvals ⭐
- **Build:** refunds above 500 in messy-shop need a manager's approval. First version: a button that writes `approved_by` to the database. Second version: the manager's client signs a canonical payload (refund ID, amount, customer, server nonce, time) with an Ed25519 key that never leaves the client; the server verifies and stores payload, signature and key ID.
- **Dispute:** edit the `approved_by` row as a database admin in version one. Try to forge or alter an approval in version two.
- **Check:** replaying an old signed approval for a new refund fails (nonce).
- **See:** README §3.

### 2. HMAC vs signature
- **Build:** messy-shop sends `order.shipped` webhooks to a "partner" service, signed first with HMAC and then with an asymmetric key (JWS or HTTP Message Signatures). The partner returns a signed receipt.
- **Dispute:** the partner claims an event was never sent; the shop claims it was. With HMAC, show that either side could have created the evidence. With signatures and receipts, show who can prove what.
- **See:** README §2.

### 3. Sign what the user sees (WebAuthn)
- **Build:** confirm a payout with a passkey, using a hash of `{amount, payee, nonce}` as the WebAuthn challenge, and show the same details on the confirmation screen.
- **Break:** change the amount on the server after the user confirms; verify the stored assertion no longer matches.
- **Think:** the browser shows your page's text, not the authenticator. What could malicious JavaScript still do, and what does that mean for a dispute?

### 4. Timestamps and long-term evidence
- **Build:** sign a PDF or JSON contract with a key and certificate from a private CA made with OpenSSL. Get an RFC 3161 timestamp token for it from a public timestamping authority (`openssl ts`), and verify it.
- **Break:** revoke the certificate "after" signing. Show how the timestamp proves the signature came first. Then try to verify with only parsed fields and no original bytes.
- **Then:** list what you'd archive so this still verifies in 10 years.
- **Optional:** sign a Git tag and a release artifact with `cosign` and find the entry in the public transparency log.
- **See:** README §2, §5.

---

## Progress

- [ ] 1. Signed approvals
- [ ] 2. HMAC vs signature
- [ ] 3. Sign what the user sees (WebAuthn)
- [ ] 4. Timestamps and long-term evidence
