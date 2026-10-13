# Page experience: hands-on practice notes

Planned labs for learning page experience by **measuring what real users and Googlebot experience, finding where the time goes, and fixing the biggest part first**. Theory is in [README.md](README.md).

These labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)). Its relevant problems: client-only rendering, a huge hero image, no CDN caching of HTML, and a firewall rule that geo-blocks non-Vietnam IPs. Tools: Lighthouse and PageSpeed Insights, WebPageTest, the `web-vitals` library, CrUX (via PageSpeed Insights or the CrUX API), `curl`, `/cdn-cgi/trace`, and the access logs.

Keep a `RESULTS.md` with before and after numbers for every change. Lab numbers are only comparable when measured the same way (same device profile, network, location).

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** Core Web Vitals audit (performance lab 13), RUM with `web-vitals` (observability labs), crawl stats and log analysis ([SEO monitoring labs](../seo-monitoring/example-notes.md)).

---

## Labs

### 1. Baseline: lab vs field ⭐
- **Do:** run Lighthouse (mobile) on the home page, a category page and a product page. Record LCP, CLS, Total Blocking Time and the LCP element.
- **Build:** add the `web-vitals` library and send LCP, INP, CLS and TTFB with the page type to a small endpoint. Generate traffic from a real phone over mobile data.
- **Check:** compare your RUM p75 values with Lighthouse. Which is worse, and why?
- **See:** README §1.

### 2. Break down LCP
- **Do:** in WebPageTest or Chrome DevTools, split LCP into TTFB, resource load delay, resource load time and render delay for the product page.
- **Fix:** the largest part first: preload or prioritise the hero image (`fetchpriority="high"`, no `loading="lazy"` on it), serve a correctly sized AVIF or WebP, server-render the main content.
- **Check:** re-measure; record each fix's effect separately.
- **See:** README §3, [Responsiveness §3.1](../../UI-UX/responsiveness/README.md#31-core-web-vitals-web).

### 3. Distance and TTFB ⭐
- **Do:** with `curl -w "%{time_connect} %{time_appconnect} %{time_starttransfer}\n"`, measure TTFB to the origin and to the CDN from a Vietnamese connection and from a US VPS. Do the same with a cold and a warm cache.
- **Then:** check `/cdn-cgi/trace` (`colo=`) from Viettel, VNPT and FPT connections if you can. Note which PoP each gets.
- **Write:** a table like README §4: location, RTT, TTFB uncached, TTFB cached.
- **See:** README §4 and §5.

### 4. Cache HTML at the edge
- **Build:** cache anonymous HTML pages at the CDN with a short TTL plus `stale-while-revalidate`; bypass the cache for logged-in users and the cart.
- **Check:** TTFB for a Hanoi user and from the US before and after. Check cache hit ratio in the CDN dashboard.
- **See:** [Performance §5.7](../../runtime-operational/performance/README.md#57-http-caching).

### 5. Googlebot gets the same page
- **Do:** find the geo-block rule in the firewall config. Request a product page from a US VPS and see what happens.
- **Fix:** remove the geo-block, or allow verified Googlebot (reverse DNS check or Google's published IP ranges) while keeping rate limits.
- **Check:** URL Inspection "Test live URL" in Search Console renders the full page; the rendered HTML matches what a Vietnamese user sees (no cloaking).
- **See:** README §4 table, [Crawlability](../crawlability/README.md).

---

## Progress

- [ ] 1. Baseline: lab vs field
- [ ] 2. Break down LCP
- [ ] 3. Distance and TTFB
- [ ] 4. Cache HTML at the edge
- [ ] 5. Googlebot gets the same page
