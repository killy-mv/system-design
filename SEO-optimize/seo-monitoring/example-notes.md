# SEO monitoring: hands-on practice notes

Planned labs for learning SEO monitoring by **breaking the site the way real releases do, and building the checks, logs and alerts that catch it before Google does**. Theory is in [README.md](README.md).

These labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)). Its relevant problems: a leftover `noindex`, missing titles, no sitemap, and robots.txt controlled by an environment variable. Search Console labs need a real domain you own (a cheap domain pointing at a small VPS is enough). Log labs need the nginx or CDN access logs. Use Prometheus and Grafana, or any stack you already know, for alerts.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** RUM with `web-vitals` ([Page experience lab 1](../page-experience/example-notes.md)), hreflang checker ([International SEO lab 3](../international-seo/example-notes.md)), alerting labs ([Monitorability](../../operations-devops/monitorability/example-notes.md)).

---

## Labs

### 1. Set up Search Console ⭐
- **Do:** verify a Domain property via DNS. Submit the sitemap. Inspect the home page and one product page with URL Inspection (live test) and compare the rendered HTML with what you expect.
- **Then:** after a week, read Performance, Page indexing and Crawl Stats. Write down three facts you didn't know.
- **See:** README §2.

### 2. Crawler log analysis
- **Build:** a script (or SQL over logs loaded into DuckDB) that finds crawler requests, **verifies** Googlebot with reverse and forward DNS, and reports requests per day by status code, path prefix and response time.
- **Check:** fake a request with a Googlebot user agent from your laptop; it must be classified as unverified.
- **Then:** what share of Googlebot hits go to parameter URLs, redirects or 404s?
- **See:** README §3.

### 3. SEO regression checks in CI ⭐
- **Build:** a test that loads one representative URL per template from a preview build and asserts: status 200, no `noindex` (meta or header), self-referencing absolute canonical, unique non-empty title, main content present in the server HTML, valid JSON-LD. Add a robots.txt diff against an approved copy.
- **Check:** introduce each bug on purpose (set the staging robots flag, delete a title, point canonical at the home page). Every one must fail the build.
- **See:** README §6.

### 4. Alerts on the production site
- **Build:** a synthetic check every 15 minutes for robots.txt hash and status, `noindex` on key pages, and 5xx rate for verified crawlers from the logs. Route alerts with a short runbook.
- **Check:** deploy the staging robots.txt to production (on your lab domain) and time how long the alert takes.
- **See:** README §5.

### 5. Explain a traffic drop
- **Do:** pull Search Console data via the API into a table. Add release annotations from your git tags and dates of announced Google updates from the Search Status Dashboard.
- **Then:** pick any dip (or simulate one by breaking a section for a week) and write a short incident note following README §7: evidence for a release, an update, seasonality, or tracking.
- **See:** README §7.

---

## Progress

- [ ] 1. Set up Search Console
- [ ] 2. Crawler log analysis
- [ ] 3. SEO regression checks in CI
- [ ] 4. Alerts on the production site
- [ ] 5. Explain a traffic drop
