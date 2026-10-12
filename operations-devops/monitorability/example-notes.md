# Monitorability: hands-on practice notes

Planned labs for learning monitoring by **defining SLOs, building alerts, and checking that they fire for real problems and stay quiet otherwise**. Theory is in [README.md](README.md).

They reuse the **demo system and stack** from the observability notes (OTel + Prometheus + Grafana), plus **Alertmanager** and an uptime checker.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** measure availability with a probe (availability lab 1), metrics done right (observability lab 2).

---

## Labs

### 1. SLOs and burn-rate alerts ⭐
- **Build:** SLIs for "checkout success" and "checkout latency < 500 ms" in Prometheus. Generate multi-window burn-rate rules with **Sloth** or **Pyrra** (or write them by hand from README §2).
- **Break:** inject 2% errors → the fast-burn page should fire within minutes. Inject 0.2% errors → only the slow-burn ticket should fire, hours later.
- **Then:** fix the fault and check that the alert resolves quickly (thanks to the short window).

### 2. Alert routing and noise
- **Build:** **Alertmanager** routing pages to one channel (a test Slack/Discord webhook or ntfy) and tickets to another, grouped by service, with inhibition (no "high latency" page when "service down" is already firing).
- **Break:** add naive threshold alerts (CPU > 70%, memory > 80%) and run a day of normal load spikes. Count pages.
- **Fix:** delete or demote them. Compare the number of pages.

### 3. Black-box monitoring
- **Build:** **Blackbox exporter** (or Uptime Kuma, or a free external uptime service) checking the public endpoint, TLS certificate expiry and a multi-step synthetic flow (k6 browser or Playwright on a schedule).
- **Break:** stop Nginx (internal metrics look fine because the app is up) and serve an expired certificate. Which alerts catch them?

### 4. Silent failures
- **Build:** monitors for: last successful backup age, queue oldest-message age, a metric that stops reporting (`absent()`), business metric drop (orders/min vs same time last week).
- **Break:** stop the backup job, stop a consumer, kill the metrics exporter, make checkout return 200 with an empty order. Check each is caught.

### 5. Monitor the monitoring
- **Build:** a Watchdog alert that always fires, sent to a dead man's switch service (e.g. healthchecks.io).
- **Break:** stop Prometheus or Alertmanager. You should get notified by the *external* service.

---

## Progress

- [ ] 1. SLOs and burn-rate alerts
- [ ] 2. Alert routing and noise
- [ ] 3. Black-box monitoring
- [ ] 4. Silent failures
- [ ] 5. Monitor the monitoring
