# How search works: hands-on practice notes

Planned labs for learning how search works by **building a toy search engine, then watching a real one discover, index and rank a deliberately broken site**. Theory is in [README.md](README.md).

## The shared practice site: `seo-shop`

All SEO-optimize labs use the same deliberately badly optimised storefront, built once:

- A **public storefront** for the existing `messy-shop` backend (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)): home, category pages, product pages, search, a small blog.
- Built with a framework that can do **both client-side and server-side rendering** (Next.js or Nuxt), so you can switch strategies per route.
- Run with **Docker Compose** behind **nginx** locally, and behind the **Cloudflare free plan** on a real, cheap domain so Google can actually crawl it.
- Seeded with ~500 fake products (Faker), Vietnamese and English descriptions, prices in VND.
- Written **badly on purpose**:
  - client-side-only rendering: the first HTML response is an empty `<div id="app">`
  - product URLs like `/p?id=123&sort=price`
  - faceted filters (colour, size, price, sort) that combine into an unlimited number of URLs
  - the same page served at `/shoes`, `/shoes/`, `http://` and `https://`
  - missing or identical `<title>` tags
  - a `<meta name="robots" content="noindex">` left over from staging
  - `robots.txt` blocking `/assets/` (CSS and JS)
  - 3–5 MB unoptimised hero images
  - an IP-based redirect sending foreign visitors to an English page
  - a firewall rule blocking all non-Vietnam IPs
  - no sitemap

**Tools:** Lighthouse and PageSpeed Insights, Google Search Console and Bing Webmaster Tools (on the real domain), Screaming Frog (free up to 500 URLs) or your own small crawler, `curl`, and the nginx access log.

Keep a `CHANGELOG-labs.md`: for each lab, what you changed and the before/after numbers (pages indexed, Lighthouse scores, crawl requests per day).

**Planned folder layout:**

```
SEO-optimize/labs/
  seo-shop/                ← the shared site
SEO-optimize/how-search-works/labs/
  01-toy-search-engine/README.md
  ...
```

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** web crawler design (scalability §18.7), Core Web Vitals (performance lab 13).

---

## Labs

### 1. Toy search engine ⭐
- **Build:** a crawler (≈100 lines) that starts from `seo-shop`'s home page, follows `<a href>` links on the same host, and stores page text. Build an inverted index (term → sorted list of doc IDs) and answer AND queries by merging posting lists.
- **Check:** which products did your crawler never find? (Client-side rendering and JS-only links will hide most of them. Keep the list for the crawlability labs.)
- **See:** README §2 and §3.

### 2. Rank the results
- **Build:** add TF-IDF or BM25 scoring, then a title-match boost.
- **Then:** compute PageRank over your crawled link graph (power iteration, d = 0.85). Combine it with the text score and compare result order with and without it.
- **See:** README §4.

### 3. Compare with what Google sees ⭐
- **Do:** deploy `seo-shop` on the real domain, verify it in Search Console, and submit the home page with URL Inspection. Look at the "crawled page" HTML and screenshot.
- **Check:** compare with `curl` output and with your browser. Write down every difference and the stage (crawl, render, index) that caused it.
- **See:** README §2.

### 4. Results depend on context
- **Do:** pick five queries your shop should rank for. Search them from a phone on Viettel 4G, a laptop on home fibre, and through a VPN in the US and Japan. Try Google, Bing and Cốc Cốc.
- **Write:** a table of differences, with the signal (location, language, device, engine) behind each.
- **See:** README §6 and §7.

### 5. Read the bots in your logs
- **Do:** after two weeks on the real domain, parse the nginx access log. Count requests per bot (Googlebot, Bingbot, coccocbot, AI crawlers, fake bots) and per URL pattern.
- **Check:** which pages does each bot fetch most? Which important pages has no bot fetched yet?
- **See:** [Crawlability](../crawlability/README.md), [SEO monitoring](../seo-monitoring/README.md).

---

## Progress

- [ ] 1. Toy search engine
- [ ] 2. Rank the results
- [ ] 3. Compare with what Google sees
- [ ] 4. Results depend on context
- [ ] 5. Read the bots in your logs
