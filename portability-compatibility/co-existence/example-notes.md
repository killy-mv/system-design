# Co-existence: hands-on practice notes

Planned labs for learning co-existence by **putting workloads next to each other, letting one misbehave, and adding isolation until the other stops noticing**. Theory is in [README.md](README.md).

They need **Docker Compose**, a local Kubernetes (**kind** or **k3d**), **k6**, **PostgreSQL**, and **stress-ng** (or any CPU/memory burner) as the noisy neighbour. The victim is the **demo system** from the [observability notes](../../operations-devops/observability/example-notes.md), so latency is already on a dashboard.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** hit hard limits on purpose (capacity lab 3), overload behaviour (capacity lab 4), connection pool exhaustion (scalability lab 4), isolate a misbehaving plugin (extensibility lab 3).

---

## Labs

### 1. Noisy neighbour on one node ⭐
- **Break:** on a single-node kind cluster, run order-service under steady k6 load and start a stress-ng pod with no requests or limits. Record p99 and any evictions.
- **Fix:** add requests to order-service, requests and limits to the noisy pod, and a low priority class. Repeat and compare.
- **Check:** `kubectl describe pod` shows the QoS class of each pod; explain why each got it.
- **See:** README §4.

### 2. Quotas per team
- **Build:** two namespaces with a `ResourceQuota` and a `LimitRange` each. Try to deploy something that exceeds the quota, and a pod with no requests.
- **Check:** the error message tells the deploying team what to change.

### 3. One database server, several applications ⭐
- **Break:** two apps share one Postgres with one user. App B opens connections until `max_connections` is reached, then runs a long unindexed query. App A fails.
- **Fix:** a role per app with `CONNECTION LIMIT` and `statement_timeout`, a schema per app, a pooler (PgBouncer) in front.
- **Check:** repeat the attack; app A's error rate stays flat.
- **See:** README §2.

### 4. Dependency and name clashes
- **Break:** install two Python tools that need incompatible versions of the same library into one global environment. Then run two copies of an app that both write `/tmp/app.lock` and use the cache key `session:<id>` in one Redis.
- **Fix:** an environment per tool (venv / pipx / containers); configurable paths; key prefixes per app.
- **See:** README §1, §3.

### 5. A widget on someone else's page
- **Build:** a small embeddable widget and a host page with aggressive global CSS (`* { box-sizing: content-box; font-size: 30px }`) and a global variable with the same name as yours.
- **Fix:** Shadow DOM or an iframe, no globals, a size budget for the script.
- **Check:** the widget looks the same on three different host pages and adds no globals (`Object.keys(window)` before and after).

---

## Progress

- [ ] 1. Noisy neighbour on one node
- [ ] 2. Quotas per team
- [ ] 3. One database server, several applications
- [ ] 4. Dependency and name clashes
- [ ] 5. A widget on someone else's page
