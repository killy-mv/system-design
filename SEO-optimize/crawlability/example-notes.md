# Crawlability: hands-on practice notes

Planned labs for learning crawlability by **finding every reason a bot can't reach `seo-shop`'s pages, then fixing the links, rules and responses one by one**. Theory is in [README.md](README.md).

All labs use the shared **`seo-shop`** site (described in the [how-search-works notes](../how-search-works/example-notes.md)), deployed on a real domain behind Cloudflare so Google can crawl it, with nginx access logs kept for at least 30 days.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** toy crawler and bot log reading (how-search-works labs 1 and 5), rendering strategies (indexability lab 1), monitoring Crawl Stats (seo-monitoring).

---

## Labs

### 1. Crawl it like a bot ⭐
- **Do:** crawl the site with Screaming Frog (JavaScript rendering off) or your toy crawler from how-search-works lab 1. Count URLs found vs products in the database.
- **Fix:** replace `onclick` navigation and hash routes with real `<a href>` links and path-based routes.
- **Check:** crawl again; the found/total ratio should be close to 100%.
- **See:** README §1.

### 2. robots.txt and sitemaps
- **Fix:** stop blocking `/assets/`; disallow cart, checkout and internal search. Generate sitemaps from the database (split per category), with `lastmod` from the product's `updated_at`.
- **Check:** every sitemap URL returns 200 and is canonical (write a script). Submit in Search Console and Bing Webmaster Tools and watch the "discovered" counts.
- **Then:** deploy a robots.txt that returns 500 for an hour on a test subdomain and note what Crawl Stats shows.
- **See:** README §2 and §3.

### 3. The geo-block trap ⭐
- **Do:** with the "block non-Vietnam IPs" rule on, run URL Inspection's live test. Then request a page through a US VPN.
- **Fix:** add a Cloudflare rule that skips the country block, challenges and rate limits for verified bots, placed first. Replace the IP-based redirect to `/en/` with a dismissible banner.
- **Check:** live test passes; nginx logs show 200s to verified Googlebot IPs; foreign visitors see the page they asked for.
- **See:** README §6.

### 4. Verify bots
- **Build:** a script that reads the access log, takes every request claiming to be Googlebot, Bingbot or coccocbot, and verifies it by reverse + forward DNS (or Google's IP range JSON).
- **Check:** what share of "Googlebot" requests are fake? Which paths do fake bots hit?
- **See:** README §6.1.

### 5. Server health as seen by bots
- **Do:** add a slow endpoint (2 s TTFB) for category pages and a deploy that returns 500s for 10 minutes. Watch Crawl Stats (average response time, host status) over the following days.
- **Fix:** cache category HTML at the edge; return 503 with `Retry-After` during deploys; remove redirect chains found by the crawler.
- **See:** README §4 and §5.

---

## Progress

- [ ] 1. Crawl it like a bot
- [ ] 2. robots.txt and sitemaps
- [ ] 3. The geo-block trap
- [ ] 4. Verify bots
- [ ] 5. Server health as seen by bots
