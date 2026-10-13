# Site architecture

> **Site architecture** = how a site's URLs are designed, how pages are grouped and linked to each other, and how URLs change over time without losing what they've earned.
> "Can users and bots reach every important page in a few clicks, does each URL make sense on its own, and do old links still work after we redesign?"

Architecture decides how internal link value flows (the [PageRank example](../how-search-works/README.md) applies inside your own site too), how fast new pages get discovered, and how much damage a redesign or migration does. It is mostly decided early, by engineers, and is expensive to change later, which makes it a system design topic, not a marketing one.

Related: [Crawlability](../crawlability/README.md), [Indexability](../indexability/README.md) (duplicates, canonicals), [Authority](../authority/README.md) (external links), [Backward compatibility](../../portability-compatibility/backward-compatibility/README.md) (URLs are a public interface), [Case study: Hostinger](../../case-study/hostinger/README.md) (one domain, path-based routing).

---

## 1. URL design

URLs are a public, long-lived interface. Bookmarks, other sites and search indexes keep them for years.

| Guideline | Good | Bad |
|---|---|---|
| Readable words, hyphen-separated | `/products/giay-chay-bo-runner-x` | `/p?id=123` |
| Lowercase, one form | `/shoes/` everywhere | `/Shoes`, `/shoes`, `/shoes/` all 200 |
| Stable: not tied to implementation | `/blog/http-caching` | `/index.php?module=blog&id=88` |
| No session IDs or tracking in paths | `/cart` | `/cart;jsessionid=AB12` |
| Parameters only for filters, sort, paging | `/shoes?page=2` | `/shoes?page=2&ref=nav&t=169...` |
| Paths, not fragments, for separate pages | `/products/runner-x` | `/#/products/runner-x` |
| Hierarchy only if it's stable | `/blog/2026/...` for news | `/men/sale/summer/shoes/...` (product moves = URL changes) |

**IDs vs slugs.** `/products/runner-x-123` combines a readable slug with a stable ID. If the name changes, look up by ID and 301 to the new slug. This avoids broken links without a giant redirect table.

**Vietnamese text in URLs.** Use ASCII slugs without diacritics (`giay-chay-bo`, not `giày-chạy-bộ`). Search engines handle both, but encoded URLs are long, ugly when shared, and easy to get wrong in redirects. See [Internationalization](../../UI-UX/internationalization/README.md).

---

## 2. Hierarchy: flat vs deep

**Click depth** is the number of clicks from the home page. Pages buried deep get crawled less often and receive less internal link value.

```
Deep (bad):  Home → Men → Footwear → Running → Road → Brand → Product   (depth 6)
Flat (good): Home → Running shoes → Product                              (depth 2)
             Home → Brand → Product                                      (depth 2)
```

| Guideline | Why |
|---|---|
| Important pages within ~3 clicks | Crawled more often, more internal link value |
| Categories match how people search ("running shoes"), not internal org charts | Category pages are the ones that rank for broad queries |
| Avoid categories with 1–2 items and categories with 5,000 | Thin pages vs unreachable products |
| Hubs: category and topic pages that link to all their children | They concentrate and redistribute link value |

---

## 3. Internal linking

Internal links do three jobs: discovery, link-value distribution, and telling the engine what a page is about (the anchor text).

| Link type | Notes |
|---|---|
| Main navigation | Links to top categories from every page; keep it small and stable |
| Breadcrumbs | Link each page to its parents ([§4](#4-breadcrumbs)) |
| Contextual links in content | Strongest topical signal: "see our guide to [choosing running shoes]" |
| Related products / articles | Spread value sideways; generate them server-side as `<a href>` |
| Pagination | Makes deep list items reachable |
| Footer | Fine for policy pages; footer link farms add little |

**Orphan pages** have no internal links pointing at them. They can only be found through sitemaps or external links and receive almost no internal value (page D in the PageRank example). Typical causes: products removed from all categories, campaign landing pages, pages linked only from JavaScript widgets.

Find orphans by comparing three lists: URLs in the database or sitemap, URLs found by a crawl from the home page, and URLs receiving traffic. Anything in the first or third list but not the second is an orphan.

Anchor text: descriptive ("Runner X trail shoes") beats generic ("click here"). Don't stuff exact keywords into every anchor; write for people.

---

## 4. Breadcrumbs

```
Home › Running shoes › Runner X
```

- Every crumb is a real `<a href>` link, except the current page.
- Mark them up with `BreadcrumbList` [structured data](../structured-data/README.md) so search results can show the path instead of the raw URL.
- A product in several categories should have one primary breadcrumb path, matching its canonical URL.

---

## 5. Redirects

| Code | Meaning | Use for |
|---|---|---|
| **301** | Moved permanently | URL changes, host/protocol consolidation |
| **308** | Permanent, keeps the HTTP method | Same as 301; APIs and forms |
| **302 / 307** | Temporary | A/B tests, short promotions, maintenance |
| Meta refresh / JS redirect | Client-side | Only when you can't do a server redirect |

Rules:

- **One hop.** `A → B → C` should become `A → C` and `B → C`. Long chains waste crawl and may not be followed to the end.
- **Redirect to the closest equivalent**, not the home page. Mass redirects to the home page are treated as soft 404s.
- **Keep redirects for at least a year**, ideally indefinitely. Old links and bookmarks never disappear.
- **Update internal links** to point straight at the new URL; don't rely on redirects internally.
- Implement redirects at the edge (CDN rules, nginx maps) so they are fast and don't depend on the application.

---

## 6. One domain, subdomains or separate domains

| Setup | Example | SEO notes |
|---|---|---|
| **Subfolders** on one domain | `shop.example/blog/` | Simplest: links to any part help the whole site; one Search Console property |
| **Subdomains** | `blog.shop.example` | Google says it handles both; in practice they're often treated more like separate sites and need their own setup (robots.txt, Search Console, internal links across) |
| **Separate domains** | `shopblog.example` | Fully separate authority; only for genuinely separate brands |

Subfolders are the safe default. Different stacks can still live on one domain with path-based routing at the reverse proxy or CDN, as in the [Hostinger case study](../../case-study/hostinger/README.md) where `/templates` is Nuxt and `/blog` is WordPress.

For countries and languages, see [International SEO](../international-seo/README.md) (ccTLDs vs subfolders vs subdomains).

---

## 7. Worked example: migrating `seo-shop` to clean URLs

The shop moves from `http://shop.example/p?id=123` and `/c?cat=5` to `https://shop.example/products/runner-x-123` and `/running-shoes/`.

### 7.1 Redirect map

Generated from the database, not written by hand:

| Old URL | New URL | Code |
|---|---|---|
| `http://shop.example/*` | `https://shop.example/*` | 301 (host rule) |
| `/p?id=123` (any extra params) | `/products/runner-x-123` | 301 |
| `/c?cat=5` | `/running-shoes/` | 301 |
| `/c?cat=9` (category merged into 5) | `/running-shoes/` | 301 |
| `/p?id=77` (product deleted, no replacement) | none | 410 |
| `/blog.php?post=12` | `/blog/how-to-choose-running-shoes` | 301 |

### 7.2 Plan

| Phase | Steps |
|---|---|
| **Before** | Crawl the old site and export all URLs (crawl + sitemap + top URLs from analytics + URLs with backlinks). Build the redirect map; every old URL maps to a new URL or a deliberate 404/410. Benchmark: indexed pages, clicks per page type, rankings for key queries |
| **Launch** | Deploy redirects at the edge. New sitemaps with new URLs. Self-canonicals point at new URLs. Internal links updated. Keep the old sitemap submitted for a few weeks so Google recrawls the old URLs and sees the redirects |
| **After** | Crawl the old URL list: every URL returns one 301 to a 200, or the planned 410. Watch Search Console daily for 404 spikes and coverage changes. Fix gaps found in logs (old URLs still being requested) |

Expect some ranking movement for a few weeks after any migration. A complete redirect map keeps it temporary. Missing redirects make losses permanent for those pages.

---

## 8. Anti-patterns

- URLs that expose implementation (`.php`, query IDs) and change when the stack changes.
- Several live variants of each URL (case, trailing slash, host).
- Deep hierarchies that mirror the org chart.
- Important pages reachable only through site search or JavaScript widgets.
- Redesigns that change every URL without a redirect map.
- Redirect chains, loops, and redirecting everything to the home page.
- Removing redirects after a few months "to clean up".
- Putting the blog on a separate subdomain or domain by default because it runs on a different stack.

---

## 9. Checklist

- [ ] URL scheme documented: lowercase, hyphenated, ASCII slugs, stable IDs, one trailing-slash rule
- [ ] All key pages within ~3 clicks of the home page
- [ ] No orphan pages (crawl vs database vs traffic comparison)
- [ ] Breadcrumbs with real links and `BreadcrumbList` markup
- [ ] Redirects are single-hop, at the edge, to the closest equivalent, and kept long-term
- [ ] Internal links point at final URLs, not redirects
- [ ] Every URL change goes through a redirect map generated from data
- [ ] Content on subfolders of one domain unless there's a real reason not to

---

## 10. Exercises

1. Design the URL scheme for a shop with products, categories, brands, a blog and Vietnamese and English versions. Write example URLs for each.
2. Take a site you use and measure click depth to three deep pages. Propose a structure that halves it.
3. Write the redirect map for merging two categories and renaming a third. How do you handle products that belonged to both merged categories?
4. Your team wants to move the blog from `shop.example/blog` to `blog.shop.example` because it's now a separate service. Argue for an alternative that keeps the subfolder.

---

## 11. Further reading

Google Search Central: "URL structure best practices", "Site moves and migrations", "Redirects and Google Search", "Link best practices", "Breadcrumb structured data"; Tim Berners-Lee, "Cool URIs don't change" (W3C, 1998); RFC 9110 (HTTP status codes, 301/302/307/308); Louis Rosenfeld, Peter Morville and Jorge Arango, *Information Architecture* (4th ed.).
