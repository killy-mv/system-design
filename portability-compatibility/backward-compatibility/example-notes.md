# Backward compatibility: hands-on practice notes

Planned labs for learning backward compatibility by **breaking old clients on purpose, then making the same change without breaking them**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) plus one extra piece built in lab 1: a **frozen old client**, a small script or test suite that calls the API the way a customer integrated a year ago, and is never updated. Later labs need **Docker Compose**, **Redpanda** (or Kafka) with a schema registry, **Redis**, and **oasdiff** and **buf** in CI.

**Suggested order:** 1 → 2 → 3 → 4 → 5, then 6–8 in any order.

**Related labs elsewhere:** zero-downtime schema change (deployability lab 6), version compatibility N / N−1 (deployability lab 8), API version translation (evolvability lab 7), contract testing between services (testability lab 4).

**Planned folder layout:**

```
portability-compatibility/backward-compatibility/labs/
  old-client/              ← frozen v1 client + its tests; never edited after lab 1
  01-break-it/README.md    ← what you changed, what the old client did
  02-expand-contract/
  ...
```

---

## Labs

### 1. Break the old client seven ways ⭐
- **Build:** the frozen client: it lists products, places an order, reads the order status with an exhaustive `switch`, and parses an error message.
- **Break:** one change at a time on the server, run the old client after each: rename a field; change `price` from float dollars to integer cents; add an enum value to `status`; make a field nullable; change the default page size; reorder a list; reword an error message.
- **Record:** which changes failed loudly, which failed silently (wrong numbers, missing items), which didn't fail.
- **See:** README §3, §4.

### 2. The same change, compatibly ⭐
- **Fix:** redo the `price` change with expand–contract: add `price_amount` and `price_currency` next to `price`, send `Deprecation` and `Sunset` headers, log which API keys still call without opting in to the new fields.
- **Check:** the frozen client passes at every step. A new client uses only the new fields.
- **See:** README §5 worked example, §8.

### 3. Breaking-change detection in CI ⭐
- **Build:** generate the OpenAPI spec from messy-shop and run **oasdiff** against the spec on `main` in every pull request. Fail on breaking changes unless the PR carries an explicit override label.
- **Check:** replay each change from lab 1 as a PR. Which does the tool catch? Which does it miss (hint: the semantic ones)? Run the frozen client's tests in CI to cover those.
- **See:** README §10.

### 4. Readers before writers: the rollback trap ⭐
- **Break:** v2 adds order status `partially_shipped` and writes it. Deploy v2, create some orders, roll back to v1. Watch v1 fail on the unknown value.
- **Fix:** split into v2a (reads the new value, never writes it) and v2b (writes it behind a flag). Repeat the rollback.
- **Also:** do the same with a cached JSON shape in Redis; fix it with a versioned cache key.
- **See:** README §9.

### 5. Schema registry and compatibility modes
- **Build:** `order-events` in Avro or Protobuf on Redpanda with the schema registry. A producer and two consumers.
- **Try:** add a field with a default; add one without; remove a field; rename a field. Do each under `BACKWARD`, `FORWARD` and `FULL_TRANSITIVE` and note which the registry rejects.
- **Check:** for each mode, which side do you have to deploy first? Verify by actually deploying in the wrong order.
- **Protobuf variant:** run `buf breaking` in CI; reuse a deleted field number and see what a consumer decodes.
- **See:** README §6.

### 6. Pinned API versions with a translation layer
- **Build:** a `Shop-Version: 2025-01-01` style header. The core serves only the newest shape; small per-version transformers convert responses (and requests) for older versions. New API keys are pinned to the current version on first use.
- **Check:** the frozen client, pinned to the oldest version, still passes after three more "breaking" changes. Count the lines of code each old version costs.
- **See:** README §7.

### 7. Deprecate and remove an endpoint
- **Do:** pick an endpoint. Add `Deprecation` / `Sunset` headers, a usage dashboard per API key, and a changelog entry. Run two brownouts (15 minutes of `410 Gone` with a link to the migration guide). Then remove it.
- **Check:** the frozen client's logs show the warning before the failure. What would a real customer have seen, and when?
- **See:** README §8.

### 8. Library compatibility
- **Build:** extract a small client library for the shop API and publish `1.0.0` locally. A separate "dependent" project uses it.
- **Break:** change a function signature, an exception type, a default argument. Classify each as patch, minor or major.
- **Fix:** deprecate in `1.1.0` with a warning, remove in `2.0.0`, and write the migration note. Add an API-surface check to CI (API Extractor, japicmp, `cargo-semver-checks`, or `griffe` for Python).
- **See:** README §7, §10.

---

## Progress

- [ ] 1. Break the old client seven ways
- [ ] 2. The same change, compatibly
- [ ] 3. Breaking-change detection in CI
- [ ] 4. Readers before writers: the rollback trap
- [ ] 5. Schema registry and compatibility modes
- [ ] 6. Pinned API versions with a translation layer
- [ ] 7. Deprecate and remove an endpoint
- [ ] 8. Library compatibility
