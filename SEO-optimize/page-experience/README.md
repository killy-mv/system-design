# Page experience

> **Page experience** = how fast, stable and comfortable a page is for the real people who load it: loading speed, responsiveness to input, visual stability, mobile usability, HTTPS and the absence of intrusive pop-ups.
> "When a real user on a real phone and network opens this page, is it fast, stable and usable, and does Google's field data agree?"

Page experience is a **modest ranking signal**. Relevance and quality decide most rankings, and a fast page with weak content will not outrank a slow page that answers the query much better. Speed works more like a tie-breaker between similarly relevant pages. It also matters through users: slow pages lose visitors and conversions whether or not Google ranks them lower.

This note covers the SEO side: which metrics Google uses, where the time goes, and how server and CDN location affect them. Frontend optimisation techniques are in [Performance](../../runtime-operational/performance/README.md) and [Responsiveness](../../UI-UX/responsiveness/README.md); this note links there instead of repeating them.

Related: [How search works](../how-search-works/README.md), [Crawlability](../crawlability/README.md) (server speed also affects crawl rate), [International SEO](../international-seo/README.md) (geo-targeting signals), [SEO monitoring](../seo-monitoring/README.md) (Core Web Vitals report, RUM), [Availability](../../runtime-operational/availability/README.md) (anycast and CDN edges).

---

## 1. Core Web Vitals

Three metrics, measured on **real users** (field data), not in a lab:

| Metric | Measures | Good | Needs improvement | Poor |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Loading: when the main content (largest image or text block) appears | ≤ 2.5 s | 2.5–4.0 s | > 4.0 s |
| **INP** (Interaction to Next Paint) | Responsiveness: delay from a tap, click or key press to the next frame | ≤ 200 ms | 200–500 ms | > 500 ms |
| **CLS** (Cumulative Layout Shift) | Visual stability: how much content jumps around | ≤ 0.1 | 0.1–0.25 | > 0.25 |

- **INP replaced FID** (First Input Delay) as a Core Web Vital in **March 2024**. FID only measured the first input; INP covers all interactions.
- A page or URL group passes when the **75th percentile** of real page loads is "good" for all three metrics. One in four visitors can have a worse experience and the page still passes; the slowest tail is ignored.
- **Mobile and desktop are assessed separately.** Most sites fail on mobile first.
- The data comes from the **Chrome UX Report (CrUX)**: anonymised measurements from opted-in Chrome users, aggregated over a rolling 28-day window. Changes therefore take weeks to show up.
- Pages with too little traffic have no URL-level data; Google falls back to groups of similar pages or the whole origin.

Details and fixes per metric: [Responsiveness §3.1](../../UI-UX/responsiveness/README.md#31-core-web-vitals-web), [Performance §1.3](../../runtime-operational/performance/README.md#13-user-centric-metrics).

### Field data vs lab data

| | Field (CrUX, RUM) | Lab (Lighthouse, WebPageTest) |
|---|---|---|
| Source | Real users, devices and networks | One simulated device and network |
| Used for ranking | Yes (CrUX) | No |
| Good for | Knowing whether you have a problem | Finding why, and testing a fix before release |
| INP | Measured | Not available; Lighthouse shows Total Blocking Time as a proxy |

A green Lighthouse score on a fast laptop in your office does not mean your users on mid-range Android phones over 4G are fine. Trust field data; use lab data to debug.

---

## 2. Other page experience factors

| Factor | What Google wants | Notes |
|---|---|---|
| **Mobile-friendly** | Readable, usable on a phone without zooming | **Mobile-first indexing applies to all sites**: Google indexes and ranks the mobile version. Content missing on mobile is effectively missing |
| **HTTPS** | Served securely, valid certificate, no mixed content | A lightweight ranking signal, but browsers mark HTTP pages "Not secure", which hurts trust |
| **No intrusive interstitials** | Main content reachable immediately | Full-screen pop-ups and app-install walls covering content after arriving from search are discouraged. Cookie consent, age verification and reasonably sized banners are fine |
| **No deceptive or harmful content** | Safe Browsing clean | Malware or phishing warnings remove you from users' reach entirely |

There is no single "page experience score". Google has said these are aspects its ranking systems reward, not one combined signal.

---

## 3. Where the time goes for LCP

LCP is the end of a chain. For an HTML page whose largest element is a hero image:

```
DNS  →  TCP  →  TLS  →  request HTML → TTFB → parse → discover image → download → render
└─────── network round trips ───────┘   └ server ┘   └────── frontend ──────────┘
```

**TTFB** (Time To First Byte) is not a Core Web Vital, but everything after it waits for it. web.dev suggests aiming for a TTFB of about 0.8 s or less at the 75th percentile. A 1.5 s TTFB leaves only 1 s for everything else before LCP fails.

TTFB itself has two parts:

- **Network**: several round trips (DNS lookup, TCP handshake, TLS handshake, the request). Each costs one RTT, so distance multiplies.
- **Server**: time to produce the HTML (database queries, rendering). Caching HTML at the edge removes most of it.

---

## 4. Geography: server and CDN location

Light in fiber travels at about **200,000 km/s**, and fiber follows coastlines and roads rather than straight lines. That sets a floor under every round trip.

### Worked example: a user in Hanoi

| Content served from | Typical RTT | Rough network cost before the first byte (≈ 4 round trips on a new connection) |
|---|---|---|
| CDN edge in Hanoi | ~2–10 ms | ~10–40 ms |
| CDN edge in Ho Chi Minh City (fiber path ~1,600–2,000 km) | ~20–35 ms | ~80–140 ms |
| Singapore | ~30–50 ms | ~120–200 ms |
| US (origin only, no CDN) | ~200–300 ms | ~0.8–1.2 s |

These are rough ranges; real values depend on the ISP and route. TLS 1.3, HTTP/2 and HTTP/3 (QUIC) reduce the number of round trips, and reused connections skip the handshakes entirely.

Conclusions:

- **Hanoi vs Ho Chi Minh City** edge: tens of milliseconds. Not noticeable for a normal website and negligible for LCP.
- **Somewhere in Vietnam or nearby vs the US**: up to a second of TTFB on uncached requests. This can push mobile LCP from "good" to "needs improvement".
- **The user's ISP route often matters more than the city.** Traffic between Vietnamese ISPs and a CDN may leave the country and come back if peering is poor.

### How location affects SEO

| Path | Who experiences it | Effect |
|---|---|---|
| Core Web Vitals (TTFB → LCP) | Real users, measured by CrUX | The main path. Small ranking effect, mostly a tie-breaker |
| Crawl rate | Googlebot, which crawls **mostly from US IP addresses** | Slow responses or timeouts make Google crawl less. Matters mostly for large sites ([Crawlability](../crawlability/README.md)) |
| Geo-targeting | Google's guess at the intended country | Server IP location is a **weak, close to negligible** signal today. ccTLD, hreflang, content language and local signals matter far more ([International SEO](../international-seo/README.md)) |
| User behaviour | Real users | Slow pages lose visitors; indirect and hard to measure |

Do not move servers for SEO alone. Put a CDN in front, cache what you can, and check the field data.

---

## 5. CDN choice

One **global CDN** serves both groups automatically: anycast or GeoDNS sends Vietnamese users to a nearby edge and Googlebot to a US edge ([Availability](../../runtime-operational/availability/README.md)). You do not need "a CDN for users and another for bots".

**Do not serve bots something different from users.** Showing crawlers different content from what people see is **cloaking**, a violation of Google's spam policies. Same content, different edge locations, is fine.

| Option | Pricing model | Vietnam notes |
|---|---|---|
| **Cloudflare** | Per plan and per feature; no charge per edge location; bandwidth not metered on standard plans. Check current pricing | Has PoPs in Vietnam, but see the caveat below |
| **CloudFront, Fastly, Akamai** | Mostly per GB and per request | Check each provider's current PoP list and ISP coverage |
| **Vietnamese CDNs** (VNETWORK, Viettel CDN, BizFly and others) | Usually per GB | Direct peering with Vietnamese ISPs; fewer edges abroad |

### The Cloudflare and Vietnam caveat

Every plan uses Cloudflare's whole network, but in regions where bandwidth is expensive, **Free and Pro traffic may be served from a farther PoP**. Vietnamese users are often reported landing in Singapore (`SIN`) or Hong Kong (`HKG`) instead of Hanoi (`HAN`) or Ho Chi Minh City (`SGN`). It varies by ISP (Viettel, VNPT, FPT) and changes over time, so measure instead of assuming:

```
curl -s https://your-domain.example/cdn-cgi/trace | grep colo
colo=SIN
```

Run it from several Vietnamese networks (mobile and fixed, different ISPs). `SIN` or `HKG` usually adds about 30–60 ms per round trip: worth knowing, rarely decisive for SEO.

A second Vietnam-specific reason to have an edge inside the country: **undersea cable faults** (AAG, APG, IA and others) happen regularly and slow international routes for weeks. Content served from a domestic PoP is unaffected; content that must travel to Singapore or the US is not.

---

## 6. Anti-patterns

- Optimising a lab Lighthouse score while field data (CrUX) stays red.
- Single origin far from users, no CDN, uncached HTML.
- Giant unoptimised hero image as the LCP element, loaded lazily or discovered late.
- A desktop site with a stripped-down mobile version that lacks content (mobile-first indexing indexes the stripped version).
- Full-screen newsletter or app-install pop-ups for visitors arriving from search.
- Serving crawlers a fast pre-rendered page and users a slow different one (cloaking).
- Firewall rules that block or challenge non-Vietnamese IPs, which also block Googlebot.
- Assuming "Cloudflare has a Hanoi PoP" means your users are served from Hanoi.
- Moving hosting countries for SEO while the real problem is a 2 MB JavaScript bundle.

---

## 7. Checklist

- [ ] Core Web Vitals report in Search Console reviewed for mobile and desktop
- [ ] LCP, INP and CLS good at the 75th percentile in CrUX or your own RUM
- [ ] TTFB measured from your main user regions and ISPs; HTML cached at the edge where possible
- [ ] CDN in front of the site; edge location verified (`colo=` or provider headers) from real networks
- [ ] Googlebot not blocked or challenged by firewall, bot protection or geo rules
- [ ] Same content for users and crawlers
- [ ] Mobile version has the same content, structured data and metadata as desktop
- [ ] HTTPS everywhere, HTTP redirects to HTTPS, no mixed content
- [ ] No intrusive interstitials on landing pages
- [ ] Performance budget in CI ([Performance](../../runtime-operational/performance/README.md))

---

## 8. Exercises

1. Pick a site you use. Compare its PageSpeed Insights field data (top section) with the lab data (Lighthouse section). Where do they disagree, and why?
2. Estimate TTFB for a Hanoi user to an uncached page on an origin in Virginia, with and without a CDN edge in Vietnam. Count the round trips explicitly. Then repeat assuming TLS 1.3 and a reused connection.
3. A product manager wants to move hosting from Singapore to Hanoi "for SEO". What would you measure first, and what would you expect to change?
4. Your site's mobile LCP is 3.2 s at p75. List the four parts of LCP (TTFB, resource load delay, resource load time, render delay) and how you would find which one dominates.

---

## 9. Further reading

web.dev articles on Core Web Vitals, LCP, INP, CLS and "Optimize Time to First Byte"; Google Search Central documentation "Understanding page experience in Google Search results" and "Mobile-first indexing best practices"; Google's spam policies (cloaking); Chrome UX Report documentation; Cloudflare's documentation on network and data center selection; Ilya Grigorik, *High Performance Browser Networking* (latency, TCP, TLS, HTTP/2); the [Performance](../../runtime-operational/performance/README.md) and [Responsiveness](../../UI-UX/responsiveness/README.md) notes in this repo.
