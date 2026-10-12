# Authorization: hands-on practice notes

Planned labs for learning authorization by **reading other people's data, then making that impossible at more than one layer**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)), extended to be **multi-tenant**: add a `tenant_id` to shops, users, orders and products, and seed **two tenants with two users each**. Its access control is bad on purpose: handlers check only "is logged in", admin pages are hidden in the UI but not protected, and updates bind the whole request body onto the model.

Tools: **Docker Compose**, **PostgreSQL**, **Burp Suite Community** (with the Autorize extension) or **OWASP ZAP**, **Open Policy Agent** or **Cedar**, **OpenFGA** or **SpiceDB**, and **LocalStack** or a cloud sandbox account.

Only attack systems you own. Every lab here runs on your own machine.

**Suggested order:** 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8.

**Related labs elsewhere:** safe support actions and read-only impersonation (supportability labs 3–4), each module owns its data (modularity lab 5), sessions done right (authentication lab 2), audit events (auditability lab 1).

---

## Labs

### 1. IDOR hunt ⭐
- **Break:** as user A, request user B's order, invoice PDF and address by changing IDs. Then do the same across tenants. Write a script that walks sequential order IDs.
- **Fix:** scoped queries plus a policy check per object, `404` for "not yours" (README §4 worked example).
- **Check:** rerun the script. Switch IDs to UUIDs and explain why that alone wouldn't have been a fix.

### 2. Function- and property-level holes
- **Break:** call the hidden admin endpoints directly as a normal user. Send `{"role": "admin"}` and `{"tenant_id": "<other>"}` in a profile update. Find internal fields in API responses.
- **Fix:** default-deny middleware where every route declares a permission, explicit input and output schemas.
- **Check:** add a test that fails when a route has no declared permission.

### 3. Roles and permissions, done properly ⭐
- **Build:** a permission list (`order.read`, `order.refund`, ...), roles as sets of permissions per tenant, and one `can(user, action, resource)` function. Replace every `role == "admin"` in the codebase.
- **Then:** add a new role "finance" without touching any handler. Count the files changed.
- **See:** README §3.

### 4. The permission matrix test ⭐
- **Build:** a table-driven test: every role × every endpoint × (own resource, another user's, another tenant's) → expected status code. Generate the endpoint list from the router.
- **Break:** remove one check from a handler and confirm the matrix catches it.
- **Optional:** replay a browsing session with another user's cookie using Autorize.
- **See:** README §12.

### 5. Row-level security as a safety net ⭐
- **Build:** enable RLS on tenant tables with a `tenant_id` policy; set the tenant per transaction; make the app role a non-owner (README §5 worked example).
- **Break:** delete the `WHERE tenant_id = ...` from a query on purpose. Run the app as the table owner and see the policy bypassed. Use a connection pool without transaction-scoped settings and look for a leak between requests.
- **Measure:** query plans and latency with and without RLS.

### 6. Policy as code
- **Build:** move the refund rule ("support agents may refund up to 50 in their own tenant with MFA; above that needs finance") into OPA/Rego or Cedar. Call it from the service; write policy unit tests and run them in CI.
- **Check:** change the limit by editing only the policy. Log every decision with its inputs.
- **Then:** stop the policy engine and confirm the service fails closed.
- **See:** README §6.

### 7. Sharing with relationships
- **Build:** "shared wishlists" in messy-shop: owner, editors, viewers, and team-level sharing. Model it in OpenFGA or SpiceDB and answer both "may Alice view list 7?" and "which lists may Alice view?".
- **Break:** remove Alice from a team, add an item to the list straight away, and check whether a cached decision still shows it to her.
- **See:** README §11 (the "new enemy" problem).

### 8. Least privilege in the cloud
- **Break:** give the image-upload service a wildcard storage policy and an endpoint that fetches a user-supplied URL. Use it to read the (emulated) instance metadata credentials and then another service's bucket.
- **Fix:** a per-workload role limited to one bucket prefix, URL allow-listing, metadata service protection (IMDSv2 hop limit).
- **Check:** repeat the attack; each defence should stop it independently.
- **See:** README §8, §9.

---

## Progress

- [ ] 1. IDOR hunt
- [ ] 2. Function- and property-level holes
- [ ] 3. Roles and permissions, done properly
- [ ] 4. The permission matrix test
- [ ] 5. Row-level security as a safety net
- [ ] 6. Policy as code
- [ ] 7. Sharing with relationships
- [ ] 8. Least privilege in the cloud
