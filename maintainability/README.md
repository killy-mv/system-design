# Maintainability

> **Maintainability** = how easily a system can be **operated**, **understood**, and **changed** over its lifetime by the people who work on it.
> "Can we keep running it, fixing it, and evolving it — cheaply and safely — for years?"

Most of a system's cost is not building it, but **maintaining** it: fixing bugs, keeping it running, adapting to new requirements, paying off tech debt, upgrading dependencies, onboarding new engineers. A system that is fast, scalable, and available today but impossible to change will fail tomorrow.

Related: [Availability](../availability/README.md), [Reliability](../reliability/README.md) (most outages come from changes — maintainable systems change safely), [Performance](../performance/README.md) and [Scalability](../scalability/README.md) (observability is how you find bottlenecks).

Kleppmann's three design principles for maintainability:
1. **Operability** — make it easy for operations teams to keep the system running smoothly.
2. **Simplicity** — make it easy for new engineers to understand the system.
3. **Evolvability** — make it easy to change the system in the future.

This README also covers **cost efficiency**, since sustaining a system includes being able to afford it.

---

## Table of contents

1. [Measuring maintainability](#1-measuring-maintainability)
2. [Simplicity: managing complexity](#2-simplicity-managing-complexity)
3. [Modularity, coupling, and cohesion](#3-modularity-coupling-and-cohesion)
4. [Architecture styles](#4-architecture-styles)
5. [Domain-driven design and boundaries](#5-domain-driven-design-and-boundaries)
6. [APIs and contracts](#6-apis-and-contracts)
7. [Data and schema evolution](#7-data-and-schema-evolution)
8. [Code quality and technical debt](#8-code-quality-and-technical-debt)
9. [Testing strategy](#9-testing-strategy)
10. [Observability](#10-observability)
11. [Operability: deployment, configuration, automation](#11-operability-deployment-configuration-automation)
12. [Documentation and knowledge sharing](#12-documentation-and-knowledge-sharing)
13. [Dependencies, upgrades, and security maintenance](#13-dependencies-upgrades-and-security-maintenance)
14. [Teams and organisation](#14-teams-and-organisation)
15. [Cost efficiency (FinOps)](#15-cost-efficiency-finops)
16. [Evolving legacy systems](#16-evolving-legacy-systems)
17. [Approaches by system type](#17-approaches-by-system-type)
18. [Trade-offs](#18-trade-offs)
19. [Anti-patterns](#19-anti-patterns)
20. [Checklist](#20-checklist)
21. [Further reading](#21-further-reading)

---

## 1. Measuring maintainability

Maintainability is hard to measure directly; use proxies.

### 1.1 DORA metrics (delivery performance)

| Metric | Elite performers (approx.) | What it reveals |
|---|---|---|
| **Deployment frequency** | On demand, multiple times/day | How easy it is to ship |
| **Lead time for changes** | < 1 day (commit → production) | Pipeline and process friction |
| **Change failure rate** | 0–15% | Quality and safety of changes |
| **Time to restore service** | < 1 hour | Operability, observability |
| *(Rework rate — newer)* | Low | Unplanned work from defects |

Research (*Accelerate*) shows speed and stability are **not** a trade-off — high performers are better at both, because the same practices (small changes, automation, testing, observability) enable both.

### 1.2 Other useful signals
- **Time to onboard** a new engineer to first meaningful change.
- **Time to implement** a typical feature (and how it trends over time — rising time = growing complexity/debt).
- **Bug rate / escaped defects**, recurring incidents.
- **Toil**: % of ops time spent on manual, repetitive, automatable work (Google SRE target: < 50%).
- **Alert volume** and % actionable alerts.
- **Code metrics** (use with caution): cyclomatic complexity, file churn × complexity hot spots, test coverage trends, dependency freshness, build time.
- **Developer experience surveys** (SPACE framework: Satisfaction, Performance, Activity, Communication, Efficiency).
- **Cognitive load**: how much does a team need to know to change their part safely?

---

## 2. Simplicity: managing complexity

> "Simplicity is prerequisite for reliability." — Edsger Dijkstra

### 2.1 Essential vs accidental complexity
- **Essential complexity**: inherent in the problem (tax rules, payment flows).
- **Accidental complexity**: introduced by our solution (unnecessary layers, tools, abstractions, inconsistent conventions, premature distribution).

Goal: minimise accidental complexity. Every component, technology, and abstraction must pay for itself.

### 2.2 Symptoms of complexity (Ousterhout, *A Philosophy of Software Design*)
- **Change amplification**: a simple change requires edits in many places.
- **Cognitive load**: you need to know a lot to make a change.
- **Unknown unknowns**: it's not obvious what you need to change or know.

### 2.3 Principles
- **KISS** — keep it simple; choose boring technology where possible ("Choose Boring Technology", Dan McKinley — you have limited "innovation tokens").
- **YAGNI** — don't build for hypothetical future requirements.
- **DRY** — but don't over-apply it; wrong abstraction is worse than duplication ("prefer duplication over the wrong abstraction" — Sandi Metz). Rule of three.
- **Deep modules**: simple interfaces hiding powerful implementations (e.g., Unix file I/O). Avoid shallow modules (interface as complex as implementation).
- **Information hiding / encapsulation**: hide design decisions likely to change behind stable interfaces.
- **Principle of least astonishment**: things behave as people expect.
- **Consistency**: one way to do things (logging, error handling, config, naming) across the codebase.
- **Make illegal states unrepresentable** (types, state machines).
- **Fewer moving parts**: each additional database, queue, language, or service adds operational surface. Can Postgres do it (queue, search, JSON, pub/sub) before adding a new system?

---

## 3. Modularity, coupling, and cohesion

### 3.1 Coupling and cohesion
- **High cohesion**: things that change together live together.
- **Low coupling**: modules depend on each other as little as possible and through stable interfaces.

Types of coupling (worst → best, roughly):
| Coupling | Example |
|---|---|
| Shared database | Two services read/write the same tables |
| Temporal / synchronous | Service A fails when B is down |
| Content / implementation | Depending on another module's internals |
| Data / schema | Sharing a data format |
| Message / contract | Communicating via well-defined events/APIs |

### 3.2 SOLID (object-oriented, applicable more broadly)
- **S**ingle Responsibility — one reason to change.
- **O**pen/Closed — extend without modifying.
- **L**iskov Substitution — subtypes are substitutable.
- **I**nterface Segregation — small, focused interfaces.
- **D**ependency Inversion — depend on abstractions; high-level policy doesn't depend on low-level detail.

### 3.3 Layered and ports-and-adapters designs
- **Layered** (presentation → application → domain → infrastructure).
- **Hexagonal / ports and adapters / clean architecture / onion**: business logic at the centre, independent of frameworks, databases, and UIs. External concerns plug in via adapters. Benefits: testable domain logic, swappable infrastructure.

```
        ┌─────────────── Adapters ────────────────┐
        │  HTTP API   CLI   Queue consumer        │
        │      │       │         │                │
        │   ┌──▼───────▼─────────▼──┐             │
        │   │   Application / Domain │ ◄── Ports  │
        │   └──┬───────┬─────────┬──┘             │
        │      │       │         │                │
        │  Postgres  Stripe   Email provider      │
        └─────────────────────────────────────────┘
```

### 3.4 Enforcing boundaries
- Package/module visibility, architecture tests (ArchUnit, dependency-cruiser, import-linter), monorepo tooling (Nx, Bazel, Turborepo) with dependency rules.

---

## 4. Architecture styles

| Style | Description | Maintainability strengths | Maintainability costs | Fits |
|---|---|---|---|---|
| **Monolith** | One deployable | Simple dev, debug, deploy, refactor; one codebase | Can become a "big ball of mud"; whole-app deploys; team contention at scale | Startups, small–medium teams |
| **Modular monolith** | One deployable, strict internal modules | Monolith simplicity + clear boundaries; easy to extract services later | Discipline needed to keep boundaries | Most systems — a strong default |
| **Microservices** | Many independently deployable services | Team autonomy, independent deploys, isolated tech choices | Distributed systems complexity, observability, data consistency, infra overhead, versioning | Large orgs, many teams, divergent scaling needs |
| **Service-oriented (SOA)** | Coarser services, often shared ESB | Reuse | ESB becomes bottleneck/coupling point | Enterprises (legacy) |
| **Event-driven** | Components communicate via events | Loose coupling, easy to add consumers | Hard to trace flows, eventual consistency, event schema evolution | Integrations, workflows, analytics |
| **Serverless** | Functions + managed services | No servers to manage; scales automatically | Vendor lock-in, local testing difficulty, distributed tracing, cold starts, many small pieces | Event-driven glue, spiky workloads, small teams |
| **Cell-based** | Many copies of the full stack serving subsets of users | Blast radius containment | Deployment orchestration across cells | Large-scale SaaS |

Guidelines:
- **Start with a (modular) monolith** unless you have a strong reason not to. "Microservices are a solution to an organisational scaling problem."
- Extract a service when a module has: a separate team, a very different scaling/availability/security profile, a different release cadence, or a different tech need.
- Avoid the **distributed monolith**: services that must be deployed together, share a database, or call each other synchronously in long chains — all the costs of microservices, none of the benefits.

---

## 5. Domain-driven design and boundaries

DDD helps decide **where to draw boundaries** so the system stays changeable.

- **Ubiquitous language**: the same terms in code, conversations, and docs as the business uses.
- **Bounded context**: a boundary within which a model and its language are consistent. "Customer" in Billing ≠ "Customer" in Support — and that's fine.
- **Context map**: relationships between contexts (shared kernel, customer–supplier, anticorruption layer, open host service, published language).
- **Anticorruption layer (ACL)**: translate between your model and an external/legacy system's model so their concepts don't leak into yours.
- **Aggregates**: clusters of entities that change together under one transactional boundary (e.g., Order + OrderLines). Reference other aggregates by ID.
- **Domain events**: `OrderPlaced`, `PaymentFailed` — make business-relevant changes explicit.
- **Core vs supporting vs generic subdomains**: invest custom engineering in the **core** (your differentiator); buy or use off-the-shelf for generic (auth, email, payments).

Bounded contexts are natural candidates for modules in a modular monolith and later for services.

---

## 6. APIs and contracts

APIs are the hardest things to change once published. Design them carefully.

### 6.1 Design principles
- **Consistent conventions**: naming, pagination (cursor-based), filtering, error format (e.g., RFC 9457 Problem Details), timestamps (ISO 8601 UTC), IDs (opaque strings).
- **Resource-oriented** (REST) or clear RPC methods (gRPC); avoid leaking internal DB structure.
- **Idempotency** for unsafe operations (see [Reliability §11](../reliability/README.md#11-idempotency)).
- **Design-first** with OpenAPI / Protobuf / GraphQL schema; generate clients and docs.
- **Least surprise**: return what clients need; don't overload endpoints.

### 6.2 Evolution and versioning
- **Additive changes** are safe: new optional fields, new endpoints, new enum values (if clients handle unknown values).
- **Breaking changes**: removing/renaming fields, changing types/semantics, making optional fields required.
- **Tolerant reader / Postel's law**: clients ignore unknown fields; servers accept old formats.
- Versioning strategies: URI (`/v2/`), header, media type, or date-based (Stripe). Prefer evolving without new versions where possible.
- **Deprecation policy**: announce, mark deprecated (`Deprecation`/`Sunset` headers), monitor usage, give migration time, then remove.
- **Consumer-driven contract tests** (Pact) to catch breaking changes before deploy.
- **Schema registry** for event schemas (Avro/Protobuf/JSON Schema) with compatibility checks (backward/forward/full).

### 6.3 Internal vs public APIs
- Public APIs: stability above all; versioned, documented, rate-limited.
- Internal APIs: can evolve faster, but still use compatible changes because services deploy independently.

---

## 7. Data and schema evolution

Data outlives code. Schema changes are among the riskiest operations.

- **Migrations as code** (Flyway, Liquibase, Alembic, Rails/Django migrations, Atlas), versioned and reviewed, run automatically in CI/CD.
- **Expand–contract (parallel change)**:
  1. Expand: add new column/table (nullable / with default).
  2. Deploy code that writes both old and new.
  3. Backfill existing data (in batches).
  4. Deploy code that reads new.
  5. Stop writing old.
  6. Contract: drop old column.
- **Online schema changes** for big tables (gh-ost, pt-online-schema-change, `CREATE INDEX CONCURRENTLY`).
- **Backward/forward compatible encodings** (Protobuf/Avro rules: never reuse field numbers/names; add fields as optional with defaults).
- **Database per service / module ownership**: only the owning module writes its tables; others use its API or events.
- **Data lineage and catalogue** for analytics data (DataHub, OpenMetadata, Unity Catalog).
- **Data retention and deletion** policies (GDPR "right to be forgotten") built in, not bolted on.

---

## 8. Code quality and technical debt

### 8.1 Practices
- **Code review**: small PRs, clear descriptions, review for design and readability, not just correctness.
- **Consistent style** enforced by tools (formatters: Prettier, Black, gofmt; linters: ESLint, Ruff, golangci-lint).
- **Static analysis and type checking** (TypeScript strict mode, mypy/pyright, SonarQube, Semgrep).
- **Meaningful names**, small focused functions, clear error handling.
- **Comments explain why**, not what.
- **Refactor continuously** (boy scout rule: leave code better than you found it).
- **Trunk-based development** with short-lived branches and feature flags → fewer painful merges.

### 8.2 Technical debt
Debt is a trade-off, not inherently bad. Fowler's **technical debt quadrant**:

| | Reckless | Prudent |
|---|---|---|
| **Deliberate** | "We don't have time for design" | "We must ship now and will deal with consequences" |
| **Inadvertent** | "What's layering?" | "Now we know how we should have done it" |

Managing it:
- Make debt **visible** (tickets, ADRs, debt register) with an estimate of its "interest" (how much it slows you down).
- Prioritise debt in **hot spots** — code that changes often *and* is complex (use `git log` churn + complexity analysis; CodeScene).
- Reserve capacity (e.g., 15–20% of each cycle) for maintenance.
- Pay it down as part of feature work in the same area.
- Delete dead code and unused features (feature flags left forever are debt).

---

## 9. Testing strategy

Tests are what make change **safe** — the core of evolvability. (Reliability-focused testing techniques are in [Reliability §16](../reliability/README.md#16-testing-for-reliability).)

```
           ▲ slower, costlier, more realistic
          ╱ ╲        E2E / UI tests (few)
         ╱───╲       Contract & integration tests
        ╱─────╲      Component / service tests
       ╱───────╲     Unit tests (many)
      ▼ faster, cheaper, more isolated
```
Alternative shapes (testing trophy, honeycomb for microservices) emphasise integration tests — choose what gives confidence per unit of cost.

Maintainability-specific guidance:
- Test **behaviour**, not implementation details (so refactoring doesn't break tests).
- Fast feedback: unit tests in seconds, full CI in minutes. Slow, flaky suites get ignored.
- **Quarantine and fix flaky tests** immediately.
- Use real dependencies in integration tests where practical (Testcontainers) rather than deep mocks.
- Test data builders/factories to keep tests readable.
- Coverage is a guide, not a goal; focus on critical paths and complex logic.
- **Preview / ephemeral environments** per PR for realistic testing.

---

## 10. Observability

> Observability = being able to understand what's happening inside the system from its outputs, including for problems you didn't predict.

It's the foundation of operability and of lowering MTTR.

### 10.1 The signals

| Signal | What | Good for | Tools |
|---|---|---|---|
| **Metrics** | Numeric time series (counters, gauges, histograms) | Dashboards, alerting, trends; cheap at scale | Prometheus, Grafana, Datadog, CloudWatch, VictoriaMetrics |
| **Logs** | Timestamped event records | Detailed context, debugging, audit | Loki, Elasticsearch/OpenSearch, Splunk, CloudWatch Logs |
| **Traces** | Request path across services with timings (spans) | Latency breakdown, dependency maps, finding the failing hop | OpenTelemetry, Jaeger, Tempo, Zipkin, Honeycomb, X-Ray |
| **Profiles** | CPU/memory usage by code path | Performance hot spots in production | Pyroscope, Parca, cloud profilers |
| **Events / wide events** | Rich structured events per request with many fields | High-cardinality exploration ("which customers on which version see errors?") | Honeycomb, ClickHouse-based stacks |
| **Real User Monitoring / synthetic** | Client-side experience; scripted probes | User-perceived performance and availability | Sentry, Datadog RUM, Checkly |

**OpenTelemetry** is the vendor-neutral standard for instrumenting all three main signals — use it to avoid lock-in.

### 10.2 What to measure
- **RED** for every service: Rate, Errors, Duration.
- **USE** for every resource: Utilisation, Saturation, Errors.
- **Four golden signals**: latency, traffic, errors, saturation.
- **Business metrics**: sign-ups, orders/minute, payment success rate — often the first to reveal problems.
- **Dependency health**: latency/errors of each downstream call.
- **Saturation of pools/queues**: thread pools, connection pools, queue depth, consumer lag.

### 10.3 Logging best practices
- **Structured logs** (JSON) with consistent fields: timestamp, level, service, version, environment, `trace_id`, `span_id`, request ID, user/tenant ID (if allowed).
- **Correlation IDs** propagated across services (W3C Trace Context `traceparent` header).
- Log levels used consistently; avoid logging in tight loops.
- **Never log secrets or sensitive PII** (redaction, allowlists).
- Retention tiers (hot/warm/cold) and sampling to control cost.

### 10.4 Dashboards
- Service overview dashboard per service (RED + saturation + deploy markers).
- User journey dashboards (end-to-end flows).
- Don't build hundreds of dashboards no one looks at; link dashboards from alerts and runbooks.

### 10.5 Alerting
- **Alert on symptoms** (users affected: SLO burn rate, error rate, latency), not causes (CPU high) — cause-based signals go on dashboards.
- **Every alert must be actionable** and have a **runbook**.
- **Multi-window, multi-burn-rate** SLO alerts (fast burn → page; slow burn → ticket).
- Route by severity: page (wake someone up) vs ticket (next business day).
- Review alert noise regularly; delete or fix alerts that fire without action.

---

## 11. Operability: deployment, configuration, automation

### 11.1 CI/CD
- **Continuous integration**: every commit builds and runs tests automatically; main branch always releasable.
- **Continuous delivery/deployment**: automated pipeline to production with gates (tests, security scans, approvals where required).
- **Build once, deploy many**: same artifact (container image) promoted through environments.
- **Immutable artifacts** with versions tied to commits; reproducible builds.
- Progressive delivery: canary, blue–green, automated rollback (see [Availability §11](../availability/README.md#11-safe-deployments--change-management)).
- **GitOps** (Argo CD, Flux): desired state in Git; controllers reconcile clusters to match.

### 11.2 Infrastructure as Code (IaC)
- Terraform/OpenTofu, Pulumi, CloudFormation, CDK, Crossplane.
- Everything in version control, reviewed, with plan previews. No "click-ops" in production.
- Modules/templates for consistent environments; drift detection.
- Environments (dev/staging/prod) created from the same code with different parameters.

### 11.3 Configuration management
- **12-factor app**: config in environment, not code; strict separation of build and run; stateless processes; logs as event streams; dev/prod parity; disposability.
- Centralised config with validation, versioning, and gradual rollout (config changes are deploys).
- **Secrets management**: Vault, AWS Secrets Manager, GCP Secret Manager, SOPS, sealed secrets. Never commit secrets; rotate regularly.
- **Feature flags** (LaunchDarkly, Unleash, OpenFeature, Flagsmith): decouple deploy from release; kill switches; percentage rollouts. Clean up stale flags.

### 11.4 Platforms and containers
- **Containers** (Docker) for consistent runtime environments.
- **Orchestration** (Kubernetes, ECS, Nomad, Cloud Run): self-healing, rolling deploys, service discovery, autoscaling. Kubernetes is powerful but complex — managed services or simpler platforms may be more maintainable for small teams.
- **Internal developer platform (IDP)** / platform engineering: golden paths and templates (Backstage scaffolder) so teams don't reinvent CI, observability, deployment for each service.
- **Managed services** (RDS, managed Kafka, managed Redis) reduce operational toil at higher cost and less control.

### 11.5 Automation and toil reduction
- Automate repetitive operational tasks: certificate renewal (cert-manager/ACME), backups, scaling, failover, log rotation, dependency updates.
- **Self-healing**: health checks + restarts, auto-replacement of unhealthy nodes, automated remediation for known issues.
- **ChatOps** and runbook automation for incident response.

### 11.6 Operational readiness
Before launching a service, a **production readiness review**:
- Owner and on-call defined
- SLOs, dashboards, alerts with runbooks
- Capacity tested
- Backups and restore tested
- Security review done
- Rollback plan
- Dependencies and their failure modes documented

### 11.7 Incident management
- Clear severity levels, incident commander role, communication templates, status page.
- **Blameless postmortems** with timeline, contributing factors, and tracked action items.
- Track incident trends to direct investment.

---

## 12. Documentation and knowledge sharing

Documentation reduces cognitive load and bus factor.

| Doc type | Purpose |
|---|---|
| **README per repo/service** | What it is, how to run, test, deploy; owners; links |
| **Architecture diagrams** (C4 model: Context, Container, Component, Code) | Shared mental model at the right zoom level |
| **ADRs** (Architecture Decision Records) | Why decisions were made; context, options, consequences — short, immutable, numbered |
| **Design docs / RFCs** | Proposals reviewed before big changes |
| **Runbooks / playbooks** | Step-by-step operational procedures for alerts and tasks |
| **API docs** | Generated from OpenAPI/Protobuf/GraphQL schemas |
| **Onboarding guide** | Getting a new engineer productive fast |
| **Service catalogue** (Backstage, OpsLevel, Cortex) | Ownership, dependencies, docs, scorecards for all services |
| **Glossary** | Ubiquitous language |

Principles:
- **Docs as code**: in the repo, reviewed with code changes, so they stay current.
- Keep docs close to what they describe; delete outdated docs.
- Diagrams as code (Mermaid, PlantUML, Structurizr) to keep them versioned.
- Write for the reader who is on call at 3 a.m.

---

## 13. Dependencies, upgrades, and security maintenance

Software rots if not maintained: dependencies get vulnerabilities, runtimes reach end of life, APIs deprecate.

- **Automated dependency updates**: Dependabot, Renovate — small, frequent upgrades are easier than big jumps.
- **Lock files** for reproducible builds.
- **Vulnerability scanning**: SCA (Snyk, GitHub Advanced Security, Trivy, Grype), container image scanning, SAST/DAST.
- **SBOM** (software bill of materials) and supply-chain security (SLSA, signed artifacts with Sigstore/cosign).
- **Track EOL dates** for languages, frameworks, OS images, databases (endoflife.date).
- **Minimise dependencies**: each one is code you maintain indirectly. Evaluate maintenance health before adopting.
- **Wrap third-party SDKs** behind your own interface (adapter) so they can be swapped/upgraded in one place.
- Regular **patching cadence** for OS and base images (rebuild images, don't patch in place).
- Security as part of maintainability: least privilege, secret rotation, audit logs, threat modelling for new designs.

---

## 14. Teams and organisation

### 14.1 Conway's law
> Organisations design systems that mirror their communication structures.

- **Inverse Conway manoeuvre**: shape teams to match the architecture you want.
- Misaligned teams and architecture → constant cross-team coordination, slow changes.

### 14.2 Team Topologies (Skelton & Pais)
| Team type | Role |
|---|---|
| **Stream-aligned** | Owns a business flow end-to-end (most teams) |
| **Platform** | Provides internal self-service platform to reduce stream-aligned teams' cognitive load |
| **Enabling** | Helps other teams adopt new skills/practices |
| **Complicated-subsystem** | Owns parts needing deep specialist knowledge (e.g., video codec, ML ranking) |

Interaction modes: collaboration, X-as-a-service, facilitating.

### 14.3 Ownership
- **Every service, dataset, and dashboard has a clear owner** (service catalogue).
- **"You build it, you run it"**: teams operate what they build → incentive to make it operable.
- Sustainable **on-call**: reasonable rotation size, compensated, low alert noise, follow-the-sun if possible.
- Reduce **bus factor**: pairing, rotation, documentation, shared code ownership.

### 14.4 Engineering culture
- Psychological safety (people report mistakes and risks).
- Small batches, fast feedback, continuous improvement.
- Inner source and shared libraries with clear owners.

---

## 15. Cost efficiency (FinOps)

A system that costs too much to run isn't sustainable. Cost is a design dimension like latency.

### 15.1 Visibility
- **Tag/label** every resource with owner, service, environment, cost centre.
- Cost dashboards per team/service; **unit economics** (cost per request, per user, per order, per GB).
- Budgets and anomaly alerts.

### 15.2 Common optimisations

| Area | Techniques |
|---|---|
| **Compute** | Right-sizing; autoscaling (incl. scale to zero for dev/test); ARM instances; spot/preemptible for stateless and batch; reserved instances / savings plans for steady baseline |
| **Storage** | Lifecycle policies (hot → infrequent → archive); delete orphaned volumes/snapshots; compression; appropriate replication levels |
| **Data transfer** | Keep chatty traffic in-zone; CDN for egress; compress; avoid unnecessary cross-region replication; watch NAT gateway costs |
| **Databases** | Right-size; read replicas only where needed; serverless/auto-pause for spiky/dev workloads; archive cold data |
| **Observability** | Log sampling, retention tiers, drop noisy logs, control metric cardinality — observability bills can rival compute |
| **Architecture** | Managed vs self-hosted trade-off (people cost vs infra cost); serverless for spiky low-volume, containers/VMs for steady high-volume; caching reduces backend cost |
| **Environments** | Shut down non-prod at night/weekends; ephemeral preview environments |
| **Licensing** | Audit SaaS and licence usage |

### 15.3 Total cost of ownership
Include **engineering time**: a cheap self-hosted Kafka that needs half an engineer to operate may cost more than a managed service. Optimise the total, not just the cloud bill.

---

## 16. Evolving legacy systems

Most engineering work is on existing systems. Rewrites are risky ("the second-system effect"; Netscape's rewrite).

- **Strangler fig pattern**: put a façade/proxy in front of the legacy system; route features one by one to new implementations; retire the old system gradually.
- **Branch by abstraction**: introduce an abstraction over the component to replace, switch implementations behind it, remove the old one.
- **Anticorruption layer** to isolate new code from legacy models.
- **Characterisation tests** (Feathers, *Working Effectively with Legacy Code*): capture current behaviour before changing it.
- **Seams**: find places where behaviour can be altered without editing in place (dependency injection points).
- **Parallel run / shadow traffic**: run old and new side by side, compare outputs (Scientist library pattern).
- **Data migration**: dual writes or CDC from old to new store, backfill, verify, cut over, keep rollback path.
- **Incremental modernisation** with measurable milestones; avoid big-bang cutovers.

---

## 17. Approaches by system type

### 17.1 Startup / MVP / early-stage product
- **Speed of learning** is the priority. Monolith, one database (Postgres), managed services, a PaaS (Render, Fly.io, Heroku, Vercel, Railway, Cloud Run).
- Keep it simple but **not sloppy**: tests on critical paths, CI, error tracking (Sentry), basic metrics, IaC from early on is cheap.
- Accept deliberate, documented debt; avoid irreversible decisions (lock-in that's hard to undo, poor data model).

### 17.2 Growing SaaS / mid-size product (5–50 engineers)
- **Modular monolith** with enforced boundaries; extract services only where justified.
- Invest in CI/CD, observability (OpenTelemetry), feature flags, migrations discipline, on-call and runbooks.
- Introduce ADRs and a service catalogue as teams multiply.

### 17.3 Large-scale / multi-team enterprise
- Microservices or larger "macro-services" aligned with bounded contexts and teams.
- **Platform team** providing golden paths (templates, CI, deployment, observability, security baseline).
- API governance, schema registry, contract testing, service ownership scorecards.
- Strong change management and progressive delivery across many services.

### 17.4 Platforms and public APIs (developer platforms, payment APIs)
- **API stability is paramount**: versioning strategy, long deprecation windows, changelogs, SDKs generated from specs, sandbox environments.
- Backward-compatibility testing in CI; API usage analytics to understand impact of changes.

### 17.5 Open-source libraries / SDKs
- Semantic versioning; minimal dependencies; clear public vs internal API; CHANGELOG; contribution guide; CI across supported versions/platforms; deprecation warnings before removal.

### 17.6 Content sites / CMS (e.g., WordPress)
- Maintainability risks: plugin sprawl, unmaintained themes/plugins, direct edits in production, core updates.
- Version-control themes/custom plugins; use Composer/Bedrock-style dependency management; staging environment; automated updates with testing; minimal plugins; managed hosting; headless CMS when frontend and content teams need independence.

### 17.7 Data platforms / analytics / pipelines
- **Data as code**: dbt models with tests and docs; orchestration (Airflow, Dagster, Prefect) with lineage.
- **Data contracts** between producers and consumers; schema registry; ownership per dataset (data mesh ideas for large orgs).
- Idempotent, re-runnable jobs; observability for data freshness and quality (Monte Carlo, Elementary).
- Catalogue and documentation of datasets.

### 17.8 ML / AI systems
- **MLOps**: versioned data, code, and models (DVC, MLflow, model registry); reproducible training pipelines; feature stores for consistency between training and serving.
- Monitoring for drift and quality; automated retraining with evaluation gates; shadow and canary model deployments.
- LLM apps: prompt and model versioning, evaluation suites as regression tests, tracing of LLM calls (Langfuse, OpenTelemetry GenAI conventions), abstraction over model providers.
- "Hidden technical debt in ML systems" — glue code, pipeline jungles, undeclared consumers.

### 17.9 Mobile apps
- Long tail of old app versions in the wild → backend APIs must support old clients for months/years; **server-driven UI / remote config / feature flags** to change behaviour without app releases; forced-upgrade mechanism.
- Crash reporting (Crashlytics, Sentry); staged rollouts in app stores.

### 17.10 Embedded / IoT / firmware
- Over-the-air (OTA) updates with A/B partitions and rollback; device fleet management; remote logging/telemetry with bandwidth limits; long-term support of hardware revisions; strict versioning of device–cloud protocols.

### 17.11 Regulated industries (finance, healthcare, government)
- Audit trails, change approval workflows, traceability from requirements to tests to deployments, compliance as code (policy checks in pipelines), documented controls, data retention policies.

### 17.12 Internal tools
- Low-code/internal tool builders (Retool, Appsmith) or simple frameworks; prioritise ease of change and low ops over scale.

### Summary table

| Context | Priority | Key maintainability practices |
|---|---|---|
| Startup/MVP | Speed of iteration | Monolith, PaaS, managed DB, basic CI + error tracking |
| Growing SaaS | Sustainable velocity | Modular monolith, CI/CD, observability, feature flags, ADRs |
| Large enterprise | Team autonomy at scale | Bounded-context services, platform team, contracts, catalogue |
| Public API/platform | Stability | Versioning, deprecation policy, generated SDKs, contract tests |
| OSS library | Compatibility | SemVer, minimal deps, changelog |
| CMS/content | Safe updates | Version control, staging, minimal plugins |
| Data platform | Trustworthy data | dbt tests, data contracts, lineage, ownership |
| ML/AI | Reproducibility | MLOps, model registry, evals, drift monitoring |
| Mobile | Old-client support | Backward-compatible APIs, remote config, staged rollouts |
| IoT/embedded | Safe remote updates | OTA with rollback, fleet management |
| Regulated | Auditability | Audit trails, compliance as code |

---

## 18. Trade-offs

| Maintainability choice | What it costs / risks |
|---|---|
| Microservices for team autonomy | Operational complexity, network latency, consistency challenges, more infra |
| Monolith for simplicity | Team contention and coupled deploys at large scale |
| Abstractions/layers | Indirection, performance overhead, harder to follow if overdone |
| Managed services | Higher cost, vendor lock-in, less control |
| Standardising on few technologies | Sometimes not the best tool for a specific job |
| Extensive tests | Time to write/maintain; slow suites if unmanaged |
| Rich observability | Significant cost (storage, licences), performance overhead |
| Strict API compatibility | Slower evolution; carrying legacy fields/endpoints |
| Feature flags | Code complexity, combinatorial testing, stale flags |
| Highly optimised code (for performance) | Readability, harder to change |
| Eventual consistency / async (for scale) | Harder debugging and reasoning |
| Cost optimisation (spot, aggressive right-sizing) | Less headroom, more interruptions, engineering effort |

---

## 19. Anti-patterns

- **Big ball of mud**: no clear structure or boundaries.
- **Distributed monolith**: microservices that share a DB or must deploy together.
- **Resume-driven development**: adopting complex technology without need.
- **Premature abstraction / over-engineering** for imaginary requirements.
- **Golden hammer**: forcing one tool onto every problem.
- **Snowflake servers / click-ops**: manually configured infrastructure that no one can reproduce.
- **Shared mutable database** as an integration mechanism between teams.
- **Tribal knowledge**: undocumented systems only one person understands.
- **Alert fatigue** and dashboards nobody reads.
- **Logs without correlation IDs**; unstructured logs; logging secrets.
- **Big-bang rewrites** of working systems.
- **Long-lived feature branches** and painful merges.
- **Ignoring dependency updates** until a critical CVE forces a massive upgrade.
- **Flaky tests** tolerated → tests ignored.
- **No owner** for a service or dataset.
- Treating **tech debt as invisible** until velocity collapses.

---

## 20. Checklist

**Simplicity & structure**
- [ ] Architecture diagram (C4) up to date
- [ ] Clear module/service boundaries aligned with domains and teams
- [ ] Boundaries enforced (architecture tests / dependency rules)
- [ ] Number of technologies justified ("innovation tokens")

**Evolvability**
- [ ] API contracts defined (OpenAPI/Protobuf), versioning and deprecation policy
- [ ] Contract tests / schema registry compatibility checks
- [ ] Migrations automated; expand–contract for breaking schema changes
- [ ] Fast, reliable test suite; flaky tests fixed
- [ ] Feature flags with cleanup process
- [ ] Tech debt tracked and budgeted

**Operability**
- [ ] CI/CD with automated tests, security scans, progressive delivery, rollback
- [ ] Infrastructure as code; no manual production changes
- [ ] Config and secrets managed centrally, validated, versioned
- [ ] Observability: metrics (RED/USE), structured logs with trace IDs, distributed tracing
- [ ] SLO-based, actionable alerts with runbooks
- [ ] Production readiness review before launch
- [ ] Incident process and blameless postmortems

**Knowledge & ownership**
- [ ] README, runbooks, ADRs per service
- [ ] Service catalogue with clear owners
- [ ] Onboarding guide; bus factor > 1 for every critical system
- [ ] Sustainable on-call

**Sustainability**
- [ ] Automated dependency updates; vulnerability scanning; EOL tracking
- [ ] Cost visibility per service/team; unit economics tracked
- [ ] Toil measured and reduced

---

## 21. Further reading

- *Designing Data-Intensive Applications* (Kleppmann) — ch. 1 (maintainability), ch. 4 (encoding and evolution)
- *A Philosophy of Software Design* (John Ousterhout) — complexity, deep modules
- *Accelerate* (Forsgren, Humble, Kim) — DORA metrics and what drives delivery performance
- *Site Reliability Engineering* & *The SRE Workbook* (Google) — toil, monitoring, alerting, postmortems
- *Building Microservices* (Sam Newman) and *Monolith to Microservices* (Newman)
- *Domain-Driven Design* (Eric Evans) / *Implementing DDD* (Vaughn Vernon) / *Learning DDD* (Vlad Khononov)
- *Team Topologies* (Skelton & Pais)
- *Working Effectively with Legacy Code* (Michael Feathers)
- *Refactoring* (Martin Fowler) and martinfowler.com (strangler fig, branch by abstraction, tech debt quadrant)
- *Fundamentals of Software Architecture* (Richards & Ford) — architecture styles and trade-offs
- *Observability Engineering* (Majors, Fong-Jones, Miranda)
- The Twelve-Factor App — 12factor.net
- C4 model — c4model.com; ADRs — adr.github.io
- "Choose Boring Technology" (Dan McKinley)
- "Hidden Technical Debt in Machine Learning Systems" (Sculley et al., 2015)
- FinOps Foundation — finops.org; AWS Well-Architected (Operational Excellence & Cost Optimization pillars)
