# Observability: hands-on practice notes

Planned labs for learning observability by **instrumenting a small system, breaking it, and finding the cause using only telemetry**. Theory is in [README.md](README.md).

All labs share one **demo system** and one **observability stack**, built in lab 1 and reused:

```
Demo system:  web/API gateway → order-service → payment-service (calls a fake "psp")
                                     │                  │
                                 Postgres            Kafka/Redpanda → email-worker

Stack:        OpenTelemetry Collector → Prometheus (metrics) · Loki (logs) · Tempo or Jaeger (traces)
              · Pyroscope (profiles) → Grafana
```

Shortcut: the **`grafana/otel-lgtm`** Docker image bundles an OTel Collector, Prometheus, Loki, Tempo, Pyroscope and Grafana in one container. It's perfect for learning.

Every service should have a **fault switch** (env var or HTTP endpoint) to inject latency, errors or bad behaviour, so you can create incidents on demand.

**Suggested order:** 1 → 2 → 3 → 4 → 7 → 6 → 5, then the rest.

**Planned folder layout:**

```
operations-devops/observability/labs/
  demo-system/             ← the services above + docker-compose.yml + fault switches
  stack/                   ← collector config, Grafana dashboards
  01-auto-instrumentation/
    README.md              ← steps + screenshots/notes of what you found
  02-metrics-done-right/
  ...
```

---

## Labs

### 1. Auto-instrumentation in 30 minutes ⭐
- **Build:** the demo system with OpenTelemetry **auto-instrumentation** (Java agent, `opentelemetry-instrument` for Python, `@opentelemetry/auto-instrumentations-node`), exporting OTLP to the stack.
- **See:** traces across all services in Grafana/Jaeger, the service map, and automatic HTTP/DB metrics, without writing instrumentation code.
- **Note what's missing:** business context (order ID, tenant), the Kafka hop (often missing), custom metrics. Labs 2–4 fix these.
- **See:** README §8.

### 2. Metrics done right
- **Build:** add manual metrics: a request **histogram** with buckets around your SLO, counters for business events (orders, payments by provider), gauges for queue depth and pool usage.
- **Make:** a RED dashboard per service, with p50/p95/p99 computed from histograms (`histogram_quantile`) and a business panel.
- **Experiment:** compute "average of per-instance p99" vs the real p99 from merged histograms, and see how wrong the first is.
- **See:** README §3.

### 3. Structured logs + trace correlation ⭐
- **Build:** JSON logs with `trace_id`, `span_id`, `service`, `version` (via OTel log bridge or your logger's trace context integration), shipped to Loki.
- **Make:** Grafana links from a trace span → its logs, and from a log line → its trace.
- **Then:** add one **canonical log line** per request with ~20 fields (README §4.3).
- **See:** README §4, §7.

### 4. Propagation through a queue
- **Break:** with only auto-instrumentation, the trace often ends at the Kafka producer; the email worker starts a new, unrelated trace.
- **Fix:** inject `traceparent` into message headers on produce and extract it on consume (or use span **links** for batch consumers).
- **See:** one trace from the HTTP request all the way to "email sent". README §5.1.

### 5. Cardinality explosion
- **Break:** add a `user_id` label to the request counter and generate traffic for 50,000 fake users.
- **Measure:** active series in Prometheus (`prometheus_tsdb_head_series`), memory use, query speed.
- **Fix:** remove the label; put `user_id` on spans instead; add exemplars to link the latency histogram to traces. Add a collector processor that drops disallowed labels.
- **See:** README §12.1.

### 6. Sampling: head vs tail
- **Build:** set head sampling to 1% and inject a rare error (0.5% of requests). Try to find an example trace of the error.
- **Fix:** switch to **tail sampling** in the OTel Collector (`tail_sampling` processor): keep all errors, all requests > 500 ms, 2% of the rest. Generate span metrics **before** sampling (`spanmetrics` connector) so rates stay exact.
- **Measure:** stored spans per minute before and after.
- **See:** README §5.3, §5.4.

### 7. Find the bug: high-cardinality slicing ⭐
- **Build:** recreate the README §11 incident: a fault that only affects one provider × one country × one version. Add these as attributes on spans/wide events.
- **Practice:** someone else (or a script with a random seed) turns on a hidden fault. Using only telemetry, find it and write a short timeline like §11.
- **Bonus:** load your canonical log lines into **DuckDB** or **ClickHouse** and answer questions with `GROUP BY` over any field.

### 8. Continuous profiling
- **Build:** add **Pyroscope** profiling to one service. Ship a version with a deliberately slow function.
- **See:** compare flame graphs for v1 vs v2 and find the regression. Link from a slow span to the profile at that time.
- **See:** README §6, and performance lab 4 for profiling basics.

### 9. Frontend: RUM and errors
- **Build:** a small web page calling the API. Add the `web-vitals` library and OTel browser SDK (or Sentry's free tier) for page loads, Core Web Vitals and JS errors, with `traceparent` sent on fetch calls.
- **See:** one trace that starts in the **browser** and continues into the backend. Upload source maps and compare a minified vs mapped stack trace.

### 10. eBPF: telemetry without code changes
- **Build:** add a service with **no instrumentation**. Run **Grafana Beyla** (or Pixie on k8s) next to it.
- **See:** RED metrics and spans for that service appear without touching its code. Compare what you get with manual instrumentation.

### 11. Observe a data pipeline
- **Build:** a nightly job loading orders into a reporting table.
- **Instrument:** job duration, success, rows processed, freshness ("newest row is X minutes old"), null rate on key columns.
- **Break:** make the upstream silently send 0 rows. Does anything alert? Add a volume/freshness check that would.

### 12. Survive the observability outage
- **Break:** stop the stack while the demo system runs.
- **Check:** does the app slow down or crash (blocking exporters, unbounded buffers)? Fix with async batch exporters and bounded queues.
- **Discuss:** where would you run the stack in production so it doesn't fail together with the app? (README §9.)

---

## Bonus: cost estimate

Using your lab numbers (spans/request, log bytes/request, series count), estimate monthly telemetry volume at 1,000 and 10,000 RPS. Price it with the published per-GB / per-series prices of one SaaS vendor and compare with self-hosting. Then apply the levers in README §12.2 and estimate again.

---

## Tools

| Tool | Used for |
|---|---|
| Docker Compose | Running the demo system and stack |
| OpenTelemetry SDKs + auto-instrumentation | Generating telemetry |
| OpenTelemetry Collector | Pipeline: batching, sampling, redaction, span metrics |
| `grafana/otel-lgtm` (or Prometheus + Loki + Tempo + Pyroscope + Grafana) | Storage and UI |
| Jaeger | Alternative trace UI |
| Redpanda / Kafka | Queue hop for propagation |
| k6 | Generating traffic |
| DuckDB / ClickHouse | Querying wide events |
| Sentry / OTel browser SDK + web-vitals | Frontend errors and RUM |
| Grafana Beyla | eBPF auto-instrumentation |

## Progress

- [ ] 1. Auto-instrumentation in 30 minutes
- [ ] 2. Metrics done right
- [ ] 3. Structured logs + trace correlation
- [ ] 4. Propagation through a queue
- [ ] 5. Cardinality explosion
- [ ] 6. Sampling: head vs tail
- [ ] 7. Find the bug: high-cardinality slicing
- [ ] 8. Continuous profiling
- [ ] 9. Frontend: RUM and errors
- [ ] 10. eBPF: telemetry without code changes
- [ ] 11. Observe a data pipeline
- [ ] 12. Survive the observability outage
- [ ] Bonus: cost estimate
