# Observability

> **Observability** = how well you can understand what's happening **inside** a system from the data it **emits** (logs, metrics, traces, profiles), including for problems you didn't predict.
> "Something is wrong for some users. Can we find out what, where and why, without shipping new code to find out?"

Related: [Monitorability](../monitorability/README.md) (alerting and SLOs built on top of observability), [Debuggability](../debuggability/README.md) (investigating a specific problem), [Maintainability §10](../../development-evolution/maintainability/README.md#10-observability) (short overview), [Performance](../../runtime-operational/performance/README.md) (finding bottlenecks), [Availability §13](../../runtime-operational/availability/README.md#13-operations-monitoring-alerting-incident-response) (lowering MTTR).

---

## Table of contents

1. [Observability vs monitoring](#1-observability-vs-monitoring)
2. [The signals](#2-the-signals)
3. [Metrics](#3-metrics)
4. [Logs](#4-logs)
5. [Distributed tracing](#5-distributed-tracing)
6. [Profiles and other signals](#6-profiles-and-other-signals)
7. [Correlation: connecting the signals](#7-correlation-connecting-the-signals)
8. [OpenTelemetry](#8-opentelemetry)
9. [Architecture and tools](#9-architecture-and-tools)
10. [What to instrument](#10-what-to-instrument)
11. [The debugging workflow](#11-the-debugging-workflow)
12. [Cost and cardinality](#12-cost-and-cardinality)
13. [Approaches by system type](#13-approaches-by-system-type)
14. [Trade-offs](#14-trade-offs)
15. [Anti-patterns](#15-anti-patterns)
16. [Real-world stories](#16-real-world-stories)
17. [Checklist](#17-checklist)
18. [Exercises](#18-exercises)
19. [Further reading](#19-further-reading)

---

## 1. Observability vs monitoring

| | Monitoring | Observability |
|---|---|---|
| Question | "Is the thing I expected to break, broken?" | "What is happening, and why?", for anything |
| Handles | **Known unknowns**: failure modes you predicted (disk full, high error rate) | **Unknown unknowns**: new failure modes nobody predicted |
| Typical tools | Dashboards, threshold alerts, health checks | Ad-hoc queries over rich telemetry, traces, high-cardinality slicing |
| Output | A signal that something is wrong | An explanation |

You need both: monitoring tells you **that** something is wrong (and wakes you up); observability lets you find out **why**.

Modern distributed systems fail in new ways all the time (one tenant, one region, one app version, one combination of feature flags). Pre-built dashboards can't anticipate those. The key property is being able to **slice telemetry by any dimension** after the fact: "errors, but only for users on iOS 18 with app version 5.2 in Vietnam using payment provider X".

Two terms you'll see everywhere:
- **Cardinality**: the number of distinct values a field has. `http_method` = low (~5). `user_id` = high (millions).
- **Dimensionality**: how many fields each event has. Rich events have dozens or hundreds.

High cardinality and high dimensionality are what make unknown-unknowns debuggable, and also what make telemetry expensive (§12).

---

## 2. The signals

| Signal | What it is | Good at | Weak at | Cost driver |
|---|---|---|---|---|
| **Metrics** | Numbers aggregated over time (counts, gauges, histograms) | Trends, alerting, dashboards, cheap long retention | Explaining *why*; high-cardinality detail | Number of time series |
| **Logs** | Timestamped records of discrete events | Detail about a specific event, errors, audits | Aggregation at scale; cost at high volume | Bytes ingested and stored |
| **Traces** | The path of one request across services, as a tree of timed spans | Where time goes, which service failed, dependencies | Aggregates (unless derived); volume | Spans stored (sampling) |
| **Profiles** | Where CPU / memory / locks are spent inside a process | Code-level hot spots | Request-level context | Low with continuous profilers |
| **Events (wide events)** | One rich, structured record per unit of work, with many fields | Unknown-unknowns, arbitrary slicing | Needs a columnar/event store | Events × width |

Plus user-side signals: **RUM** (real user monitoring: page loads, Core Web Vitals, JS errors from real browsers) and **synthetic checks** (scripted probes from outside).

"Three pillars" (logs, metrics, traces) is the classic framing. A more useful view: they're **different views of the same events**, and the value comes from **connecting** them (§7).

---

## 3. Metrics

### 3.1 Metric types

| Type | Meaning | Example |
|---|---|---|
| **Counter** | Only goes up (resets on restart); you look at its *rate* | `http_requests_total`, `errors_total` |
| **Gauge** | Current value, goes up and down | `queue_depth`, `memory_bytes`, `active_connections` |
| **Histogram** | Counts of observations in buckets → percentiles can be computed and **aggregated across instances** | `http_request_duration_seconds` |
| **Summary** | Percentiles computed in the client | Avoid for multi-instance services: **percentiles can't be averaged** |

Use histograms for latency. Choose buckets around your SLO (e.g. `…, 0.1, 0.2, 0.3, 0.5, 1, …` for a 300 ms target), or use **native/exponential histograms** (Prometheus native histograms, OTel exponential histograms) that pick buckets automatically.

### 3.2 What to measure

| Method | For | Signals |
|---|---|---|
| **RED** | Request-driven services | **R**ate, **E**rrors, **D**uration |
| **USE** | Resources (CPU, disk, pools, queues) | **U**tilisation, **S**aturation, **E**rrors |
| **Four golden signals** | Any user-facing system | Latency, traffic, errors, saturation |

Plus **business metrics** (orders/min, sign-ups, payments succeeded). They often catch problems technical metrics miss ("no errors, but orders dropped to zero").

### 3.3 Labels and naming

- Labels (dimensions) let you slice: `http_requests_total{service="checkout", route="/pay", status="500"}`.
- Use **low-cardinality** labels only: method, route *template* (`/users/:id`, not `/users/42`), status class, region, version.
- **Never** put user IDs, emails, request IDs, full URLs or error messages in metric labels. Each unique combination is a new time series (§12).
- Follow naming conventions: unit in the name (`_seconds`, `_bytes`), `_total` for counters, OTel semantic conventions.

### 3.4 Pull vs push

- **Pull** (Prometheus scrapes `/metrics`): simple, the server knows when a target is down. Hard for short-lived jobs.
- **Push** (OTLP, StatsD, Pushgateway for batch jobs): works through firewalls and for ephemeral workloads.

---

## 4. Logs

### 4.1 Structured logging

```json
{"ts":"2026-10-08T14:02:11.482Z","level":"error","service":"checkout","version":"1.42.0",
 "trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7",
 "user_id":"u_8812","tenant":"acme","route":"/pay","msg":"payment provider timeout",
 "provider":"psp-x","duration_ms":3002,"attempt":2}
```

- **JSON (or logfmt)**, not free text: machines must parse, filter and aggregate it.
- Always include: timestamp (UTC), level, service, version, environment, **trace_id/span_id**, request ID.
- Log **context**, not just messages: IDs, durations, counts, the decision taken.
- Consistent field names across services (OTel semantic conventions help).

### 4.2 What to log

| Do log | Don't log |
|---|---|
| Errors with context and stack traces | Passwords, tokens, API keys, full card numbers |
| State changes and important decisions | Personal data you don't need (GDPR); mask or hash it |
| Requests to external systems (at sampled rate) | Every loop iteration / debug noise in prod |
| Security events (logins, permission changes) | Huge payloads (log size + reference instead) |
| Startup config (without secrets), version | The same error 10,000 times a second (rate-limit) |

**Levels:** `ERROR` (needs attention), `WARN` (unexpected but handled), `INFO` (important business/lifecycle events), `DEBUG` (off in prod by default; turn on dynamically for one service/user).

### 4.3 Canonical log lines / wide events

Instead of 20 small log lines per request, emit **one wide line at the end of each request** with everything: route, status, duration, user, tenant, version, feature flags, DB time, cache hits, downstream calls, error. (Stripe calls these *canonical log lines*.)

Benefits: one record answers most questions; easy to aggregate; cheap to sample consistently. This is the core idea behind event-based observability tools (Honeycomb, Scuba).

### 4.4 Log pipeline

```
App (stdout) → Agent (Fluent Bit, Vector, OTel Collector) → Buffer (optional: Kafka)
  → Store (Loki, Elasticsearch/OpenSearch, ClickHouse, cloud logging) → Query / alert
```

- Log to **stdout** in containers; let the platform ship logs.
- Never let logging **block** the app (async, bounded buffers, drop on overload).
- Set **retention tiers**: hot (7–30 days, searchable), cold/archive (months, cheap object storage) for audit/compliance.

---

## 5. Distributed tracing

### 5.1 Concepts

```
Trace 4bf92f…  (one user request)
├─ span: GET /checkout                    api-gateway        320 ms
│  ├─ span: POST /orders                  order-service      290 ms
│  │  ├─ span: SELECT cart                postgres            12 ms
│  │  ├─ span: POST /charge               payment-service    240 ms
│  │  │  └─ span: HTTP POST psp-x.com     (external)         225 ms   ← here
│  │  └─ span: publish OrderCreated       kafka                4 ms
```

- **Trace**: the whole request. **Span**: one operation with start time, duration, status, attributes, events, parent.
- **Context propagation**: the trace ID travels with the request via headers (**W3C Trace Context**: `traceparent`, plus `tracestate` and **baggage**). Every service must forward it, including through **queues** (put it in message headers) and async jobs.
- **Span attributes** carry high-cardinality context (user ID, tenant, order ID) because spans are individual events, not aggregated time series.

### 5.2 What traces reveal

- The **critical path**: which span actually determines total latency.
- **Serial calls** that could run in parallel (a staircase in the waterfall).
- **N+1 patterns**: 40 identical DB spans under one request.
- **Which dependency** failed or timed out, and retries.
- The real **service dependency graph** (service maps), generated from data instead of outdated diagrams.

### 5.3 Sampling

Tracing everything is expensive. Options:

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Head sampling** | Decide at the start (keep 1%), propagate the decision | Simple, cheap | Misses most rare errors and slow requests |
| **Tail sampling** | Buffer complete traces in a collector, decide at the end | Keep **all** errors and slow traces + a % of normal ones | Needs a collector tier with memory; more complex |
| **Rate-limited / adaptive** | Keep N traces/s per service or route | Predictable cost; rare routes still represented | Uneven representation |

Common setup: tail sampling that keeps 100% of errors, 100% of requests slower than the SLO, and 1–10% of everything else. Record the **sample rate** so counts can be re-weighted.

### 5.4 Worked example: trace volume

```
10,000 RPS × 30 spans per request × ~1 KB per span
= 300 MB/s ≈ 26 TB per day
```

Keeping everything is rarely worth it. With tail sampling (all errors + slow + 1% of the rest), and an error/slow rate of 2%:

```
≈ 3% of traces kept → ~0.8 TB/day
```

Cost drops ~30×, and every interesting trace is still there. Metrics (computed **before** sampling, e.g. by the collector's span-metrics connector) still give exact rates and latencies.

---

## 6. Profiles and other signals

- **Continuous profiling** (Pyroscope, Parca, Datadog/Google Cloud Profiler): sample CPU/memory/lock profiles from production all the time at ~1–2% overhead. Answers "which function got slower in v1.42?". Now an OpenTelemetry signal too.
- **eBPF-based telemetry** (Grafana Beyla, Pixie, Cilium Hubble): observe network calls, latency and syscalls from the kernel **without changing code**. Great for uninstrumented services and network-level visibility.
- **RUM**: browser/mobile SDKs record page loads, Core Web Vitals, JS errors, user sessions.
- **Synthetic monitoring**: scripted checks from many locations (black-box view, works even with zero traffic).
- **Error tracking** (Sentry, Rollbar, Bugsnag): groups exceptions, with stack traces, release and breadcrumbs.
- **Audit logs**: who did what, when (security and compliance; different retention and integrity needs).

---

## 7. Correlation: connecting the signals

The power comes from jumping between signals with one click:

```
Alert (metric) ──exemplar──► Trace ──trace_id──► Logs of every service in that request
      │                        │
      └── deploy marker        └── span → profile of that service at that time
```

How to make it work:
- **Same trace ID everywhere**: in spans, in every log line, in error reports, in the response header (`traceparent` or `X-Request-Id`) so support can quote it.
- **Exemplars**: attach a sample trace ID to a metric data point ("this p99 bucket was hit by trace 4bf92f…").
- **Same resource attributes on all signals**: `service.name`, `service.version`, `deployment.environment`, `cloud.region`, `k8s.pod.name`.
- **Deploy and feature-flag change markers** on dashboards. Most incidents start with a change.

---

## 8. OpenTelemetry

**OpenTelemetry (OTel)** is the vendor-neutral standard for generating and shipping telemetry (a CNCF project, merging the earlier OpenTracing and OpenCensus).

| Part | Role |
|---|---|
| **API** | What your code calls (create span, record metric) |
| **SDK** | Implementation in your process: sampling, batching, exporting |
| **Auto-instrumentation** | Agents/libraries that instrument HTTP, gRPC, DB clients, queues without code changes (Java agent, Python/Node/.NET auto-instrumentation, eBPF for Go) |
| **OTLP** | The wire protocol for logs, metrics, traces, profiles |
| **Collector** | A pipeline service: receive → process (batch, filter, redact, sample, enrich) → export to any backend |
| **Semantic conventions** | Standard attribute names (`http.request.method`, `db.system`, `service.name`) |

Why it matters: instrument **once**, send to any backend, switch vendors without touching code. Start with auto-instrumentation, then add **manual spans and attributes** for business context (order ID, tenant, plan, feature flags).

---

## 9. Architecture and tools

```
Services (OTel SDK / auto-instrumentation)    Infra (node exporter, kube-state-metrics, eBPF)
          │ OTLP                                          │
          ▼                                               ▼
   OTel Collector (agent per node) ──► OTel Collector (gateway: tail sampling, redaction, routing)
                                                │
          ┌───────────────┬─────────────────────┼─────────────────┬──────────────┐
          ▼               ▼                     ▼                 ▼              ▼
       Metrics          Logs                 Traces           Profiles     Long-term archive
   (Prometheus/Mimir) (Loki/OpenSearch)  (Tempo/Jaeger)     (Pyroscope)    (object storage)
          └───────────────┴──────── Grafana / query UI / alerting ┘
```

| Category | Open source | Commercial / managed |
|---|---|---|
| Metrics | Prometheus, Thanos, Mimir, VictoriaMetrics | Datadog, Grafana Cloud, New Relic, cloud-native (CloudWatch, Cloud Monitoring, Azure Monitor) |
| Logs | Loki, Elasticsearch/OpenSearch (ELK), ClickHouse | Datadog, Splunk, Elastic Cloud, cloud logging |
| Traces | Jaeger, Tempo, Zipkin | Honeycomb, Datadog APM, Lightstep/ServiceNow, Dynatrace |
| All-in-one | SigNoz, HyperDX, Grafana LGTM stack, OpenObserve | Datadog, New Relic, Dynatrace, Honeycomb |
| Errors | Sentry (self-hosted) | Sentry, Rollbar, Bugsnag |
| Profiling | Pyroscope, Parca | Datadog, Polar Signals, cloud profilers |

**Build vs buy:** SaaS is fast to start and has great UX, but cost scales with volume and can surprise you. Self-hosting costs engineer time and needs its own reliability. Either way, **OTel at the edges keeps you portable**.

**Don't let observability depend on what it observes**: run it in a separate failure domain (other cluster/account/region, or a SaaS), or you lose visibility exactly when you need it.

---

## 10. What to instrument

**At every boundary:**
- Incoming requests (HTTP/gRPC handlers, consumers): RED metrics + a span + one wide log line.
- Outgoing calls (HTTP clients, DB, cache, queues, third-party APIs): latency, errors, retries, timeouts.
- Queues: produce/consume rates, **lag**, age of oldest message, DLQ size.
- Resource pools: utilisation and wait time of thread pools, connection pools, semaphores.

**Context to attach (as span attributes / wide event fields, not metric labels):**
- user ID, tenant/account, plan tier, region, device/app version
- feature flags evaluated, experiment variants
- build version, commit SHA, deployment ID
- business IDs (order ID, payment ID) for support and audits

**Business and product signals:** orders, sign-ups, payments by provider, conversion steps. Alert on business drops too.

**Build info:** expose `service.version` and commit SHA on all telemetry and on a `/version` or `/healthz` endpoint.

---

## 11. The debugging workflow

### Worked example: from alert to fix in 12 minutes

```
14:05  ALERT: checkout SLO burning error budget 14× too fast (metric)
14:06  RED dashboard: /pay 5xx rate 4% (normal 0.05%), started 14:02
       Deploy marker: checkout v1.42.0 rolled out at 14:00              ← suspicious
14:07  Slice errors by attributes (wide events / trace search):
         by version   → 100% of errors on v1.42.0
         by provider  → 100% on psp-x (other providers fine)
         by country   → only VN, TH (the regions routed to psp-x)
14:08  Open an exemplar trace: POST psp-x.com span fails after 3 s: TLS handshake error
14:09  Logs for that trace_id in payment-service:
         "x509: certificate signed by unknown authority"
14:10  Diff of v1.42.0: base image updated → CA bundle missing psp-x's intermediate cert
14:12  Roll back to v1.41.3 → error rate back to normal. Fix forward later with the right CA bundle.
```

What made it fast:
- An **SLO alert** on a user-facing symptom, not on CPU.
- **Deploy markers** on the dashboard.
- **High-cardinality slicing** (version × provider × country) to narrow down in one minute. A dashboard with pre-built panels wouldn't have had "provider × country".
- **Trace ↔ log correlation** to jump from "where" to "why".
- **Mitigate first** (rollback), then fix.

This is the core loop: **start from the symptom → slice by dimensions until the difference between good and bad requests is obvious → drill into examples → confirm the cause**.

---

## 12. Cost and cardinality

Observability bills can rival compute bills. Main cost drivers and controls:

### 12.1 Worked example: cardinality explosion

```
http_requests_total{method, route, status, instance}
  5 methods × 50 routes × 10 statuses × 20 instances = 50,000 series   (fine)

Someone adds a user_id label (200,000 active users):
  up to 50,000 × 200,000 = 10 billion possible series                 (not fine)
```

In practice only combinations that occur get created, but even a few million series makes Prometheus run out of memory (each active series costs memory) or makes a SaaS bill jump, since many vendors bill per custom metric series.

Fix: keep `user_id` **out of metric labels**. Put it on **spans and wide events**, where high cardinality is cheap, and use exemplars to link metrics to examples.

### 12.2 Controls

| Lever | How |
|---|---|
| Cardinality | Label allowlists; route templates; drop/aggregate labels in the collector; alert on series count |
| Sampling | Tail sampling for traces; sample successful-request logs; keep 100% of errors |
| Log volume | Levels; no debug in prod by default; deduplicate/rate-limit repeated errors |
| Retention | Short hot retention, cheap cold archive; downsample old metrics |
| Aggregation | Pre-aggregate at the collector (span metrics, log-to-metric) |
| Ownership | Cost per team/service visible; budgets and alerts |

---

## 13. Approaches by system type

| System | What to focus on |
|---|---|
| **Monolith / small team** | Structured logs with request IDs, RED metrics, error tracking (Sentry), uptime checks. A single all-in-one tool. Add tracing when you split services. |
| **Microservices** | Distributed tracing is essential; propagation through every hop and queue; service maps; consistent resource attributes; per-service SLOs |
| **Frontend / web** | RUM (Core Web Vitals, JS errors, slow resources), source maps for stack traces, session context; link browser spans to backend traces |
| **Mobile apps** | Crash reporting (Crashlytics, Sentry), app start time, network error rates by app version/OS/device; telemetry batched and sent when online |
| **Data pipelines / batch** | Freshness, volume, schema changes, null rates, job duration and success (data observability: dbt tests, Great Expectations, Monte Carlo, Elementary); lineage |
| **Streaming / queues** | Consumer lag, age of oldest message, throughput per partition, rebalances, DLQ size; trace context in message headers |
| **Databases** | Slow query log, `pg_stat_statements`, lock waits, replication lag, connection count, cache hit ratio |
| **Kubernetes** | Node/pod resource metrics, restarts, OOM kills, pending pods, events; kube-state-metrics; eBPF for network |
| **Serverless** | Cold starts, duration, throttles, concurrency; vendor tracing (X-Ray) or OTel layers; cost per invocation |
| **ML / LLM apps** | Model latency, token usage and cost per request, prompt/response traces (with PII care), evaluation scores, drift; OTel GenAI semantic conventions |
| **Security** | Audit logs, auth events, anomaly detection; longer retention, tamper-proof storage (SIEM) |

**By maturity:**
1. Logs and basic uptime checks.
2. RED/USE metrics, dashboards, structured logs with request IDs, error tracking.
3. Distributed tracing, SLOs and burn-rate alerts, deploy markers, trace-log correlation.
4. Wide events with high-cardinality querying, continuous profiling, tail sampling, cost governance, observability-driven development (instrument while writing code, check telemetry after every deploy).

---

## 14. Trade-offs

| Choice | Cost |
|---|---|
| More telemetry | Money (ingest, storage), performance overhead, noise |
| High-cardinality metrics | Memory, cost explosion; prefer traces/events |
| Head sampling | Cheap, but misses rare errors |
| Tail sampling | Complex collector tier, memory, latency before export |
| SaaS | Speed and UX vs vendor lock-in and volume-based bills |
| Self-hosted | Control and cost at scale vs operational burden |
| Rich context in logs | Debuggability vs privacy/compliance risk (PII) |
| Long retention | Historical analysis and compliance vs storage cost |
| Auto-instrumentation | Fast coverage vs generic spans with little business context |

---

## 15. Anti-patterns

- Logs as unstructured text with no request/trace ID.
- Averages instead of percentiles; percentiles averaged across instances.
- User IDs, URLs or error messages as metric labels.
- Hundreds of dashboards nobody looks at, and none for the question you have during an incident.
- Alerting on causes (CPU 80%) instead of symptoms (users failing).
- Trace context lost at queues and async boundaries, so traces end at the first hop.
- Observability stack in the same cluster/region as the app, so it dies with it.
- Logging secrets or personal data.
- Debug logging left on in production; nobody owns the bill.
- Instrumentation added only after an incident, every time.

---

## 16. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Google Dapper (2010 paper)** | Google described its always-on distributed tracing system, which used very low sampling rates (as low as 1 in 1,000+) and still found most performance problems. It inspired Zipkin (Twitter, 2012), Jaeger (Uber) and eventually OpenTelemetry. | Tracing at scale is possible with sampling and propagation built into shared libraries. |
| **Facebook Scuba (2013 paper) → Honeycomb** | Facebook used Scuba, an in-memory store of wide events queried interactively by any dimension, to debug production. Engineers who used it at Parse/Facebook later founded Honeycomb and popularised the term "observability" for software. | Wide events + high-cardinality slicing solve unknown-unknowns that dashboards can't. |
| **Stripe canonical log lines (2019 blog)** | Stripe described emitting one dense, structured log line per request with all key fields, making most production questions answerable with simple queries. | You don't need a fancy tool to start: one wide log line per request is a big step. |
| **Uber: Jaeger and M3** | With thousands of microservices, Uber built Jaeger (tracing) and M3 (a metrics platform for billions of series) and open-sourced both. | At high scale, telemetry is a big data problem of its own. |
| **Coinbase's Datadog bill (reported 2023)** | Coinbase was reported to have run up an observability bill of about **$65M** in a year, after rapid growth in 2021. | Telemetry cost needs governance (sampling, cardinality, retention, ownership) from early on. |
| **AWS S3 outage (Feb 2017)** | The AWS status dashboard depended on S3, so it couldn't show that S3 was down. | Observability and status tooling must not depend on the system they observe. |
| **Facebook outage (Oct 2021)** | When the backbone went down, internal tools and dashboards went down too, slowing diagnosis and recovery. | Keep out-of-band access and observability for the worst case. |

---

## 17. Checklist

**Signals**
- [ ] RED metrics for every service; USE metrics for key resources; business metrics
- [ ] Latency as histograms; percentiles computed from aggregated histograms
- [ ] Structured logs with trace ID, request ID, service, version
- [ ] Distributed tracing with propagation through HTTP, gRPC **and** queues
- [ ] Error tracking with releases and source maps; RUM for frontends

**Correlation and context**
- [ ] Same `service.name`, `service.version`, environment and region on all signals
- [ ] Logs ↔ traces ↔ metrics (exemplars) linked
- [ ] Deploy and feature-flag markers on dashboards
- [ ] High-cardinality context (user, tenant, version, flags) on spans/events, not metric labels

**Operations**
- [ ] OpenTelemetry for instrumentation; collector pipeline with redaction
- [ ] Sampling strategy keeps all errors and slow requests
- [ ] Retention tiers; cardinality and cost monitored per team
- [ ] Observability stack in a separate failure domain
- [ ] No secrets or unnecessary personal data in telemetry

---

## 18. Exercises

1. **Signal choice.** For each question, which signal answers it best: "Is error rate rising?", "Why was this one checkout slow?", "Which function got slower in v1.42?", "Which users were affected by yesterday's incident?", "Is the site up from Asia?"
2. **Cardinality.** Count the series for `http_request_duration_seconds` (a histogram with 12 buckets + sum + count) with labels method (4), route (80), status class (5), region (3), version (2). Is it OK? What if someone adds `customer_id` (5,000 customers)?
3. **Design a canonical log line** for a checkout request: list 25 fields you'd include and why. Which of them would *also* be metric labels?
4. **Propagation.** Draw a request that goes API → Kafka → worker → email service. Where must trace context be injected and extracted? What does the trace look like if one step forgets?
5. **Sampling policy.** 5,000 RPS, 20 spans/request, budget of 100 GB/day of traces at ~1 KB/span. Design a tail-sampling policy that fits and keeps the interesting traces.
6. **Debug walkthrough.** Rewrite the §11 example for an incident in a system you know. Which dimensions would you slice by? Which of them do you actually record today?
7. **Instrument a gap.** Pick a service you built. List every boundary from §10 and mark which ones are instrumented. Fix the most important missing one.
8. **Read** the Dapper paper (sections 1–4) or Stripe's canonical log lines post. What design choices made it work at scale?

---

## 19. Further reading

- *Observability Engineering* (Charity Majors, Liz Fong-Jones, George Miranda): the core book on modern observability
- *Distributed Tracing in Practice* (Parker, Spoonhower, Mace, Sigelman, Isaacs)
- Google SRE book ch. 6 "Monitoring Distributed Systems" (four golden signals) — free at sre.google
- "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" (Google, 2010)
- "Scuba: Diving into Data at Facebook" (2013)
- Stripe blog: "Fast and flexible observability with canonical log lines" (Brandur Leach)
- OpenTelemetry documentation (opentelemetry.io): concepts, collector, semantic conventions
- Brendan Gregg: USE method, flame graphs, eBPF tools
- Tom Wilkie: "The RED Method"
- Honeycomb blog and Charity Majors's writing on observability vs monitoring
