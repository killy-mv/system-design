# Authority

> **Authority** = how much the rest of the web vouches for a page or site, mainly through links from other relevant, trusted sites, and through being known and referenced as a source.
> "If two pages answer the query equally well, why should a search engine trust this one more?"

Relevance ([On-page relevance](../on-page-relevance/README.md)) decides *which* queries a page can rank for. Authority is one of the main things that separates pages that are all relevant. It is also the part of SEO you control least: you can't create authority directly, only earn it.

Related: [How search works](../how-search-works/README.md) (PageRank and ranking), [Site architecture](../site-architecture/README.md) (internal links distribute authority inside your site), [Local SEO](../local-seo/README.md) (prominence and citations), [SEO monitoring](../seo-monitoring/README.md) (Links report).

---

## 1. Links as votes

Google's original idea: a link from page A to page B is a **vote** for B. Not all votes are equal.

- A vote from an important page counts more.
- A page's voting power is **split among all its outbound links**: a link from a page with 3 links passes more than one from a page with 300.
- Importance flows through the whole graph, so it's computed iteratively until it settles.

This is **PageRank** (formula and intuition in [How search works](../how-search-works/README.md)). Google still uses link analysis, but PageRank is now one signal among many and the public "toolbar PageRank" score was retired long ago. Third-party scores like Domain Rating (Ahrefs), Domain Authority (Moz) or Authority Score (Semrush) **are not Google metrics**. They're each tool's own estimate from its own crawl.

### Anchor text

The clickable text of a link tells search engines what the target page is about. "Toshiba rice cooker review" says more than "click here". Natural link profiles have varied anchors (brand names, URLs, descriptive phrases). Large numbers of identical, keyword-exact anchors from other sites look manipulated.

---

## 2. What makes a link valuable

| Factor | Stronger link | Weaker link |
|---|---|---|
| **Relevance** | A cooking blog linking to a rice cooker review | A car forum linking to it |
| **Authority of the linking page** | A national newspaper or a known industry site | A brand-new blog with no links of its own |
| **Editorial choice** | The author chose to link because it helps their readers | Self-added: forum signatures, comment links, profile pages |
| **Placement** | In the main content, in context | Footer, sidebar, sitewide "partners" block |
| **Traffic** | People actually click it | Nobody ever sees it |
| **Diversity** | Links from many different sites | Hundreds of links from one site |
| **`rel` attribute** | No qualifier | `nofollow`, `sponsored`, `ugc` (§3) |

One relevant editorial link from a respected site is worth more than hundreds of low-quality ones, and the low-quality ones can be ignored or treated as spam.

---

## 3. `rel` attributes

| Attribute | Meaning | Use it for |
|---|---|---|
| `rel="sponsored"` | The link is paid or part of an advertising or affiliate deal | Ads, sponsored posts, affiliate links |
| `rel="ugc"` | The link is in user-generated content | Comments, forum posts, reviews written by users |
| `rel="nofollow"` | "Don't associate me with this link" | Any other link you don't want to vouch for |

Since 2019 Google treats all three as **hints**, not strict commands: it may still use them for discovery or understanding. They can be combined (`rel="ugc nofollow"`). For your own site this matters in two directions:
- **Outbound:** mark paid and user-generated links, or your site may be seen as selling links.
- **Inbound:** a link to you with `sponsored` or `nofollow` passes little or no ranking credit, which is correct.

---

## 4. What not to do

Google's spam policies call these **link spam**:

| Practice | Why it fails |
|---|---|
| **Buying or selling links** that pass ranking credit (no `sponsored`/`nofollow`) | Violates spam policies; can lead to the links being ignored or a manual action |
| **Link exchanges** at scale ("link to me and I'll link to you") | Excessive reciprocal linking is called out in the policies |
| **Private blog networks (PBNs)** | Sites built only to link to a money site. Detectable by shared hosting, owners, templates and patterns |
| **Automated links** | Comment spam, forum profiles, directory blasts, auto-generated widgets with hidden links |
| **Guest posts or press releases with keyword-rich, followed links** at scale | Treated as link schemes |
| **Expired domain abuse** | Buying an old domain for its links and reusing it for unrelated content |

Google's link spam systems (SpamBrain) mostly **neutralize** spammy links, so they simply don't count. Money spent on them is usually wasted, and in worse cases a manual action appears in Search Console.

### The disavow tool

Search Console has a disavow tool that tells Google to ignore specific inbound links. **Google says most sites never need it.** Use it only if you have a manual action for unnatural links or know of large-scale paid links pointing at you. Disavowing good links by mistake removes their benefit.

---

## 5. Earning links

Links come from making something worth referencing and making sure the right people see it.

| Approach | Example for `seo-shop` |
|---|---|
| **Original data** | "We tested 20 rice cookers: cooking time and power use" with downloadable results |
| **Useful tools** | A "rice-to-water ratio calculator" or a size picker by household size |
| **Definitive guides** | The best explanation of choosing a rice cooker, kept up to date |
| **Digital PR** | Pitching a genuinely newsworthy finding to journalists and bloggers |
| **Relationships** | Brands you sell, local partners, suppliers' "where to buy" pages |
| **Reclaim mentions** | Sites that mention your brand without linking: ask them to add a link |
| **Fix broken links to you** | Old URLs that other sites link to but now return 404: redirect them ([Site architecture](../site-architecture/README.md)) |

Links earned this way are relevant and editorial by nature, so they're also the ones search engines value.

---

## 6. Brand and mentions

Authority isn't only links. Search engines also learn about entities from how they're discussed: brand searches ("seo-shop rice cooker"), unlinked mentions, reviews on other platforms, a Wikipedia article, a knowledge panel. Google doesn't publish exactly how these are used, so don't optimize for them directly. Building a real reputation that people talk about tends to produce them.

---

## 7. Internal link equity

Links inside your own site also pass signals. Inbound authority usually lands on a few pages (homepage, popular guides). Internal links carry it to the rest.

```
External links ──► Homepage ──► Category: Rice cookers ──► Product: Toshiba RC-18
                      │                     │
                      └──► Guide: "How to choose a rice cooker" ──┘ (contextual link)
```

- Important pages should be **few clicks from the homepage** and linked from relevant content.
- **Orphan pages** (no internal links) get little authority and may not be found at all.
- Use descriptive anchor text internally too.
- Don't waste equity on redirect chains or links to pages that return 404.

See [Site architecture](../site-architecture/README.md) for structure, depth and orphan detection.

---

## 8. Measuring authority

| Source | What you get | Limits |
|---|---|---|
| **Search Console → Links** | Top linking sites, top linked pages, top anchor texts, as Google sees them | A sample; no quality score; no history beyond what's shown |
| **Bing Webmaster Tools → Backlinks** | Inbound links, free | Bing's crawl, not Google's |
| **Ahrefs Webmaster Tools (free for verified sites), Moz, Semrush** | Link lists, new and lost links, their own authority scores | Their own crawls and scores; never complete; not used by Google |
| **Referral traffic in analytics** | Which links actually send visitors | Only links people click |

Track trends (referring domains over time, links to key pages) rather than chasing a third-party score.

---

## 9. Anti-patterns

- Buying "100 DA50+ backlinks" packages.
- Chasing a third-party authority score as if it were a Google metric.
- Exact-match keyword anchors in every link you request.
- Sitewide footer links to partners without `sponsored`.
- Letting user comments carry followed links (spam magnet).
- Disavowing links "just in case".
- Changing URLs without redirects and losing every link to the old ones.
- Getting links to the homepage only and never linking internally to the pages that need them.

---

## 10. Checklist

- [ ] Inbound links reviewed in Search Console and one third-party tool; trend tracked monthly
- [ ] Outbound paid and affiliate links use `rel="sponsored"`; user content uses `rel="ugc"`
- [ ] No link buying, exchanges or networks; any agency's methods checked against spam policies
- [ ] At least one linkable asset (data, tool, guide) planned or maintained
- [ ] Unlinked brand mentions and broken inbound links reclaimed
- [ ] Old URLs with inbound links redirect to the right new page
- [ ] Important pages linked internally with descriptive anchors; no orphans
- [ ] Manual actions report checked; disavow used only with a reason

---

## 11. Exercises

1. Compute PageRank by hand for a 4-page graph (damping 0.85, three iterations). Then add a link from the highest-ranked page to the lowest and recompute. What changed, and why?
2. Look at the Links report (or a free backlink tool) for a site you know. Classify its top 20 referring domains by relevance and editorial quality.
3. Design a linkable asset for a small shop with no budget for PR. Who would link to it, and why?
4. An agency offers "guaranteed page-one rankings via our private network". Explain the risks using Google's spam policies.
5. A competitor points thousands of spammy links at your site. What do you do, and what don't you do?

---

## 12. Further reading

Brin and Page, "The Anatomy of a Large-Scale Hypertextual Web Search Engine" (1998); Page, Brin, Motwani and Winograd, "The PageRank Citation Ranking: Bringing Order to the Web" (1999); Google Search Central: "Spam policies for Google web search" (link spam section), "Qualify your outbound links to Google", "Disavow links to your site", and the 2019 blog post "Evolving 'nofollow'"; Search Console Help on the Links report; Moz, "The Beginner's Guide to SEO" (link building chapter).
