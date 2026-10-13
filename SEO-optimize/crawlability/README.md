# Crawlability

> **Crawlability** = search engine bots can **discover** your URLs, are **allowed** to fetch them, and **get a fast, correct response** when they do.
> "If Googlebot started from our home page right now, could it reach every page we want found, and would our servers answer it properly?"

Crawling is the first stage of the [search pipeline](../how-search-works/README.md). A page that is never fetched can't be indexed or ranked, no matter how good it is. Most crawl problems are infrastructure problems: firewalls, redirects, status codes, server speed and links built in JavaScript. That makes crawlability mostly an engineering responsibility.

Related: [How search works](../how-search-works/README.md), [Indexability](../indexability/README.md) (what happens after the fetch), [Site architecture](../site-architecture/README.md) (internal links), [SEO monitoring](../seo-monitoring/README.md) (Crawl Stats, log analysis), [Availability](../../runtime-operational/availability/README.md) (CDN, anycast).

---

## 1. How bots discover URLs

| Source | Notes |
|---|---|
| **Links from your own pages** | The main source. Only `<a href="...">` links are reliably followed |
| **Links from other sites** | How a brand-new site gets found at all |
| **XML sitemaps** | A list of URLs you want crawled, with optional `lastmod` |
| **Manual submission** | URL Inspection "Request indexing" in Search Console; low volume only |
| **IndexNow** | Push notification of changed URLs, supported by Bing and some other engines (not Google) |

Links built only in JavaScript are a common failure:

```html
<a href="/products/red-shoes">Red shoes</a>                 <!-- followed -->
<span onclick="go('/products/red-shoes')">Red shoes</span>  <!-- not a link -->
<a onclick="go('/products/red-shoes')">Red shoes</a>        <!-- no href: not followed -->
<a href="#/products/red-shoes">Red shoes</a>               <!-- fragment: same page as "/" -->
```

Google ignores the part of a URL after `#`, so hash-based routing (`/#/products/1`) looks like one page.

---

## 2. robots.txt

A plain-text file at the root of each host (`https://shop.example/robots.txt`) that tells bots which paths they may **crawl**.

```
User-agent: *
Disallow: /cart
Disallow: /checkout
Disallow: /search?          # internal search results: infinite, low value
Allow: /

Sitemap: https://shop.example/sitemap.xml
```

Rules that cause most mistakes:

| Fact | Consequence |
|---|---|
| robots.txt controls **crawling, not indexing** | A blocked URL can still appear in results (with no snippet) if other pages link to it |
| `noindex` must be **seen** to work | If robots.txt blocks a page, Google can't read its `noindex`. To remove a page from the index, allow crawling and use `noindex` ([Indexability](../indexability/README.md)) |
| Blocking CSS and JS breaks rendering | Google can't render the page as users see it. Don't block `/assets/`, `/_next/`, `/_nuxt/` |
| It is **per host and protocol** | `www.shop.example` and `shop.example` each need their own |
| A **5xx** on robots.txt | Google treats the site as temporarily fully disallowed and pauses crawling |
| A **4xx** (except 429) on robots.txt | Treated as "no rules": everything may be crawled |
| It is public and advisory | It is not access control. Bad bots ignore it; don't list secret paths in it |

Bot-specific groups (`User-agent: Googlebot`, `User-agent: GPTBot`) let you allow search crawlers while restricting others. `Google-Extended` controls use of content for Google's AI models; it does not affect Search crawling or ranking.

---

## 3. XML sitemaps

```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://shop.example/products/red-shoes</loc>
    <lastmod>2026-10-01</lastmod>
  </url>
</urlset>
```

- Limits: **50,000 URLs** or **50 MB uncompressed** per file. Larger sites use a **sitemap index** pointing at many files (e.g. one per category or per day).
- List only **canonical, indexable, 200-status** URLs. A sitemap full of redirects and `noindex` pages teaches the engine to trust it less.
- Google **ignores `changefreq` and `priority`**. It uses `lastmod` only if it is consistently accurate, so set it from the real content-change time, not the build time.
- Reference sitemaps in robots.txt and submit them in Search Console and Bing Webmaster Tools.
- Generate sitemaps from the database, not by crawling your own site, so new products appear immediately.

A sitemap helps discovery. It does not guarantee indexing, and it doesn't replace internal links.

---

## 4. Crawl budget

Google crawls each site at a rate limited by two things:

| Factor | Meaning |
|---|---|
| **Crawl capacity limit** | How hard Google can crawl without hurting your server. It drops when responses get slow or return 5xx/429 |
| **Crawl demand** | How much Google wants to crawl: popular and frequently changing pages get more |

Google says crawl budget mainly matters for **very large sites (around a million pages or more)** or **medium-to-large sites with content that changes very often (around 10,000+ pages changing daily)**. For a 2,000-page shop it is almost never the bottleneck. Getting pages discovered and making them worth indexing matters more.

When it does matter, budget is usually wasted on:

- Faceted navigation and sort parameters producing endless URL combinations
- Session IDs and tracking parameters in URLs
- Redirect chains and soft 404s
- Infinite calendars and internal search result pages
- Duplicate hosts (`http`/`https`, `www`/non-`www`)

---

## 5. Status codes and server health

What bots do with each response:

| Response | Bot behaviour |
|---|---|
| **200** | Content processed |
| **301 / 308** | Permanent redirect: target becomes canonical over time |
| **302 / 307** | Temporary redirect: original URL usually kept |
| **404 / 410** | Page removed from the index over time (Google treats all 4xx except 429 the same way) |
| **429** | Treated like a server error: crawl rate reduced |
| **500 / 502 / 503 / 504** | Crawl rate reduced; persistent errors drop pages from the index |
| **503 + `Retry-After`** | The right way to signal planned maintenance |
| **Timeout / slow TTFB** | Fewer pages crawled per day |

The crawler is just another client. The same [availability](../../runtime-operational/availability/README.md) and [performance](../../runtime-operational/performance/README.md) work that protects users protects crawling: cache pages at the edge, keep origin TTFB low, and don't let a deploy return 500s for an hour.

---

## 6. Where Googlebot comes from

Googlebot crawls **mostly from IP addresses in the United States**. Google also does some locale-adaptive crawling from other countries, but you should design as if all crawling comes from the US.

Two common traps for sites aimed at one country:

| Trap | What happens | Fix |
|---|---|---|
| **Geo-blocking** non-Vietnam IPs at the firewall or CDN (often added to stop attacks) | Googlebot gets 403 or timeouts; pages drop out of the index | Allow **verified** search bots before the country rule; rate-limit instead of blocking |
| **Redirecting by IP** (foreign visitors → `/en/`) | Googlebot is always "foreign", so it only ever sees the English page | Don't force redirects. Show a dismissible language banner and use `hreflang` ([International SEO](../international-seo/README.md)) |

Serving bots different content from users on purpose is cloaking ([Indexability](../indexability/README.md)). The goal is the opposite: make sure bots get exactly what users get.

### 6.1 Verify bots, don't trust the user agent

Anyone can send `User-Agent: Googlebot`. To verify:

```
$ host 66.249.66.1
1.66.249.66.in-addr.arpa domain name pointer crawl-66-249-66-1.googlebot.com.
$ host crawl-66-249-66-1.googlebot.com
crawl-66-249-66-1.googlebot.com has address 66.249.66.1     ← must match the original IP
```

1. Reverse DNS lookup of the IP must end in `googlebot.com`, `google.com` or `googleusercontent.com`.
2. A forward lookup of that name must return the same IP.

Or match against Google's published IP range lists (JSON files in Search Central). Bing documents the same approach for `search.msn.com`.

### 6.2 With a CDN or WAF

CDNs and firewalls block bots too eagerly as often as they let bad ones through.

- **Cloudflare**: verified search bots are recognised (the verified bots list, usable in WAF rule expressions). Put an **allow / skip** rule for verified bots *above* country blocks, rate limits and challenge rules. Bot Fight Mode and JavaScript challenges can block crawlers if they're applied to everything.
- **Any WAF**: log blocked requests by user agent and check whether real (verified) search bots appear in them.
- **Test** from outside your country: use the URL Inspection live test in Search Console, which fetches from Google's own infrastructure.

---

## 7. Worked example: the shop that vanished

A Hanoi shop's organic traffic falls 80% over three weeks. Nothing changed in the content.

| Step | Finding |
|---|---|
| Search Console → Pages | "Server error (5xx)" and "Blocked due to access forbidden (403)" rising since the 1st |
| Search Console → Crawl Stats | Crawl requests fell from ~3,000/day to ~200/day; response codes mostly 403 |
| Change log | On the 1st, ops added a Cloudflare rule: block all countries except Vietnam, after a DDoS |
| URL Inspection → live test | "Failed: blocked (403)" |
| Access log | Verified Googlebot IPs (US) receiving 403 |

Fix: add a rule *before* the country block that skips it for verified bots, replace the country block with rate limiting, then request reindexing of key pages. Crawl rate recovered within days; rankings over the following weeks. The lesson: **any change to firewall, CDN or redirect rules is an SEO change** and needs a check from outside the country.

---

## 8. Anti-patterns

- Geo-blocking or IP-redirecting without an exception for verified search bots.
- Navigation built from `onclick` handlers or hash routes instead of `<a href>`.
- Blocking CSS/JS in robots.txt.
- Using robots.txt to "remove" pages from Google (use `noindex` or delete them).
- A staging robots.txt (`Disallow: /`) deployed to production.
- Sitemaps listing redirects, 404s and `noindex` pages, or `lastmod` set to the build time.
- Returning 200 with an error page during outages instead of 503.
- Trusting the user agent string to identify Googlebot.

---

## 9. Checklist

- [ ] Every important page is reachable from the home page by `<a href>` links
- [ ] robots.txt is valid, doesn't block CSS/JS, and returns 200 (never 5xx)
- [ ] Sitemaps generated from the database, contain only canonical 200 URLs, with accurate `lastmod`
- [ ] Sitemaps submitted in Search Console and Bing Webmaster Tools
- [ ] Verified search bots bypass country blocks, challenges and aggressive rate limits
- [ ] No forced redirects based on IP or `Accept-Language`
- [ ] Maintenance returns 503 with `Retry-After`
- [ ] Crawl Stats and server logs monitored for 4xx/5xx to bots ([SEO monitoring](../seo-monitoring/README.md))
- [ ] Firewall, CDN and redirect changes include an SEO check

---

## 10. Exercises

1. Write a robots.txt for a shop with cart, checkout, account, internal search and faceted filters. For each line, say whether it saves crawl budget, protects nothing, or causes harm.
2. Design a sitemap generator for 2 million products that change price daily. How many files, how is `lastmod` computed, and how does it stay fast?
3. Your security team wants to block all traffic from outside Vietnam. Write the rule order you would propose for Cloudflare and explain each rule.
4. Explain why a page blocked in robots.txt can still appear in Google results, and what you would do to really remove it.

---

## 11. Further reading

Google Search Central: "Overview of crawling and indexing", "robots.txt introduction and specifications", "Build and submit a sitemap", "Large site owner's guide to managing your crawl budget", "How HTTP status codes and network errors affect Google Search", "Verifying Googlebot and other Google crawlers", "Locale-adaptive pages"; RFC 9309 (Robots Exclusion Protocol); sitemaps.org protocol; Cloudflare documentation on verified bots and WAF custom rules; Bing Webmaster Guidelines and IndexNow.
