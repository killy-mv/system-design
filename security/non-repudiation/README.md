# Non-repudiation

> **Non-repudiation** = a party **cannot credibly deny** having performed an action or sent a message, because there is evidence that only they could have produced and that a third party can check.
> "The customer says 'I never approved that transfer.' What can we show a bank, an auditor or a court?"

Related: [Auditability](../auditability/README.md) (recording actions; non-repudiation is the stronger claim), [Integrity](../integrity/README.md) (the content wasn't changed), [Authentication](../authentication/README.md) (the actor was verified), [Compliance](../compliance/README.md) (legal requirements for signatures and records).

---

## 1. How it differs from its neighbours

| Property | What it gives you | Who it convinces |
|---|---|---|
| **Authentication** | "The request came from Alice's session" | Your server, at that moment |
| **Integrity** | "The message wasn't altered" | Anyone holding the key or hash |
| **Auditability** | "Our log says Alice did it" | Anyone who **trusts your log** |
| **Non-repudiation** | "Here is evidence only Alice could have created" | A **third party who trusts neither side** |

The gap between the last two is the point. An audit log is the operator's own record: in a dispute, the customer can say the operator made a mistake or made it up. Non-repudiation needs evidence the operator **could not have forged**.

Two directions matter:

- **Non-repudiation of origin**: the sender can't deny sending (a signed payment order).
- **Non-repudiation of receipt**: the receiver can't deny receiving (a signed acknowledgement or receipt).

---

## 2. Why a MAC isn't enough

| Mechanism | Who can create a valid tag? | Non-repudiation? |
|---|---|---|
| **HMAC** (shared secret) | Both parties | **No.** Either side could have produced it, so it proves nothing to a third party |
| **Digital signature** (private key) | Only the private key's holder | **Yes**, if the key was under the signer's sole control |
| **Session cookie / bearer token** | Anyone who holds it, including the server that issued it | No |
| **Password entry** | The server sees the password and could replay it | No |

So the building blocks are:

1. **An asymmetric signature** over the exact content (Ed25519, ECDSA, RSA-PSS).
2. **A private key only the signer controls**: in a secure element, security key, smart card or HSM, not on the operator's server.
3. **A binding from the key to an identity**: a certificate from a certificate authority, or a registration ceremony you can evidence.
4. **A trusted timestamp** (RFC 3161 timestamping authority) proving the signature existed at a time when the key was valid, so that a later "my key was stolen" or certificate expiry doesn't undo it.
5. **Preserved evidence**: the signed bytes, signature, certificate chain, timestamp and revocation status, kept for as long as a dispute is possible.

---

## 3. Worked example: approving a payment

A weak design and a strong one for the same action.

**Weak:** the user clicks "Approve" in a logged-in session. The server records `approved_by=alice` in its database.
In a dispute the only evidence is the operator's own row. A stolen session, a malicious insider or a bug could have written it.

**Strong:** the approval is signed on the user's device over a canonical description of *what* is being approved.

```
payload = {
  "action":  "payment.approve",
  "payment": "p_7731",
  "amount":  "12500.00",
  "currency":"EUR",
  "payee":   "DE89 3704 0044 0532 0130 00",
  "nonce":   "c5f1…",                 ← from the server, single use (no replay)
  "time":    "2026-03-10T14:02:11Z"
}

client:  signature = Sign(device_private_key, SHA-256(canonical(payload)))
server:  Verify(alice_public_key, payload, signature)   → store payload + signature + key ID
         + timestamp token from a timestamping authority
```

What each part contributes:

| Element | Protects against |
|---|---|
| Amount and payee inside the signed payload | "I approved a different payment" (what-you-see-is-what-you-sign) |
| Server nonce | Replaying an old approval |
| Device-held private key, unlocked by biometric or PIN | The operator or a session thief forging the approval |
| Trusted timestamp | Backdating; later key revocation |
| Stored original bytes | Arguments about what exactly was signed |

This is the idea behind **dynamic linking** in the EU's strong customer authentication rules for payments (PSD2): the authentication code must be bound to the specific amount and payee. WebAuthn can provide the device signature, with a hash of the transaction details used as the challenge.

---

## 4. Where it's used

| Area | Mechanism |
|---|---|
| **Electronic signatures on documents** | PDF/XML signatures (PAdES, XAdES). The EU's eIDAS regulation defines *simple*, *advanced* and *qualified* electronic signatures; a qualified signature has the legal effect of a handwritten one. In the US, the ESIGN Act and UETA give electronic signatures legal standing |
| **Payments and banking** | Signed payment orders, dynamic linking, card chip cryptograms (EMV) |
| **B2B messaging and APIs** | Signed requests and responses (JWS, HTTP Message Signatures, RFC 9421), signed receipts in protocols such as AS2 and Peppol e-invoicing |
| **Software supply chain** | Signed commits and tags, signed artifacts and provenance, public transparency logs (Sigstore Rekor) |
| **Email** | S/MIME and PGP signatures. DKIM signs for the *domain*, not the individual |
| **Public records** | Certificate Transparency: append-only public logs that anyone can audit, so a CA can't deny having issued a certificate |

---

## 5. Limits

Non-repudiation is partly cryptographic and partly legal and procedural.

- **A signature proves the key was used, not that the person intended it.** Malware, coercion or a shared device can all produce valid signatures. Evidence of key custody and user intent (clear display of what's signed, deliberate user action) matters.
- **Key compromise** is the standard defence. You need revocation, timestamps and a record of when compromise was reported.
- **If the server generates or stores the user's private key**, the operator could have signed, so the property is weak.
- **Canonicalisation**: both sides must sign exactly the same bytes. Keep the original signed bytes; don't re-serialise.
- **Long-term validity**: algorithms weaken and certificates expire. Long-lived records need archival timestamps and periodic re-timestamping.
- **Privacy tension**: undeniable evidence is not always wanted. Messaging protocols such as Signal are designed for *deniability* on purpose.
- **Cost and friction**: use it where disputes are likely and expensive (money, contracts, regulated records, releases), not for every click.

---

## 6. Anti-patterns

- Calling an application log entry "non-repudiation".
- HMAC or a shared API key used where a third party may need to be convinced.
- Signing a hash or ID the user never saw, instead of the meaningful content.
- Private keys generated, escrowed or usable by the operator.
- No timestamp, so nobody can show the signature predates a key revocation.
- Discarding the original signed bytes and keeping only parsed fields.
- A "type your name to sign" box with no evidence of who typed it.
- Shared accounts or shared signing keys.

---

## 7. Checklist

- [ ] Actions that need non-repudiation identified (money movement, contracts, approvals, releases)
- [ ] Asymmetric signatures over the full, canonical, human-meaningful content
- [ ] Private keys under the signer's sole control (device, security key, HSM); never on the verifier's side
- [ ] Key-to-identity binding evidenced (certificate or recorded enrolment)
- [ ] Nonce or sequence number to prevent replay
- [ ] Trusted timestamps; revocation process with recorded times
- [ ] Original signed bytes, signature, key/certificate and timestamp retained for the dispute period
- [ ] Signed receipts where proof of delivery matters

---

## 8. Exercises

1. Your payment provider signs webhooks to you with HMAC. Six months later you dispute whether a "payout completed" event was ever sent. What can each side prove? What would change with asymmetric signatures and signed acknowledgements?
2. Design the evidence package you'd keep for a signed loan agreement that must hold up for 10 years.
3. A user claims malware approved a transfer on their phone. Which parts of the §3 design help decide the dispute, and which don't?
4. Compare "click to accept terms" with an advanced electronic signature. For which of your product's actions is the first enough?

---

## 9. Further reading

ISO/IEC 13888 (non-repudiation mechanisms); NIST FIPS 186-5 (Digital Signature Standard); RFC 3161 (Time-Stamp Protocol); RFC 7515 (JSON Web Signature) and RFC 9421 (HTTP Message Signatures); the EU eIDAS Regulation (910/2014) and ETSI standards for PAdES/XAdES; the EU regulatory technical standards on strong customer authentication (dynamic linking); W3C WebAuthn specification; Sigstore documentation (Rekor transparency log); Ross Anderson, *Security Engineering* (chapters on banking and on what signatures do and don't prove).
