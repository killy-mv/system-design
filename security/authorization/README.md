# Authorization

> **Authorization** (authz, access control) = deciding whether an authenticated subject **may perform this action on this resource**, and enforcing that decision on every request.
> "We know it's Alice. Is Alice allowed to refund *this* order?"

**Broken access control** has been the number one category in the OWASP Top 10 since 2021, and broken object-level authorization is number one in the OWASP API Security Top 10.

Related: [Authentication](../authentication/README.md) (who is asking), [Confidentiality](../confidentiality/README.md) and [Integrity](../integrity/README.md) (what authorization protects), [Auditability](../auditability/README.md) (recording decisions), [Supportability](../../operations-devops/supportability/README.md) (safe staff access), [Modularity](../../development-evolution/modularity/README.md) (one place for the rules).

---

## Table of contents

1. [The question authorization answers](#1-the-question-authorization-answers)
2. [Principles](#2-principles)
3. [Access control models](#3-access-control-models)
4. [The three levels of checks](#4-the-three-levels-of-checks)
5. [Where to enforce](#5-where-to-enforce)
6. [Policy as code and authorization services](#6-policy-as-code-and-authorization-services)
7. [Multi-tenancy](#7-multi-tenancy)
8. [Delegation: scopes, tokens and the confused deputy](#8-delegation-scopes-tokens-and-the-confused-deputy)
9. [Infrastructure and cloud IAM](#9-infrastructure-and-cloud-iam)
10. [Privileged and staff access](#10-privileged-and-staff-access)
11. [Performance, caching and revocation](#11-performance-caching-and-revocation)
12. [Testing authorization](#12-testing-authorization)
13. [Approaches by system type](#13-approaches-by-system-type)
14. [Trade-offs](#14-trade-offs)
15. [Anti-patterns](#15-anti-patterns)
16. [Real-world stories](#16-real-world-stories)
17. [Checklist](#17-checklist)
18. [Exercises](#18-exercises)
19. [Further reading](#19-further-reading)

---

## 1. The question authorization answers

Every decision has the same shape:

```
May  <subject>  perform  <action>  on  <resource>  in  <context>?
     alice      refund    order 8812            from the office network, with MFA, at 14:02
```

| Part | Examples |
|---|---|
| **Subject** | User, service, API key, device; plus its roles, groups and attributes |
| **Action** | Read, create, update, delete, approve, export, impersonate |
| **Resource** | A specific order, document, tenant, field or endpoint |
| **Context** | Time, location, device, authentication strength, request risk |

Authorization bugs are rarely about "is the user logged in". They are about the **resource**: the user is logged in, has the right role, and asks for *someone else's* order.

---

## 2. Principles

| Principle | Meaning |
|---|---|
| **Deny by default** | No rule means no access. New endpoints and resources start closed |
| **Least privilege** | Each subject gets the minimum access needed, for the minimum time |
| **Complete mediation** | *Every* request is checked, on the server, every time. Never rely on hidden buttons or unguessable URLs |
| **Separation of duties** | No single person can both request and approve a sensitive action |
| **Defence in depth** | More than one layer checks (gateway, service, database), so one missed check isn't a breach |
| **Centralised, readable rules** | Policies live in one place that can be reviewed and tested, not scattered through `if` statements |
| **Fail closed** | If the check errors or times out, the answer is no |

---

## 3. Access control models

| Model | Rule looks like | Good for | Weakness |
|---|---|---|---|
| **ACL** (access control list) | "Alice: read, write" attached to each resource | Files, simple sharing | Hard to manage and review at scale |
| **RBAC** (role-based) | "Editors can update articles" | Most business applications; easy to explain and audit | **Role explosion** when rules depend on the resource ("editor of project X") |
| **ABAC** (attribute-based) | "Doctors may read records of patients in their department during their shift" | Fine-grained, context-dependent rules | Hard to answer "who can access this?"; policies get complex |
| **ReBAC** (relationship-based) | "Users who are *members* of a *team* that *owns* the *folder* containing the document may view it" | Sharing, hierarchies, collaboration (Google Docs, GitHub) | Needs a relationship store and graph queries |
| **MAC** (mandatory) | Labels and clearances set centrally (secret, top secret) | Military, regulated environments, SELinux | Rigid |

Most real systems combine them: **RBAC inside a tenant**, plus **ownership or relationship** checks per resource, plus a few **attribute** conditions (MFA present, IP range).

Design roles from **permissions**, not the other way round:

```
permission:  order.refund          ← code checks this
role:        support_agent = { order.read, order.refund, customer.read }
assignment:  alice → support_agent in tenant acme
```

Code that checks `user.can("order.refund", order)` survives role changes. Code that checks `user.role == "admin"` has to be found and edited every time a role is added.

---

## 4. The three levels of checks

A request can be wrong at three levels. Each needs its own check.

| Level | Question | OWASP API name | Typical bug |
|---|---|---|---|
| **Function** | May this subject call this operation at all? | Broken Function Level Authorization | A normal user calls `POST /admin/users` because only the menu item was hidden |
| **Object** | May this subject touch *this particular* record? | Broken Object Level Authorization (BOLA, or IDOR) | `GET /orders/8813` returns someone else's order |
| **Property** | May this subject read or write *this field*? | Broken Object Property Level Authorization | The response includes `password_hash`; the request sets `role: "admin"` (mass assignment) |

### Worked example: the missing object check

```python
# Vulnerable: authenticated, but any user can read any order by changing the ID
@app.get("/orders/{order_id}")
def get_order(order_id: int, user = Depends(current_user)):
    return db.query(Order).get(order_id)

# Fixed: scope the query to what the subject may see, then check the action
@app.get("/orders/{order_id}")
def get_order(order_id: int, user = Depends(current_user)):
    order = db.query(Order).filter_by(id=order_id, tenant_id=user.tenant_id).first()
    if order is None or not policy.can(user, "order.read", order):
        raise HTTPException(404)          # same answer for "missing" and "not yours"
    return OrderOut.from_orm(order)       # explicit output schema: no internal fields
```

Three things changed: the query is **scoped** to the tenant, the **policy** is asked about this object, and the response goes through an **allow-list of fields**. Returning `404` instead of `403` avoids confirming that the record exists.

Random IDs (UUIDs) instead of sequential ones make guessing harder, but they are **not** access control: IDs leak through links, logs and other API responses.

---

## 5. Where to enforce

| Layer | Good at | Not enough alone because |
|---|---|---|
| **UI** | Hiding what the user can't do (usability) | Anyone can call the API directly |
| **Edge / API gateway** | Authentication, coarse rules ("only admins reach `/admin/*`"), scopes | It doesn't know which order belongs to whom |
| **Service / domain layer** | Object- and property-level decisions with business context | A single forgotten check in one handler |
| **Data layer** | A safety net under all code paths: row-level security, scoped repositories, views | Complex business rules are awkward in SQL |

Enforce at the service layer, with the data layer as the net underneath.

### Worked example: PostgreSQL row-level security

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

```python
with db.transaction():
    db.execute("SELECT set_config('app.tenant_id', %s, true)", [user.tenant_id])  # true = this transaction only
    rows = db.execute("SELECT * FROM orders")     # only this tenant's rows, even if a WHERE is forgotten
```

The application's database role must not be the table owner or a superuser, or the policy is bypassed. With connection pooling, set the value per transaction so it can't leak to the next request.

---

## 6. Policy as code and authorization services

As a system grows, `if` statements spread across services drift apart. The usual answer separates **deciding** from **enforcing**:

| Component | Job |
|---|---|
| **PEP** (policy enforcement point) | The code in the request path that asks and obeys |
| **PDP** (policy decision point) | Evaluates policy and answers allow or deny |
| **PAP / PIP** (administration / information points) | Where policies are managed, and where attributes and relationships come from |

| Approach | Examples | Fits |
|---|---|---|
| **Library in the application** | Casbin, CASL, Pundit, Django permissions, Spring Security | One application, rules close to the domain |
| **Policy engine** | Open Policy Agent (Rego), Cedar (Amazon Verified Permissions) | Policies shared across services, infrastructure and Kubernetes admission |
| **Relationship store (Zanzibar-style)** | SpiceDB, OpenFGA, Permify | Sharing and hierarchies at scale; "list everything Alice can see" |

A small Cedar policy shows the idea of rules as reviewable, testable text:

```
permit (
  principal in Role::"support_agent",
  action == Action::"refund",
  resource is Order
) when {
  resource.tenant == principal.tenant && resource.amount <= 50 && context.mfa == true
};
```

Benefits: policies are versioned, reviewed and tested like code; decisions are logged in one format; "who can do X?" becomes answerable. Costs: another critical dependency in the request path ([§11](#11-performance-caching-and-revocation)).

---

## 7. Multi-tenancy

In a multi-tenant product, the worst authorization failure is **one customer seeing another's data**.

| Isolation model | How | Notes |
|---|---|---|
| **Shared tables + `tenant_id`** | Every row carries its tenant | Cheapest; depends on every query being scoped. Add row-level security |
| **Schema or database per tenant** | Separate namespaces | Stronger isolation; more operational work |
| **Stack per tenant** | Separate deployments or accounts | Strongest; most expensive. For regulated or very large customers |

Rules of thumb:

- Take the tenant from the **authenticated session**, never from a request parameter or header the client controls.
- Make the scoped path the **only** path: a repository that requires a tenant, RLS underneath.
- Scope **everything**: caches, search indexes, object-storage prefixes, queues, background jobs, exports, logs.
- Test with **two tenants** in every integration test, and assert tenant B never sees tenant A's data.

---

## 8. Delegation: scopes, tokens and the confused deputy

When an application acts **on behalf of** a user, two limits apply at once:

```
effective access = what the user may do  ∩  what the application was granted (scopes)
```

- **Scopes limit the client, not the user.** A token with `orders:read` doesn't mean the user may read *all* orders. The API still has to check the object.
- **Check the audience.** A token minted for service A must be rejected by service B.
- **Don't put fine-grained permissions in long-lived tokens.** They go stale the moment a role changes. Put identity and coarse scopes in the token and look up the rest.
- **Confused deputy**: a service with broad rights is tricked into using them for a caller with fewer. Classic forms are SSRF (the server fetches a URL the attacker chose, such as the cloud metadata endpoint) and a service that calls downstream with its *own* identity instead of the user's.
- **Propagate the user's identity** through service chains (token exchange, RFC 8693) so each service can decide for the real subject, not just for "order-service".

---

## 9. Infrastructure and cloud IAM

The same principles apply to machines, pipelines and engineers.

- **One role per workload** with only the actions and resources it needs. No wildcard `*:*` policies.
- **Short-lived credentials** from workload identity instead of static keys ([Authentication](../authentication/README.md)).
- **Start narrow and widen from evidence**: tools such as AWS IAM Access Analyzer generate policies from actual usage.
- **Guardrails above teams**: organisation-level policies (AWS SCPs, Azure Policy, GCP organisation policies) that nobody in an account can override, such as "no public buckets".
- **Network position is not authorization.** Require credentials even inside the VPC. Protect the instance metadata service (IMDSv2 on AWS) against SSRF.
- **Kubernetes**: namespaced RBAC, no `cluster-admin` for workloads, admission policies (OPA Gatekeeper, Kyverno).
- **Database**: separate roles per service, no shared superuser ([Integrity](../integrity/README.md)).

---

## 10. Privileged and staff access

Administrators and support staff are the highest-value accounts, and many large breaches go through internal tools.

| Practice | Meaning |
|---|---|
| **No standing privilege** | Elevated access is requested, approved, time-limited and expires on its own (just-in-time access) |
| **Break-glass accounts** | For emergencies; sealed, alerting loudly when used, reviewed afterwards |
| **Two-person rule** | A second approver for dangerous actions: production data changes, large refunds, permission grants |
| **Scoped support tools** | Buttons for specific, limited actions instead of database access; read-only impersonation ([Supportability](../../operations-devops/supportability/README.md)) |
| **Access reviews** | Regular review of who has what; removal on role change and departure |
| **Everything audited** | Each privileged action recorded with who, what, why ([Auditability](../auditability/README.md)) |

---

## 11. Performance, caching and revocation

Authorization runs on every request, so it has to be fast, and it has to be *current*.

- **Latency**: a remote decision call adds a network hop. Options: run the engine as a sidecar or library with policies pushed to it, batch checks, cache decisions.
- **Caching vs revocation**: any cache, and any permission baked into a token, keeps working after access is removed. Decide the acceptable delay per action. Seconds may be fine for reading a dashboard, not for a fired employee's admin access.
- **The "new enemy" problem** (from Google's Zanzibar paper): if Alice is removed from a document and then new content is added, a stale permission check must not let Alice see the new content. Zanzibar-style systems solve it with consistency tokens that say "evaluate at least as fresh as this change".
- **Lists are harder than checks**: "may Alice read document 7?" is one lookup. "List the documents Alice may read, page 3, sorted by date" must be pushed into the query (filter by tenant, owner, relationships), not done by loading everything and filtering in memory.
- **Availability**: if the decision service is down, fail closed, and design it to be at least as available as the services that depend on it ([Availability](../../runtime-operational/availability/README.md)).

---

## 12. Testing authorization

Functional tests check that allowed things work. Authorization tests check that **forbidden things fail**, and those are the tests that are usually missing.

- **A permission matrix** as a table-driven test: every role × every endpoint × own/other's/other tenant's resource → expected status.
- **Two users, two tenants** in integration tests. For every endpoint that takes an ID, request user B's ID as user A.
- **Default-deny test**: a test that fails when a new route has no authorization rule declared.
- **Policy unit tests** (`opa test`, Cedar's validator and test tools) run in CI.
- **Response-shape tests**: internal fields never appear; unknown input fields are rejected.
- **Automated scans and manual review**: tools like Burp's Autorize replay requests with another user's session; penetration tests focus on access control.
- **In production**: log and alert on spikes of denied requests and on sequential ID access patterns.

---

## 13. Approaches by system type

| System | Authorization focus |
|---|---|
| **Monolithic web app** | Framework policy layer (policies/guards/voters), RBAC + ownership checks, default-deny middleware |
| **Multi-tenant SaaS** | Tenant scoping everywhere, RLS, per-tenant roles, customer-defined roles later, audit logs customers can read |
| **Collaboration products** (documents, repositories) | ReBAC: sharing, groups, inheritance through folders; a relationship store |
| **Microservices** | Coarse checks at the gateway, fine-grained in each owning service; propagated user identity; shared policy engine |
| **Public APIs** | Scoped keys and OAuth scopes, per-key limits, object checks behind every scope |
| **Internal / admin tools** | SSO groups mapped to roles, just-in-time elevation, two-person approval |
| **Cloud infrastructure** | Per-workload roles, organisation guardrails, policy as code in CI for Terraform |
| **Healthcare / finance** | ABAC with context (treating relationship, purpose), break-glass with review, separation of duties |
| **Data platforms** | Column- and row-level policies, masking by role, purpose-based access |
| **LLM agents and tools** | The agent acts with the *user's* permissions, never broader ones; tool calls are authorized like any other request; retrieved documents filtered by the user's access |

---

## 14. Trade-offs

| Choice | Cost |
|---|---|
| Fine-grained permissions | Precise, but hard to understand, review and explain to users |
| Few broad roles | Simple, but everyone ends up over-privileged |
| Central authorization service | Consistent and auditable; adds latency and a critical dependency |
| Checks in each service | Fast and local; rules drift apart and are hard to audit |
| Permissions inside tokens | No lookup needed; stale until the token expires |
| Aggressive caching | Fast; slow revocation |
| Just-in-time access | Small attack surface; friction during incidents |
| Customer-defined roles | Flexible for enterprises; a much larger test and support surface |
| Database row-level security | Strong net; harder debugging and query planning |

---

## 15. Anti-patterns

- Authentication checked, authorization forgotten: any logged-in user can do anything by changing an ID.
- Security in the UI only: hidden buttons, disabled fields, client-side route guards.
- `if user.role == "admin"` scattered through the code.
- The tenant or user ID taken from the request body, a query parameter or a client-set header.
- Relying on unguessable IDs or URLs as access control.
- Binding request bodies straight onto models (mass assignment); returning whole database rows.
- Allow by default: a new endpoint is open until someone remembers to protect it.
- Treating an OAuth scope as proof that the user may access the object.
- Wildcard cloud policies; one shared role for every service.
- Permanent admin rights "just in case"; never reviewing access.
- Internal tools with full power, no approval and no audit.
- Loading everything and filtering by permission in memory.
- Failing open when the policy check errors.

---

## 16. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **GitHub mass assignment (2012)** | A researcher used a mass-assignment weakness in a Rails form to add his own SSH key to the Rails organisation and push a commit to the project's repository. Rails later changed its defaults to require explicit allow-lists (strong parameters). | Bind only fields you list. Property-level authorization matters. |
| **Facebook "View As" (2018)** | Facebook reported that an interaction of several bugs in the "View As" feature generated access tokens for the *viewed* user, exposing tokens for tens of millions of accounts. | Impersonation-style features are high-risk. A token must only ever be issued for the authenticated subject. |
| **USPS Informed Visibility API (2018)** | As reported by Brian Krebs, an API let any logged-in user query account details of other users, with wildcard search, affecting around 60 million accounts. | Being authenticated isn't being authorized. Check the object on every call. |
| **First American Financial (2019)** | As reported by Brian Krebs and later in regulatory actions, hundreds of millions of mortgage-related documents could be viewed by changing a number in a document URL, without logging in. | Sequential IDs with no access check is the classic IDOR. |
| **Capital One (2019)** | According to court documents and the company's statements, an attacker exploited a misconfigured web application firewall to obtain credentials for its cloud role, which had permission to list and read many storage buckets. Data on about 100 million people was taken. | Least privilege for workload roles. A component that only filters traffic shouldn't be able to read customer data. |
| **Twitter internal tools (2020)** | Twitter reported that attackers social-engineered employees to gain access to internal support tools and used them to take over high-profile accounts. | Limit, approve and audit staff tooling; no standing access to powerful actions. |
| **Optus (2022)** | The Australian regulator's court filings describe a coding error that left an internet-facing API without working access controls for years, exposing records of millions of customers. | Inventory every exposed endpoint, and test that access controls actually reject. |

---

## 17. Checklist

- [ ] Deny by default: every route and operation declares its rule; a test fails if one doesn't
- [ ] Function-, object- and property-level checks on the server for every request
- [ ] Queries scoped by tenant and owner; tenant taken from the session
- [ ] Explicit input and output schemas (no mass assignment, no leaking fields)
- [ ] Code checks permissions, not role names; rules live in one reviewable place
- [ ] A data-layer safety net (RLS or mandatory scoped repositories) for multi-tenant data
- [ ] Tokens validated for audience; scopes treated as a limit on the client, not a grant to the user
- [ ] Per-workload cloud roles with least privilege; metadata service protected; organisation guardrails
- [ ] Staff access just-in-time, approved, time-limited and audited; break-glass procedure tested
- [ ] Revocation delay known and acceptable for each kind of access
- [ ] Negative tests: permission matrix, two users and two tenants, policy unit tests in CI
- [ ] Denied requests logged and monitored; regular access reviews

---

## 18. Exercises

1. **Find the IDOR.** Take an API you built. For every endpoint with an ID in the path or body, write down which check proves the caller may access that object. How many have none?
2. **Model choice.** Choose RBAC, ABAC or ReBAC (or a mix) for: a blog CMS, a hospital record system, a Google-Drive-like product, an internal deploy tool. Justify each.
3. **Role design.** Design permissions and roles for messy-shop: customer, support agent, warehouse staff, finance, admin. Which actions need a second approver?
4. **Tenant leak hunt.** List every place in a multi-tenant app where data from two tenants sits side by side (tables, caches, search, files, queues, logs). For each, what enforces isolation?
5. **Revocation.** An employee is dismissed at 14:00. With 1-hour JWTs that carry roles, and a 5-minute decision cache, until when can they act? Redesign so it's under one minute for admin actions.
6. **Confused deputy.** An image-resizing service fetches any URL the user provides and runs with a cloud role. Describe the attack and three independent defences.

---

## 19. Further reading

- OWASP Top 10 ("Broken Access Control") and OWASP API Security Top 10 (2023)
- OWASP Cheat Sheets: Authorization, Insecure Direct Object Reference Prevention, Mass Assignment, Multi-Tenant Security
- OWASP ASVS (access control chapter)
- NIST SP 800-162, *Guide to Attribute Based Access Control*; the NIST RBAC model (Sandhu, Ferraiolo, Kuhn)
- Google, "Zanzibar: Google's Consistent, Global Authorization System" (2019)
- Saltzer and Schroeder, "The Protection of Information in Computer Systems" (1975): least privilege, complete mediation, fail-safe defaults
- Norm Hardy, "The Confused Deputy" (1988)
- Open Policy Agent, Cedar, SpiceDB and OpenFGA documentation
- PostgreSQL documentation, "Row Security Policies"
- NIST SP 800-207, *Zero Trust Architecture*
- RFC 8693 (OAuth 2.0 Token Exchange); RFC 9700 (OAuth 2.0 Security Best Current Practice)
- AWS IAM documentation, "Security best practices in IAM"
