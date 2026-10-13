# Structured data: hands-on practice notes

Planned labs for learning structured data by **generating schema.org markup from the product data model, validating it, and keeping it consistent with the visible page in CI**. Theory is in [README.md](README.md).

The labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)), which starts with no structured data and one hand-pasted, outdated JSON-LD block on a single product. Tools: **Schema Markup Validator** (validator.schema.org), **Rich Results Test**, **Search Console** (Enhancements and Shopping reports, if the site is deployed and verified), and your test runner (Vitest/Jest or pytest).

Keep a `CHANGELOG-labs.md`: for each lab, what you changed and what the validators reported before and after.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** rewriting thin product pages (on-page-relevance lab 4), `LocalBusiness` for a physical store (local-seo labs), contract tests between services (interoperability labs).

---

## Labs

### 1. Read other people's markup
- **Do:** open the source of three real product pages from different shops. Extract their JSON-LD and run each through the Schema Markup Validator and the Rich Results Test.
- **Write:** a table: types used, fields present, errors and warnings, whether visible values match the markup.
- **See:** README §3–4.

### 2. Generate Product and BreadcrumbList ⭐
- **Build:** `toProductJsonLd(product)` and `toBreadcrumbJsonLd(path)` on the server, rendered into every product page. Remove the hand-pasted block.
- **Check:** `curl` a product page and confirm the JSON-LD is in the raw HTML. Validate three products, including one out of stock and one with no reviews (no `aggregateRating` block).
- **See:** README §6.

### 3. Break consistency on purpose
- **Do:** change a price in the database while the page is cached at the edge or in an in-memory cache. Compare the visible price, the markup price and the price shown after cache expiry.
- **Then:** decide on a cache TTL or purge-on-update rule for product pages, and write it down with the reason.
- **See:** README §5–6.

### 4. Validation in CI ⭐
- **Build:** unit tests for each generator (required fields, `priceCurrency` = `VND`, numeric `price`, availability URL). Add an end-to-end test that renders three product pages, parses every `application/ld+json` block and asserts the markup price equals the visible price.
- **Check:** introduce a bug (format `price` as `"1.290.000 ₫"`) and confirm CI fails.
- **See:** README §7.

### 5. Organization and the deprecated types
- **Build:** `Organization` markup on the homepage (name, logo, URL, contact point).
- **Do:** add `FAQPage` markup to one product page and run the Rich Results Test. Record what it says about eligibility, then remove it or keep it with a written reason.
- **See:** README §4.

---

## Progress

- [ ] 1. Read other people's markup
- [ ] 2. Generate Product and BreadcrumbList
- [ ] 3. Break consistency on purpose
- [ ] 4. Validation in CI
- [ ] 5. Organization and the deprecated types
