# Replaceability

> **Replaceability** = how easily one component or product can be **swapped for another that does the same job** in the same environment.
> "Our payment provider doubles its prices (or our database vendor is acquired, or a library is abandoned). How long does it take to switch, and what breaks?"

Related: [Vendor independence](../../business-project/vendor-independence/README.md) (the business case and exit plans), [Portability](../portability/README.md) (moving the whole system), [Interoperability](../interoperability/README.md) (standard interfaces make swaps possible), [Modularity](../../development-evolution/modularity/README.md), [Evolvability §7](../../development-evolution/evolvability/README.md#7-migration-patterns) (migration patterns).

---

## 1. Two directions

| Direction | Question | Examples |
|---|---|---|
| **Your product replaces another** | Can a customer switch *to* you without rewriting their side? | MariaDB for MySQL, Valkey for Redis, OpenTofu for Terraform, Podman for the Docker CLI, MinIO and others speaking the S3 API, Redpanda speaking the Kafka protocol |
| **A component inside your system gets replaced** | Can you switch database, queue, payment provider, auth provider, or a library without touching everything? | Changing email provider; moving from a self-run queue to a managed one |

The first is a product strategy: implement the incumbent's interface and import its data. The rest of this page is mostly about the second.

---

## 2. What makes replacement hard

| Obstacle | Why |
|---|---|
| **Leaked interface** | The vendor's types, error codes and concepts appear all over your code, not in one place |
| **Proprietary features** | You use the things only this product has (a special query language, a unique consistency mode) |
| **Behaviour, not just API** | Two products with the "same" API differ in ordering, limits, defaults, error handling and timing. Callers depend on those ([Hyrum's Law](../backward-compatibility/README.md)) |
| **Data gravity** | Terabytes to move, with history, while the system stays up |
| **Identifiers stored elsewhere** | The old provider's customer IDs and tokens are in your database, emails and partners' systems |
| **Operational knowledge** | Runbooks, dashboards, alerts and people's experience are all tied to the old thing |
| **Contracts and compliance** | Notice periods, data deletion, re-certification |

An ORM does not make the database replaceable. SQL dialects, types, locking and default isolation levels differ (PostgreSQL defaults to Read Committed, MySQL/InnoDB to Repeatable Read), and so does everything outside the ORM: migrations, raw queries, operations.

---

## 3. Designing for replacement

- **Ports and adapters**: the application defines the interface it needs in its own words (`PaymentGateway.charge(order)`), and one adapter per provider implements it. Vendor SDK types never cross the port.
- **Prefer standard interfaces** when the choice is close: SQL, the S3 API, the Kafka protocol, AMQP, OpenTelemetry, OIDC, SMTP. Then the replacement already exists.
- **Own your identifiers**: store your own IDs and map them to the provider's; don't use theirs as primary keys.
- **Keep your data exportable**, and test the export.
- **A conformance test suite** for each port, run against every adapter. It defines what "does the same job" means, including the awkward behaviour.
- **Don't abstract everything.** A port that hides every difference also hides every useful feature. Rank dependencies by *likelihood of replacement × cost of replacement* and protect the top few.

| Dependency | Likelihood of change | Cost if tangled | Worth a port? |
|---|---|---|---|
| Payment / email / SMS provider | High (price, outages, new markets) | Medium | Yes |
| Feature-flag or analytics vendor | Medium | Low–medium | Thin wrapper |
| Primary database engine | Low | Very high | Usually no: commit to it and use it fully |
| Programming language, main framework | Very low | Very high | No |

---

## 4. Worked example: replacing a payment provider

messy-shop calls provider A's SDK from four places and stores A's charge IDs on orders. The business wants provider B for lower fees.

```python
# The port: what the shop needs, in the shop's words
class PaymentGateway(Protocol):
    def charge(self, order_id: str, amount: Money, token: str) -> ChargeResult: ...
    def refund(self, payment_id: str, amount: Money) -> RefundResult: ...
    def parse_webhook(self, headers, body) -> PaymentEvent: ...

# One adapter per provider; SDK types stay inside
class ProviderAGateway: ...
class ProviderBGateway: ...
```

```
Step 1  Introduce the port; move the four call sites behind ProviderAGateway. No behaviour change.
Step 2  Write conformance tests against the port: success, decline, timeout, duplicate request
        (idempotency key), partial refund, webhook signature, webhook arriving before the API response.
Step 3  Implement ProviderBGateway until the same tests pass in B's sandbox.
Step 4  Add a payments table with our own payment_id and a (provider, provider_ref) pair.
Step 5  Route 1% of new payments to B behind a flag; compare success rate, latency, fees.
Step 6  Ramp up. Old payments stay with A: refunds route by the stored provider.
Step 7  Remove A's adapter when its last refundable payment has aged out.
```

What the tests found that the API docs didn't suggest: B rounds some currencies differently, reports declines with different codes, and retries webhooks on a different schedule. This is branch by abstraction; each step ships on its own and can be reversed.

---

## 5. Anti-patterns

- Vendor SDK types and error codes in business logic.
- "We use an ORM, so we can change databases any time."
- A generic abstraction over a dependency nobody will ever replace.
- A lowest-common-denominator interface that forbids using what you pay for.
- The provider's IDs as your primary keys.
- Big-bang switch-over on a weekend with no way back.
- An exit plan that has never been tried, even in a sandbox.

---

## 6. Checklist

- [ ] The dependencies most likely to be replaced are named
- [ ] Each of those sits behind a port defined in the application's own terms
- [ ] A conformance suite exists per port and runs against every adapter
- [ ] Your own identifiers are stored; provider references are mapped
- [ ] Data export from each critical provider has been tested
- [ ] Replacements are done incrementally, with routing by flag and a rollback path

---

## 7. Exercises

1. List the external products a system you know depends on. For each: how long would a replacement take, and what's the hardest part?
2. Design a port for sending email that works for SMTP and for two API-based providers. Which provider features did you decide *not* to expose, and why?
3. Two message brokers both promise "at-least-once delivery". List five behaviours that could still differ and break a consumer.
4. Your product is a drop-in replacement for a popular open-source database. What do you need besides wire-protocol compatibility for customers to trust the switch?

---

## 8. Further reading

Alistair Cockburn, "Hexagonal Architecture"; Martin Fowler, "BranchByAbstraction" and Paul Hammant's writing on the same pattern; *Monolith to Microservices* (Sam Newman) for migration patterns; Eric Evans, *Domain-Driven Design* (anticorruption layer); Gregor Hohpe, "Don't get locked up into avoiding lock-in" (martinfowler.com, 2019); Hyrum Wright, hyrumslaw.com; ISO/IEC 25010 (replaceability).
