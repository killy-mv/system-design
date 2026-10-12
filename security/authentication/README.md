# Authentication

> **Authentication** (authn) = verifying that a user, device or service **is who it claims to be**, and keeping that proof valid for the rest of the session.
> "Is this really Alice, and is it still Alice ten requests later?"

In **ISO/IEC 25010**, this is *authenticity*: the degree to which the identity of a subject or resource can be proved to be the one claimed.

Related: [Authorization](../authorization/README.md) (what Alice may do, once we know it's Alice), [Confidentiality](../confidentiality/README.md) (protecting the secrets involved), [Auditability](../auditability/README.md) (recording logins), [Non-repudiation](../non-repudiation/README.md), [Memorability §4](../../UI-UX/memorability/README.md#4-memorability-of-credentials) (the human side of credentials).

---

## Table of contents

1. [Identification, authentication, authorization](#1-identification-authentication-authorization)
2. [Factors and assurance levels](#2-factors-and-assurance-levels)
3. [Passwords](#3-passwords)
4. [Multi-factor authentication](#4-multi-factor-authentication)
5. [Passkeys and WebAuthn](#5-passkeys-and-webauthn)
6. [Sessions and tokens](#6-sessions-and-tokens)
7. [Federation: OAuth 2, OpenID Connect, SAML](#7-federation-oauth-2-openid-connect-saml)
8. [Service and machine authentication](#8-service-and-machine-authentication)
9. [The account lifecycle](#9-the-account-lifecycle)
10. [Attacks and defences](#10-attacks-and-defences)
11. [Approaches by system type](#11-approaches-by-system-type)
12. [Trade-offs](#12-trade-offs)
13. [Anti-patterns](#13-anti-patterns)
14. [Real-world stories](#14-real-world-stories)
15. [Checklist](#15-checklist)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Identification, authentication, authorization

| Step | Question | Example |
|---|---|---|
| **Identification** | Who do you claim to be? | Typing `alice@example.com` |
| **Authentication** | Can you prove it? | Password, passkey, client certificate |
| **Authorization** | What are you allowed to do? | Alice may read her own orders |

HTTP naming is confusing: status **401 Unauthorized** means "not authenticated", and **403 Forbidden** means "authenticated, but not allowed".

Authentication has two halves, and both must hold:

1. **Proving identity once** (login).
2. **Carrying that proof** on every later request (session or token). A perfect login is worthless if the session cookie can be stolen and reused.

---

## 2. Factors and assurance levels

| Factor | Examples | Typical weakness |
|---|---|---|
| **Something you know** | Password, PIN | Guessed, reused, phished |
| **Something you have** | Phone, security key, smart card | Lost, stolen, SIM-swapped |
| **Something you are** | Fingerprint, face | Can't be changed after a leak; best used to unlock a device-held key |

**Multi-factor** means factors of *different kinds*. A password plus a security question is two things you know: one factor.

**NIST SP 800-63B** defines Authenticator Assurance Levels:

| Level | Requirement (simplified) | Typical use |
|---|---|---|
| **AAL1** | Single factor | Low-risk consumer accounts |
| **AAL2** | Two factors; a phishing-resistant option must be offered | Most business and financial applications |
| **AAL3** | Hardware-based, phishing-resistant authenticator with a non-exportable key | Administrators, high-value systems |

Match the level to the damage a takeover would cause, and raise it for risky actions ([§9](#9-the-account-lifecycle), step-up).

---

## 3. Passwords

Passwords will be around for a long time. Handle them properly, and offer something better next to them.

### 3.1 Storage

Never store passwords, and never store them encrypted or with a fast hash (MD5, SHA-1, SHA-256). Use a **slow, salted, memory-hard password hash**:

| Algorithm | Notes |
|---|---|
| **Argon2id** | First choice. OWASP's minimum: 19 MiB memory, 2 iterations, 1 thread |
| **scrypt** | Good alternative where Argon2 isn't available |
| **bcrypt** | Fine for existing systems; cost ≥ 10; only the first 72 bytes of input are used |
| **PBKDF2-HMAC-SHA-256** | When FIPS compliance is required; needs a very high iteration count |

```python
from argon2 import PasswordHasher
ph = PasswordHasher()                      # Argon2id with sensible defaults

stored = ph.hash(password)                 # "$argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>"

def login(user, password):
    try:
        ph.verify(user.password_hash, password)
    except VerifyMismatchError:
        return fail()                      # same response and timing as "no such user"
    if ph.check_needs_rehash(user.password_hash):
        user.password_hash = ph.hash(password)   # upgrade parameters at next login
    return start_session(user)
```

The salt is random per password and stored inside the hash string. The parameters are stored too, so you can raise them later and rehash at the next login. Tune the cost so one hash takes roughly 100–500 ms on your servers, and remember that this makes the login endpoint expensive: rate limit it ([Availability §9](../../runtime-operational/availability/README.md#9-overload-protection)).

### 3.2 Policy

Current NIST SP 800-63B guidance reverses many old habits:

| Do | Don't |
|---|---|
| Require **length** (15+ characters when a password is the only factor; 8+ when used with MFA) | Composition rules ("one uppercase, one symbol") |
| Allow long passwords (64+), all characters, and **paste** | Block paste or password managers |
| Check new passwords against **breached-password lists** (e.g. Have I Been Pwned's k-anonymity API) | Security questions |
| Force a change **only on evidence of compromise** | Scheduled expiry every 90 days |
| Rate limit and throttle attempts | Password hints |

---

## 4. Multi-factor authentication

| Method | Phishing-resistant? | Notes |
|---|---|---|
| **SMS / voice code** | No | SIM swapping, SS7 interception, real-time phishing. Better than nothing; avoid for high-value accounts |
| **Email code / magic link** | No | Only as strong as the mailbox |
| **TOTP app** (RFC 6238) | No | Codes can be phished and relayed in real time; the shared secret must be stored server-side |
| **Push approval** | No | "MFA fatigue": users approve after repeated prompts. Use **number matching** and show location/app |
| **Passkey / security key** (WebAuthn) | **Yes** | The credential is bound to the site's origin, so a fake site gets nothing |
| **Smart card / client certificate** | Yes | Common in government and enterprise |

MFA is one of the most effective controls available, but only phishing-resistant methods stop a real-time phishing proxy. Also plan for:

- **Recovery codes** issued at enrolment, stored hashed.
- **More than one authenticator** per account, so losing a phone isn't a lockout.
- **MFA on recovery and on changing MFA**, otherwise the attacker goes around it.

---

## 5. Passkeys and WebAuthn

A **passkey** is a WebAuthn (FIDO2) credential: a key pair created for one site, with the private key held by the user's device or password manager and unlocked by a local biometric or PIN.

```
Registration:  server sends challenge → authenticator creates key pair for "shop.example"
               → server stores public key + credential ID

Login:         server sends random challenge
               → authenticator checks the origin is shop.example, user unlocks with fingerprint/PIN
               → authenticator signs (challenge + origin + counter/flags)
               → server verifies the signature with the stored public key
```

Why this is stronger than a password plus a code:

- **Nothing reusable on the server**: a database leak exposes only public keys.
- **Origin binding**: the browser won't use the `shop.example` credential on `sh0p.example`, so phishing fails.
- **No shared secret travels** over the network; each login signs a fresh challenge.
- It's **two factors in one step**: the device (have) plus biometric or PIN (are/know).

Practical points: **synced passkeys** (iCloud Keychain, Google Password Manager, 1Password) survive device loss and suit consumers; **device-bound keys** (YubiKey) suit administrators and AAL3. Use a maintained server library, support several passkeys per account, and keep a recovery path that isn't weaker than the passkey itself.

---

## 6. Sessions and tokens

### 6.1 Two models

| | **Server-side session** | **Self-contained token (JWT)** |
|---|---|---|
| What the client holds | A random, meaningless session ID | A signed set of claims |
| Server state | Session store (Redis, database) | None to verify; a key to check the signature |
| Revocation | Delete the session: immediate | Hard: valid until expiry unless you add a deny-list or check |
| Size | Small | Larger, sent on every request |
| Good for | Web applications with one backend | APIs across many services, short-lived access |

A common, sound combination: a **short-lived access token** (5–15 minutes) plus a **refresh token** that is stored server-side, rotated on every use, and revoked for the whole family if an old one is ever reused (a sign of theft).

### 6.2 Cookies in the browser

```
Set-Cookie: __Host-session=<128+ random bits>; Path=/; Secure; HttpOnly; SameSite=Lax
```

- `HttpOnly`: JavaScript can't read it, so XSS can't simply copy it.
- `Secure`: sent only over HTTPS.
- `SameSite=Lax` or `Strict`: the main defence against CSRF; add CSRF tokens for sensitive actions.
- `__Host-` prefix: bound to this exact host, no `Domain` attribute.

Avoid keeping tokens in `localStorage`: any XSS can read them. For single-page apps, the **backend-for-frontend** pattern keeps tokens on the server and gives the browser only a cookie.

### 6.3 Session rules

- **New session ID at login** and at every privilege change (prevents session fixation).
- **Idle timeout and absolute timeout**, chosen by risk.
- **Logout destroys the session on the server**, not just the cookie.
- **Revoke all sessions** on password change, MFA change and suspected compromise; show users their active sessions.

### 6.4 Validating a JWT

Use a library, and still check: the signature with a **pinned algorithm** (never trust the token's own `alg`; reject `none`), `iss`, `aud`, `exp`/`nbf`, and that the key came from *your* trusted key set. A JWT is signed, not encrypted: anyone can read its payload.

---

## 7. Federation: OAuth 2, OpenID Connect, SAML

Federation lets one **identity provider (IdP)** authenticate users for many applications. You handle fewer passwords, and users get single sign-on.

| Protocol | Purpose | Token |
|---|---|---|
| **OAuth 2.0 / 2.1** | **Delegated authorization**: let an app call an API on the user's behalf | Access token |
| **OpenID Connect (OIDC)** | **Authentication** layer on top of OAuth | ID token (a JWT about the user) |
| **SAML 2.0** | Enterprise single sign-on (XML-based) | Signed assertion |

OAuth by itself is **not** authentication: an access token says "the bearer may call this API", not "the bearer is Alice". Use OIDC for login.

The flow to use for users is **Authorization Code with PKCE**:

```
1. App → browser redirect to IdP: client_id, redirect_uri, scope, state, code_challenge
2. User authenticates at the IdP (the app never sees the password)
3. IdP → redirect back with a one-time code (+ state)
4. App → IdP (back channel): code + code_verifier  →  ID token, access token, refresh token
5. App validates the ID token (signature, iss, aud, exp, nonce) and starts its own session
```

Rules that prevent most real-world bugs:

- **Exact-match redirect URIs**; no wildcards.
- Check **`state`** (CSRF) and **`nonce`** (replay).
- **PKCE for every client**, including confidential ones. OAuth 2.1 drops the implicit and password grants entirely.
- Identify users by **`iss` + `sub`**, not by email (emails change and can be reassigned).
- Check **`aud`**: a token issued for another application must be rejected.
- For enterprise customers, expect SSO (OIDC/SAML) and **SCIM** for provisioning and, more importantly, *de*provisioning.

---

## 8. Service and machine authentication

Services, jobs and CI pipelines authenticate too, and there's no human to type a code.

| Method | How | Notes |
|---|---|---|
| **API key** | Long random string in a header | Simple; long-lived and easily leaked. Store hashed, show once, scope it, make rotation easy |
| **Signed requests** | HMAC over the request (AWS SigV4) | The secret never travels; includes replay protection |
| **mTLS** | Both sides present certificates | Strong; needs certificate issuance and rotation (service mesh, SPIFFE/SPIRE) |
| **OAuth client credentials** | Service gets a short-lived token from the IdP | Central control, scopes |
| **Workload identity** | The platform vouches for the workload (cloud IAM roles, Kubernetes service account tokens) | No stored secret at all |
| **OIDC federation for CI** | The CI system's signed token is exchanged for cloud credentials | Removes long-lived cloud keys from CI |

Direction of travel: **short-lived, automatically issued credentials** instead of static secrets. In a **zero-trust** design every call is authenticated, and "it came from inside the network" proves nothing.

---

## 9. The account lifecycle

Attackers go for the weakest path, and that is rarely the login form.

| Stage | Risks | Good practice |
|---|---|---|
| **Registration** | Fake accounts, squatting on someone's email | Verify email/phone before trusting it; rate limits; bot defences |
| **Login** | Stuffing, phishing, enumeration | See [§10](#10-attacks-and-defences) |
| **Recovery** | The easiest bypass of everything else | Single-use, short-lived, hashed reset tokens; identical response whether or not the account exists; invalidate sessions after reset; don't let recovery skip MFA |
| **Credential change** | Attacker locks the owner out | Re-authenticate first; notify the *old* email address; delay or allow reversal |
| **Step-up** | A stolen session used for a high-risk action | Fresh authentication for payouts, exports, changing MFA, deleting the account |
| **Support channel** | Social engineering | Scripted verification; staff can't reset MFA on a phone call alone |
| **Offboarding** | Ex-employees and stale accounts keep access | Automatic deprovisioning (SCIM), disable dormant accounts, rotate shared secrets |

---

## 10. Attacks and defences

| Attack | How it works | Defences |
|---|---|---|
| **Credential stuffing** | Passwords leaked elsewhere are replayed here | MFA, breached-password checks, rate limits per account and per IP, bot detection, passkeys |
| **Password spraying** | A few common passwords tried on many accounts | Block common passwords, detection across accounts, MFA |
| **Brute force** | Many guesses on one account | Throttling with backoff; slow password hashing limits offline guessing |
| **Phishing / real-time proxy** | A fake site relays the password *and* the code | Phishing-resistant MFA (WebAuthn) |
| **MFA fatigue** | Repeated push prompts until the user approves | Number matching, prompt limits, passkeys |
| **SIM swap** | The attacker takes over the phone number | Don't rely on SMS for valuable accounts |
| **Session theft** | Cookie or token stolen by XSS, malware or a leaked log | `HttpOnly` cookies, short lifetimes, rotation, re-authentication for sensitive actions, device-bound sessions where available |
| **Session fixation** | The victim logs in on a session ID the attacker already knows | New session ID at login |
| **CSRF** | Another site uses the victim's cookies | `SameSite`, CSRF tokens |
| **User enumeration** | Different errors or timing reveal which accounts exist | Uniform messages and timing on login, signup and reset |
| **Token forgery** | Weak or leaked signing key, `alg` confusion | Strong keys in a KMS, pinned algorithms, key rotation |

Hard account lockout after N failures lets an attacker lock out real users. Prefer progressive delays and risk signals.

---

## 11. Approaches by system type

| System | Authentication focus |
|---|---|
| **Consumer web / mobile** | Passkeys and social login alongside passwords; low-friction MFA; strong recovery; bot and stuffing defences |
| **B2B SaaS** | SSO (OIDC/SAML) per tenant, SCIM, enforced MFA policies, session controls for admins |
| **Banking / payments** | Strong customer authentication, step-up per transaction, device binding, signed transactions ([Non-repudiation](../non-repudiation/README.md)) |
| **Internal tools** | Company IdP with phishing-resistant MFA, device posture checks, no local accounts |
| **Public APIs** | Scoped API keys or OAuth; hashed keys; easy rotation; per-key rate limits |
| **Microservices** | mTLS or workload identity between services; user identity passed as a verified token, not a trusted header |
| **CI/CD and infrastructure** | OIDC federation, short-lived credentials, no shared admin accounts, hardware keys for production access |
| **IoT / devices** | Per-device keys or certificates provisioned at manufacture; no shared default passwords |

Build versus buy: an identity provider (Keycloak, Auth0, Okta, Microsoft Entra ID, Amazon Cognito) or a well-used framework module is almost always the right choice. Writing your own password storage, session handling and OIDC client is a long list of chances to make a known mistake.

---

## 12. Trade-offs

| Choice | Cost |
|---|---|
| Stronger authentication | More friction, more lockouts, more support tickets |
| Long sessions | Convenient, but a stolen session lives longer |
| Stateless JWTs | No session lookup, but slow revocation and bigger requests |
| Server-side sessions | Instant revocation, but a store that every request depends on |
| SMS codes | Everyone can use them; weak against SIM swap and phishing |
| Synced passkeys | Recoverable and easy; the private key leaves the original device |
| External identity provider | Less to build and maintain; a dependency whose outage or breach becomes yours |
| Easy account recovery | Fewer locked-out users; an easier takeover path |

---

## 13. Anti-patterns

- Passwords stored in plain text, encrypted, or with a fast or unsalted hash.
- Composition rules and forced periodic changes instead of length and breach checks.
- Different responses for "unknown user" and "wrong password".
- Password reset that is weaker than login: guessable tokens, tokens that never expire, security questions.
- MFA that can be removed with only the password, or skipped through recovery.
- JWTs with no expiry, in `localStorage`, validated without checking `aud`/`iss`, or trusting the token's `alg`.
- Logout that only deletes the cookie.
- Using an OAuth access token as proof of identity; matching federated users by email.
- Long-lived API keys shared by many systems, committed to Git, never rotated.
- Trusting an `X-User-Id` header from inside the network.
- Shared accounts ("admin", "deploy") that hide who did what.
- Writing your own cryptography or session scheme.

---

## 14. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **LinkedIn (2012)** | Password hashes stored as unsalted SHA-1 were stolen. About 6.5 million appeared online in 2012; in 2016 a much larger set from the same breach, over 100 million accounts, was offered for sale. Most were cracked quickly. | Fast, unsalted hashes are close to plain text once the database leaks. |
| **Google security keys (reported 2018)** | Google told journalist Brian Krebs that after requiring physical security keys for its 85,000+ employees in 2017, it had no confirmed account takeovers from phishing. | Phishing-resistant authentication removes a whole class of attack. |
| **Colonial Pipeline (2021)** | According to the company's testimony to the US Congress, attackers entered through a legacy VPN account that used a single password and had no MFA. Fuel deliveries on the US East Coast were disrupted for days. | One forgotten account without MFA is enough. Inventory and retire old access paths. |
| **Twilio and Cloudflare (2022)** | The same SMS phishing campaign targeted both. Twilio reported that attackers used employee credentials to reach internal systems. Cloudflare reported that some employees entered their passwords too, but logins required hardware security keys, so the attackers got no further. | Passwords and one-time codes can be phished; origin-bound keys can't. |
| **Uber (2022)** | Uber stated that an attacker with a contractor's password sent repeated MFA push requests until one was accepted, then moved through internal systems. | Simple push approval is vulnerable to fatigue. Use number matching or passkeys, and limit what one account can reach. |
| **Microsoft Storm-0558 (2023)** | Microsoft reported that an attacker obtained a consumer account signing key and used it to forge tokens that were accepted for enterprise email accounts, due to a validation flaw. | Signing keys are the root of trust. Protect, scope and rotate them, and validate issuer and audience strictly. |
| **Okta support system (2023)** | Okta reported that an attacker accessed files in its customer support system, including HAR files that customers had uploaded for troubleshooting and that contained session tokens. | Session tokens are credentials. Keep them out of logs and uploads, and bind or shorten them where you can. |

---

## 15. Checklist

- [ ] Passwords hashed with Argon2id (or scrypt/bcrypt) with per-password salts and upgradable parameters
- [ ] Length-based policy, breached-password check, paste and password managers allowed
- [ ] MFA available to everyone and required for administrators; a phishing-resistant option offered
- [ ] Passkeys supported, several per account
- [ ] Session cookies `Secure`, `HttpOnly`, `SameSite`; new ID at login; idle and absolute timeouts
- [ ] Server-side logout and "revoke all sessions"; sessions revoked on credential change
- [ ] Tokens short-lived; refresh tokens rotated with reuse detection; JWT `alg`, `iss`, `aud`, `exp` validated
- [ ] OIDC with Authorization Code + PKCE, exact redirect URIs, `state` and `nonce` checked
- [ ] Recovery and credential changes at least as strong as login; users notified
- [ ] Uniform responses and timing; rate limits on login, signup and reset
- [ ] Services use short-lived credentials (mTLS, workload identity, OIDC federation); no secrets in Git
- [ ] Login, failure, MFA and recovery events recorded ([Auditability](../auditability/README.md)) and alerted on
- [ ] Automatic deprovisioning when people leave

---

## 16. Exercises

1. **Cracking cost.** A database with 1 million password hashes leaks. Estimate how long it takes to test the 10,000 most common passwords against every account if hashes are (a) unsalted SHA-1 at 10 billion guesses/s, (b) Argon2id at 10 guesses/s per GPU. Why does the salt matter in (a)?
2. **Recovery review.** Write down every way to get into an account in an app you use without the password (reset email, SMS, support, social login). Which is the weakest? How would you attack it?
3. **Session design.** Design sessions for a banking mobile app and for an internal wiki: lifetimes, storage, revocation, step-up. Justify the differences.
4. **Spot the bugs.** An API accepts JWTs and checks only the signature using the algorithm named in the token header. List three attacks.
5. **Phishing walk-through.** Describe step by step how a real-time phishing proxy defeats password + TOTP, and at which step WebAuthn stops it.
6. **Machine identity.** A GitHub Actions workflow deploys to AWS with an access key stored as a secret. Redesign it without any long-lived secret.

---

## 17. Further reading

- NIST SP 800-63B, *Digital Identity Guidelines: Authentication and Authenticator Management*
- OWASP Cheat Sheets: Authentication, Password Storage, Session Management, Forgot Password, Multifactor Authentication, JSON Web Token
- OWASP ASVS (authentication and session management chapters); OWASP Top 10 ("Identification and Authentication Failures")
- W3C Web Authentication (WebAuthn) specification; passkeys.dev; FIDO Alliance documentation
- RFC 6749 (OAuth 2.0), RFC 7636 (PKCE), RFC 9700 (OAuth 2.0 Security Best Current Practice), the OAuth 2.1 draft
- OpenID Connect Core 1.0
- RFC 8725, *JSON Web Token Best Current Practices*; RFC 6238 (TOTP)
- *OAuth 2 in Action* (Justin Richer, Antonio Sanso)
- *Designing Secure Software* (Loren Kohnfelder)
- Troy Hunt's blog and Have I Been Pwned (Pwned Passwords API)
- Google BeyondCorp papers (zero trust)
