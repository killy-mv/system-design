# SEO monitoring

> **SEO monitoring** = continuously measuring how search engines crawl, index and rank your site and how much traffic it brings, and catching regressions before they cost weeks of traffic.
> "If a release accidentally adds `noindex` to every product page tonight, how soon do we find out, and from where?"

SEO problems are slow and quiet. A broken page throws errors immediately; a broken canonical or a `Disallow: /` in robots.txt only shows up days or weeks later as pages dropping out of the index and traffic sliding. Recovery also takes time, because search engines have to recrawl. Good monitoring catches the cause (the release) rather than the symptom (lost traffic).

Related: [Crawlability](../crawlability/README.md), [Indexability](../indexability/README.md), [Page experience](../page-experience/README.md), [Monitorability](../../operations-devops/monitorability/README.md) (alerting principles), [Observability](../../operations-devops/observability/README.md) (RUM, logs), [Deployability](../../operations-devops/deployability/README.md) (checks in the pipeline).

---

## 1. What to watch, layer by layer

| Layer | Question | Main sources |
|---|---|---|
| **Crawling** | Can search engines fetch our pages, fast and without errors? | Search Console Crawl Stats, server logs, robots.txt monitoring |
| **Indexing** | Are the pages we want in the index, and only those? | Search Console Page indexing, URL Inspection, sitemaps report |
| **Rendering and markup** | Do crawlers see the content, titles, canonicals and structured data we intend? | URL Inspection (rendered HTML), Rich results reports, CI checks |
| **Experience** | Are real users getting good Core Web Vitals? | Search Console Core Web Vitals, CrUX, your own RUM |
| **Visibility** | Are we shown for the queries we care about? | Search Console Performance (impressions, average position), rank trackers |
| **Outcome** | Do searchers click and convert? | Search Console clicks and CTR, web analytics (organic segment) |

Watch from the top down when something looks wrong: traffic falls (outcome), but the cause is usually lower (an indexing or crawling change).

---

## 2. Google Search Console

Free, and the only first-party view of how Google sees your site. Verify the **Domain property** (DNS verification) so all subdomains and protocols are covered.

| Report | Shows | Use it for |
|---|---|---|
| **Performance** | Clicks, impressions, CTR and average position by query, page, country, device, search appearance, date | Traffic trends, which queries and pages win or lose, country breakdown for [international](../international-seo/README.md) sites |
| **Page indexing** | Indexed vs not indexed URLs, with reasons ("Excluded by noindex", "Duplicate without user-selected canonical", "Crawled – currently not indexed", "Soft 404", server errors) | Spotting sudden changes in indexed count and why |
| **Sitemaps** | Submitted sitemaps, discovered URLs, read errors | Checking that generated sitemaps are fetched |
| **URL Inspection** | For one URL: indexed or not, Google-selected canonical, last crawl, rendered HTML and screenshot (live test) | Debugging a specific page; checking what Googlebot actually renders |
| **Core Web Vitals** | Groups of URLs rated good, needs improvement, poor, for mobile and desktop, from CrUX | Finding page templates that fail ([Page experience](../page-experience/README.md)) |
| **Crawl Stats** (under Settings) | Total crawl requests, download size, **average response time**, breakdown by response code, file type, purpose and **Googlebot type**, host status (robots.txt fetch, DNS, connectivity) | Seeing whether the server is slow or erroring for Googlebot |
| **Enhancements / rich results** | Valid and invalid items per structured-data type | Catching broken structured data after releases |
| **Manual actions, Security issues** | Penalties and hacks | Should always be empty; check after any traffic collapse |

Things to know:

- **Data is delayed** (typically a couple of days) and some reports are sampled or limited to the top rows. The UI export is capped; use the Search Console API or the BigQuery bulk export for full data.
- **Average position** is an average across all impressions, including personalised and local results. A page ranking 3rd in Hanoi and 30th in Ho Chi Minh City shows a blended number.
- Anonymised queries are omitted from query-level data for privacy, so query rows don't sum to the total.

Other engines have their own tools: **Bing Webmaster Tools** (also covers results that use Bing's index) and, for the Vietnamese market, Cốc Cốc has offered webmaster tools; check what is currently available.

---

## 3. Server logs: what crawlers really do

Search Console tells you Google's summary. Your access logs tell you every request.

```
66.249.66.1 - - [13/Oct/2026:03:12:45 +0700] "GET /vi/ao-thun-trang HTTP/2" 200 18342 "-" "Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) ... (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
```

What to extract:

| Question | How |
|---|---|
| Which sections does Googlebot crawl, and how often? | Group crawler requests by path prefix and day |
| Is crawl budget wasted? | Share of crawler hits on parameter URLs, faceted filters, redirects, 404s |
| Are crawlers getting errors or slow responses? | Status codes and response times for crawler requests vs users |
| Are new pages discovered quickly? | Time from publishing to first Googlebot hit |
| Is that really Googlebot? | **Verify**: reverse DNS lookup resolves to `googlebot.com` or `google.com`, and forward lookup matches the IP; or compare against Google's published IP ranges. User agents are trivially faked |

With a CDN, crawler requests served from cache never reach the origin. Use the CDN's logs (Cloudflare Logpush, CloudFront logs) for the full picture.

---

## 4. Rank tracking and its limits

Rank trackers query search engines for a keyword list and record your position.

- Results are **personalised and localised**: device, location (down to city or neighbourhood for local queries), language and history change them. A tracker reports one simulated searcher, not your users.
- Track from the **locations that matter** (Hanoi and Ho Chi Minh City separately if both matter) and on mobile.
- Treat ranks as a trend indicator for a fixed keyword set. Search Console's impressions and clicks are the ground truth for actual traffic.
- Results pages now include many features (AI-generated summaries, local packs, videos, "People also ask"). Position 1 in the web results may sit far down the screen. Some trackers record which features appear.

---

## 5. Alerts

Alert on causes you can act on, following the usual alerting rules ([Monitorability](../../operations-devops/monitorability/README.md)): few, actionable, with a runbook.

| Alert | Trigger (example) | Likely causes |
|---|---|---|
| **robots.txt changed** | Hash of production robots.txt differs from the approved version | Staging robots.txt deployed (`Disallow: /`), CMS plugin change |
| **robots.txt not fetchable** | Non-200 or 5xx | Google may pause crawling the site while robots.txt returns server errors |
| **noindex on key pages** | Synthetic check finds `noindex` (meta tag or `X-Robots-Tag` header) on home, category or top product pages | Staging flag left on; template change |
| **5xx or timeouts for crawlers** | Crawler error rate above a threshold in logs | Origin overload, firewall or bot protection blocking Googlebot |
| **Indexed pages drop** | Page indexing "indexed" count falls by more than N% week on week (via API or manual weekly check) | Canonical bug, noindex, mass 404s, redirects |
| **Organic clicks drop** | Clicks down more than N% vs the same weekday, sustained for several days | Any of the above, a Google update, seasonality |
| **Core Web Vitals regression** | RUM p75 LCP or INP crosses the "good" threshold after a release | New script, unoptimised image, slower backend |

---

## 6. SEO regression tests in CI

Most SEO incidents are self-inflicted by releases. Check the obvious things before deploy, on a preview or staging build with production settings:

| Check | Example assertion |
|---|---|
| Status codes | Key URLs return 200; old URLs return 301 to the right target; deleted products return 404 or 410 |
| Indexing directives | No `noindex` or `X-Robots-Tag: noindex` on indexable templates |
| Canonical | Each template's canonical is absolute, self-referencing (or the expected target), and returns 200 |
| Title and description | Present, unique per page, within reasonable length |
| hreflang | Reciprocal and valid ([International SEO](../international-seo/README.md)) |
| Rendered content | Main content and internal links are in the server HTML, not only after JavaScript |
| Structured data | Valid JSON-LD for the template's types |
| robots.txt | Diff against the approved file; production must not contain `Disallow: /` |
| Sitemap | Generates, valid XML, only canonical 200 URLs |
| Performance budget | Lighthouse CI thresholds for LCP, CLS, total JavaScript size |

These are cheap: a script with an HTTP client and an HTML parser, run against a list of representative URLs per template.

---

## 7. Explaining traffic changes

When organic traffic changes, work out **why** before reacting:

| Check | If it matches |
|---|---|
| Your own release log, robots.txt and CMS changes | A deploy on the day before the drop: start there |
| Search Console Page indexing and Crawl Stats | Fewer pages indexed or more errors: technical cause |
| Which pages, queries, countries and devices lost | Concentrated in one template: likely technical. Spread across the site: likely an algorithm change |
| Google Search Status Dashboard (announced core, spam and other updates) | Drop starts on a rollout date and nothing changed on your side: probably the update; improve content quality rather than technical tweaks |
| Seasonality and demand | Same period last year; Google Trends for your main queries |
| Results page changes | New features above results (AI summaries, local pack) can cut CTR while position stays the same |
| Tracking | Analytics tag removed or consent changes reduce measured traffic while Search Console clicks are unchanged |

Annotate your dashboards with releases and known Google updates so future readers can see both.

---

## 8. Anti-patterns

- Looking at Search Console only when traffic has already collapsed.
- Not verifying the domain property, so one subdomain or protocol is unmonitored.
- Trusting the Googlebot user agent in logs without verification.
- Ranking reports from one location and desktop only for a mobile-first local business.
- No checks in CI; robots.txt and noindex controlled by an environment variable nobody tests.
- Reacting to every core update with site-wide rewrites before checking for technical causes.
- Treating Lighthouse scores as Core Web Vitals.

---

## 9. Checklist

- [ ] Search Console domain property verified; owners and alerts email set; Bing Webmaster Tools set up
- [ ] Sitemaps submitted and generated automatically
- [ ] Weekly review: Performance, Page indexing, Crawl Stats, Core Web Vitals, enhancements
- [ ] Search Console data exported (API or BigQuery) for history beyond the UI limits
- [ ] Crawler requests analysed from origin and CDN logs; Googlebot verified
- [ ] Alerts on robots.txt changes, noindex on key pages, crawler 5xx, indexed-page drops, organic click drops
- [ ] SEO regression checks in CI for each page template
- [ ] RUM for Core Web Vitals per template and country
- [ ] Release and Google-update annotations on dashboards

---

## 10. Exercises

1. Open the Crawl Stats report for a site you own (or study Google's documentation screenshots). What is the average response time, and what share of requests are not 200?
2. Write a log query that counts verified Googlebot requests per day by status code and top-level path.
3. Design the CI check list for an e-commerce site with home, category, product, search and blog templates. Which checks apply to which template?
4. Organic clicks fell 25% on Tuesday. Walk through §7 in order and write down what evidence would point to a release, an algorithm update, or a tracking problem.

---

## 11. Further reading

Google Search Central documentation for Search Console (each report has a help page), "Crawl Stats report", "Verifying Googlebot and other Google crawlers" and "Debugging drops in Google Search traffic"; Google Search Status Dashboard; Search Console API and bulk data export to BigQuery; Bing Webmaster Tools documentation; Lighthouse CI; the [Monitorability](../../operations-devops/monitorability/README.md) and [Observability](../../operations-devops/observability/README.md) notes in this repo.
