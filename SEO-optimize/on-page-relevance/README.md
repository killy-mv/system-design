# On-page relevance

> **On-page relevance** = the page clearly answers what the searcher wants, in the words they use, and the main signals on the page (title, headings, body text, images, URL) all point at the same topic.
> "If someone typed this query, would they be glad they landed on this page, and can a machine tell what it's about?"

Crawling and indexing ([Crawlability](../crawlability/README.md), [Indexability](../indexability/README.md)) only get a page *into* the index. Relevance decides which queries it can rank for at all, and quality decides how high. No amount of speed or links makes a page rank for a query it doesn't answer.

Related: [How search works](../how-search-works/README.md) (retrieval and ranking), [Structured data](../structured-data/README.md) (machine-readable facts about the page), [Site architecture](../site-architecture/README.md) (one page per topic, internal links), [Accessibility](../../UI-UX/accessibility/README.md) (headings and alt text serve both), [Usability](../../UI-UX/usability/README.md).

---

## 1. Search intent

Every query has a goal behind it. The page type has to match the goal before the wording matters.

| Intent | The user wants to... | Example query | Page that fits |
|---|---|---|---|
| **Informational** | Learn something | "how to choose a rice cooker" | Guide, article, comparison table |
| **Navigational** | Reach a specific site or page | "seo-shop login" | The login page, the homepage |
| **Commercial investigation** | Compare before buying | "best rice cooker under 2 million" | Reviews, comparisons, category page with filters |
| **Transactional** | Buy or do something now | "buy Toshiba RC-18 rice cooker" | Product page with price, stock, buy button |

**How to find the intent:** search the query yourself and look at what already ranks. If the top ten are all guides, a product page will struggle, and the reverse. Google has already learned what people click for that query.

Mixed intent is common ("rice cooker" could be any of the four). Category pages often serve it best: a short intro, filters, and products.

---

## 2. Keyword research: finding the user's words

Keyword research is not about stuffing words in. It's about learning **how real people describe what you offer**, which is often different from how your company describes it.

| Source | What it tells you |
|---|---|
| **Search Console → Performance → Queries** | Queries you already appear for, with impressions, clicks, CTR, average position. The best source for an existing site |
| **Google autocomplete and "People also ask"** | Common phrasings and follow-up questions |
| **Google Trends** | Relative interest over time and by region; seasonality; which of two phrasings is more common |
| **Google Ads Keyword Planner** | Rough search volume ranges (needs an Ads account; ranges are wide without spend) |
| **Site search logs, support tickets, reviews** | The words your customers actually use |
| **Competitor pages that rank** | Subtopics they cover that you don't |

**Group keywords by topic, not by exact string.** "rice cooker 1.8L", "1.8 litre rice cooker" and "nồi cơm điện 1.8 lít" are one topic and belong on one page (for Vietnamese and English, see [International SEO](../international-seo/README.md)). Search engines understand synonyms and paraphrases. You don't need a page per variant, and making one is harmful (§6).

---

## 3. Page elements

| Element | Purpose | Good practice | Notes |
|---|---|---|---|
| **`<title>`** | Strong relevance signal; usually the clickable headline in results | Unique per page; main topic first; brand last; roughly 50–60 characters so it isn't cut | Google may rewrite the title link from the page's headings or anchor text if the title is missing, stuffed or misleading |
| **Meta description** | Candidate snippet text under the title | Unique, a plain summary that makes people want to click; about 1–2 sentences | **Not a ranking factor.** Affects click-through. Google often generates its own snippet from the body text |
| **Meta keywords** | None | Leave it out | Ignored by Google |
| **Headings (`<h1>`–`<h6>`)** | Structure for readers, screen readers and search engines | One clear `<h1>` that states the page topic; `<h2>`s for sections; don't skip levels for styling | Google says multiple `<h1>`s are fine. Clear structure matters more than the count ([Accessibility](../../UI-UX/accessibility/README.md)) |
| **Body text** | The actual answer | Cover the topic fully: the facts, specs, steps and questions a searcher would need. Use the words people search with, naturally | Length is not a target. Answer completely, then stop |
| **Images and `alt` text** | Image search; context for the page; accessibility | Descriptive `alt` ("Toshiba RC-18 rice cooker, white, 1.8 L"), descriptive file names, captions where useful | Empty `alt=""` for purely decorative images |
| **URL** | Small signal; shown in results as breadcrumbs | Short, readable, words separated by hyphens: `/rice-cookers/toshiba-rc-18` | Don't change working URLs for SEO alone; redirects lose time ([Site architecture](../site-architecture/README.md)) |
| **Internal links and anchor text** | Tell search engines what the target page is about | Descriptive anchors ("Toshiba rice cookers") instead of "click here" | See [Site architecture](../site-architecture/README.md) |

**Server-render the important parts.** Title, headings, main text and links should be in the HTML the server returns, not only added later by JavaScript ([Indexability](../indexability/README.md)).

---

## 4. Content quality and E-E-A-T

Google's ranking systems try to reward **helpful, reliable, people-first content**. The "helpful content system" that targeted unhelpful, search-engine-first content was folded into the core ranking systems in March 2024, so this is now part of normal ranking, not a separate filter.

**E-E-A-T** (Experience, Expertise, Authoritativeness, Trustworthiness) comes from Google's **Search Quality Rater Guidelines**. Human raters use it to evaluate results, and their ratings help Google test changes to its systems. **Raters don't change rankings directly**, and E-E-A-T isn't a single score or ranking factor. Treat it as a description of what the systems aim to reward.

| Part | What it looks like on a page |
|---|---|
| **Experience** | First-hand use: own photos, real measurements, "we cooked 20 batches of rice with it" |
| **Expertise** | Accurate, detailed, written or reviewed by someone who knows the subject |
| **Authoritativeness** | The site or author is a known source on this topic (see [Authority](../authority/README.md)) |
| **Trustworthiness** | The most important part: accurate information, clear ownership and contact details, honest prices, secure checkout, clear returns policy |

The bar is higher for **YMYL** ("Your Money or Your Life") topics: health, finance, safety, legal.

### Google's self-check questions (paraphrased)

- Does the content provide original information, research or analysis?
- Would you bookmark it, share it, or recommend it?
- Does it leave the reader feeling they've learned enough to achieve their goal?
- Was it made mainly to attract search visits, or mainly to help people who come to it?
- Is it clear who made it and why?

---

## 5. Spam policies and AI-generated content

Google's spam policies (updated in 2024) target manipulation, not tools:

| Policy | What it means |
|---|---|
| **Scaled content abuse** | Producing many pages mainly to rank, with little value to users, **whether made by AI, people, or both**. Example: thousands of auto-generated "best X in [city]" pages |
| **Site reputation abuse** | Publishing third-party content on a trusted site mainly to exploit its ranking signals (e.g., unrelated coupon or casino pages on a news site) |
| **Expired domain abuse** | Buying an expired domain for its old reputation and filling it with low-value content |
| **Keyword stuffing** | Repeating keywords unnaturally, lists of cities or phrases |
| **Cloaking** | Showing search engines different content from users |
| **Hidden text and links** | Text the same color as the background, off-screen, etc. |

**AI-generated content is allowed** if it's helpful and accurate. Using it to mass-produce pages to manipulate rankings is scaled content abuse. Edit and fact-check AI drafts the same way you would a junior writer's. Violations can lead to lower rankings or a **manual action** shown in Search Console.

---

## 6. Keyword cannibalization

Several pages on your site target the same query and intent, so they compete with each other. The usual result is that neither ranks well, and the page Google picks keeps changing.

| Symptom | Fix |
|---|---|
| Two blog posts on the same topic | Merge into one stronger page; 301-redirect the other |
| A category page and a blog post both targeting "rice cooker" | Make their intents different (category = transactional, post = informational) and link between them |
| Filter/sort pages competing with the main category | Canonicalize or `noindex` them ([Indexability](../indexability/README.md)) |
| Near-identical product variant pages | One product page with variant selection; canonical to it |

**Detect it:** in Search Console, filter Performance by a query and look at the **Pages** tab. Several of your URLs with impressions for the same query is a warning sign.

---

## 7. Worked example: rewriting a thin product page

Before (typical auto-generated page on `seo-shop`):

```html
<title>Product 1042 | SEO Shop</title>
<meta name="description" content="SEO Shop - best shop. Rice cooker rice cooker cheap.">
<h1>RC-18</h1>
<p>Rice cooker. Good quality. Buy now.</p>
<img src="/img/1042.jpg">
```

Problems: title says nothing; stuffed description; `<h1>` is a code; no specs, no reason to choose it; image without `alt`; nothing a searcher would need.

After:

```html
<title>Toshiba RC-18 Rice Cooker 1.8 L – Price & Specs | SEO Shop</title>
<meta name="description" content="Toshiba RC-18 1.8 L rice cooker with ceramic pot and 12-hour keep-warm. 1,290,000 ₫, free delivery in Hanoi and HCMC, 24-month warranty.">
<h1>Toshiba RC-18 Rice Cooker (1.8 L)</h1>
<img src="/img/toshiba-rc-18-white.jpg" alt="Toshiba RC-18 rice cooker, white, with ceramic inner pot">
<h2>Key specs</h2>        <!-- capacity, power, pot material, dimensions, warranty -->
<h2>Who it's for</h2>     <!-- serves 4–6 people; compare with the 1.0 L model (internal link) -->
<h2>Our test notes</h2>   <!-- first-hand: cooking times, cleaning, noise -->
<h2>Questions</h2>        <!-- "Can it cook brown rice?" taken from support tickets -->
<h2>Delivery & returns</h2>
```

Plus Product structured data ([Structured data](../structured-data/README.md)) generated from the same database fields, so price and stock always match the visible page.

**Fix it at the template level.** The real fix is in the product template and the product data model (adding `brand`, `capacity_l`, `test_notes`), not in hand-editing one page. A template fix improves every product at once.

---

## 8. Anti-patterns

- Writing for a keyword list instead of for the person behind the query.
- One page per keyword variant ("rice cooker hanoi", "rice cooker ha noi", ...).
- The same title and description on every page, or the brand name first on every title.
- Keyword stuffing in titles, `alt` text and footers.
- Auto-generated pages with nothing but a heading and a product grid.
- Manufacturer descriptions copied word for word, with nothing added.
- Hiding the main content behind tabs or clicks that only load it via JavaScript later.
- Publishing AI drafts unchecked, at scale.
- Fake "experience": stock photos presented as own tests, invented authors.

---

## 9. Checklist

- [ ] Each important page targets one topic and one main intent, checked against what currently ranks
- [ ] Keyword research uses Search Console queries and real customer language, grouped by topic
- [ ] Titles unique and descriptive, generated from data in templates; no empty or duplicate titles
- [ ] Meta descriptions unique, plain summaries; meta keywords removed
- [ ] One clear `<h1>`; logical heading structure
- [ ] Main content, title, headings and links are in the server-rendered HTML
- [ ] Meaningful images have descriptive `alt` text
- [ ] Content shows first-hand experience and clear ownership (about, contact, policies)
- [ ] No scaled, near-duplicate or stuffed pages; AI-assisted content is reviewed
- [ ] Cannibalization checked in Search Console for top queries

---

## 10. Exercises

1. Pick three queries for a site you know. Classify the intent of each by looking at the current top ten. Does the site's page type match?
2. Export Search Console queries for one page. Which queries have high impressions but low CTR? Rewrite the title and description and predict the effect.
3. Find two pages on one site that compete for the same query. Decide: merge, differentiate, or canonicalize. Justify it.
4. Rewrite a thin category page so it serves mixed intent, without pushing products below the fold.
5. Read Google's "Creating helpful, reliable, people-first content" page and score one of your own pages against its questions.

---

## 11. Further reading

Google Search Central: "SEO Starter Guide", "Creating helpful, reliable, people-first content", "Influencing your title links in search results", "Control your snippets in search results", "Spam policies for Google web search", and "Google Search's guidance about AI-generated content"; Google's Search Quality Rater Guidelines (PDF, updated periodically); the Google Search Central Blog post on the March 2024 core update and spam policies; Moz, "The Beginner's Guide to SEO" (chapters on keyword research and on-page); Ahrefs blog articles on search intent.
