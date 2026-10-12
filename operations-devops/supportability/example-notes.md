# Supportability: hands-on practice notes

Planned labs for learning supportability by **building the tools support needs, and timing how fast a ticket gets answered**. Theory is in [README.md](README.md).

They reuse the **demo system** from the observability notes (orders, payments, emails).

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** the reference ID (debuggability lab 1), error tracking (debuggability lab 2), soft delete and undo (UI-UX user error protection).

---

## Labs

### 1. Specific errors with reference codes
- **Build:** a consistent API error format (`code`, human message, `reference` = trace ID; for example RFC 9457 "Problem Details") and an error page showing the reference.
- **Check:** every error path in the demo system returns a specific code. Make a short list of codes and their meaning for support.

### 2. Entity timeline ⭐
- **Build:** an `order_events` table (or read from the event stream / outbox) and an admin page showing an order's full timeline, like the README §2 worked example.
- **Practice:** recreate "my order disappeared" with a failed payment webhook. Time how long it takes to answer using only the admin page.

### 3. Safe support actions
- **Build:** admin actions "resend payment link", "refund up to $50" and "unlock account", with role-based permissions, a required reason, limits, and an audit log entry for each.
- **Check:** a support user can't do anything outside those actions; every action is visible in the audit log.

### 4. Read-only impersonation
- **Build:** "view as user": a support user gets a time-limited (15 min), read-only session as the customer, requiring a ticket number. Write actions are blocked; the session is audited and shown in the customer's security log.
- **Check:** try to perform a write while impersonating, and confirm it's blocked and logged.

---

## Progress

- [ ] 1. Specific errors with reference codes
- [ ] 2. Entity timeline
- [ ] 3. Safe support actions
- [ ] 4. Read-only impersonation
