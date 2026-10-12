# Adaptability: hands-on practice notes

Planned labs for learning adaptability by **running one unchanged image in very different environments and making it behave well in each**. Theory is in [README.md](README.md).

They need **Docker Compose** (with `--cpus` / `--memory` limits), **k6**, and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)). **toxiproxy** or `tc` is used to make the network bad.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** measure capacity at the SLO (capacity lab 1), validate config and fail fast (configurability lab 1), move the storage (portability lab 5).

---

## Labs

### 1. Small box, big box ⭐
- **Break:** run messy-shop with fixed settings (16 workers, large cache) under `--cpus=1 --memory=512m`. Watch for the OOM kill (`docker inspect` → `OOMKilled`).
- **Fix:** read the cgroup CPU and memory limits at start-up, derive worker count, pool sizes and cache size from them, and log the chosen values. Keep env-var overrides.
- **Check:** the same image is stable at 1 CPU / 512 MB and uses the extra resources at 4 CPU / 4 GB (compare k6 throughput).
- **See:** README §3.

### 2. Optional dependencies
- **Build:** make Redis optional. With it: caching and rate limiting use Redis. Without it: an in-process fallback, and a clear line in the start-up log and the health endpoint about what is degraded.
- **Break:** stop Redis while the app runs. It must keep serving.
- **See:** README §2 (graceful degradation).

### 3. Bad network
- **Do:** put toxiproxy between the app and the fake payment provider: add 2 s latency, then 30% connection resets, then a full outage.
- **Fix:** timeouts, retries with backoff, and an "order accepted, payment pending" path instead of an error page.
- **Check:** behaviour at each level is acceptable and documented.

### 4. Profiles and a small test matrix
- **Build:** three presets (`small`, `standard`, `ha`) that set the dozen relevant options together. Reject contradictory combinations at start-up.
- **Then:** a CI job that runs the smoke tests against `small` without Redis and `standard` with Redis, rather than every combination.
- **See:** README §4.

---

## Progress

- [ ] 1. Small box, big box
- [ ] 2. Optional dependencies
- [ ] 3. Bad network
- [ ] 4. Profiles and a small test matrix
