# Deployability

> **Deployability** = how easily, quickly and **safely** you can get a change into production, and get it back out if it's bad.
> "Can we ship a one-line fix in 15 minutes, on a Friday, without fear? If it breaks, can we undo it in 1 minute?"

Related: [Availability §11](../../runtime-operational/availability/README.md#11-safe-deployments--change-management) (safe deployment strategies), [Maintainability §1.1](../../development-evolution/maintainability/README.md#11-dora-metrics-delivery-performance) (DORA metrics) and [§11](../../development-evolution/maintainability/README.md#11-operability-deployment-configuration-automation) (CI/CD overview), [Configurability](../configurability/README.md) (config and flags are deploys too), [Observability](../observability/README.md) (knowing whether a deploy is healthy).

---

## Table of contents

1. [Why deployability matters](#1-why-deployability-matters)
2. [Deploy vs release](#2-deploy-vs-release)
3. [The deployment pipeline](#3-the-deployment-pipeline)
4. [Continuous integration](#4-continuous-integration)
5. [Deployment strategies](#5-deployment-strategies)
6. [Feature flags](#6-feature-flags)
7. [Verification and automated rollback](#7-verification-and-automated-rollback)
8. [Compatibility during rollouts](#8-compatibility-during-rollouts)
9. [Database and schema changes](#9-database-and-schema-changes)
10. [Infrastructure, config and GitOps](#10-infrastructure-config-and-gitops)
11. [Deploying safely at scale: waves and blast radius](#11-deploying-safely-at-scale-waves-and-blast-radius)
12. [Artifacts and supply chain](#12-artifacts-and-supply-chain)
13. [Approaches by system type](#13-approaches-by-system-type)
14. [Trade-offs](#14-trade-offs)
15. [Anti-patterns](#15-anti-patterns)
16. [Real-world stories](#16-real-world-stories)
17. [Checklist](#17-checklist)
18. [Exercises](#18-exercises)
19. [Further reading](#19-further-reading)

---

## 1. Why deployability matters

- **Changes cause most outages.** Google's SRE book estimates about 70% of outages come from changes to a live system. Safe deploys are the biggest availability lever you control.
- **Speed and stability go together.** The DORA research (*Accelerate*) found that teams deploying more often also have **fewer** failures and recover **faster**, because small changes are easy to test, review, diagnose and undo.
- **Fast recovery depends on fast deploys.** If a fix takes 3 hours to reach production, your MTTR is at least 3 hours.

**DORA metrics** measure it:

| Metric | Elite teams (roughly) |
|---|---|
| Deployment frequency | On demand, many times a day |
| Lead time for changes (commit → production) | Less than a day |
| Change failure rate | Low (single-digit to ~15%) |
| Time to restore service | Less than an hour |

---

## 2. Deploy vs release

| | Deploy | Release |
|---|---|---|
| Meaning | New code is **running** in production | Users **see** the new behaviour |
| Controlled by | Pipeline | Feature flags, config, gradual rollout |
| Risk | Technical (crashes, performance) | Product and technical |

**Decouple them.** Deploy code "dark" behind a flag, verify it's healthy, then release to 1% → 10% → 100% of users, and turn it off instantly without a deploy if needed. This turns a scary "big launch deploy" into a boring deploy plus a flag flip.

---

## 3. The deployment pipeline

```
commit ──► CI: build + unit tests + lint + security scan
       ──► artifact (container image, tagged with git SHA, immutable)
       ──► deploy to staging ──► integration / e2e / smoke tests
       ──► deploy to production (progressively) ──► automated verification
       ──► done, or automatic rollback
```

Principles:
- **Build once, deploy many.** The same artifact goes to staging and production; only config differs. Never rebuild for prod.
- **Immutable, versioned artifacts.** Tag with the commit SHA; never overwrite `latest` in production use.
- **Everything automated.** No manual steps (no "SSH in and run this"). Humans approve, machines execute.
- **Same deploy process everywhere**, including hotfixes. An emergency is the worst time to use an untested path.
- **Fast pipeline.** Aim for commit → production in < 1 hour; CI feedback in < 10 minutes.
- **Production-like staging**, but don't rely on it: real verification happens in production with progressive delivery.

---

## 4. Continuous integration

- **Trunk-based development**: short-lived branches (hours to a day or two), merge to main often. Long-lived branches = painful merges and big risky deploys.
- **Small batches**: small PRs are reviewed better, fail less, and are easy to revert.
- **Fast, reliable tests**: flaky tests destroy trust in the pipeline. Quarantine and fix them.
- **Merge queues** (GitHub merge queue, Bors) for busy repos: test the merged result before it lands on main.
- **Main is always deployable.** Incomplete features are hidden behind flags, not kept on branches.

**Continuous delivery** = every change on main *can* be deployed with one click. **Continuous deployment** = every change on main *is* deployed automatically.

---

## 5. Deployment strategies

| Strategy | How | Rollback | Extra capacity | Good for |
|---|---|---|---|---|
| **Recreate** | Stop all old, start all new | Redeploy old | None | Dev environments; apps that can't run two versions |
| **Rolling** | Replace instances a few at a time | Roll back gradually | Small (surge) | Default for stateless services (Kubernetes Deployment) |
| **Blue–green** | Run full new environment, switch all traffic at once | Switch back instantly | 2× during switch | Fast, clean cut-over and rollback |
| **Canary** | Send 1% → 5% → 25% → 100% to the new version, check metrics at each step | Route back to old | Small | Most production services: limits blast radius |
| **Shadow / dark** | Copy real traffic to the new version, discard its responses | N/A | Extra for the copy | Risky rewrites, performance validation |
| **A/B / experiment** | Different versions for user groups, measure product metrics | Turn off the variant | Small | Product decisions (usually via flags) |
| **Waves** | Region by region / cell by cell with bake time | Stop the wave | None | Large multi-region systems (§11) |

Supporting requirements for any zero-downtime strategy:
- **Readiness checks** so new instances get traffic only when ready.
- **Graceful shutdown**: on `SIGTERM`, stop accepting, finish in-flight work, then exit.
- **Two versions running at once** must work (§8).

Tools: Kubernetes Deployments, **Argo Rollouts**, **Flagger**, Spinnaker, AWS CodeDeploy, service mesh traffic splitting (Istio, Linkerd).

---

## 6. Feature flags

| Flag type | Lifetime | Example |
|---|---|---|
| **Release flag** | Days–weeks, then delete | Hide the new checkout until it's ready |
| **Ops flag / kill switch** | Long-lived | Turn off recommendations under heavy load |
| **Experiment flag** | Weeks | A/B test of a new pricing page |
| **Permission / entitlement flag** | Long-lived | Feature only for the Enterprise plan |

Good practice:
- Use a flag service or standard (**OpenFeature** API with flagd, Unleash, LaunchDarkly, Flagsmith, GrowthBook) rather than ad-hoc `if` on env vars.
- **Safe default** if the flag service is unreachable (cache last known values).
- **Remove release flags** after full rollout. Flag debt makes code unreadable and dangerous.
- **Never reuse a flag name** for a different feature (the Knight Capital lesson, §16).
- Flag changes are production changes: audit log, owner, gradual rollout, and monitoring.

---

## 7. Verification and automated rollback

A deploy isn't done when the pods are running. It's done when you know it's **healthy**.

- **Automated canary analysis**: compare the canary's error rate, latency and saturation with the baseline (old version, same traffic slice). Tools: Argo Rollouts analysis, Flagger, Kayenta.
- **SLO-based gates**: halt if the error budget burn rate spikes during rollout.
- **Bake time**: wait long enough at each step for slow problems (memory leaks, cron jobs, cache expiry, daily batch jobs).
- **Business metrics** too: errors can be zero while orders silently drop.
- **Automatic rollback** on failure. Humans investigate afterwards.

**Rollback vs roll forward:**
- **Roll back** first when in doubt: it's the fastest mitigation, and it's always the same action.
- **Roll forward** (ship a fix) only when rollback is impossible (e.g. an irreversible data migration) or the fix is trivial and the pipeline fast.
- Make rollback **always possible**: keep previous artifacts, keep schema backward compatible (§9).

---

## 8. Compatibility during rollouts

During any non-recreate deploy, **old and new versions run at the same time**, and clients may be older still.

- **APIs**: additive changes only; new fields optional; don't remove or rename fields in one step; version breaking changes.
- **Messages/events**: consumers must accept both old and new formats (schema registry with compatibility rules for Avro/Protobuf/JSON Schema).
- **Caches and sessions**: a new version must read entries written by the old one (and vice versa during rollback).
- **Clients**: mobile apps and browser tabs may run old versions for weeks/hours. The server must support them.
- Rule of thumb: version **N** must work with **N−1** and **N+1** on the other side of every interface.

---

## 9. Database and schema changes

The hardest part of deploys: you can roll back code in seconds, but not data.

**Expand–contract (parallel change)**, e.g. renaming `name` → `full_name`:

```
1. Expand:   add column full_name (nullable)                   ← deploy schema
2. Write both: app writes name AND full_name                   ← deploy app
3. Backfill: copy name → full_name for old rows (in batches)    ← job
4. Read new: app reads full_name                               ← deploy app
5. Stop writing old: app stops writing name                    ← deploy app
6. Contract: drop column name (later, after a safe period)     ← deploy schema
```

Every step is backward compatible, so any step can be rolled back.

Also:
- **Separate schema deploys from code deploys**; schema changes go first and must be compatible with the running code.
- **Online schema change** tools for big tables: gh-ost, pt-online-schema-change, `CREATE INDEX CONCURRENTLY` (Postgres). Watch for long locks (`ALTER TABLE` with a lock timeout).
- **Backfills in batches** with throttling, idempotent and resumable.
- Migration tools: Flyway, Liquibase, Alembic, Prisma Migrate, Atlas.

---

## 10. Infrastructure, config and GitOps

- **Infrastructure as code** (Terraform/OpenTofu, Pulumi, CloudFormation, CDK): reviewed in PRs, `plan` before `apply`, applied by the pipeline, not from laptops.
- **GitOps** (Argo CD, Flux): Git holds the desired state of the cluster; an agent continuously applies it and reports **drift**. Rollback = `git revert`.
- **Config changes are deploys**: validate (schema), review, roll out gradually, monitor, and be able to revert. Many of the largest outages were config pushes (§16, [Configurability](../configurability/README.md)).
- **Environment parity**: same IaC modules for staging and prod, different parameters.

---

## 11. Deploying safely at scale: waves and blast radius

Large systems deploy in **waves**, from smallest to largest exposure, with bake time in between:

```
1 instance (canary) → 1 zone → 1 small region → remaining regions in batches → all
     bake 15 min        bake 1 h     bake 2 h            bake ...
```

Plus: **cells** (independent copies of the stack serving a subset of users) so one bad deploy affects one cell; **deploy freezes** around peak events; error-budget policies that slow deploys when reliability is poor.

### Worked example: blast radius

A bad version fails **every** request it handles. Monitoring detects it **5 minutes** after it starts taking traffic, and rollback takes **2 minutes**.

**Rolling to 100% in 5 minutes, no canary:**

```
Traffic hits the bad version from minute 0, reaching 100% by minute 5
Detection at ~minute 5, rollback complete at minute 7
User impact ≈ ramp to 100% over 5 min + 100% for 2 min ≈ 2.5 + 2 = 4.5 "full-traffic minutes"
```

**Canary at 1% first:**

```
1% of traffic for 5 min until detection + 2 min to roll back
User impact ≈ 1% × 7 min = 0.07 "full-traffic minutes"
```

About **60× less user impact**, for the cost of a slower full rollout. The same logic applies at every wave: the earlier a stage catches a problem, the smaller the stage, the less it hurts.

---

## 12. Artifacts and supply chain

- **Artifact registry** (GHCR, ECR, Artifact Registry, Harbor) with immutable tags and retention.
- **Reproducible builds** with pinned dependencies (lockfiles) and pinned base images (by digest).
- **SBOM** (software bill of materials: Syft, CycloneDX) and **vulnerability scanning** (Trivy, Grype) in CI.
- **Signing and provenance** (Sigstore/cosign, SLSA levels); verify signatures at deploy time (admission policies).
- **Versioning**: SemVer for libraries and APIs; commit SHA or CalVer for services. Show the version on `/version` and in all telemetry.

---

## 13. Approaches by system type

| System | Key deploy concerns |
|---|---|
| **Stateless web services** | Rolling or canary with readiness checks, graceful shutdown; many deploys per day |
| **Monolith** | One deploy unit: make it fast and safe (canary by instance), feature flags for independent releases of modules |
| **Microservices** | Independent deploys per service; compatibility between versions; avoid "deploy these 5 together" (a distributed monolith) |
| **Frontend / SPA** | Hashed asset filenames, keep old assets for a while (avoid "chunk load failed" for open tabs), CDN invalidation of HTML only, flags for UI changes |
| **Mobile apps** | App store review (hours–days), **staged rollouts** (1% → 100%), can't roll back installed apps → server-driven flags, kill switches, minimum-version enforcement |
| **Databases / stateful systems** | Rolling upgrades one replica at a time, failover before upgrading the primary, expand–contract for schemas |
| **Serverless** | Versions + aliases with weighted traffic shifting (Lambda), fast rollback by alias |
| **ML models** | Shadow mode and canary on model metrics (accuracy proxies, latency, cost), model registry with versions, rollback to previous model |
| **Data pipelines** | Deploy new version alongside, compare outputs on the same input, then switch; backfill strategy |
| **IoT / firmware** | A/B partitions with automatic rollback if the new image fails to boot, staged rollouts by device cohort, signed updates |
| **Libraries / SDKs** | SemVer, deprecation periods, changelogs; you don't control when users upgrade |
| **On-prem / customer-hosted software** | Upgrade scripts tested across versions, skip-version upgrades, rollback guides, long support windows |

---

## 14. Trade-offs

| Choice | Cost |
|---|---|
| Frequent small deploys | Pipeline investment, need for strong automation and testing |
| Canary / waves | Slower full rollout; needs good metrics and traffic splitting |
| Blue–green | Double capacity during switch; data/schema still shared |
| Feature flags | Code complexity, flag debt, combinations to test |
| Expand–contract migrations | Several deploys for one schema change |
| Strict compatibility (N±1) | Slower API evolution, temporary duplicate code |
| Deploy freezes | Changes pile up → bigger, riskier deploys after the freeze |
| Continuous deployment | Requires high trust in tests and automated verification |

---

## 15. Anti-patterns

- Manual deploy steps, SSH into servers, snowflake environments.
- Rebuilding artifacts per environment.
- Big-bang releases after weeks of changes ("release day").
- Deploying everything to all regions at once.
- No automated health verification ("pods are running, ship it").
- Schema changes that make rollback impossible.
- Hotfix process that skips the pipeline.
- Long-lived feature branches instead of flags.
- Flags never removed; flag names reused.
- "Don't deploy on Friday" as a substitute for safe deploys (fine as a team preference, a smell if deploys are dangerous).

---

## 16. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Amazon (2011)** | A widely cited Velocity talk reported Amazon deploying to production about **every 11.6 seconds** on average, across many teams and services. | High frequency is possible when services are independent and deploys automated. |
| **Etsy (early 2010s)** | Etsy became a well-known example of continuous deployment (dozens of deploys a day with its "Deployinator" tool), with heavy use of feature flags and dashboards watched during deploys. | Small changes + one-button deploy + visible metrics changed engineering culture. |
| **Knight Capital (Aug 2012)** | A manual deploy reached 7 of 8 servers. A **reused feature flag** switched on old dead code on the 8th server, which lost about $440M in 45 minutes. | Automate deploys, verify they complete everywhere, never reuse flags, delete dead code. |
| **Cloudflare WAF (Jul 2019)** | A new WAF rule (a regular expression) was deployed **globally at once** and took CPU to 100% on every edge server. Cloudflare then added staged rollouts for these rules. | Config/rules deploys need canaries and waves too. |
| **CrowdStrike (Jul 2024)** | A content update for the Falcon sensor was pushed to all Windows hosts at once and crashed about 8.5 million machines. Afterwards CrowdStrike introduced staged deployment for these updates and gave customers control over timing. | "It's just data/content" still needs staged rollout. |
| **Google Cloud (Jun 2025)** | A new code path in a core API-management service was deployed **without a feature flag**. Weeks later, a policy change with unexpected blank fields replicated globally within seconds, triggered the path and crashed the service in every region. | Put new code paths behind flags; don't replicate config globally at once. |

Common thread: the biggest incidents came from changes that reached **everything at once**. Staged rollout + fast rollback would have turned each into a minor incident.

---

## 17. Checklist

**Pipeline**
- [ ] Every change deployed by the same automated pipeline (including hotfixes)
- [ ] Build once; immutable artifacts tagged by commit SHA
- [ ] CI feedback < 10 min; commit → production < 1 day (ideally < 1 hour)
- [ ] Trunk-based development; small PRs; main always deployable

**Safety**
- [ ] Canary or waves with automated analysis and automatic rollback
- [ ] Readiness checks and graceful shutdown on every service
- [ ] N/N−1 compatibility for APIs, messages, caches, schemas
- [ ] Expand–contract migrations; schema changes separate from code
- [ ] Feature flags with safe defaults, owners, and cleanup
- [ ] Config and infrastructure changes reviewed, staged and revertible

**Visibility**
- [ ] Deploy markers on dashboards; version on all telemetry
- [ ] DORA metrics tracked
- [ ] Previous version always available for rollback; rollback rehearsed

---

## 18. Exercises

1. **Map your pipeline.** For a project you built, write every step from commit to production. Which are manual? How long does each take? What's your lead time?
2. **Blast radius.** Redo the §11 worked example with: canary 5%, detection in 10 minutes, rollback 5 minutes. Then with waves 1% → 10% → 50% → 100%. Compare user impact.
3. **Choose a strategy** for: a payments API, an internal admin tool, a mobile app's new onboarding screen, a ML ranking model, a Postgres major version upgrade. Justify each.
4. **Expand–contract.** Plan the steps to split a `users.address` text column into `street`, `city`, `postal_code` without downtime and with rollback possible at every step.
5. **Compatibility.** A service publishes `OrderCreated` events consumed by 4 services. You need to rename `amount` to `total_amount` and change it from float to integer cents. Plan the rollout.
6. **Flag hygiene.** Design a policy for feature flags: naming, owners, expiry dates, how stale flags are detected and removed.
7. **Mobile rollback.** Your app v5.0 has a crash on login for 3% of users, and it's already on 40% of devices. You can't un-install it. What can you do in the next hour, day and week?
8. **Read** the Cloudflare July 2019 postmortem or the Google Cloud June 2025 incident report. Which pipeline safeguard would have caught it earliest?

---

## 19. Further reading

- *Continuous Delivery* (Jez Humble, David Farley): the foundational book
- *Accelerate* (Forsgren, Humble, Kim) and the yearly **DORA State of DevOps** reports
- Google SRE book: ch. 8 "Release Engineering"; SRE workbook ch. 16 "Canarying Releases"
- Amazon Builders' Library: "Automating safe, hands-off deployments", "Ensuring rollback safety during deployments"
- Martin Fowler: "Feature Toggles (aka Feature Flags)" (Pete Hodgson), "BlueGreenDeployment", "ParallelChange"
- Argo Rollouts / Flagger documentation (progressive delivery)
- *Refactoring Databases* (Ambler, Sadalage): evolutionary schema changes
- SLSA framework (slsa.dev) and Sigstore for supply chain security
