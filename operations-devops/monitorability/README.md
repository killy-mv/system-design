# Monitorability

> **Monitorability** = how well the system's health can be **watched continuously** and how reliably the **right people are alerted** when users are affected.
> "If checkout starts failing at 3am, who finds out first: us, or our customers on social media?"

Related: [Observability](../observability/README.md) (the telemetry that monitoring uses; monitoring covers *known* failure modes, observability helps with *unknown* ones), [Availability §1.4](../../runtime-operational/availability/README.md#14-sli-slo-sla-error-budget) (SLI, SLO, error budget), [Availability §13](../../runtime-operational/availability/README.md#13-operations-monitoring-alerting-incident-response) (incident response).

---

## 1. What to monitor

| Layer | Examples | Why |
|---|---|---|
| **User experience** | Request success rate, latency at the edge, synthetic checks, RUM | This is what matters; alert here |
| **Services** | RED per service and endpoint | Locate which service is the problem |
| **Resources** | USE: CPU, memory, disk, pools, queues | Find saturation before it hurts |
| **Dependencies** | Error rate/latency of DB, cache, third-party APIs | "It's not us, it's the payment provider" |
| **Business** | Orders/min, sign-ups, payments by provider | Catches silent failures with no errors |
| **Platform** | Node health, pod restarts, certificate expiry, quotas, backups succeeded | Slow-burning problems |

**White-box** monitoring uses internal metrics; **black-box** monitoring tests from outside like a user (synthetic probes, uptime checks). You need both: black-box catches DNS, CDN, TLS and "everything looks fine inside but users can't reach us".

---

## 2. SLOs: the backbone of good alerting

1. **SLI**: a ratio of good events to all events, measured close to users. *"% of checkout requests that succeed in < 500 ms."*
2. **SLO**: the target over a window. *"99.9% over 30 days."*
3. **Error budget**: 100% − SLO. 0.1% of 30 days ≈ **43 minutes** of badness per month.

Alert on how fast you're **spending the budget**, not on raw thresholds.

### Worked example: burn-rate alerts

**Burn rate** = how fast you consume the budget relative to "exactly on target". Burn rate 1 = the budget runs out exactly at the end of 30 days.

```
SLO 99.9% over 30 days (720 h). Error budget = 0.1%.
Current error rate 1.44%  →  burn rate = 1.44 / 0.1 = 14.4
At burn rate 14.4, 1 hour consumes 14.4 / 720 = 2% of the monthly budget
→ the whole budget is gone in ~50 hours
```

The Google SRE workbook's recommended **multi-window, multi-burn-rate** setup:

| Severity | Burn rate | Long window | Short window | Budget consumed |
|---|---|---|---|---|
| **Page** | 14.4 | 1 h | 5 min | 2% in 1 hour |
| **Page** | 6 | 6 h | 30 min | 5% in 6 hours |
| **Ticket** | 1 | 3 days | 6 h | 10% in 3 days |

The **short window** makes the alert stop quickly once the problem is fixed. This catches both fast outages and slow leaks, with very few false alarms.

---

## 3. Alerting principles

- **Alert on symptoms, not causes.** "Checkout success rate dropping" pages someone; "CPU 85%" is at most a dashboard panel or a ticket.
- **Every page must be actionable and urgent.** If the right response is "wait and see", it's not a page.
- **Page vs ticket vs log**: page (needs a human now), ticket (needs a human this week), dashboard only (useful context).
- **Every alert links to a runbook** and a dashboard.
- **Route by ownership**: the team that owns the service gets the alert.
- **Measure alert quality**: pages per shift, % actionable, false positives. Delete or fix noisy alerts. **Alert fatigue** is how real alerts get ignored.
- **Deduplicate and group**: one incident should page once, not 200 times.

Also alert on things that fail silently and slowly: **certificate expiry**, **backup failures**, **disk/quotas filling**, **queue age**, **"no data"** (a metric that stopped reporting is often the worst sign).

---

## 4. Dashboards

- **Top-level**: SLOs and error budgets per user journey (login, search, checkout). One glance = "are users OK?".
- **Service dashboards**: RED + dependencies + saturation + deploy markers, the same layout for every service.
- **Drill-down**: from a journey to services to instances.
- Fewer, consistent dashboards beat hundreds of one-off ones. Keep them in code (Grafana as code, Jsonnet, Terraform).

---

## 5. Monitoring the monitoring

- **Dead man's switch / watchdog**: an alert that always fires; if it *stops* arriving, the alerting pipeline is broken.
- Run monitoring and alerting in a **separate failure domain** from what it watches.
- Probe from **outside** (multiple regions, a third-party uptime service).
- Test alerts: fire them on purpose (game days, chaos tests).

---

## 6. On-call and response

- A sustainable rotation (enough people, follow-the-sun if possible, compensation).
- Clear **escalation** and **severity** levels.
- **Status page** for customers; incident channel for the team.
- Track: MTTD, MTTA (acknowledge), MTTR, pages per week.
- Postmortems that improve alerts (missing alert? noisy alert?).

---

## 7. Anti-patterns

- Alerting on every metric threshold (CPU, memory, disk) → hundreds of pages, none meaningful.
- No alert on user-facing symptoms; customers report outages first.
- Alerts without runbooks or owners.
- Monitoring hosted on the same infrastructure it monitors.
- Dashboards nobody looks at; no top-level "are users OK?" view.
- Ignoring "no data" conditions.

---

## 8. Checklist

- [ ] SLIs and SLOs for each critical user journey
- [ ] Multi-window burn-rate alerts for pages; tickets for slow burns
- [ ] Black-box synthetic checks from multiple locations
- [ ] Every page actionable, owned, with runbook and dashboard links
- [ ] Alerts for silent killers: certificates, backups, quotas, queue age, missing data
- [ ] Dead man's switch; monitoring in a separate failure domain
- [ ] Alert quality reviewed regularly (noise, false positives)

---

## 9. Exercises

1. Write SLIs and SLOs for three journeys of an e-commerce site. Which share an SLO and which need their own?
2. With a 99.5% SLO over 28 days, what burn rate consumes 5% of the budget in 6 hours? What error rate is that?
3. Review the alerts of a system you know (or a template like a default Prometheus/Kubernetes alert pack). Classify each as page, ticket or delete.
4. List 5 ways your system could fail silently (no errors, no alerts). Add a monitor for each.

---

## 10. Further reading

Google SRE book ch. 4 "Service Level Objectives", ch. 6 "Monitoring Distributed Systems"; SRE workbook ch. 2 "Implementing SLOs" and ch. 5 "Alerting on SLOs"; Rob Ewaschuk, "My Philosophy on Alerting"; *Implementing Service Level Objectives* (Alex Hidalgo); Sloth / Pyrra (SLO generators for Prometheus).
