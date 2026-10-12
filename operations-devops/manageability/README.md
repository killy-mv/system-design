# Manageability

> **Manageability** (or **administrability**) = how easy it is for operators to **run the system day to day**: provision it, upgrade it, scale it, patch it, back it up, rotate secrets, and keep many environments consistent, with as little manual work as possible.
> "How many hours a week does it take to keep this running, and how much of that is a human typing commands?"

Related: [Deployability](../deployability/README.md), [Configurability](../configurability/README.md), [Maintainability §11](../../development-evolution/maintainability/README.md#11-operability-deployment-configuration-automation) (operability overview), [Efficiency](../../runtime-operational/efficiency/README.md) (cost management).

---

## 1. Toil

Google SRE defines **toil** as operational work that is **manual, repetitive, automatable, reactive, without lasting value, and growing with the size of the service**. Examples: restarting stuck jobs, manually renewing certificates, creating accounts by hand, resizing disks, copying data between environments.

Rule of thumb from Google SRE: keep toil **below 50%** of an operations engineer's time; the rest goes to engineering that removes toil.

### Worked example: is it worth automating?

Manual TLS certificate renewal: **30 minutes** per certificate, **200** certificates per year.

```
Manual: 200 × 0.5 h = 100 hours/year, plus the risk of a forgotten one expiring (an outage)
Automation (cert-manager / ACME): ~2–3 days to set up, near zero afterwards
```

It pays for itself in the first year, and removes a whole class of outages. The xkcd "Is It Worth the Time?" table is a good quick check; also count the **risk** of the manual version, not just the hours.

---

## 2. Practices

| Area | Practice |
|---|---|
| **Provisioning** | Infrastructure as code (Terraform/OpenTofu, Pulumi); no click-ops in production consoles |
| **Environments** | Same modules for dev/staging/prod; ephemeral preview environments per PR |
| **Cluster/app state** | GitOps (Argo CD, Flux): desired state in Git, drift detected automatically |
| **Upgrades and patching** | Automated OS/base-image patching, dependency update bots (Renovate, Dependabot), managed services where possible, rolling node upgrades |
| **Certificates and secrets** | Automatic issuance and renewal (cert-manager, ACME), secret rotation, expiry alerts |
| **Backups** | Scheduled, monitored, restore-tested automatically ([Recoverability](../../runtime-operational/recoverability/README.md)) |
| **Scaling** | Autoscaling instead of manual resizing ([Elasticity](../../runtime-operational/elasticity/README.md)) |
| **Runbooks → automation** | Every repeated runbook becomes a script, then a button, then an automatic response |
| **Admin interfaces** | CLIs/APIs for operational tasks instead of direct DB edits; audited and permissioned |
| **Policy as code** | Guardrails (OPA/Gatekeeper, Kyverno, cloud policies): no public buckets, required tags, resource limits |
| **Self-service platform** | Golden paths and templates (Backstage, internal developer platform) so teams create services, databases and pipelines safely without tickets |
| **Inventory** | Know what runs where, who owns it, which version (service catalog, tags) |

**Managed services** (managed databases, queues, Kubernetes control planes) trade money and some control for a lot of manageability: patching, backups and failover become the provider's job.

---

## 3. Anti-patterns

- "Pet" servers configured by hand that nobody dares to rebuild.
- Changes made in the cloud console, not in code (drift).
- Runbooks with 30 manual steps executed under pressure.
- One person who knows how to do the upgrade.
- Patching and upgrades postponed until they become huge, risky projects.
- No owner or tags on resources.

---

## 4. Checklist

- [ ] All infrastructure in code; production changes only through the pipeline
- [ ] Environments created from the same templates; drift detection
- [ ] Automated patching, dependency updates, certificate renewal, secret rotation
- [ ] Backups automated and restore-tested
- [ ] Toil measured; top toil items have automation tickets
- [ ] Admin tasks via audited tools, not direct DB/console access
- [ ] Service catalog with owners

---

## 5. Exercises

1. List every manual operational task for a system you run, with frequency and time per occurrence. Rank by hours/year × risk. Automate the top one.
2. Could you rebuild your production environment from scratch, from code, today? What's missing?
3. Design a self-service flow for "a team needs a new Postgres database" that's safe (backups, monitoring, access control included by default).
4. Compare running Kafka yourself vs a managed Kafka service: which operational tasks move to the provider?

---

## 6. Further reading

Google SRE book ch. 5 "Eliminating Toil" and ch. 7 "The Evolution of Automation at Google"; *Infrastructure as Code* (Kief Morris); *Team Topologies* (platform teams); *The Phoenix Project* / *The DevOps Handbook*; Backstage documentation; xkcd 1205 "Is It Worth the Time?".
