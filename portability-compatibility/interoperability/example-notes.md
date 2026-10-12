# Interoperability: hands-on practice notes

Planned labs for learning interoperability by **making independently written systems talk, and finding where they misunderstand each other**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) plus a second system built in lab 1: **`warehouse`**, a small service written in a *different language* from messy-shop, with its own data model. Using a second language is the point: it's what exposes assumptions. Later labs need **Docker Compose**, **Redpanda** (or Kafka), **Keycloak**, **WireMock** or **Prism**, and **Pact**.

**Suggested order:** 1 → 2 → 3 → 4 → 5, then 6–8 in any order.

**Related labs elsewhere:** outgoing webhooks (extensibility lab 4), contract testing between services (testability lab 4), propagation through a queue (observability lab 4), schema registry and compatibility modes (backward compatibility lab 5), a port and a second provider (replaceability lab 1).

**Planned folder layout:**

```
portability-compatibility/interoperability/labs/
  warehouse/               ← the second system, in another language
  01-spec-first/README.md  ← steps + what went wrong
  02-semantics/
  ...
```

---

## Labs

### 1. Specification first, two languages ⭐
- **Build:** write an OpenAPI document for "shop sends an order to the warehouse; warehouse reports shipment". Generate the client in messy-shop's language and the server stub in warehouse's language. Run a mock (Prism) from the spec before the real server exists.
- **Check:** both sides were built against the document only. List every question the spec didn't answer.
- **See:** README §5, §8.

### 2. Semantic traps ⭐
- **Break:** send through the integration, one at a time: an order ID above 2⁵³; a timestamp without an offset across a daylight-saving change; a weight with no unit; a price in JPY and one in KWD using "2 decimals"; a product name with emoji and combining characters; `null` vs missing vs `""` in a partial update.
- **Record:** which were rejected, which were silently wrong.
- **Fix:** strings for IDs, RFC 3339 timestamps, units in field names, integer minor units + ISO 4217, UTF-8 declared and tested, explicit partial-update rules. Put each rule in the OpenAPI document.
- **See:** README §6 worked example.

### 3. Tolerant reader, strict writer
- **Build:** validate every outgoing message against the schema in tests. On the receiving side, ignore unknown fields but reject malformed ones with an RFC 9457 error body.
- **Break:** add a new field on one side and deploy only that side. Then send a malformed date. Check the first works and the second fails clearly, with a counter in the metrics.
- **See:** README §7.

### 4. Consumer-driven contracts ⭐
- **Build:** Pact tests in messy-shop (the consumer) describing what it needs from warehouse. Verify them in warehouse's CI.
- **Break:** in warehouse, rename a field messy-shop uses, and separately one it doesn't. Only the first should fail the build.
- **See:** README §8.

### 5. An unreliable partner behind an anticorruption layer
- **Build:** a fake carrier API (WireMock) with its own vocabulary ("consignment", "pieces"), random latency, `429` with `Retry-After`, occasional `500`, and duplicate webhook deliveries in random order.
- **Fix:** one adapter that maps the carrier's model to the shop's, with timeouts, backoff, idempotency keys, webhook signature checks and deduplication by event ID. Store the raw payloads.
- **Check:** grep the shop's domain code for the word "consignment". It must not appear.
- **See:** README §9.

### 6. Same data, three integration styles
- **Do:** deliver stock levels from warehouse to messy-shop three ways: a nightly CSV file, a REST endpoint the shop polls, and events on Redpanda in a CloudEvents envelope.
- **Compare:** freshness, load on each side, what happens when one side is down for an hour, and how hard each was to get right (try a CSV with commas, quotes and a non-UTF-8 export).
- **See:** README §3.

### 7. Standard identity between the two systems
- **Build:** Keycloak as the identity provider. Warehouse calls the shop's API with an OAuth 2.0 client-credentials token; the shop validates it with a standard library and checks scopes. Add "sign in with Keycloak" (OpenID Connect) to the shop's admin.
- **Break:** an expired token, a token for the wrong audience, a missing scope. Each must give a distinct, correct error.
- **See:** README §10.

### 8. Reconciliation
- **Build:** a nightly job that compares orders shipped according to the shop with shipments according to the warehouse, and reports differences.
- **Break:** drop one webhook, deliver one twice, and change one quantity by hand.
- **Check:** all three show up in the report, with enough detail to fix them.
- **See:** README §8, §9.

---

## Progress

- [ ] 1. Specification first, two languages
- [ ] 2. Semantic traps
- [ ] 3. Tolerant reader, strict writer
- [ ] 4. Consumer-driven contracts
- [ ] 5. An unreliable partner behind an anticorruption layer
- [ ] 6. Same data, three integration styles
- [ ] 7. Standard identity between the two systems
- [ ] 8. Reconciliation
