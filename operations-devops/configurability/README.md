# Configurability

> **Configurability** = the system's behaviour can be **changed without changing code**: per environment, per customer, or at runtime, **safely**.
> "Can we raise a timeout, turn off a feature or change a limit in minutes, without a rebuild, and without breaking production?"

Related: [Deployability](../deployability/README.md) (config changes are deploys), [Manageability](../manageability/README.md) (managing many environments), [Maintainability §11.3](../../development-evolution/maintainability/README.md#113-configuration-management) (short overview).

---

## 1. Kinds of configuration

| Kind | Changes | Examples | Where it lives |
|---|---|---|---|
| **Build-time** | Rarely, needs a rebuild | Compiler flags, bundled defaults | Code / build files |
| **Deploy-time** | Per environment, at deploy | DB host, region, replica count, log level | Env vars, config files, Helm values, IaC |
| **Runtime / dynamic** | Any time, no deploy | Feature flags, rate limits, timeouts, kill switches | Flag service, config service (Consul, etcd, AppConfig) |
| **Secrets** | Rotated regularly | Passwords, API keys, certificates | Secret manager (Vault, AWS/GCP/Azure secret managers, Kubernetes Secrets + external-secrets) |
| **Tenant / user settings** | By customers or admins | Plan limits, branding, SSO settings | Application database |

**12-factor rule**: keep deploy-time config **out of the code** (in the environment), so the same artifact runs everywhere.

---

## 2. Principles

- **Safe defaults**: the system works sensibly with no config; missing optional config never crashes it.
- **Validate on load**: schema-check every value (types, ranges, units, required fields). **Fail fast at startup** on invalid config, rather than at 3am when that code path runs.
- **Explicit units**: `timeout_ms: 300` or `timeout: "300ms"`, never a bare `timeout: 300`.
- **Config is code**: in version control, reviewed in PRs, with history and an easy revert.
- **Roll out config gradually**: one instance / one region first, like a code deploy. A bad config pushed everywhere at once is one of the most common causes of global outages.
- **Last known good**: if the config service is unreachable or the new config is invalid, keep running on the last valid config.
- **Observable**: log the effective config at startup (without secrets), expose the config version, mark config changes on dashboards.
- **Secrets never** in code, images, logs or plain environment dumps; rotate them; least-privilege access.

### Worked example: a unit mistake

A team changes a payment client's timeout from `30` to `3000` "to be safe", intending milliseconds (3 s). The code reads the value as **seconds**.

```
Payment provider slows down → requests now wait up to 3000 s (50 minutes) instead of 30 s
→ threads and connections pile up waiting → the thread pool is exhausted
→ checkout stops completely, not just for slow payments
```

What would have prevented it:
- A typed value with units (`3s`) and a **range check** (`1ms ≤ timeout ≤ 30s`) at load time.
- Rolling the config to **one instance first** with an automated health check.
- A **review** that showed the effective value in human terms ("50 minutes").

---

## 3. How much configurability?

Every config option is a **code path that must be tested** and a decision someone must understand.

- Too little: every small change needs a deploy; operators can't react during incidents (no kill switches, no tunable limits).
- Too much: thousands of options nobody understands, untested combinations, "configuration as a programming language" (the *configuration complexity clock*: config grows rules, then logic, then becomes a bad DSL).

Good targets for configurability: **operational levers** (timeouts, limits, pool sizes, log levels, kill switches), **environment differences**, and **feature rollout**. Business logic usually belongs in code.

---

## 4. Anti-patterns

- Hard-coded hosts, credentials or limits.
- Different artifacts per environment instead of different config.
- Config with no validation, no units, no review.
- Global, instant config pushes.
- Secrets in Git, Docker images or logs.
- Dozens of undocumented options; config nobody dares to change.
- Config drift: production changed by hand and no longer matches the repo.

---

## 5. Checklist

- [ ] Deploy-time config in the environment, same artifact everywhere
- [ ] Every config value validated (type, range, units) at load; fail fast
- [ ] Config in version control with review; changes logged and visible on dashboards
- [ ] Runtime config rolled out gradually, with last-known-good fallback
- [ ] Secrets in a secret manager, rotated, never logged
- [ ] Kill switches and tunable limits for incident response
- [ ] Options documented with defaults; unused options removed

---

## 6. Exercises

1. List every config value of a service you built. Classify each as build-time, deploy-time, runtime, secret or tenant setting. Is any in the wrong place?
2. Write a validation schema (JSON Schema, Pydantic, Zod) for that config with units and ranges.
3. Design the rollout process for a change to a global rate limit used by every service.
4. Which 5 runtime levers would you want during an incident on an e-commerce site?

---

## 7. Further reading

The Twelve-Factor App, "III. Config" (12factor.net); Google SRE workbook ch. 14 "Configuration Design and Best Practices" and ch. 15 "Configuration Specifics"; Mike Hadlow, "The Configuration Complexity Clock"; Facebook's "Holistic Configuration Management at Facebook" (SOSP 2015); HashiCorp Vault / external-secrets documentation.
