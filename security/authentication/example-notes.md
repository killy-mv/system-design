# Authentication: hands-on practice notes

Planned labs for learning authentication by **building each mechanism, attacking it, and fixing what the attack finds**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)). Its login is bad on purpose for these labs: SHA-1 password hashes, a session ID that doesn't change at login, a JWT in `localStorage`, different errors for unknown users, and a reset token built from the timestamp.

Tools: **Docker Compose**, **Keycloak** (identity provider), **hashcat** or **John the Ripper**, **Burp Suite Community / OWASP ZAP**, a browser with passkey support (Chrome's DevTools has a virtual WebAuthn authenticator), and **step-ca** or **mkcert** for certificates.

Only attack systems you own. Every lab here runs on your own machine.

**Suggested order:** 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8.

**Related labs elsewhere:** secrets done right (configurability lab 3), read-only impersonation (supportability lab 4), rate limiting and overload behaviour (capacity lab 4), object-level checks (authorization lab 1).

---

## Labs

### 1. Crack your own password database ⭐
- **Break:** create 1,000 users with passwords from a common-password list plus a few strong ones. Dump the SHA-1 hashes and run `hashcat` with a wordlist. Record how many fall and how fast.
- **Fix:** migrate to Argon2id with rehash-on-login. Run the same attack and compare guesses per second.
- **Then:** add a breached-password check at signup using the Pwned Passwords range API (only the first 5 hash characters leave your server).
- **See:** README §3.

### 2. Sessions done right ⭐
- **Break:** demonstrate session fixation (set a known session ID, have the "victim" log in, reuse it). Steal the JWT from `localStorage` with a planted XSS payload. Log out and show the token still works.
- **Fix:** server-side sessions in Redis, `__Host-` cookie with `Secure; HttpOnly; SameSite=Lax`, new ID at login, idle and absolute timeouts, server-side logout, a "sign out everywhere" button.
- **Check:** repeat each attack.
- **See:** README §6.

### 3. Stuffing, enumeration and rate limits
- **Break:** write a script that finds valid emails from the different error messages and response times, then replays a leaked-credentials list against them.
- **Fix:** uniform responses and timing, per-account and per-IP throttling with backoff, alerts on many failures across accounts.
- **Check:** the script no longer distinguishes accounts; a real user who mistypes twice isn't locked out.
- **See:** README §10.

### 4. TOTP, recovery codes and a phishing proxy
- **Build:** TOTP enrolment (QR code, verify a code before enabling), hashed recovery codes, MFA required to disable MFA.
- **Break:** build a tiny reverse proxy on another local hostname that relays the login form, password and code to the real app and keeps the session cookie.
- **See:** why TOTP doesn't stop real-time phishing. README §4.

### 5. Passkeys ⭐
- **Build:** WebAuthn registration and login with a server library (SimpleWebAuthn, py_webauthn, webauthn4j). Support several passkeys per account. Test with Chrome's virtual authenticator and a real phone or security key.
- **Break:** point the phishing proxy from lab 4 at the passkey login.
- **Check:** the browser refuses to use the credential on the wrong origin. Dump the credentials table: what could an attacker do with it?
- **See:** README §5.

### 6. Single sign-on with OIDC ⭐
- **Build:** run Keycloak, register messy-shop as a client, and implement Authorization Code + PKCE *by hand* once (build the redirect, exchange the code, validate the ID token) before switching to a library.
- **Break:** remove the `state` check and perform a login CSRF; register a wildcard redirect URI and steal a code; accept an ID token issued for a different client.
- **Fix:** each one, and key users by `iss` + `sub`.
- **See:** README §7.

### 7. Account recovery review
- **Break:** predict messy-shop's timestamp-based reset token. Reuse a token twice. Reset a password and show old sessions still work. Use recovery to get around MFA.
- **Fix:** random single-use tokens stored hashed with a 15–30 minute expiry, sessions revoked on reset, MFA still required afterwards, notification to the account's email.
- **See:** README §9.

### 8. Service-to-service: from API key to mTLS
- **Build:** order-service calls payment-service, first with a static API key in an env var, then with mTLS using certificates from `step-ca` that expire after 24 hours and renew automatically.
- **Break:** call payment-service directly from another container, with and without a certificate. Let a certificate expire.
- **Optional:** replace a long-lived cloud key in a GitHub Actions workflow with OIDC federation.
- **See:** README §8.

---

## Progress

- [ ] 1. Crack your own password database
- [ ] 2. Sessions done right
- [ ] 3. Stuffing, enumeration and rate limits
- [ ] 4. TOTP, recovery codes and a phishing proxy
- [ ] 5. Passkeys
- [ ] 6. Single sign-on with OIDC
- [ ] 7. Account recovery review
- [ ] 8. Service-to-service: from API key to mTLS
