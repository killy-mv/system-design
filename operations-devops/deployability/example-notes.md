# Deployability: hands-on practice notes

Planned labs for learning deployability by **building a real pipeline, deploying under load, and making bad releases harmless**. Theory is in [README.md](README.md).

Every deploy lab runs **k6 traffic during the deploy** and records errors and latency. A good deploy shows **zero** failed requests and no latency spike; that's how you know a strategy works.

They need: a **GitHub repo** (Actions + GHCR are free for public repos), **Docker**, **kind** or **k3d**, and later **Argo CD**, **Argo Rollouts** and **flagd**. Reuse the demo system from the observability notes if you have it (its telemetry is what the canary analysis reads).

**Suggested order:** 1 → 2 → 4 → 5 → 6 → 3, then the rest.

**Related labs elsewhere:** zero-downtime deploy and graceful shutdown (availability lab 8), rolling back a bad deploy with expand–contract (recoverability lab 3), common-mode config fault (fault-tolerance lab 3).

**Planned folder layout:**

```
operations-devops/deployability/labs/
  app/                     ← a small service with /version, /healthz, a "bad version" switch
  .github/workflows/       ← (in your GitHub repo) CI/CD pipelines
  k8s/                     ← manifests / Helm / Kustomize
  01-ci-pipeline/
    README.md              ← steps + results
  ...
```

---

## Labs

### 1. A real CI/CD pipeline ⭐
- **Build:** GitHub Actions: lint → test → build a container image **once** → tag with the commit SHA → push to GHCR → deploy to a kind cluster (or to a cheap VM) → smoke test `/version`.
- **Measure:** lead time (commit → running in "prod") and CI duration. Try to get CI under 5 minutes (caching, parallel jobs).
- **See:** README §3, §4.

### 2. Rolling update under load ⭐
- **Build:** a Kubernetes Deployment with 4 replicas. Run k6 during `kubectl set image`.
- **Break:** remove the readiness probe and graceful shutdown → see failed requests during the rollout.
- **Fix:** readiness probe, `preStop` sleep, `SIGTERM` handling, `maxUnavailable: 0`, `maxSurge: 1`. Rerun until k6 shows zero errors.
- **See:** README §5.

### 3. Blue–green
- **Build:** two Deployments (blue, green) and a Service whose selector you switch. Script the switch and the switch back.
- **Measure:** errors during switch and rollback time (should be seconds).
- **Discuss:** what happens with a DB schema change in blue–green?

### 4. Canary with automated analysis and rollback ⭐
- **Build:** **Argo Rollouts** (or Flagger) with steps 10% → 30% → 60% → 100%, and an **AnalysisTemplate** that queries Prometheus for error rate and p99 of the canary.
- **Break:** release a version with a 5% error rate (the "bad version" switch). Watch the rollout **abort and roll back automatically**.
- **Then:** a version that's only slow (no errors). Does your analysis catch it?
- **See:** README §7, §11 worked example.

### 5. Feature flags: deploy dark, release gradually ⭐
- **Build:** add **OpenFeature** SDK + **flagd** (or Unleash). Put a new checkout flow behind a release flag, and recommendations behind a kill switch.
- **Do:** deploy with the flag off → enable for internal users → 10% → 100%. Then flip the kill switch under load.
- **Break:** stop flagd. Does the app fall back to safe defaults?
- **Cleanup:** remove the release flag from code after 100% and redeploy.
- **See:** README §2, §6.

### 6. Zero-downtime schema change
- **Build:** rename a column with expand–contract (README §9) while k6 reads and writes continuously. Each step is a separate deploy.
- **Break first:** do it the naive way (`ALTER TABLE ... RENAME` + deploy) and count errors.
- **Check:** at every step, roll back the app one version. It must still work.

### 7. GitOps with Argo CD
- **Build:** a Git repo with the cluster manifests; Argo CD syncing it to kind.
- **Do:** deploy by PR merge; roll back by `git revert`. Then `kubectl edit` something by hand and watch Argo CD detect (and optionally fix) the **drift**.
- **See:** README §10.

### 8. Version compatibility (N / N−1)
- **Build:** a producer and consumer exchanging JSON (or Protobuf) events.
- **Break:** rename a field and deploy the producer first → the old consumer fails. Then try deploying the consumer first.
- **Fix:** additive changes, tolerant readers, a schema registry with compatibility checks in CI.
- **See:** README §8.

### 9. Frontend deploy pitfalls
- **Build:** a SPA (Vite/React) with hashed chunks served via Nginx with long cache headers on assets and `no-cache` on `index.html`.
- **Break:** open the app, deploy a new version that deletes old chunks, then navigate in the old tab → "Failed to fetch dynamically imported module".
- **Fix:** keep old assets for N days (upload new assets without deleting old ones), and/or reload on chunk errors.

### 10. Waves with bake time
- **Build:** three namespaces as "regions". A script (or Argo Rollouts/Kargo) deploys region by region with bake time and checks error rate before continuing.
- **Break:** a bad version that only fails in region 2 (env-specific config). Confirm the wave stops there.
- **See:** README §11.

### 11. Supply chain basics
- **Build:** add **Syft** (SBOM), **Trivy** (vulnerability scan, fail on critical) and **cosign** (sign the image) to the pipeline. Optionally verify signatures in the cluster with Kyverno or policy-controller.
- **See:** README §12.

### 12. Measure your DORA metrics
- **Build:** a script using the GitHub API: deployment frequency (deploy workflow runs), lead time (PR first commit → deploy), change failure rate (deploys followed by a revert/rollback), time to restore.
- **See:** README §1. Track them for a month while you do the other labs.

---

## Tools

| Tool | Used for |
|---|---|
| GitHub Actions + GHCR | CI/CD and artifact registry |
| Docker, kind / k3d | Building images, local Kubernetes |
| k6 | Traffic during deploys |
| Argo Rollouts or Flagger | Canary, blue–green, automated analysis |
| Prometheus | Metrics for canary analysis |
| OpenFeature + flagd (or Unleash) | Feature flags |
| Argo CD (or Flux) | GitOps |
| Flyway / Alembic / Atlas | Schema migrations |
| Syft, Trivy, cosign, Kyverno | SBOM, scanning, signing, policy |

## Progress

- [ ] 1. A real CI/CD pipeline
- [ ] 2. Rolling update under load
- [ ] 3. Blue–green
- [ ] 4. Canary with automated analysis and rollback
- [ ] 5. Feature flags: deploy dark, release gradually
- [ ] 6. Zero-downtime schema change
- [ ] 7. GitOps with Argo CD
- [ ] 8. Version compatibility (N / N−1)
- [ ] 9. Frontend deploy pitfalls
- [ ] 10. Waves with bake time
- [ ] 11. Supply chain basics
- [ ] 12. Measure your DORA metrics
