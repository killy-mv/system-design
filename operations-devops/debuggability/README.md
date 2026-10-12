# Debuggability

> **Debuggability** = how quickly and safely engineers can find the **cause of a specific problem**, especially in production, where you can't just attach a debugger and pause.
> "A customer says 'payment failed at 14:03'. How long until we know exactly why?"

Related: [Observability](../observability/README.md) (the telemetry you debug with), [Supportability](../supportability/README.md) (getting the right information from users), [Maintainability](../../development-evolution/maintainability/README.md) (readable code is debuggable code).

---

## 1. Why production debugging is different

- You usually **can't reproduce** it locally: different data, traffic, timing, config, versions.
- You **can't pause** the process (it's serving other users) and often can't SSH in.
- The bug may involve **several services** and only one request in a million.
- What you'll have is what the system **recorded at the time**, plus what you can safely inspect now.

So debuggability is mostly designed in **before** the bug happens.

---

## 2. Design for debuggability

| Practice | Why it helps |
|---|---|
| **Request / trace ID everywhere** (logs, traces, error responses, support tools) | Find everything about one request in seconds |
| **Rich error messages** with context (what, which IDs, which input, which dependency) | "Error 500" vs "psp-x timeout after 3s for payment p_123, attempt 2" |
| **Version + config on every signal** | "Only v1.42 fails" is often the whole answer |
| **Error tracking** with stack traces, releases, breadcrumbs (Sentry, etc.) | Groups errors, shows first/last seen, which release introduced them |
| **Dynamic log levels** (per service, per user, per request via a header or flag) | Turn on debug detail for one customer without flooding logs |
| **Debug endpoints** (internal only): `/healthz`, `/version`, `/debug/pprof` (Go), JMX/actuator, cache stats | Inspect live state safely |
| **Deterministic, explicit state** (state machines, idempotent operations, event logs) | You can see the exact sequence of states and replay it |
| **Audit / event history** of business entities | "What happened to order 123?" has a clear answer |
| **Fail fast with clear errors** instead of silently continuing | The error appears where the bug is, not 3 services later |

---

## 3. Techniques when investigating

- **Start from the symptom and slice** (by version, region, tenant, endpoint) until good and bad requests differ in one clear way ([Observability §11](../observability/README.md#11-the-debugging-workflow)).
- **Find a single example** and follow its trace and logs end to end.
- **Diff**: what changed? Deploys, config, flags, dependency versions, traffic patterns, data.
- **Reproduce** with captured inputs: replay the request (with PII removed) in staging or a local test.
- **Snapshot debuggers / non-breaking breakpoints** (Lightrun, Rookout, Cloud debuggers): capture variables at a code line in production without stopping it.
- **Core dumps / heap dumps / profiles** for crashes, leaks and CPU problems.
- **Traffic shadowing** to send real traffic to a debug build without affecting users.
- **Record and replay** (rr for C/C++, deterministic simulation) for nasty concurrency bugs.

### Worked example: "payment failed at 14:03"

**Without debuggability:** support forwards "payment failed around 2pm". An engineer greps logs on 12 servers for the customer's email (not logged), finds nothing, asks the customer for screenshots, waits a day. Total time: 2 days.

**With debuggability:**

```
1. The error page showed: "Payment failed. Reference: 4bf92f35"   (the trace ID)
2. Support pastes 4bf92f35 into the trace search                  → 30 seconds
3. Trace: payment-service → psp-x returned 402 "card_declined: do_not_honor"
4. Linked logs: our retry logic retried a decline 3 times (it shouldn't retry declines)
5. Error tracking: this exception group started with release 1.42, 214 occurrences
```

Total time: about 10 minutes to the root cause, plus a clear fix (don't retry non-retryable errors). The difference is entirely things designed in **before** the bug: a visible reference ID, traces, linked logs and release tracking.

---

## 4. Safety

- Debug tools in production must be **access-controlled and audited** (debug endpoints are a classic security hole).
- No PII or secrets in debug output; redact by default.
- Read-only by default; anything that changes state (e.g. "fix this order") goes through reviewed tooling ([Supportability](../supportability/README.md)).

---

## 5. Anti-patterns

- "Something went wrong" with no ID or context, to users and in logs.
- Swallowing exceptions or logging them without stack traces.
- Needing SSH and `grep` across servers to investigate.
- Debug endpoints exposed to the internet.
- Can't tell which version or config a request ran with.
- Bugs only investigated by adding logs and redeploying, again and again.

---

## 6. Checklist

- [ ] Trace/request ID in all logs, traces, error responses and support tools
- [ ] Error tracking with releases, stack traces and source maps
- [ ] Version, config version and flags attached to telemetry
- [ ] Dynamic log levels per service / user / request
- [ ] Safe, authenticated debug endpoints (version, health, profiling)
- [ ] Event/audit history for important business entities
- [ ] Ability to replay or reproduce a request outside production

---

## 7. Exercises

1. Take a recent bug you fixed. Write down how long each step took (finding the request, finding the cause, reproducing). What single change would have halved it?
2. Design the error response format for an API so that support and engineers can find the request instantly, without leaking internals.
3. Design a way to enable debug logging for **one** customer's requests for 30 minutes, safely.
4. Which debug endpoints would you expose for a service, and how would you protect them?

---

## 8. Further reading

*Debugging* (David Agans, the nine rules); *Observability Engineering* ch. on debugging from first principles; Julia Evans's zines and blog on debugging; Go `net/http/pprof` docs; Sentry docs on releases and source maps; *Why Programs Fail* (Andreas Zeller).
