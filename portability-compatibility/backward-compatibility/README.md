# Backward compatibility

> **Backward compatibility** = a new version of a system keeps working with the **clients, data and integrations built for older versions**.
> "We ship a change on Tuesday. Does every caller, stored record and saved file from before Tuesday still work, without anyone else having to do anything?"

Related: [Evolvability §6](../../development-evolution/evolvability/README.md#6-evolving-interfaces-and-data) (evolving interfaces and data over years), [Deployability §8](../../operations-devops/deployability/README.md#8-compatibility-during-rollouts) (old and new versions running together during a rollout), [Maintainability §6](../../development-evolution/maintainability/README.md#6-apis-and-contracts) (API design), [Interoperability](../interoperability/README.md), [Reliability §13.4](../../runtime-operational/reliability/README.md#134-schema-evolution) (schema evolution and data integrity).

---

## Table of contents

1. [Vocabulary](#1-vocabulary)
2. [Where compatibility is promised](#2-where-compatibility-is-promised)
3. [What counts as a breaking change](#3-what-counts-as-a-breaking-change)
4. [Hyrum's Law: behaviour is part of the contract](#4-hyrums-law-behaviour-is-part-of-the-contract)
5. [Making changes compatibly](#5-making-changes-compatibly)
6. [Schema compatibility rules](#6-schema-compatibility-rules)
7. [Versioning](#7-versioning)
8. [Deprecation and removal](#8-deprecation-and-removal)
9. [Old and new running together](#9-old-and-new-running-together)
10. [Detecting breaking changes automatically](#10-detecting-breaking-changes-automatically)
11. [Approaches by system type](#11-approaches-by-system-type)
12. [Trade-offs](#12-trade-offs)
13. [Anti-patterns](#13-anti-patterns)
14. [Real-world stories](#14-real-world-stories)
15. [Checklist](#15-checklist)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Vocabulary

| Term | Meaning | Memory aid |
|---|---|---|
| **Backward compatible** | New code works with old data and old callers | New server, old client. New reader, old data |
| **Forward compatible** | Old code copes with data or requests produced by newer code | Old reader, new data. Needed for rollbacks and for clients that update late |
| **Full compatibility** | Both | Any mix of old and new works |
| **Breaking change** | A change after which something that worked before no longer does | |

Compatibility also has layers:

| Layer | Breaks when |
|---|---|
| **Source** | Code that compiled against the old version no longer compiles |
| **Binary (ABI)** | Already-compiled code can no longer link or run against the new version |
| **Wire / storage** | Bytes written by the old version can't be read by the new one (or the reverse) |
| **Semantic / behavioural** | Everything still parses, but means or does something different |

Semantic breaks are the dangerous ones: nothing fails loudly.

---

## 2. Where compatibility is promised

Anything someone else depends on is an interface, whether or not you meant it to be.

| Surface | Who depends on it | How long old versions live |
|---|---|---|
| Public HTTP API, webhooks | Customers' code | Years |
| Mobile and desktop clients calling your backend | Your own old releases | Months to years (users don't update) |
| Events and messages | Other teams' consumers; stored event logs | As long as the retention, sometimes forever |
| Database schema | Every reader of the tables, reports, the previous app version | Until all readers are migrated |
| File and export formats | Users' saved files, backups | Decades |
| Library / SDK API | Dependents' source and binaries | Until they upgrade |
| CLI flags and output | Scripts and CI pipelines parsing it | Unknown |
| Configuration files and environment variables | Every installation | Until upgraded |
| URLs | Bookmarks, links, search engines | Indefinitely |
| Caches, sessions, serialised jobs in queues | Your own previous deploy | Minutes to days |

---

## 3. What counts as a breaking change

For a JSON-over-HTTP API:

| Usually safe | Breaking |
|---|---|
| Add an optional request field | Add a required request field, or make an optional one required |
| Add a response field (clients must ignore unknown fields) | Remove or rename a response field |
| Add a new endpoint | Remove an endpoint or change its URL or method |
| Relax validation (accept more) | Tighten validation (reject what was accepted) |
| Add an optional query parameter with the old behaviour as default | Change a default (page size, sort order, time zone) |
| | Change a field's type or format (`"42"` → `42`, seconds → milliseconds) |
| | Change status codes or the error body shape |
| | Change meaning while keeping the name (`amount` in dollars → cents) |
| | Change authentication, rate limits or ordering that clients rely on |

Two that surprise people:

- **Adding a value to an enum in a response** breaks every client with an exhaustive `switch` and no default branch. Treat response enums as open, and say so in the docs from day one.
- **Making a response field nullable** breaks clients that never checked for null.

"Safe" always assumes **tolerant readers**: consumers that ignore what they don't understand ([§5](#5-making-changes-compatibly)).

---

## 4. Hyrum's Law: behaviour is part of the contract

> "With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody." (Hyrum Wright)

Things clients come to depend on without being told they can: the order of items in an unordered list, the exact text of error messages, response timing, the length or format of IDs, undocumented fields, bugs.

Defences:

- **Document what is *not* guaranteed** (ordering, ID format, message text).
- **Make the unpromised visibly unstable**: Go randomises map iteration order so nobody can rely on it. Opaque, prefixed IDs (`ord_8sK2…`) discourage parsing.
- **Give machines a stable alternative**: error *codes* next to human messages, `--output json` next to human-readable CLI output.
- **Measure before changing**: log which clients use the old behaviour.

---

## 5. Making changes compatibly

- **Add, don't change.** A new field, endpoint or event type next to the old one.
- **Tolerant reader**: ignore unknown fields, don't depend on field order, handle unknown enum values with a default.
- **Conservative writer**: keep emitting what old readers need for as long as they exist.
- **Defaults**: a new optional input behaves exactly like before when absent.
- **Never reuse** a field name, field number, error code or event type for a different meaning.
- **Expand–contract (parallel change)** for anything that must actually change: add the new form, migrate callers and data, then remove the old form. Steps and a schema example are in [Deployability §9](../../operations-devops/deployability/README.md#9-database-and-schema-changes).
- **Translate at the edge**: keep the core on the newest model and convert for old versions in one layer ([Evolvability §6](../../development-evolution/evolvability/README.md#6-evolving-interfaces-and-data)).

### Worked example: price from float dollars to integer cents

`GET /products/7` returns `{"id": 7, "price": 19.99}`. The team needs integer minor units and multiple currencies.

```
Breaking way:   {"id": 7, "price": 1999, "currency": "USD"}
                → every existing client now shows $1,999.00. Nothing errors.

Compatible way:
Step 1  Expand    {"id": 7, "price": 19.99,
                   "price_amount": 1999, "price_currency": "USD"}
Step 2  Announce  docs, changelog, Deprecation header on responses that include "price"
Step 3  Measure   which API keys still read "price"? (hard to see for response fields:
                  offer an opt-in ?fields= or a new API version and track who has moved)
Step 4  Migrate   contact the remaining consumers; give a date
Step 5  Contract  remove "price" in the next major version only
```

For non-USD products there is no honest value for the old `price` field. Decide explicitly: omit those products from old-version responses, or return an error old clients already handle. Don't return a number in the wrong currency.

---

## 6. Schema compatibility rules

### Protocol Buffers

| Change | Safe? |
|---|---|
| Add a field with a new field number | Yes. Old readers skip it; new readers see the default when it's absent |
| Remove a field | Yes, if you `reserved` its number and name so they're never reused |
| Rename a field | Safe on the binary wire (only numbers are sent). Breaks JSON encoding and generated code |
| Change a field number | No. It's a different field |
| Change a field's type | Only between a few compatible types; otherwise no |
| Add an enum value | Wire-safe; readers must handle unknown values |
| `required` (proto2) | Avoid: a required field can never be removed safely. proto3 dropped it |

### Avro

Avro resolves a **writer schema** against a **reader schema**. Adding a field **with a default** is both backward and forward compatible; adding one without a default is not backward compatible; removing a field without a default is not forward compatible. Renames need aliases.

### Schema registry modes

A schema registry (Confluent Schema Registry, Apicurio, Buf Schema Registry) rejects incompatible schemas at publish time.

| Mode | Guarantee | Upgrade first |
|---|---|---|
| **BACKWARD** | Consumers on the new schema can read data written with the previous one | Consumers |
| **FORWARD** | Consumers on the previous schema can read data written with the new one | Producers |
| **FULL** | Both | Either |
| **…_TRANSITIVE** | The same, checked against *all* earlier versions, not just the last | |

Use a transitive mode when old data is kept and re-read (event sourcing, long retention, replays).

### Databases

The schema is an interface between the database and every version of every application reading it. Use expand–contract, and remember that reports, ETL jobs and other teams' queries are consumers too.

---

## 7. Versioning

Versioning doesn't prevent breaking changes; it **labels** them and lets old and new exist side by side.

**Semantic Versioning** (libraries, tools): `MAJOR.MINOR.PATCH`. Breaking change → major; compatible feature → minor; fix → patch. It only works if you define what your public API *is*.

**API versioning styles:**

| Style | Example | Notes |
|---|---|---|
| URL path | `/v1/orders` | Visible, cache-friendly, easy to route; encourages rare, big versions |
| Header / media type | `Accept: application/vnd.shop.v2+json` | Cleaner URLs; harder to try in a browser |
| Date-based, pinned per account | `Stripe-Version: 2024-06-20` | Many small versions; each account stays on the one it integrated with until it opts to upgrade; needs a translation layer |
| No versions, only additive change | Many internal APIs; GraphQL schemas with `@deprecated` | Works when you can reach every consumer |

Guidelines:

- A new major version is expensive for everyone. Exhaust additive options first.
- Version the **contract**, not the implementation. One codebase should serve all live versions.
- Every version you publish is one you must run, test, secure and eventually retire. Set the end date when you publish it.

---

## 8. Deprecation and removal

Compatibility forever isn't free either. Removal needs a process:

1. **Announce**: changelog, docs, email to affected owners. Say what replaces it and by when.
2. **Signal in-band**: the `Deprecation` response header (RFC 9745) and `Sunset` header (RFC 8594, the date it stops working); compiler deprecation annotations in libraries; warnings in CLI output on stderr.
3. **Measure usage** per consumer. You can't remove what you can't see.
4. **Help migration**: guides, codemods, a compatibility shim, direct contact with the heaviest users.
5. **Brownouts**: switch the old thing off for short, announced windows so the remaining users notice before the final date.
6. **Remove**, and keep a clear error (`410 Gone` with a link) rather than a generic failure.

State the policy up front (for example "deprecated features keep working for at least 12 months"). A written policy is what lets you remove things at all.

---

## 9. Old and new running together

Even with no external consumers, **your own previous version** is a client:

- **Rolling deploys**: N and N−1 serve traffic at once ([Deployability §8](../../operations-devops/deployability/README.md#8-compatibility-during-rollouts)).
- **Rollback**: after rolling back, the *old* code meets data the *new* code wrote. That requires **forward** compatibility.
- **Queues and scheduled jobs**: messages serialised by the old version are processed by the new one, hours later.
- **Caches and sessions**: entries written before the deploy are read after it. Version the cache key or tolerate both shapes.
- **Mobile apps and open browser tabs**: old front ends call the new backend for a long time.

### Worked example: a new enum value and a rollback

```
v41 writes order.status ∈ {pending, paid, shipped}
v42 adds "partially_shipped" and starts writing it

Deploy v42 → 300 orders get status "partially_shipped"
A bug appears → roll back to v41
v41 loads those orders → "unknown status" exception → order pages return 500
```

The safe sequence is two releases:

```
v42a  can READ "partially_shipped" (treats it sensibly), never writes it
v42b  starts WRITING it, behind a flag
```

Rolling back v42b lands on v42a, which understands the data. The rule: **teach readers before writers.**

---

## 10. Detecting breaking changes automatically

| Tool / technique | Catches |
|---|---|
| **OpenAPI diff in CI** (oasdiff, openapi-diff) | Removed or changed endpoints, fields, types |
| **`buf breaking`** | Protobuf wire and source incompatibilities |
| **Schema registry compatibility checks** | Incompatible Avro / Protobuf / JSON Schema versions before they're published |
| **API surface reports** for libraries (API Extractor for TypeScript, japicmp / revapi for Java, `cargo-semver-checks` for Rust) | Source and binary breaks against the last release |
| **Consumer-driven contract tests** (Pact) | Breaks to what consumers actually use, not just what you documented |
| **Old client tests against the new server** | Behavioural breaks |
| **Upgrade tests from real old data** (golden files, a snapshot of an old database) | Storage-format and migration breaks |
| **Schema migration linters** (Squawk, Atlas lint) | Migrations that break the running version or lock tables |

Put at least one of these in CI for each surface in [§2](#2-where-compatibility-is-promised). Reviewers don't spot renamed fields reliably.

---

## 11. Approaches by system type

| System | Typical approach |
|---|---|
| **Public API / developer platform** | Very strict. Additive changes only, long-lived versions, pinned versions per account, published deprecation policy, usage telemetry per key |
| **Internal services in one company** | Additive by default; expand–contract with known consumers; contract tests; breaking changes are possible but coordinated |
| **Mobile backend** | Assume clients from two years ago. Minimum supported app version, a forced-upgrade screen built in from version 1, server-driven feature flags |
| **Event-driven / streaming** | Schema registry in a transitive mode; new event types rather than changed ones; upcasters for old events in an event store |
| **Libraries and SDKs** | SemVer, a defined public surface, deprecate in a minor then remove in a major, changelog and migration guide, codemods where possible |
| **Operating systems, languages, runtimes** | Compatibility as a core promise (Linux syscall ABI, Go 1, Java); opt-in mechanisms for new behaviour (editions, feature flags) |
| **Self-hosted / packaged software** | Supported upgrade paths, config and data migrations, old config keys accepted with warnings ([Installability](../installability/README.md)) |
| **Databases and file formats** | Format version in the header; readers for every old version; writers for the current one |
| **Early-stage product, no external consumers** | Break freely *inside* the team, but still respect N/N−1 for your own deploys |

---

## 12. Trade-offs

| Tension | Notes |
|---|---|
| **Compatibility vs simplicity** | Every preserved quirk is code to keep, test and explain. Without removal, complexity only grows ([Simplicity](../../development-evolution/simplicity/README.md)) |
| **Compatibility vs speed of change** | Strict promises slow redesigns; no promises make every consumer slow instead |
| **Compatibility vs security** | Old protocols and ciphers (TLS 1.0, weak hashes) must eventually be turned off even though clients break |
| **Many small versions vs few big ones** | Small steps are easier for consumers; each needs translation code. Big versions are simpler to run but migrations stall |
| **Strictness vs tolerance** | Rejecting unknown input finds client bugs early; accepting it keeps old and new working together. Be tolerant in what you *read*, exact in what you *write* |
| **Fixing bugs vs preserving behaviour** | Somebody depends on the bug. Sometimes the fix needs a version or a flag |

---

## 13. Anti-patterns

- Changing what a field means while keeping its name.
- Renaming or removing in one step, with no overlap period.
- "It's just an internal API" with consumers nobody has listed.
- A `/v2` that is a full copy of the code, with fixes applied to only one.
- Versions published with no end date, then supported forever by accident.
- Removing something without measuring who still uses it.
- Deprecation notices only in a blog post nobody's code reads.
- Reusing Protobuf field numbers, error codes or event names.
- Clients that fail on unknown fields or enum values.
- Deploying a writer of a new format before every reader understands it.
- Bumping the major version for every release, so the number means nothing.

---

## 14. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Linux kernel: "we do not break userspace"** | The kernel treats a change that breaks existing user programs as a kernel bug, even when the program relied on undocumented behaviour. Linus Torvalds has stated the rule bluntly on the mailing list many times. | A firm, simple compatibility promise lets a huge ecosystem upgrade without fear. |
| **Windows application compatibility** | Windows ships compatibility shims that reproduce old behaviour for specific applications. A well-known example told by Raymond Chen and Joel Spolsky: Windows 95 detected SimCity and changed memory-allocator behaviour for it, because the game read memory it had already freed. | Hyrum's Law at scale: users blame the platform when their old program breaks, so platforms preserve even bugs. It is costly. |
| **Go 1 compatibility promise (2012)** | The Go team promised that programs written to the Go 1 specification would keep compiling and running with later Go 1.x releases, and has kept to that for over a decade. | A clear written promise, with stated exceptions, is itself a feature. |
| **Python 2 → 3** | A deliberately incompatible release that took the ecosystem over a decade to adopt (details in [Evolvability §14](../../development-evolution/evolvability/README.md#14-real-world-stories)). | Break many things at once only if you can afford years of running both. |
| **Stripe API versions** | Each account is pinned to the API version it first used; later changes are applied through per-version transformation modules (2017 engineering blog). | You can evolve quickly behind a translation layer and still keep old integrations working. |
| **Protocol Buffers `required`** | proto2 allowed `required` fields. Google's own guidance came to discourage them because a required field can't be removed without breaking readers, and proto3 removed the feature. | Strictness in a schema becomes a compatibility trap later. Prefer optional with validation in code. |
| **AngularJS → Angular (2016)** | Angular 2 was a different framework from AngularJS 1.x, with no drop-in upgrade; many teams stayed on 1.x for years or moved elsewhere. AngularJS reached end of life in 2022. | An incompatible "next version" is really a new product competing for your own users. |

---

## 15. Checklist

- [ ] Every interface others depend on is listed, with its consumers
- [ ] What is and isn't guaranteed is documented (ordering, ID formats, error text)
- [ ] Changes are additive by default; readers are tolerant
- [ ] Field names, numbers, codes and event types are never reused
- [ ] Breaking changes go through expand–contract or a new version, never in place
- [ ] Readers are deployed before writers of any new format or value
- [ ] Each deploy works with N−1 on the other side; rollback is tested against new data
- [ ] Breaking-change detection runs in CI for APIs, schemas and libraries
- [ ] A written deprecation policy; `Deprecation` / `Sunset` signals; usage measured per consumer
- [ ] Every published version has an owner and a planned end date

---

## 16. Exercises

1. **Spot the break.** For each change, say whether it's breaking and for whom: adding a response field; adding an enum value; changing a default page size from 50 to 20; making a field nullable; changing an error message; returning items in a different order.
2. **Expand–contract.** An event `OrderPlaced` has `customer_name`. You need `first_name` and `last_name`. Plan the change with a schema registry in `FULL_TRANSITIVE` mode and a 30-day topic retention.
3. **Rollback.** Your new version writes a new JSON shape into a Redis cache with a 24-hour TTL. What happens on rollback? Give two designs that make it safe.
4. **Mobile.** 8% of users are on an app version from 18 months ago that calls an endpoint you want to remove. Design the path to removal.
5. **Hyrum's Law.** List five undocumented behaviours of an API you maintain that a client might depend on. How would you find out?
6. **Deprecation policy.** Write a one-page deprecation policy for a public API: notice period, signals, brownouts, exceptions for security.
7. **SemVer.** A library fixes a bug where a function returned results in the wrong order. Some users sort the output themselves; some rely on the old order. Patch, minor or major? Argue both sides.

---

## 17. Further reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 4 "Encoding and Evolution"
- Hyrum Wright, hyrumslaw.com; *Software Engineering at Google* (Winters, Manshreck, Wright), chapters on Hyrum's Law, deprecation and large-scale changes
- Semantic Versioning 2.0.0 (semver.org)
- Protocol Buffers documentation: "Updating a Message Type", "Proto Best Practices"; Buf docs on breaking-change detection
- Apache Avro specification, "Schema Resolution"; Confluent docs, "Schema Evolution and Compatibility"
- RFC 8594 (the `Sunset` header), RFC 9745 (the `Deprecation` header)
- Stripe blog: "APIs as infrastructure: future-proofing Stripe with versioning" (2017)
- Martin Fowler: "TolerantReader", "ParallelChange"; Pact documentation (consumer-driven contracts)
- "Go 1 and the Future of Go Programs" (go.dev/doc/go1compat); Russ Cox, "Backward Compatibility, Go 1.21, and Go 2" (2023)
- Raymond Chen, *The Old New Thing* (Windows compatibility stories); Joel Spolsky, "How Microsoft Lost the API War" (2004)
- Rich Hickey, "Spec-ulation" (talk, 2016): growth vs breakage
