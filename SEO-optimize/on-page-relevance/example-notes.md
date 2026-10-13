# On-page relevance: hands-on practice notes

Planned labs for learning on-page relevance by **matching pages to real queries and intents, then fixing titles, headings and content in the templates that generate them**. Theory is in [README.md](README.md).

The labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)). It has missing and duplicate titles, thin auto-generated category pages and copied product descriptions on purpose. Tools: **Search Console** (a verified property, or the demo account's data if your site has no traffic yet), **Google Trends**, **Google Ads Keyword Planner**, and **Lighthouse** (SEO audit).

Keep a `CHANGELOG-labs.md`: for each lab, what you changed and the before/after numbers.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** structured data generated from the product model (structured-data lab 2), internal links and orphan pages (site-architecture labs), headings and `alt` text audit (accessibility labs).

---

## Labs

### 1. Intent map ⭐
- **Do:** list 15 queries a shopper might use for `seo-shop`'s products (informational, commercial and transactional). Search each and record the page types in the top ten.
- **Write:** a table: query, intent, page type that ranks, `seo-shop` page that should serve it (or "missing").
- **See:** README §1.

### 2. Keyword research from real words
- **Do:** collect phrasings from Google autocomplete, "People also ask", Google Trends (compare two phrasings by region) and Keyword Planner ranges. If you have Search Console data, export the Queries report.
- **Write:** group the phrasings into topics; one topic = one page. Note the Vietnamese and English variants.
- **See:** README §2.

### 3. Fix titles and headings in templates ⭐
- **Do:** crawl `seo-shop` (Screaming Frog free tier or a small script) and list pages with missing, duplicate or too-long titles, missing descriptions and missing `<h1>`.
- **Fix:** generate titles and descriptions from product and category data in the templates. Make sure they're in the server-rendered HTML (`curl` the page, don't trust DevTools).
- **Check:** re-crawl: zero duplicates. Run the Lighthouse SEO audit on three page types.
- **See:** README §3.

### 4. Rewrite a thin page
- **Do:** pick one product and one category page. Rewrite them following README §7: specs, who it's for, first-hand notes, questions from fake support tickets, descriptive `alt` text.
- **Then:** add the new fields to the data model and template so every product gets the structure, not just one.
- **Check:** compare word count, unique content (not copied from the manufacturer) and heading outline before and after.

### 5. Find cannibalization
- **Do:** create a blog post and a category page that both target "rice cooker" (the deliberate mistake). With Search Console data, filter by the query and open the Pages tab; without it, use a `site:` search to see which URL appears.
- **Fix:** differentiate their intents and link between them, or merge and 301-redirect.
- **See:** README §6.

---

## Progress

- [ ] 1. Intent map
- [ ] 2. Keyword research from real words
- [ ] 3. Fix titles and headings in templates
- [ ] 4. Rewrite a thin page
- [ ] 5. Find cannibalization
