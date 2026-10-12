# Interoperability

> **Interoperability** = two or more systems can **exchange information and correctly use what they exchanged**, without one being built specifically for the other.
> "A partner's system, written by people we've never met in a language we don't use, needs to place orders with us. Can it, and will both sides mean the same thing by 'order'?"

Related: [Backward compatibility](../backward-compatibility/README.md) (staying interoperable with older versions of the *same* system), [Co-existence](../co-existence/README.md) (the other half of ISO/IEC 25010 *compatibility*), [Replaceability](../replaceability/README.md), [Extensibility](../../development-evolution/extensibility/README.md) (webhooks, plugins, public APIs as extension points), [Maintainability §6](../../development-evolution/maintainability/README.md#6-apis-and-contracts) (API design), [Internationalization](../../UI-UX/internationalization/README.md) (text, time and formats across locales).

---

## Table of contents

1. [Levels of interoperability](#1-levels-of-interoperability)
2. [Why it matters](#2-why-it-matters)
3. [Integration styles](#3-integration-styles)
4. [Protocols and API styles](#4-protocols-and-api-styles)
5. [Data formats and schemas](#5-data-formats-and-schemas)
6. [Semantic interoperability: agreeing on meaning](#6-semantic-interoperability-agreeing-on-meaning)
7. [Robustness: how strict to be](#7-robustness-how-strict-to-be)
8. [Contracts and testing](#8-contracts-and-testing)
9. [Integrating with systems you don't control](#9-integrating-with-systems-you-dont-control)
10. [Identity and trust across systems](#10-identity-and-trust-across-systems)
11. [Standards worth knowing](#11-standards-worth-knowing)
12. [Approaches by system type](#12-approaches-by-system-type)
13. [Trade-offs](#13-trade-offs)
14. [Anti-patterns](#14-anti-patterns)
15. [Real-world stories](#15-real-world-stories)
16. [Checklist](#16-checklist)
17. [Exercises](#17-exercises)
18. [Further reading](#18-further-reading)

---

## 1. Levels of interoperability

Two systems can fail to understand each other at four different levels. Each one depends on the level above it in this table.

| Level | Question | Examples of agreement |
|---|---|---|
| **Technical / transport** | Can bytes get from A to B? | TCP/IP, TLS, HTTP, AMQP, MQTT |
| **Syntactic** | Can B parse what A sent? | JSON, XML, Protobuf, Avro, CSV; a schema |
| **Semantic** | Does B understand it the way A meant it? | Units, time zones, identifiers, what "active customer" means |
| **Organisational / process** | Do the two organisations agree on who does what, when, and what happens on failure? | SLAs, retry rules, onboarding, dispute handling, legal agreements |

Most engineering effort goes into the first two. Most integration failures happen in the last two.

---

## 2. Why it matters

- **No system is alone.** A typical product depends on payment, email, identity, analytics, shipping and accounting systems, and is depended on by others.
- **The N × M problem.** With N systems that each need to talk to M others, pairwise custom integrations grow as N × M. A shared standard turns that into N + M.
- **Ecosystems.** Products with good APIs become platforms others build on.
- **Regulation.** Some sectors require it: open banking APIs, health data exchange, data portability rights.
- **Leaving and joining.** Standard interfaces make components replaceable ([Replaceability](../replaceability/README.md)) and reduce lock-in ([Vendor independence](../../business-project/vendor-independence/README.md)).

---

## 3. Integration styles

From *Enterprise Integration Patterns* (Hohpe and Woolf):

| Style | How | Good | Bad |
|---|---|---|---|
| **File transfer** | A writes a file (CSV, XML, Parquet); B picks it up (SFTP, object storage) | Simple, decoupled in time, good for bulk | Slow (batch), partial files, weak schema, hard error handling |
| **Shared database** | Both read and write the same tables | Immediate, no API to build | The schema becomes everyone's interface; no encapsulation; changes break others ([Modularity](../../development-evolution/modularity/README.md)) |
| **Remote procedure call / API** | A calls B and waits (REST, gRPC, GraphQL, SOAP) | Immediate answer, clear contract | Couples availability and latency: if B is down, A is affected |
| **Messaging** | A publishes messages or events; B consumes later (Kafka, RabbitMQ, SQS, MQTT) | Decoupled in time and availability, many consumers | Eventual consistency, duplicates and ordering to handle ([Reliability §10](../../runtime-operational/reliability/README.md#10-reliable-messaging)) |

**Webhooks** are RPC in the other direction: the provider calls the consumer's URL when something happens ([Reliability §10.5](../../runtime-operational/reliability/README.md#105-webhooks-sending-to-external-systems)).

Between organisations, expect a mix: APIs for requests, webhooks for notifications, and files for bulk and reconciliation.

---

## 4. Protocols and API styles

| Style | Strengths | Typical use |
|---|---|---|
| **REST over HTTP + JSON** | Understood by every language and tool; caches, proxies and browsers work with it | Public APIs, most web backends |
| **gRPC (HTTP/2 + Protobuf)** | Compact, fast, strongly typed, streaming, generated clients | Internal service-to-service calls |
| **GraphQL** | Clients ask for exactly the data they need; one endpoint; typed schema | Front ends and mobile apps over many backend resources |
| **SOAP / XML web services** | Formal contracts (WSDL), mature security standards | Banking, government, older enterprise systems |
| **Webhooks** | Push without polling | Notifications from SaaS providers |
| **Messaging protocols** (AMQP, MQTT, the Kafka protocol) | Asynchronous, buffering, fan-out; MQTT suits constrained devices | Event-driven systems, IoT |
| **Files / EDI** (X12, EDIFACT) | Very widely deployed in logistics, retail, healthcare billing | Business-to-business document exchange |

Choosing: use the most **boring, widely implemented** option the other side can consume. For a public API that is nearly always HTTP + JSON, whatever you use internally.

---

## 5. Data formats and schemas

| Format | Notes |
|---|---|
| **JSON** | Universal, human-readable. No date, decimal or binary type; numbers are often parsed as 64-bit floats, so large integers and money lose precision unless sent as strings or integer minor units |
| **XML** | Namespaces, mature schema and signing standards; verbose |
| **Protocol Buffers / Avro** | Compact binary with schemas and defined evolution rules ([Backward compatibility](../backward-compatibility/README.md)) |
| **CSV** | Everyone can produce it; almost nobody agrees on quoting, delimiter, encoding and header rules (RFC 4180 is the closest thing to a standard) |
| **Parquet / Arrow** | Columnar formats for analytics exchange |

**Describe the contract in a machine-readable way:**

| Spec | For |
|---|---|
| **OpenAPI** | HTTP APIs: generates docs, clients, server stubs, mocks and tests |
| **AsyncAPI** | Event and message interfaces |
| **JSON Schema** | JSON document structure and validation |
| **Protobuf IDL / Avro schemas** | Typed messages with evolution rules |
| **CloudEvents** (CNCF) | A common envelope for events: `id`, `source`, `type`, `time`, plus data |
| **RFC 9457 Problem Details** | A standard shape for HTTP API errors |

A schema answers the syntactic question. It does not tell the other side what the fields *mean*.

---

## 6. Semantic interoperability: agreeing on meaning

Both sides parse the message perfectly and still disagree. The usual places:

| Topic | What goes wrong | Agree on |
|---|---|---|
| **Units** | Metres vs feet, kg vs lb, seconds vs milliseconds | The unit in the field name or schema (`weight_grams`, `timeout_ms`) |
| **Money** | Floats; missing currency; minor units that aren't 2 decimals (JPY has 0, KWD has 3) | Integer minor units + ISO 4217 currency code |
| **Time** | Local time without an offset; `03/04/2025`; daylight-saving gaps | RFC 3339 / ISO 8601 timestamps in UTC or with an offset; IANA zone names when the zone matters |
| **Text** | Encoding mismatches; different Unicode normalisation; length limits in bytes vs characters | UTF-8 everywhere; stated limits |
| **Identifiers** | Leading zeros dropped by a spreadsheet; integer IDs overflowing in JavaScript; case sensitivity | IDs as strings; stated format and case rules |
| **Null vs missing vs empty** | `"phone": null`, `"phone": ""` and no `phone` at all treated differently by each side | Explicit rules, especially for partial updates |
| **Enumerations and codes** | "status 3"; free-text country names | Named values; ISO 3166 countries, ISO 639 / BCP 47 languages, E.164 phone numbers |
| **Business meaning** | "Order date" = created, paid or shipped? "Customer" = person or company? Is the price with tax? | A written glossary with examples |
| **Precision and rounding** | Totals differ by a cent between systems | Who rounds, when, and by which rule |

### Worked example: one payload, five misunderstandings

A warehouse partner sends this, and the shop accepts it without errors:

```json
{
  "order_id": 100200300400500600,
  "shipped": "03/04/2025 14:30",
  "weight": 2.5,
  "cost": 12.1,
  "country": "Georgia"
}
```

| Field | Problem |
|---|---|
| `order_id` | Larger than 2⁵³: a JavaScript consumer silently reads a different number |
| `shipped` | 3 April or 4 March? In which time zone? |
| `weight` | Kilograms or pounds? |
| `cost` | A float, no currency; with or without tax? |
| `country` | The country, or the US state? |

The same data with the ambiguity removed:

```json
{
  "order_id": "100200300400500600",
  "shipped_at": "2025-04-03T14:30:00+02:00",
  "weight_grams": 2500,
  "shipping_cost": { "amount_minor": 1210, "currency": "EUR", "tax_included": false },
  "country_code": "GE"
}
```

None of the fixes needed a new protocol. They needed the two teams to write down what they meant.

---

## 7. Robustness: how strict to be

**Postel's law** (the robustness principle, from the early TCP specifications): *be conservative in what you send, be liberal in what you accept.*

It made the early internet work between imperfect implementations. It also has a cost, argued in RFC 9413 ("Maintaining Robust Protocols"): if receivers tolerate malformed input, senders never fix their bugs, the deviations become de facto standard, and every new implementation has to copy them. Lenient parsing has also been a source of security bugs when two components interpret the same input differently.

A workable position:

- **Sending**: follow the spec exactly. Validate your own output against the schema.
- **Receiving, unknown additions**: ignore fields you don't recognise (the **tolerant reader**). This is what lets the other side evolve.
- **Receiving, malformed or ambiguous input**: reject it with a clear error. Don't guess.
- **Never "repair" silently**: if you coerce or default something, log and count it.

---

## 8. Contracts and testing

| Technique | What it gives |
|---|---|
| **Specification first** (OpenAPI / AsyncAPI / Protobuf reviewed before code) | Both sides can build in parallel against the same document |
| **Generated clients and servers** | Fewer hand-written mismatches |
| **Schema validation at the boundary** | Bad messages rejected at the edge with a useful error, not deep in business logic |
| **Consumer-driven contract tests** (Pact, Spring Cloud Contract) | Each consumer records what it relies on; the provider verifies those expectations in its CI |
| **Sandbox / test environment** with realistic behaviour and test data | Partners can integrate without touching production |
| **Conformance test suites** | Independent implementations of one standard can prove they interoperate |
| **Mock servers and recorded fixtures** (WireMock, Prism) | Tests that don't depend on the partner being up |
| **Reconciliation** | A periodic comparison of both sides' records catches what tests didn't |

---

## 9. Integrating with systems you don't control

External systems are slower, less available and more surprising than your own.

- **Anticorruption layer**: translate the partner's model into yours in one adapter. Their names and quirks stay out of your domain code (Evans, *Domain-Driven Design*).
- **Canonical data model**, sparingly: with many partners, map each to one internal format instead of every pair to each other. Don't let it become a giant schema that fits nobody.
- **Assume failure**: timeouts, retries with backoff and jitter, circuit breakers ([Fault tolerance](../../runtime-operational/fault-tolerance/README.md)).
- **Idempotency**: the network will deliver duplicates. Send and honour idempotency keys ([Reliability §11](../../runtime-operational/reliability/README.md#11-idempotency)).
- **Respect their limits**: rate limits, `429` and `Retry-After`, pagination, batch sizes.
- **Webhooks in**: verify signatures, respond fast and process asynchronously, deduplicate by event ID, and don't rely on order.
- **Store the raw message** alongside your parsed version, so you can re-process after fixing a mapping bug.
- **Observe every integration**: per-partner success rate, latency, schema-validation failures, and a correlation ID that both sides log ([Observability](../../operations-devops/observability/README.md)).
- **Version awareness**: know which version of their API you use and when it ends.

---

## 10. Identity and trust across systems

Interoperating usually means agreeing *who is calling* and *what they may do*.

| Need | Standard |
|---|---|
| Delegated API access | OAuth 2.0 (and 2.1 drafts) |
| User sign-in across organisations | OpenID Connect; SAML 2.0 in enterprises |
| User and group provisioning | SCIM |
| Service-to-service identity | mutual TLS; signed tokens (JWT); SPIFFE in service meshes |
| Message integrity | HMAC signatures on webhooks; JWS; XML Signature in SOAP |

Use the standards and well-tested libraries. Home-made authentication schemes are both the least interoperable and the least secure choice. Details: [Authentication](../../security/authentication/README.md), [Authorization](../../security/authorization/README.md).

---

## 11. Standards worth knowing

| Area | Standards |
|---|---|
| Web and APIs | HTTP, URI, JSON, OpenAPI, GraphQL, WebSocket |
| Time, place, language, money | RFC 3339 / ISO 8601, IANA time zone database, ISO 3166, BCP 47, ISO 4217, E.164 |
| Text | Unicode, UTF-8 |
| Events and telemetry | CloudEvents, OpenTelemetry (OTLP) |
| Email and calendars | SMTP, IMAP, MIME, iCalendar |
| Payments and banking | ISO 20022, ISO 8583 (card messages) |
| Healthcare | HL7 v2, HL7 FHIR, DICOM (imaging) |
| Commerce and logistics | EDI (X12, EDIFACT), GS1 identifiers (GTIN barcodes) |
| Developer tooling | Language Server Protocol, OCI image and distribution specs, SQL |
| De facto standards | The S3 API, the Kafka protocol, the PostgreSQL and Redis wire protocols |

A **de facto** standard is one product's interface that others copied. It works, but one vendor controls its direction.

---

## 12. Approaches by system type

| System | Typical approach |
|---|---|
| **Public API / developer platform** | HTTP + JSON, OpenAPI, SDKs in the main languages, webhooks with signatures, sandbox, strict backward compatibility, a status page |
| **Internal microservices** | gRPC or HTTP internally, events for decoupling, schema registry, contract tests, a shared glossary per domain |
| **B2B / enterprise integration** | Whatever the partner has: SFTP files, EDI, SOAP, an iPaaS or integration layer; onboarding checklists; reconciliation |
| **Regulated domains** (health, banking) | Mandated standards (FHIR, ISO 20022, open banking APIs), certification or conformance testing, audit trails |
| **IoT and devices** | MQTT or CoAP, small binary payloads, long-lived old firmware, a device gateway that translates to internal formats |
| **Data platforms** | Open table and file formats (Parquet, Iceberg), data contracts between producers and consumers, catalogues |
| **Legacy modernisation** | Anticorruption layers, change data capture out of the old system, strangler-fig routing ([Evolvability §7](../../development-evolution/evolvability/README.md#7-migration-patterns)) |
| **Early-stage product** | Plain REST + JSON and a few well-chosen integrations; don't build a general integration framework yet |

---

## 13. Trade-offs

| Tension | Notes |
|---|---|
| **Standards vs fit** | A standard is a compromise: it covers common cases and may be awkward for yours. Extensions help, but each one reduces interoperability |
| **Interoperability vs performance** | Text formats and generic protocols cost CPU and bytes compared with a custom binary format. Use the efficient one internally, the common one at the boundary |
| **Interoperability vs security** | Every integration is attack surface, and supporting old protocol versions keeps old weaknesses alive |
| **Interoperability vs speed of change** | A published interface is hard to change ([Backward compatibility](../backward-compatibility/README.md)) |
| **Tolerance vs correctness** | Accepting anything keeps partners working today and hides their bugs until later |
| **Openness vs business model** | Easy export and standard APIs make it easier for customers to leave, and easier for them to choose you in the first place |
| **Canonical model vs point-to-point** | A shared model saves mappings with many partners; with two or three it's overhead |

---

## 14. Anti-patterns

- **Integration through a shared database.**
- Field names without units, timestamps without offsets, money as floats.
- A "standard" format with so many custom extensions that no other implementation can read it.
- Inventing a protocol or auth scheme where a standard exists.
- Exposing your internal data model directly as your public API.
- Letting a partner's model and vocabulary spread through your codebase.
- Accepting malformed input silently and guessing what was meant.
- No sandbox: partners test in production.
- Documentation that isn't generated from, or tested against, the real API.
- Polling every few seconds where a webhook or event stream exists.
- No reconciliation between systems that exchange money or stock.
- One enormous canonical schema that every team must change at once.

---

## 15. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Mars Climate Orbiter (1999)** | NASA's investigation found that ground software from one team reported thruster impulse in pound-force seconds, while the navigation software from another expected newton-seconds, as the interface specification required. The trajectory errors accumulated and the spacecraft was lost at Mars arrival. | Syntactic interoperability without semantic agreement. Put units in the interface and test the interface end to end. |
| **Internet email and the web** | SMTP, MIME, HTTP and HTML let software from thousands of independent vendors interoperate for decades, with open specifications and multiple implementations. | Open standards with several implementations outlive any single product. |
| **Language Server Protocol (2016)** | Microsoft published LSP so that an editor and a language's tooling talk through one protocol. Instead of every editor needing a plugin per language, each side implements the protocol once. | A shared protocol turns N × M integrations into N + M. |
| **Browser interoperability** | Through the 2000s, sites were often built for one browser's quirks. Shared specifications and the cross-vendor web-platform-tests suite, and later the yearly Interop project, moved browsers toward the same behaviour. | A specification is not enough; a shared conformance test suite is what makes implementations agree. |
| **The S3 API** | Amazon's object storage API was copied by many other storage products and services, so tools written for S3 work against them. | A de facto standard can emerge from one product's API, and it makes the whole category more replaceable. |
| **SWIFT's move to ISO 20022** | Cross-border payment messaging has been migrating from the older MT formats to ISO 20022 over several years, with a long coexistence period in which both formats were carried and translated. | Changing a format shared by thousands of organisations takes years, translation services and a published timeline. |
| **Health data exchange (HL7 FHIR)** | Health systems historically exchanged data through many local variants of HL7 v2. FHIR defines web-style resources and APIs, and regulators in several countries now require FHIR-based access. | Where the market doesn't produce interoperability, regulation and a modern standard together can. |

---

## 16. Checklist

- [ ] Every external interface has a machine-readable contract (OpenAPI, AsyncAPI, Protobuf, JSON Schema)
- [ ] Standard formats for time, money, countries, languages, phone numbers and text encoding
- [ ] Units and meanings written down; a glossary for business terms
- [ ] Large integers and identifiers sent as strings
- [ ] Inputs validated at the boundary; unknown fields ignored; malformed input rejected with a clear error
- [ ] Each external system wrapped in an adapter / anticorruption layer
- [ ] Timeouts, retries with backoff, idempotency and rate-limit handling on every integration
- [ ] Standard identity protocols (OAuth 2.0, OIDC, mTLS); signed webhooks
- [ ] A sandbox and test data for partners; contract tests in CI
- [ ] Per-integration monitoring and a shared correlation ID
- [ ] Reconciliation for money, stock and other critical data
- [ ] Versions and end-of-life dates of the APIs you consume are tracked

---

## 17. Exercises

1. **Find the ambiguity.** Take an API response from a system you built. List every field whose unit, time zone, precision or meaning a stranger could get wrong. Rewrite it.
2. **Integration style.** A retailer must send you 2 million price updates each night, and you must tell them within seconds when an item sells out. Choose a style for each direction and justify it.
3. **Anticorruption layer.** A shipping partner's API models a "consignment" containing "pieces"; your model has "orders" with "items" that may ship separately. Design the mapping and say where it lives.
4. **Robustness.** A partner sends dates as `2025-4-3`. Do you accept and parse, or reject? What do you do in each case so the problem gets fixed?
5. **N × M.** Five internal services exchange customer data in five slightly different shapes. Compare point-to-point mappings with a canonical model: number of mappings, who owns what, how a new field is added.
6. **Webhook receiver.** Design a receiver that is safe against duplicates, out-of-order events, forged requests and a slow downstream database.
7. **Standards.** For a domain you know (travel, education, finance, health), find the main data-exchange standard. What does it cover well, and what does everyone extend?

---

## 18. Further reading

- Gregor Hohpe, Bobby Woolf, *Enterprise Integration Patterns* (and enterpriseintegrationpatterns.com)
- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 4 (encoding, dataflow between services)
- Eric Evans, *Domain-Driven Design* (anticorruption layer, context mapping)
- Sam Newman, *Building Microservices*, 2nd ed., chapters on communication styles and contracts
- RFC 9413, "Maintaining Robust Protocols" (2023); RFC 1122 §1.2.2 (the robustness principle)
- RFC 3339 (timestamps), RFC 9457 (Problem Details for HTTP APIs), RFC 8259 (JSON, including its notes on number interoperability)
- OpenAPI, AsyncAPI and CloudEvents specifications; Pact documentation
- Zalando RESTful API Guidelines; Microsoft REST API Guidelines; Google API Improvement Proposals (aip.dev)
- NASA, *Mars Climate Orbiter Mishap Investigation Board Phase I Report* (1999)
- Language Server Protocol specification (microsoft.github.io/language-server-protocol); web-platform-tests (wpt.fyi)
- Martin Fowler, "TolerantReader"; "Consumer-Driven Contracts" (Ian Robinson, martinfowler.com)
