# Replaceability: hands-on practice notes

Planned labs for learning replaceability by **swapping real components and finding out what "the same job" hides**. Theory is in [README.md](README.md).

They need **Docker Compose** and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)), which has a fake payment provider and a `switch` on payment method in four places.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** branch by abstraction (evolvability lab 4), parallel run (evolvability lab 5), shotgun surgery → registry (modifiability lab 1), move the storage (portability lab 5).

---

## Labs

### 1. A port and a second provider ⭐
- **Build:** a `PaymentGateway` port in the shop's own terms. Move the existing fake provider behind an adapter. Write a second fake provider with deliberately different behaviour: different decline codes, different rounding, webhooks that can arrive before the API response.
- **Check:** no provider type or error code appears outside the adapters (grep for it; add an import rule in CI).
- **See:** README §3, §4.

### 2. Conformance suite ⭐
- **Build:** one test suite written against the port: success, decline, timeout, duplicate request with the same idempotency key, partial refund, bad webhook signature. Run it against both adapters.
- **Expect:** the second adapter fails several tests the first time. Each failure is a behavioural difference the API didn't show.
- **See:** README §2 ("behaviour, not just API").

### 3. Switch under traffic
- **Do:** add our own `payment_id` with a `(provider, provider_ref)` mapping. Route a percentage of new payments to the second provider by flag while k6 sends orders. Refunds must go to the provider that took the payment.
- **Break:** flip the flag back mid-run. Nothing may be lost or double-charged.
- **See:** README §4 steps 4–7.

### 4. Drop-in replacement, tested
- **Do:** run messy-shop's cache and queue on **Redis**, then on **Valkey**, changing only the image. Run the product-image storage on **MinIO** through the S3 API.
- **Record:** what worked unchanged, and what needed checking (versions, config options, client library behaviour).
- **Then:** try the harder one: point the app at MySQL instead of Postgres and list everything that breaks. Stop when the list is long enough to make the point.
- **See:** README §1, §2.

---

## Progress

- [ ] 1. A port and a second provider
- [ ] 2. Conformance suite
- [ ] 3. Switch under traffic
- [ ] 4. Drop-in replacement, tested
