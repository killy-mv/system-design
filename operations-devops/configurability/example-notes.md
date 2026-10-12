# Configurability: hands-on practice notes

Planned labs for learning configurability by **making config changes safe: validated, reviewed, staged and revertible**. Theory is in [README.md](README.md).

They need **Docker Compose**, optionally **kind/k3d**, and **HashiCorp Vault** (dev mode) or another secret store.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** feature flags (deployability lab 5), common-mode config fault with staged rollout (fault-tolerance lab 3).

---

## Labs

### 1. Validate and fail fast ⭐
- **Build:** a service loading config from env vars + a YAML file, validated with a schema (Pydantic, Zod, Go struct tags) with types, units and ranges.
- **Break:** recreate the README §2 worked example: `timeout: 3000` read as seconds. Show the thread pool exhaustion under a slow dependency.
- **Fix:** typed durations (`"3s"`), range checks, fail at startup with a clear error message. Log the effective config (minus secrets) at startup.

### 2. Dynamic config with last known good
- **Build:** runtime config (rate limit, timeout, log level) from **Consul KV**, **etcd** or a flagd file, reloaded without restart.
- **Break:** push an invalid value, then stop the config service.
- **Fix:** validate before applying, keep the last valid config, expose `config_version` as a metric and mark changes on a Grafana dashboard.

### 3. Secrets done right
- **Build:** move a DB password from an env file into **Vault** (dev mode) or Kubernetes Secrets via external-secrets. The app reads it at startup (or uses Vault dynamic DB credentials).
- **Do:** rotate the password without downtime. Scan the repo history with **gitleaks** for leaked secrets.

### 4. Staged config rollout
- **Build:** 3 "regions" (containers or namespaces) reading config from Git. A script that applies a config change to region 1, checks health and error rate, waits, then continues, with automatic revert on failure.
- **Break:** a config change that makes region 1 fail. Confirm it stops there and reverts.

---

## Progress

- [ ] 1. Validate and fail fast
- [ ] 2. Dynamic config with last known good
- [ ] 3. Secrets done right
- [ ] 4. Staged config rollout
