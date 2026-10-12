# Supportability

> **Supportability** (or **serviceability**) = how easily support and engineering teams can **help users with problems**: understand what happened to a specific customer, fix their issue safely, and communicate clearly.
> "A customer writes 'my order disappeared'. Can support answer in minutes without paging an engineer?"

Related: [Debuggability](../debuggability/README.md) (engineers finding root causes), [Observability](../observability/README.md), [User error protection](../../UI-UX/user-error-protection/README.md) (preventing and recovering from user mistakes).

---

## 1. Why it matters

- Support cost grows with users; each ticket that needs an engineer is expensive and slow.
- Customers judge you by how problems are handled, not only by how rarely they happen.
- Good support tooling **protects engineers' time** and reduces risky manual database fixes.

---

## 2. Design for support

| Practice | Example |
|---|---|
| **Reference IDs users can see** | "Something went wrong. Reference: 4bf92f35" on error pages, in emails and receipts |
| **Clear, specific error messages** with error codes | "Card declined by your bank (code: do_not_honor)" instead of "Error" |
| **Customer timeline / audit history** | Support sees: signed up → upgraded plan → payment failed → retried → succeeded |
| **Support/admin console** | Search by email, order ID, reference ID; see account state, recent errors, plan, flags |
| **Safe support actions** | Resend email, refund, unlock account, retry a stuck job, through buttons with permissions, limits and audit logs, not raw SQL |
| **"View as user" (impersonation)** | Read-only by default, requires a reason, time-limited, audited, and ideally visible to the customer |
| **Per-customer feature flags** | Turn off a broken feature or enable a fix early for one customer |
| **Diagnostics bundles** | "Send diagnostic info" in apps / CLI command for self-hosted software: version, config (no secrets), recent logs |
| **Status page and known issues** | Customers see ongoing incidents before opening tickets |
| **Versions visible** | App version, API version, plan, region shown in support tools |

### Worked example: "my order disappeared"

**Low supportability:** support can't see orders → escalates to engineering → an engineer queries the production DB by hand → finds the order was cancelled by a payment webhook → writes a reply → 2 days, 1 hour of engineer time, plus the risk of a mistaken query on prod.

**High supportability:** support searches the customer's email in the admin console:

```
Order #8812 timeline
14:01  created (card ending 4242)
14:01  payment pending (psp-x)
14:03  webhook: payment failed: insufficient_funds
14:03  order auto-cancelled; email "payment failed" sent (delivered 14:03)
```

Support answers in 3 minutes with the exact reason, and can **resend the payment link** with one audited button. No engineer involved.

---

## 3. Support process

- **Tiers**: L1 (support team, using tools and docs) → L2 (support engineers, deeper investigation) → L3 (product engineers, bugs and code fixes).
- **Escalation** with a template: reference IDs, timestamps, customer impact, steps already taken.
- **Feedback loop**: tag tickets by cause; the top causes become product fixes, better errors or new support tools.
- **Self-service**: help center, in-app guidance, clear error messages reduce tickets in the first place.

---

## 4. Privacy and safety

- **Least privilege**: support sees what it needs, masked where possible (last 4 digits, partial emails).
- **Audit everything** support does, especially impersonation and data changes.
- **Consent and transparency** for viewing customer data, as required by GDPR and contracts.
- Dangerous actions (delete account, large refunds) need limits, confirmation or a second approver.

---

## 5. Anti-patterns

- Generic errors with no reference ID.
- Support has no tools, so every ticket becomes an engineering ticket.
- Engineers fixing customer data with ad-hoc SQL in production.
- Impersonation with full write access and no audit.
- No way to see what happened to a specific order/account over time.

---

## 6. Checklist

- [ ] User-visible reference IDs linked to traces/logs
- [ ] Specific, actionable error messages with codes
- [ ] Admin/support console with search and per-entity timelines
- [ ] Safe, permissioned, audited support actions instead of manual DB edits
- [ ] Impersonation read-only by default, audited, time-limited
- [ ] Status page; ticket causes tracked and fed back to product

---

## 7. Exercises

1. Pick your 5 most likely customer complaints for a product you built. For each, what would support need to see to answer without an engineer?
2. Design the data model for an entity timeline (order, account) that support can read. Where do the events come from?
3. Design a "view as user" feature with privacy safeguards.
4. Rewrite 5 generic error messages from an app you use into specific, supportable ones with reference codes.

---

## 8. Further reading

Intercom and Zendesk blogs on support operations; Stripe API docs (a good example of error codes and request IDs, e.g. `Request-Id` headers); NN/g "Error-Message Guidelines"; *The Effortless Experience* (Dixon, Toman, DeLisi); GDPR guidance on access to personal data by staff.
