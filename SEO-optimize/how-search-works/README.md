# How search works

> **How search works** = the pipeline a search engine runs to turn billions of web pages into ten ranked results: **crawl, render, index, retrieve, rank**.
> "When someone types a query, how does the engine decide which pages exist, which ones match, and which one goes first?"

SEO is engineering against this pipeline. Every other note in this folder targets one stage of it: [crawlability](../crawlability/README.md) gets pages fetched, [indexability](../indexability/README.md) gets them stored, and [on-page relevance](../on-page-relevance/README.md), [authority](../authority/README.md) and [page experience](../page-experience/README.md) affect how they rank. If you know which stage is failing, you know which fix to apply.

Related: [Crawlability](../crawlability/README.md), [Indexability](../indexability/README.md), [Authority](../authority/README.md), [Scalability §18.7 (crawler design)](../../runtime-operational/scalability/README.md), [Performance](../../runtime-operational/performance/README.md).

---

## 1. Vocabulary

| Term | Meaning |
|---|---|
| **Crawler / bot / spider** | Program that downloads pages and follows links (Googlebot, Bingbot, coccocbot) |
| **URL frontier** | The prioritised queue of URLs waiting to be crawled |
| **Rendering** | Running a page's JavaScript in a headless browser to see the final HTML |
| **Index** | The engine's database of processed pages, organised for fast lookup |
| **Inverted index** | Map from each term to the list of documents that contain it |
| **Query** | What the user typed, plus context (location, language, device) |
| **SERP** | Search engine results page |
| **Organic result** | A result ranked by the algorithm, not paid for |
| **Rich result** | A result with extra display (stars, price, breadcrumbs), usually from [structured data](../structured-data/README.md) |
| **Signal** | Any measurable property used in ranking (text match, links, speed, freshness) |

---

## 2. The pipeline

```
 OFFLINE (continuous)                                   ONLINE (per query, well under a second)
 ┌────────┐   ┌────────┐   ┌─────────┐   ┌─────────┐    ┌──────────┐   ┌──────────────┐
 │ Crawl  │ → │ Render │ → │ Process │ → │ Index   │ →  │ Retrieve │ → │ Rank (multi- │ → SERP
 │ fetch  │   │ run JS │   │ dedupe, │   │ inverted│    │ candidate│   │ stage)       │
 │ URLs   │   │        │   │ canonic.│   │ sharded │    │ set      │   │              │
 └────────┘   └────────┘   └─────────┘   └─────────┘    └──────────┘   └──────────────┘
     ▲                                                                       
     └──── links found in pages and sitemaps feed back into the URL frontier
```

| Stage | What happens | What breaks it |
|---|---|---|
| **Crawl** | Fetch URLs from the frontier, respect `robots.txt`, rate-limit per host | Blocked by robots.txt or firewall, server errors, no links pointing in |
| **Render** | Execute JavaScript to get the final DOM. Google does this in a second stage, sometimes later than the fetch | Content only appears after user interaction; JS errors; blocked JS/CSS files |
| **Process** | Extract text, links, title, structured data; detect language, duplicates and spam; choose a canonical URL | Duplicate URLs, thin or copied content, `noindex` |
| **Index** | Store the canonical version in the inverted index and other stores | Low quality: the engine may crawl a page and still decide not to index it |
| **Retrieve** | Pull a few thousand candidates that plausibly match the query | Page doesn't contain or mean what people search for |
| **Rank** | Score candidates with many signals; cheaper models first, expensive ones on the top few hundred | Weaker relevance, authority or experience than competitors |

---

## 3. The inverted index

A normal document store maps document → words. Search needs the reverse:

```
"pizza"   → [doc12, doc87, doc301, doc455 ...]
"recipe"  → [doc12, doc55, doc301 ...]
"hanoi"   → [doc87, doc301, doc990 ...]
```

The query "pizza recipe" becomes an intersection of two posting lists: `{doc12, doc301}`. Posting lists are sorted by document ID, so intersection is a linear merge, and they are compressed (delta encoding) to fit in memory.

At web scale the index is:

- **Sharded by document**: each shard holds the full index for a slice of documents. A query fans out to all shards in parallel, each returns its local top results, and an aggregator merges them.
- **Replicated**: each shard has many copies for throughput and [availability](../../runtime-operational/availability/README.md).
- **Tiered**: popular, high-quality pages in a fast tier; the long tail in cheaper storage.

Modern engines also keep **vector (embedding) indexes** so a query can match pages by meaning, not only by exact words. "Cheap flights" can match a page that says "low-cost airfare".

---

## 4. Ranking

### 4.1 Multi-stage ranking

Scoring every candidate with the best model is too slow. Engines use a funnel:

| Stage | Candidates | Model cost |
|---|---|---|
| Retrieval | Billions → thousands | Index lookups, cheap text scores (BM25-like) |
| First-pass ranking | Thousands → hundreds | Light models over precomputed features |
| Final ranking | Hundreds → the page of results | Expensive neural models (BERT-style and newer) |
| Post-processing | The page | Diversity, freshness boosts, removal of near-duplicates, spam and legal removals |

This is the same latency-vs-accuracy trade-off as any [performance](../../runtime-operational/performance/README.md) budget: spend expensive compute only where it changes the answer.

### 4.2 Signal groups

Google says it uses many signals and does not publish weights. The well-documented groups:

| Group | Examples | Note in this folder |
|---|---|---|
| **Relevance** | Query terms and meaning in title, headings, body; search intent match | [On-page relevance](../on-page-relevance/README.md) |
| **Authority** | Links from other sites (PageRank and successors), mentions | [Authority](../authority/README.md) |
| **Quality** | Helpful, original content; E-E-A-T concepts from the quality rater guidelines | [On-page relevance](../on-page-relevance/README.md) |
| **Page experience** | Core Web Vitals, mobile-friendly, HTTPS, no intrusive interstitials | [Page experience](../page-experience/README.md) |
| **Freshness** | Recent content for time-sensitive queries | [Site architecture](../site-architecture/README.md) |
| **Context** | User location, language, device, sometimes search history | [International SEO](../international-seo/README.md), [Local SEO](../local-seo/README.md) |

Relevance and quality dominate. Page experience mostly separates pages that are otherwise similar.

### 4.3 Worked example: PageRank on four pages

PageRank treats a link as a vote, weighted by the voter's own importance. The original formula (Brin and Page, 1998, normalised form):

```
PR(A) = (1 − d) / N  +  d × Σ PR(Ti) / C(Ti)
```

`N` = number of pages, `d` ≈ 0.85 (the damping factor: chance the "random surfer" follows a link rather than jumping to a random page), `Ti` = pages linking to A, `C(Ti)` = number of outgoing links on Ti.

Graph: `A → B`, `A → C`, `B → C`, `C → A`, `D → C`. So `N = 4`, `(1 − d)/N = 0.0375`.

```
PR(D) = 0.0375                                   (nobody links to D)
PR(B) = 0.0375 + 0.85 × PR(A)/2                  (A splits its vote between B and C)
PR(C) = 0.0375 + 0.85 × (PR(A)/2 + PR(B) + PR(D))
PR(A) = 0.0375 + 0.85 × PR(C)                    (C's only link is to A)
```

Iterating until the values stop changing (or solving the linear system) gives:

| Page | Inbound links | PageRank |
|---|---|---|
| C | 3 (A, B, D) | **0.394** |
| A | 1 (C) | **0.373** |
| B | 1 (A, shared) | 0.196 |
| D | 0 | 0.0375 |

Lessons:

- **C** wins by having the most inbound links.
- **A** has only one inbound link but almost matches C, because that link comes from the strongest page and isn't shared with anything else.
- **B** also has one inbound link, but it comes from A, which splits its vote in two.
- **D** is effectively an orphan. Pages with no inbound links get only the baseline (see [Site architecture](../site-architecture/README.md) on orphan pages).

PageRank is still used inside Google but is one signal among many. It is the clearest model of why **who links to you** matters more than **how many** links you have ([Authority](../authority/README.md)).

---

## 5. Organic results vs ads

| | Organic | Paid (Google Ads) |
|---|---|---|
| How a position is won | Ranking algorithm | Real-time auction for each query |
| Ordered by | Relevance, quality, authority, experience... | Roughly bid × quality (ad relevance, expected click-through, landing page experience) |
| Cost | Free per click; costs effort | Pay per click |
| Labelled | No | "Sponsored" |
| Stops when you stop paying | No | Yes |

Paying for ads does not improve organic ranking, and organic ranking does not reduce ad prices directly. They are separate systems on the same page.

---

## 6. Personalisation and location

The same query returns different results for different users:

- **Location**: "pizza" in Hanoi shows Hanoi restaurants and a map pack ([Local SEO](../local-seo/README.md)). Location comes from IP, device GPS or settings.
- **Language and country**: Vietnamese users see Vietnamese pages and `.vn` sites first ([International SEO](../international-seo/README.md)).
- **Device**: Google uses the **mobile** version of pages for indexing and ranking (mobile-first indexing).
- **Legal**: some results are removed per country.

So "we rank #3" is only true for a place, a language and a device. Measure rankings from the market you care about ([SEO monitoring](../seo-monitoring/README.md)).

---

## 7. Other engines and AI answers

| Engine / surface | Note |
|---|---|
| **Google** | Dominant almost everywhere, including Vietnam |
| **Bing** | Also powers several other engines and AI assistants; supports **IndexNow** (push URLs on change) |
| **Cốc Cốc** | Vietnamese browser and search engine with its own crawler (`coccocbot`); worth checking in server logs for Vietnamese sites |
| **AI answers** (Google AI Overviews, chat assistants with search) | Generate a summary from retrieved pages and link sources. They still depend on crawling and indexing, so the same fundamentals apply; clicks per query can be lower |

Most technical SEO work is engine-neutral: crawlable, fast, well-structured pages work for every engine.

---

## 8. What SEO can and can't control

| You control | You influence | You don't control |
|---|---|---|
| Whether pages can be crawled, rendered and indexed | Links from other sites | Ranking weights and algorithm updates |
| URLs, redirects, canonical tags, sitemaps | User engagement with your results | Competitors |
| Content, titles, headings, structured data | How your snippet is shown | Personalisation for each user |
| Server speed, CDN, Core Web Vitals | Brand searches and mentions | SERP layout (ads, AI answers, maps) |

Engineering owns most of the first column. That is where an engineer's SEO work has the most predictable payoff.

---

## 9. Anti-patterns

- Treating SEO as "add keywords" while pages can't be crawled or rendered.
- Assuming Google sees what your browser sees (it may not run your JS, may be blocked, and crawls from the US).
- Chasing a single ranking factor (speed, word count, backlinks) instead of the whole pipeline.
- Measuring rank from one laptop in one city and calling it "our ranking".
- Paying for ads to "help" organic ranking.
- Optimising only for Google when a local engine or AI assistant brings real traffic.

---

## 10. Checklist

- [ ] For each important page type, you know which pipeline stage it reaches (crawled? rendered? indexed? ranking?)
- [ ] Pages are reachable by links and listed in a sitemap ([Crawlability](../crawlability/README.md))
- [ ] Important content is in the HTML or reliably rendered ([Indexability](../indexability/README.md))
- [ ] Each page targets a clear search intent ([On-page relevance](../on-page-relevance/README.md))
- [ ] Rankings are measured per market, language and device ([SEO monitoring](../seo-monitoring/README.md))
- [ ] Search Console (and Bing Webmaster Tools) set up for the site

---

## 11. Exercises

1. Draw the five pipeline stages for one product page on a site you know. For each stage, write one thing that could stop the page from passing it.
2. Compute PageRank by hand for a 3-page graph `A → B`, `B → C`, `C → A`, then add `C → B`. What changes and why?
3. Build a tiny inverted index for five short documents and answer two AND queries by merging posting lists.
4. Search the same query on Google from a phone on mobile data and from a laptop with a VPN in another country. List the differences and which signal caused each.

---

## 12. Further reading

Google Search Central, "How Google Search works" and "In-depth guide to how Google Search works"; Brin and Page, "The Anatomy of a Large-Scale Hypertextual Web Search Engine" (1998); Page, Brin, Motwani and Winograd, "The PageRank Citation Ranking" (1999); Manning, Raghavan and Schütze, *Introduction to Information Retrieval* (inverted indexes, BM25, ranking); Google Search Quality Rater Guidelines; Bing Webmaster Guidelines; IndexNow documentation.
