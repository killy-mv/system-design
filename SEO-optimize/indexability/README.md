# Indexability

> **Indexability** = once a page is fetched, the search engine can **see its real content**, knows **which URL is the official one**, and is **allowed and willing** to store it in the index.
> "Of the pages Google has crawled, which ones does it actually keep, and is it keeping the URL we wanted?"

Crawled is not indexed. Between the fetch and the index sit rendering, duplicate detection, canonical selection, directives like `noindex`, and a quality judgment. Search Console's Pages report usually shows many URLs as "Crawled – currently not indexed" or "Duplicate, Google chose different canonical than user". Indexability is about making those decisions easy and predictable for the engine.

Related: [Crawlability](../crawlability/README.md) (getting fetched), [Site architecture](../site-architecture/README.md) (URL design, redirects), [How search works](../how-search-works/README.md), [Case study: Hostinger](../../case-study/hostinger/README.md) (server-rendered pages for crawlers).

---

## 1. Vocabulary

| Term | Meaning |
|---|---|
| **Indexed** | Stored by the engine and eligible to appear in results |
| **Directive** | An instruction the engine follows (`noindex`) |
| **Hint** | A signal the engine weighs but may override (`rel="canonical"`) |
| **Canonical URL** | The one URL chosen to represent a group of duplicate or near-duplicate pages |
| **Duplicate content** | The same or nearly the same content at more than one URL |
| **Rendering** | Running JavaScript to build the final page content |
| **Soft 404** | A page that says "not found" or is empty but returns status 200 |
| **Cloaking** | Deliberately showing search engines different content from users |

---

## 2. Robots directives

Two ways to say "don't index this":

```html
<meta name="robots" content="noindex">
```
```
X-Robots-Tag: noindex          ← HTTP header; works for PDFs, images, any file type
```

| Directive | Effect |
|---|---|
| `noindex` | Don't show this page in results |
| `nofollow` | Don't follow links on this page (a hint for Google) |
| `nosnippet`, `max-snippet:N` | Control the text snippet (also limits use in AI Overviews) |
| `noimageindex` | Don't index images on the page |
| `unavailable_after: <date>` | Drop from results after a date (events, offers) |

Key facts:

- **`noindex` only works if the page can be crawled.** If robots.txt blocks it, Google never sees the tag ([Crawlability §2](../crawlability/README.md)).
- Use `noindex` for: internal search results, thin tag pages, thank-you pages, staging copies, user profile pages with no content.
- A `noindex` left over from staging is one of the most common causes of a site disappearing after launch. Check it in CI.

---

## 3. Duplicates and canonicalisation

Most duplicate content isn't copied. It's the same page reachable by many URLs:

| Source | Example |
|---|---|
| Protocol and host | `http://`, `https://`, `www.`, non-`www` |
| Trailing slash and case | `/shoes`, `/shoes/`, `/Shoes` |
| Parameters | `?utm_source=fb`, `?sort=price`, `?sessionid=abc` |
| Facets | `/shoes?color=red&size=40` vs `/shoes?size=40&color=red` |
| Same product in several categories | `/men/shoes/runner-x`, `/sale/runner-x` |
| Print, AMP, mobile (`m.`) versions | `/runner-x?print=1` |
| Paginated or sorted listings | `/shoes?page=2&sort=new` |

Ways to tell the engine which URL is canonical, strongest first:

| Method | Type | Use when |
|---|---|---|
| **301/308 redirect** | Strong signal | The duplicate shouldn't exist for users either (http → https, non-www → www) |
| **`<link rel="canonical" href="...">`** | Hint | The duplicate must stay for users (sort order, tracking parameters) |
| **Sitemap inclusion** | Weak hint | List only canonical URLs |
| **Internal links** | Weak hint | Always link to the canonical form |

`rel="canonical"` is a **hint**. Google picks a different canonical when signals conflict: the canonical tag says A, internal links point at B, the sitemap lists C, and B redirects to A. Make every signal agree.

Rules:

- Use **absolute URLs** in canonical tags.
- Every indexable page should have a **self-referencing canonical**.
- Don't canonicalise page 2 of a list to page 1; they have different content.
- Don't point canonicals at `noindex` pages, redirects or 404s.

---

## 4. Rendering strategies

Google renders JavaScript with an evergreen (current) version of Chromium, but rendering happens in a **second stage** after the HTML fetch, and can be delayed. Other engines and many AI crawlers render less or not at all. The safest content is content that is **already in the HTML response**.

| Strategy | First HTML response contains | SEO risk | Typical use |
|---|---|---|---|
| **CSR** (client-side rendering) | An empty shell; JS fetches and builds content | Highest: depends on rendering, JS errors hide content, slower to index | Logged-in apps, dashboards, editors |
| **SSR** (server-side rendering) | Full content, rendered per request | Low; cost is server CPU and TTFB | Product pages with live price/stock |
| **SSG** (static generation) | Full content, built at deploy time | Lowest; served from CDN | Blogs, docs, marketing pages |
| **ISR** / stale-while-revalidate | Full content, regenerated in the background | Low | Large catalogues that change hourly |
| **Hydration / islands** | Full content, then JS makes parts interactive | Low | Most modern frameworks' default |

**Dynamic rendering** (serving pre-rendered HTML only to bots) is described by Google as a workaround, not a recommended long-term solution. Prefer SSR or SSG for everyone.

Choose per route, as the [Hostinger case study](../../case-study/hostinger/README.md) does: public pages server-rendered and cached at the edge, the logged-in editor as a client-side app.

Rendering checks:

- `curl` the page: is the title, main text and the list of product links in the response?
- Search Console URL Inspection → "View crawled page": HTML and screenshot as Google rendered them.
- Content that appears only after a click, scroll or tab switch may not be indexed. Put it in the DOM at load (hidden tabs are fine; content fetched on click is not).

---

## 5. Soft 404s and empty pages

A page that returns **200** but says "Product not found", "0 results" or shows only a header and footer is treated as a **soft 404**: wasted crawl, and the page isn't indexed.

| Case | Correct response |
|---|---|
| Product deleted, no replacement | 404 or 410 |
| Product deleted, close replacement | 301 to the replacement (not to the home page) |
| Product out of stock, coming back | 200 with the page and "out of stock" |
| Search or filter with zero results | 200 with `noindex`, or don't link to it |
| CSR page whose API call failed | Server-render, or return a real error status |

Single-page apps are prone to soft 404s because the server returns 200 for every route and the "not found" view is decided in the browser.

---

## 6. Faceted navigation, pagination and infinite scroll

### 6.1 Facets

Five filters with ten values each and a sort order produce millions of URL combinations, almost all duplicates.

| Approach | Notes |
|---|---|
| Pick a few **valuable combinations** ("red running shoes") and give them clean, indexable URLs with unique titles | These match real searches |
| Everything else: `noindex` or canonical to the unfiltered listing | Keeps the index clean |
| Don't create crawlable links for multi-filter combinations (use form controls or `rel="nofollow"` hints) | Saves crawl budget on large sites |
| Disallow parameter patterns in robots.txt only when the site is large enough for crawl budget to matter | Remember: blocked URLs can't pass their `noindex` |
| Fixed parameter order and no empty parameters | Avoids `?a=1&b=2` vs `?b=2&a=1` duplicates |

### 6.2 Pagination

- Google no longer uses `rel="prev"`/`rel="next"`. Each page in a series is a normal page.
- Give each page its own URL (`?page=2` or `/page/2`), a self-referencing canonical, and normal `<a href>` links between pages.
- **Infinite scroll** must be backed by paginated URLs that work without JavaScript, and should update the URL as the user scrolls (History API). Otherwise only the first batch of items is discoverable.

---

## 7. Cloaking

Showing search engines different content from users, to manipulate ranking, violates Google's spam policies and can get pages or the whole site removed.

| Not cloaking | Cloaking |
|---|---|
| SSR for everyone; the same HTML for bots and users | Keyword-stuffed HTML only for Googlebot |
| Personalised recommendations that differ per user | Redirecting users (but not bots) to a different site |
| Paywalled content marked up as paywalled (structured data) | Showing bots the full article and users a login wall, without markup |
| A language banner for foreign visitors | Hiding or swapping content based on the bot's user agent |

The design rule: **branch on the user's needs, never on "is this a bot?"**.

---

## 8. Worked example: 500 products, 40 indexed

`seo-shop` has 500 products. Six weeks after launch, Search Console shows 40 product URLs indexed.

| Finding (Pages report / URL Inspection) | Cause | Fix |
|---|---|---|
| Home page: "Excluded by 'noindex' tag" | Staging `noindex` shipped in the layout | Remove; add a CI check that fails if production HTML contains `noindex` on templates that should be indexed |
| 300 products: "Discovered – currently not indexed" | CSR shell, links built with `onclick` | SSR for product and category routes; `<a href>` links |
| 120 URLs: "Duplicate, Google chose different canonical" | `/p?id=123&sort=price` and `/p?id=123` both linked; no canonical | Clean URLs `/products/runner-x`, 301 old ones, self-canonical |
| 60 URLs: "Soft 404" | Deleted products returning 200 "not found" | Return 404/410 or 301 to the replacement |
| Rendered screenshot shows unstyled page | `robots.txt` blocks `/assets/` | Allow assets |

Eight weeks later: 480 products indexed. The remaining 20 are near-identical colour variants, now canonicalised to one page per model on purpose.

---

## 9. Anti-patterns

- `noindex` from staging shipped to production.
- Blocking a page in robots.txt *and* adding `noindex` (the tag is never seen).
- Canonical tags pointing at the home page, at redirects, or at relative URLs.
- Conflicting signals: canonical says one URL, links and sitemap say another.
- Client-side-only rendering for public, search-facing pages.
- Returning 200 for "not found" pages.
- Redirecting every deleted page to the home page (treated as a soft 404).
- Infinite scroll with no paginated URLs.
- Serving different content when the user agent looks like a bot.

---

## 10. Checklist

- [ ] Every indexable page has a self-referencing absolute canonical
- [ ] One host and protocol; all variants 301 to it
- [ ] Title, main content and links are present in the initial HTML for public pages
- [ ] No `noindex` on templates that should be indexed (CI check)
- [ ] Deleted pages return 404/410 or 301 to a real replacement
- [ ] Facet strategy written down: which combinations are indexable, how the rest are handled
- [ ] Paginated and infinite-scroll lists have crawlable page URLs
- [ ] Bots and users receive the same content
- [ ] Search Console Pages report reviewed regularly ([SEO monitoring](../seo-monitoring/README.md))

---

## 11. Exercises

1. List every URL at which one product on a site you know can be reached. Which should redirect, which should canonicalise, and which should be `noindex`?
2. Choose a rendering strategy (CSR, SSR, SSG, ISR) for each route of an online shop: home, category, product, search, cart, account, blog. Justify each.
3. Design the facet policy for a fashion shop with filters for brand, colour, size, price and sort. Which combinations get indexable URLs?
4. A product goes out of stock permanently, but a newer model exists. Another product is out of stock for two weeks. What does each URL return?

---

## 12. Further reading

Google Search Central: "Block indexing with noindex", "What is URL canonicalization" and "How to specify a canonical URL", "Understand JavaScript SEO basics", "Dynamic rendering as a workaround", "Managing crawling of faceted navigation URLs", "Pagination, incremental page loading, and their impact on Google Search", "Spam policies for Google web search" (cloaking); web.dev "Rendering on the Web"; the Next.js and Nuxt rendering documentation.
