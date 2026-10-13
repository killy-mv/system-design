# Structured data

> **Structured data** = machine-readable facts about a page (this is a product, it costs 1,290,000 ₫, it's in stock, it has 4.6 stars from 128 reviews), written in a standard vocabulary so search engines don't have to guess from the HTML.
> "Can a machine read the key facts on this page without guessing, and do those facts match what users see?"

Search engines can read text, but they can't always tell whether "1,290,000" is a price, a phone number or a product code. Structured data says it explicitly, using the **schema.org** vocabulary, a shared standard maintained by Google, Microsoft, Yahoo and Yandex. That makes it a form of [interoperability](../../portability-compatibility/interoperability/README.md): one agreed format, many consumers.

Related: [On-page relevance](../on-page-relevance/README.md) (the visible content the markup describes), [Local SEO](../local-seo/README.md) (`LocalBusiness`), [SEO monitoring](../seo-monitoring/README.md) (rich result reports), [Integrity](../../security/integrity/README.md) (markup must stay consistent with the data).

---

## 1. What it does and doesn't do

| It does | It doesn't |
|---|---|
| Make a page **eligible** for rich results (price, stars, stock, breadcrumbs, event dates in the result) | Guarantee rich results. Google decides per query and per page |
| Help search engines understand entities (this organization, this product, this author) | Act as a direct ranking factor |
| Often improve click-through, because the result shows more useful information | Fix a page that isn't indexed or isn't relevant |
| Feed other consumers: Merchant listings, Bing, social previews, AI assistants, aggregators | Replace good visible content |

---

## 2. Formats

| Format | How it looks | Use it? |
|---|---|---|
| **JSON-LD** | A `<script type="application/ld+json">` block, separate from the HTML | **Yes. Google recommends it.** Easy to generate from data and to test |
| **Microdata** | `itemprop` attributes mixed into HTML tags | Works, but couples markup to templates and breaks easily when the HTML changes |
| **RDFa** | Attributes in HTML, similar to microdata | Rare for SEO |

---

## 3. Example: product page with breadcrumbs

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Toshiba RC-18 Rice Cooker 1.8 L",
  "image": ["https://seo-shop.example/img/toshiba-rc-18-white.jpg"],
  "description": "1.8 L rice cooker with ceramic pot and 12-hour keep-warm.",
  "sku": "RC-18-WH",
  "gtin13": "4904530000000",
  "brand": { "@type": "Brand", "name": "Toshiba" },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": 4.6,
    "reviewCount": 128
  },
  "offers": {
    "@type": "Offer",
    "url": "https://seo-shop.example/rice-cookers/toshiba-rc-18",
    "price": 1290000,
    "priceCurrency": "VND",
    "availability": "https://schema.org/InStock",
    "itemCondition": "https://schema.org/NewCondition"
  }
}
</script>
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Kitchen", "item": "https://seo-shop.example/kitchen" },
    { "@type": "ListItem", "position": 2, "name": "Rice cookers", "item": "https://seo-shop.example/rice-cookers" },
    { "@type": "ListItem", "position": 3, "name": "Toshiba RC-18" }
  ]
}
</script>
```

Notes:
- `price` is a number without separators or currency symbols; the currency goes in `priceCurrency` as an ISO 4217 code (`VND`).
- `availability` and `itemCondition` use full schema.org URLs.
- The last breadcrumb (the current page) may omit `item`.
- The `gtin13` above is a placeholder. Use real identifiers, or leave the field out.
- `aggregateRating` must come from real reviews shown on the page.

---

## 4. Common types and their status

Google's supported features change. Check the Search Gallery in Google's documentation before building one.

| Type | Rich result | Status / notes |
|---|---|---|
| `Product` + `Offer` (+ `AggregateRating`, `Review`) | Price, availability, stars in results; merchant listings | Supported. The main one for shops |
| `BreadcrumbList` | Breadcrumb trail instead of the raw URL | Supported (on mobile, Google has simplified how it displays the URL line, but the markup still describes structure) |
| `Organization` | Logo, name and contact details in knowledge panels | Supported; put it on the homepage |
| `LocalBusiness` | Business details; helps match the website to the business | Supported; see [Local SEO](../local-seo/README.md) |
| `Article` | Better title, image and date display | Supported for news and blog pages |
| `Event` | Event dates and location in results | Supported |
| `VideoObject` | Video thumbnail, duration, key moments | Supported |
| `Review` snippet | Stars for a reviewed item | Supported, but self-serving reviews of your own organization aren't shown as stars |
| `FAQPage` | Expandable Q&A under the result | **Restricted since 2023** to well-known, authoritative government and health sites. Valid markup elsewhere usually won't show |
| `HowTo` | Steps in the result | **Removed** from Google results (2023) |

---

## 5. Rules

Google's structured data guidelines in short:

- **Describe what's visible.** Markup must match the content users see. A rating in JSON-LD that isn't on the page, or a price that differs from the page, is misleading.
- **Be accurate and current.** Out-of-stock products marked `InStock` and stale prices erode trust and can be flagged.
- **Mark up the main content of the page**, not every product on a category page as if each were the page's subject.
- **No fake reviews or ratings.**
- **Violations can lead to a manual action** that removes rich results for the site, reported in Search Console under Manual actions.

---

## 6. Generate it from the data model

Hand-written JSON-LD drifts out of sync with the page within weeks. Generate it **from the same data that renders the visible page**, on the server:

```
             ┌─────────────────────────┐
 products DB │ name, price, stock, ... │
             └────────────┬────────────┘
                          │ one product object
            ┌─────────────┴──────────────┐
            ▼                            ▼
   HTML template (visible)      toProductJsonLd(product)
   price: 1,290,000 ₫           "price": 1290000
            └─────────────┬──────────────┘
                          ▼
              server-rendered HTML response
```

```ts
// One function per type, unit-tested, used by every product page
function toProductJsonLd(p: Product, baseUrl: string) {
  return {
    "@context": "https://schema.org",
    "@type": "Product",
    name: p.name,
    image: p.images.map(i => `${baseUrl}${i.path}`),
    sku: p.sku,
    brand: { "@type": "Brand", name: p.brand },
    ...(p.reviewCount > 0 && {
      aggregateRating: { "@type": "AggregateRating", ratingValue: p.rating, reviewCount: p.reviewCount },
    }),
    offers: {
      "@type": "Offer",
      url: `${baseUrl}${p.path}`,
      price: p.priceVnd,
      priceCurrency: "VND",
      availability: p.stock > 0 ? "https://schema.org/InStock" : "https://schema.org/OutOfStock",
    },
  };
}
```

Design points:
- **Single source of truth:** visible price and markup price come from the same field, so they can't disagree ([Integrity](../../security/integrity/README.md)).
- **Omit, don't invent:** leave optional fields out when there's no data (no rating block with zero reviews).
- **Escape safely:** serialize with a JSON library and escape `</script>` sequences; product names are user-editable data.
- **Server-render it.** Google can read JSON-LD injected by JavaScript, but rendering may be delayed and other consumers may not run JavaScript at all ([Indexability](../indexability/README.md)).
- **Cache-aware:** if pages are cached at the CDN, price and stock in the markup are only as fresh as the cache.

---

## 7. Validation in CI

| Check | Tool | When |
|---|---|---|
| Syntax and schema.org vocabulary | **Schema Markup Validator** (validator.schema.org) | During development |
| Eligibility for Google rich results | **Rich Results Test** (search.google.com/test/rich-results) | Before release, on sample URLs |
| Required fields present, values consistent with the page | Unit tests on `toProductJsonLd`; an end-to-end test that parses the JSON-LD and compares `price` with the visible price | Every build |
| Live errors and warnings across the site | Search Console → Enhancements / Shopping reports | Weekly ([SEO monitoring](../seo-monitoring/README.md)) |

A simple CI test: render three known product pages, extract every `application/ld+json` block, `JSON.parse` it, assert required fields exist, and assert the markup price equals the price element's text after normalizing.

---

## 8. Anti-patterns

- Hand-pasted JSON-LD that nobody updates when prices change.
- Markup for content that isn't on the page (ratings, FAQs, prices).
- Fake or self-written reviews to get stars.
- Adding `FAQPage` or `HowTo` everywhere expecting rich results that no longer appear for most sites.
- Marking up every item on a list page as if each were the main product.
- Prices with separators or symbols in `price` (`"1.290.000 ₫"`).
- Treating structured data as a ranking trick rather than a description of the page.

---

## 9. Checklist

- [ ] JSON-LD, generated server-side from the same data as the visible page
- [ ] `Organization` on the homepage; `BreadcrumbList` on deep pages
- [ ] `Product` + `Offer` on product pages with correct currency, price and availability
- [ ] Ratings only from real, visible reviews
- [ ] No markup for invisible content; no deprecated types expected to produce rich results
- [ ] Unit tests for each generator; end-to-end consistency test in CI
- [ ] Rich Results Test passed on sample URLs before release
- [ ] Search Console enhancement reports checked regularly; manual actions none

---

## 10. Exercises

1. View the source of a large e-commerce product page. Find its JSON-LD. Which types and fields does it use? Do the values match the visible page?
2. Write `BreadcrumbList` and `Organization` markup for a site you know. Validate both with the Schema Markup Validator and the Rich Results Test. Explain any difference between the two tools' results.
3. Design how structured data stays correct when prices are cached at the CDN for 10 minutes and stock changes every few seconds. What's acceptable, and what isn't?
4. A product manager wants FAQ stars on every product page. Explain, with references to Google's documentation, what will and won't happen.

---

## 11. Further reading

Google Search Central: "Introduction to structured data markup in Google Search", "General structured data guidelines", the Search Gallery of supported features, "Product (Product, Review, Offer) structured data", and the Search Central Blog post "Changes to HowTo and FAQ rich results" (August 2023); schema.org documentation for `Product`, `Offer` and `BreadcrumbList`; the JSON-LD 1.1 W3C Recommendation; Bing Webmaster Guidelines on markup.
