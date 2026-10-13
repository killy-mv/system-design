# Indexability: hands-on practice notes

Planned labs for learning indexability by **comparing what `seo-shop` serves with what Google keeps, then removing every reason for the difference**. Theory is in [README.md](README.md).

All labs use the shared **`seo-shop`** site (described in the [how-search-works notes](../how-search-works/example-notes.md)) on its real domain, verified in Search Console. Index changes take days to weeks to show up, so start lab 1 early and run the others while waiting.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** crawl it like a bot (crawlability lab 1), clean URLs and redirects (site-architecture labs 1 and 3), Core Web Vitals of SSR vs CSR (page-experience).

---

## Labs

### 1. Same page, four rendering strategies ⭐
- **Build:** render the product page four ways behind a route flag: CSR, SSR, SSG and ISR (revalidate every 5 minutes).
- **Check:** for each, `curl` the HTML and grep for the title, price and related-product links. Run URL Inspection's live test and compare the rendered HTML and screenshot. Record TTFB and LCP too.
- **Then:** keep CSR only for cart and account; ship SSR or ISR for public routes.
- **See:** README §4.

### 2. Kill the staging noindex
- **Do:** confirm the leftover `noindex` with URL Inspection.
- **Build:** a CI test that fetches the built pages for each public template and fails if it finds `noindex` or a missing canonical.
- **See:** README §2.

### 3. Canonicalise everything ⭐
- **Do:** list every URL variant for one product (protocol, host, slash, case, `sort`, `utm_*`, category paths).
- **Fix:** 301 host/protocol/slash variants at nginx or Cloudflare; self-referencing absolute canonical on every page; strip tracking parameters from internal links; sitemap lists only canonicals.
- **Check:** after two weeks, the Pages report count for "Duplicate, Google chose different canonical than user" should fall.
- **See:** README §3.

### 4. Soft 404s
- **Do:** delete 20 products and change 10 to "replaced by" a newer model.
- **Fix:** return 410 for the deleted ones, 301 to the replacement for the others, and a real 404 status from the SSR server for unknown routes (not a 200 rendered "not found" view).
- **Check:** `curl -I` each case; Pages report soft-404 count.
- **See:** README §5.

### 5. Tame the facets
- **Do:** crawl with facets enabled and count unique URLs.
- **Fix:** choose up to 10 valuable facet pages (e.g. "giày chạy bộ nam", men's running shoes) with clean URLs and unique titles; `noindex` the rest; stop emitting crawlable links for multi-filter combinations; back infinite scroll with `?page=N` links.
- **Check:** crawl again and compare URL counts and duplicate clusters.
- **See:** README §6.

---

## Progress

- [ ] 1. Same page, four rendering strategies
- [ ] 2. Kill the staging noindex
- [ ] 3. Canonicalise everything
- [ ] 4. Soft 404s
- [ ] 5. Tame the facets
