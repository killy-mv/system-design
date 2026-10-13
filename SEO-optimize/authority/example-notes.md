# Authority: hands-on practice notes

Planned labs for learning authority by **computing PageRank yourself, auditing real link profiles, and fixing how links and their credit flow into and through a site**. Theory is in [README.md](README.md).

The labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)), which has orphan products, user reviews with followed links, and a sitewide footer of unmarked "partner" links on purpose. Tools: **Python** with **networkx** (or plain code) for the graph labs, **Search Console** Links report, **Bing Webmaster Tools** and **Ahrefs Webmaster Tools** free tiers (for a site you can verify), and a crawler (Screaming Frog free tier or a small script).

Never buy links or build link networks for these labs, even as an experiment on a throwaway domain.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** internal linking and orphan detection (site-architecture labs), redirects after URL changes (site-architecture labs), Links report monitoring (seo-monitoring labs).

---

## Labs

### 1. PageRank by hand and in code ⭐
- **Do:** compute PageRank for a 4–5 page graph by hand for three iterations (damping 0.85). Then implement the power iteration in ~30 lines and compare with `networkx.pagerank`.
- **Then:** add a page with 50 outbound links and see how little each link passes. Add a dangling page (no outbound links) and see how your code must handle it.
- **See:** README §1, [How search works](../how-search-works/README.md).

### 2. Internal PageRank of `seo-shop` ⭐
- **Do:** crawl `seo-shop`, build the internal link graph, and run PageRank on it.
- **Write:** the top 10 and bottom 10 pages. Are the most important products near the top? Which pages are orphans (no inbound internal links)?
- **Fix:** add contextual links from guides and categories, remove the sitewide footer block, re-run and compare.
- **See:** README §7.

### 3. Qualify outbound links
- **Fix:** mark partner/affiliate links `rel="sponsored"` and links in user reviews `rel="ugc nofollow"`.
- **Check:** a crawler report or a `grep` over rendered HTML shows no unqualified external links in user content or the footer.
- **See:** README §3.

### 4. Audit a real link profile
- **Do:** for a site you can verify (your blog, a friend's site), export links from Search Console and one free third-party tool. Compare the lists: overlap, differences, anchors.
- **Write:** classify the top 20 referring domains by relevance, editorial choice and placement. Note which ones send referral traffic in analytics.
- **See:** README §2, §8.

### 5. Plan a linkable asset
- **Build:** a small "rice-to-water ratio calculator" or a data page ("we timed 10 rice cookers") on `seo-shop`, with a clear URL and internal links to it.
- **Write:** a one-page outreach plan: who would link to it and why, and how you'd find unlinked brand mentions and broken inbound links to reclaim.
- **See:** README §5.

---

## Progress

- [ ] 1. PageRank by hand and in code
- [ ] 2. Internal PageRank of `seo-shop`
- [ ] 3. Qualify outbound links
- [ ] 4. Audit a real link profile
- [ ] 5. Plan a linkable asset
