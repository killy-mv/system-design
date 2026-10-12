# Efficiency: hands-on practice notes

Planned labs for learning efficiency by **measuring work per resource and cost per unit, then cutting waste**. Theory is in [README.md](README.md).

They need **Docker Compose**, **k6**, and optionally **kind/k3d** for the Kubernetes lab. No cloud account needed: estimate cost with any cloud's public price list.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** CPU profiling (performance lab 4), serialisation and compression (performance lab 9), autoscaling (scalability lab 14).

---

## Labs

### 1. Requests per core ⭐
- **Build:** the same small API in two versions: a naive one (N+1 queries, full objects serialised, no caching) and an optimised one (from your performance labs).
- **Measure:** max RPS at SLO with the container limited to `--cpus=1`, plus memory use. Compute **requests per core** and **MB per 1,000 requests/s** for each.
- **Then:** turn that into "instances needed for 10,000 RPS" and "monthly cost" using a public price list.
- **See:** README §1, unit cost.

### 2. Right-size Kubernetes requests
- **Build:** a kind/k3d cluster running a few services with deliberately oversized `requests` (2 CPU, 2 GiB each).
- **Measure:** actual usage with `kubectl top pods` (metrics-server) under load. Install **VPA** in recommendation mode and read its suggestions.
- **Fix:** lower requests and count how many more pods fit per node. Then lower them too far and watch throttling and OOM kills.
- **See:** README §2 "low bin-packing", §4 trade-offs.

### 3. Bytes per request
- **Build:** an endpoint returning a product list. Measure response size and egress per 1M requests.
- **Fix:** select only needed fields, paginate, compress (gzip/Brotli), cache with ETags. Compute egress saved per month.
- **See:** README §2 "data transfer".

### 4. Observability cost
- **Build:** a service logging every request at debug level, plus a Prometheus metric with a `user_id` label.
- **Measure:** log volume per hour and the number of metric time series after 10,000 distinct users.
- **Fix:** log levels, sampling, removing high-cardinality labels. Measure again.
- **See:** README §3 "Observability".

---

## Progress

- [ ] 1. Requests per core
- [ ] 2. Right-size Kubernetes requests
- [ ] 3. Bytes per request
- [ ] 4. Observability cost
