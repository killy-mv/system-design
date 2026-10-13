# Site architecture: hands-on practice notes

Planned labs for learning site architecture by **redesigning `seo-shop`'s URLs and links, then migrating to them without losing a single old URL**. Theory is in [README.md](README.md).

All labs use the shared **`seo-shop`** site (described in the [how-search-works notes](../how-search-works/example-notes.md)). Before starting, save a full crawl of the current site and a copy of its sitemap and access logs: lab 3 needs the old URL list.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** crawl it like a bot (crawlability lab 1), canonicalise everything (indexability lab 3), breadcrumb markup (structured-data), backward-compatible APIs (backward-compatibility).

---

## Labs

### 1. Design the URL scheme ⭐
- **Write:** a one-page URL spec: product, category, brand, blog post, pagination, filters; slug rules (lowercase, hyphens, ASCII without Vietnamese diacritics, ID suffix); trailing-slash rule.
- **Build:** slug generation from product names (`Giày chạy bộ Runner X` → `giay-chay-bo-runner-x-123`) and lookup by ID that 301s to the current slug when the name changes.
- **Check:** rename a product and request its old URL.
- **See:** README §1.

### 2. Click depth and orphans
- **Do:** crawl from the home page and record click depth for every product. Export the product list from the database and the list of product URLs that received traffic.
- **Fix:** add category hub pages, breadcrumbs and server-rendered related-product links until every product is within 3 clicks; link the orphans you found.
- **Check:** depth histogram before and after; orphan count.
- **See:** README §2, §3 and §4.

### 3. Migrate without losing URLs ⭐
- **Build:** generate the redirect map from the database (old pattern → new URL, or 410), and serve it as an nginx `map` or Cloudflare bulk redirects.
- **Check:** a script that requests every old URL (from the saved crawl, sitemap and logs) and asserts exactly one 301 to a 200, or the planned 410. No chains, no redirects to the home page.
- **Then:** after launch, watch Search Console for 404s and the access log for old URLs still requested; patch the map.
- **See:** README §5 and §7.

### 4. One domain, two stacks
- **Build:** run the blog as a separate service (e.g. a static site generator or WordPress container) and serve it at `/blog/` through path-based routing in nginx or Cloudflare.
- **Check:** shared header links, one sitemap index covering both, one Search Console property.
- **See:** README §6 and the [Hostinger case study](../../case-study/hostinger/README.md).

---

## Progress

- [ ] 1. Design the URL scheme
- [ ] 2. Click depth and orphans
- [ ] 3. Migrate without losing URLs
- [ ] 4. One domain, two stacks
