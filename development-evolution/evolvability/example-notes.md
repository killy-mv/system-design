# Evolvability: hands-on practice notes

Planned labs for learning evolvability by **making large changes incrementally, with a rollback path at every step, and automating the guardrails**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)), plus **Postgres**, **Nginx** (as a routing façade), and **k6**.

**Suggested order:** 1 → 2 → 3 → 4 → 5 → 6, then 7–8.

**Related labs elsewhere:** zero-downtime schema change (deployability lab 6), version compatibility N/N−1 (deployability lab 8), enforce boundaries in CI (modularity lab 4).

---

## Labs

### 1. Write ADRs ⭐
- **Do:** set up `docs/adr/` with the **MADR** template (or `adr-tools`). Write ADRs for 3 real decisions from your labs so far (e.g. "modular monolith", "Postgres for job queue", "OpenTelemetry for telemetry").
- **Then:** supersede one with a new ADR when you change your mind in a later lab.
- **See:** README §8.

### 2. Fitness functions in CI ⭐
- **Do:** add to messy-shop's pipeline:
  - boundary rules (from modularity lab 4)
  - API breaking-change detection with **oasdiff** on the OpenAPI spec
  - a k6 latency threshold on the main endpoints
  - a dangerous-migration linter (**squawk** for Postgres)
- **Break:** make a breaking API change, a slow query, and an `ALTER TABLE` that locks the table. Each should fail CI with a clear message.
- **See:** README §5.

### 3. Strangler fig with a routing façade ⭐
- **Do:** put **Nginx** in front of messy-shop. Build a small new "catalog" service. Route `/catalog/search` to it, then more catalog routes one by one, keeping the rest on the monolith.
- **Check:** rollback for each route is a one-line config change. Run k6 throughout: zero errors during each switch.
- **See:** README §7.

### 4. Branch by abstraction
- **Do:** replace messy-shop's email sending (direct SMTP calls in 5 places) with a `Mailer` interface → keep the SMTP implementation → add a new provider implementation (or a different library) behind a flag → switch → delete the old one.
- **Check:** main stays deployable at every commit. Count the commits and note that none of them was risky.

### 5. Parallel run (Scientist pattern)
- **Do:** write a new pricing implementation. In production-like traffic, run **both** old and new for each request, return the old result, and log differences with the inputs (use a Scientist port for your language, or 30 lines of your own code).
- **Find:** the mismatches (rounding, edge cases, time zones), fix them, then switch to the new one.
- **See:** README §7 worked example.

### 6. The 32-bit ID wall
- **Do:** create a table with `SERIAL` (32-bit) IDs, `setval` the sequence to near 2,147,483,647, and insert until it fails. Note the exact error.
- **Then:** migrate to `BIGINT` **online** on a table with a few million rows while k6 keeps writing (new column + trigger/dual write + batched backfill + swap), versus a plain `ALTER COLUMN TYPE` (watch the lock).
- **Add:** a monitor for sequence usage percentage.
- **See:** README §4 worked example.

### 7. API version translation
- **Do:** implement Stripe-style versioning: clients send a version date header; the core always uses the newest model; small "version change" modules transform responses for older versions (e.g. `price` float dollars → `amount` integer cents).
- **Check:** old-version contract tests still pass after the change.

### 8. Major dependency upgrade drill
- **Do:** pick a major framework/library upgrade for messy-shop (e.g. an old pinned version of your web framework or ORM). Use **Renovate** + your tests. Log every breaking change you hit and how long each took.
- **See:** README §9. Repeat for a smaller upgrade and compare effort.

---

## Progress

- [ ] 1. Write ADRs
- [ ] 2. Fitness functions in CI
- [ ] 3. Strangler fig with a routing façade
- [ ] 4. Branch by abstraction
- [ ] 5. Parallel run (Scientist pattern)
- [ ] 6. The 32-bit ID wall
- [ ] 7. API version translation
- [ ] 8. Major dependency upgrade drill
