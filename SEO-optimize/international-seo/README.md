# International SEO

> **International SEO** = structuring and signalling a site so that search engines show **the right language and country version** of each page to each searcher.
> "When someone in Vietnam searches in Vietnamese, and someone in the US searches in English, does each land on the version meant for them?"

Search engines don't translate your site or guess your audience reliably. You have to tell them which versions exist, what language and region each targets, and how they relate. Done badly, versions compete with each other, the wrong one ranks, or one gets treated as a duplicate of another.

This note covers the search side. Building the translations, locale formatting and right-to-left layouts is in [Internationalization](../../UI-UX/internationalization/README.md).

Related: [Indexability](../indexability/README.md) (canonicals and duplicates), [Site architecture](../site-architecture/README.md) (URL design), [Local SEO](../local-seo/README.md) (city-level targeting), [Page experience](../page-experience/README.md) (server location and speed), [Crawlability](../crawlability/README.md) (Googlebot's location).

---

## 1. Language vs country targeting

Decide what you are actually targeting before choosing URLs:

| Situation | Target | Example |
|---|---|---|
| Same content, different languages | **Language** | A Vietnamese and an English version of a SaaS product page |
| Same language, different countries with different prices, stock, law | **Language + region** | `en-US` and `en-GB` with different currencies and shipping |
| A business that only serves one country | **One country** | A Vietnamese shop that ships only within Vietnam; English version for expats still targets Vietnam |

Don't create country versions you can't make meaningfully different. Ten near-identical English pages for ten countries are mostly duplicates.

---

## 2. URL structure options

Every language or country version needs **its own URL**. Search engines index URLs, not sessions or cookies.

| Structure | Example | Geo signal | Pros | Cons |
|---|---|---|---|---|
| **ccTLD** (country domain) | `shop.vn`, `shop.co.uk` | **Strong**: Google treats a ccTLD as targeting that country | Clear to users and search engines | Separate domains to buy and run; authority split between them; some ccTLDs have registration rules |
| **Subdomain** | `vi.shop.com`, `en.shop.com` | None by itself | Easy to host separately | Somewhat separate from the main site in practice; more setup |
| **Subfolder** | `shop.com/vi/`, `shop.com/en/` | None by itself | One domain, authority consolidated, easy to run | Can't host versions in different places as easily (a CDN removes most of that need) |
| **URL parameter** | `shop.com/?lang=vi` | None | Simple to build | Not recommended: easy to mishandle, hard for users to recognise |
| **Same URL, content by cookie, IP or `Accept-Language`** | `shop.com` | None | No extra URLs | **Avoid**: Googlebot crawls mostly from the US without language preferences, so it may only ever see one version |

**Subfolders** are the usual default for most companies: one domain, one set of infrastructure, consolidated links. Use ccTLDs when you have a strong local brand or legal reasons.

Note that **generic TLDs such as `.com`, `.io`, `.ai` are not tied to a country**, and some two-letter domains (`.co`, `.io`, `.tv`, and others) are treated by Google as generic. Search Console's International Targeting report, which let you set a target country for a gTLD, was **retired in 2022**; hreflang and content now do that job.

---

## 3. hreflang

`hreflang` tells search engines "this page has equivalent versions in other languages or regions; here they are". It does not boost rankings. It helps the right version get shown.

### Example: a product page in Vietnamese and English

In the `<head>` of **both** pages:

```html
<link rel="alternate" hreflang="vi" href="https://shop.example/vi/ao-thun-trang" />
<link rel="alternate" hreflang="en" href="https://shop.example/en/white-t-shirt" />
<link rel="alternate" hreflang="x-default" href="https://shop.example/en/white-t-shirt" />
```

Rules:

| Rule | Why |
|---|---|
| **Reciprocal**: every page in the set lists all the others, and each one lists the first back | Prevents a third party from claiming your page as their translation. Non-reciprocal tags may be ignored |
| **Self-reference**: each page also lists itself | Part of the complete set |
| **Valid codes**: ISO 639-1 language (`vi`, `en`), optionally plus ISO 3166-1 region (`vi-VN`, `en-US`, `en-GB`) | `en-UK` is wrong (`GB` is the code); a region alone (`VN`) is invalid |
| **`x-default`**: the fallback for users who match no listed version | Often the English page or a language picker |
| **Absolute URLs that return 200 and are canonical** | hreflang pointing to a redirect, a 404 or a non-canonical URL is ignored |
| **Canonical points to itself**, not to the other language | A Vietnamese page canonicalised to the English one tells Google to drop the Vietnamese page |

### Where to put it

| Method | Good for |
|---|---|
| HTML `<link>` in `<head>` | Most sites |
| HTTP `Link:` header | Non-HTML files such as PDFs |
| XML sitemap (`<xhtml:link>` entries) | Large sites; keeps HTML small and centralises the mapping |

Pick one method and generate it from one source of truth (the translation table in your CMS or database). Hand-maintained hreflang breaks quickly.

---

## 4. Don't redirect by IP or browser language

Automatic redirects based on the visitor's IP address or `Accept-Language` header cause two problems:

- **Googlebot crawls mostly from US IP addresses** and usually sends no language preference. If US visitors are redirected to `/en/`, Googlebot may never see `/vi/`, and the Vietnamese pages never get indexed.
- **Users are wrong-guessed**: a Vietnamese person travelling in Japan, or an expat in Hanoi who wants English.

Instead:

- Serve the requested URL as-is.
- Show a **dismissible banner** ("Bạn đang ở Việt Nam? Xem trang tiếng Việt") linking to the matching version.
- Remember the user's explicit choice in a cookie, and still keep every version reachable by its own URL.

The same applies to firewall rules that block non-local IPs: they block Googlebot too.

---

## 5. Signals that actually indicate the target market

| Signal | Strength |
|---|---|
| ccTLD (`.vn`) | Strong for country |
| hreflang | Strong for choosing between your own versions |
| Content language (the actual text) | Strong for language |
| Local details: currency (₫), addresses, phone numbers (+84), local shipping, local references | Strong supporting evidence |
| Links from sites in the target country | Moderate |
| Google Business Profile ([Local SEO](../local-seo/README.md)) | Strong for local results |
| **Server IP location** | **Weak, close to negligible.** CDNs make it meaningless; Google has said it relies on other signals |

So hosting in the US with a CDN does not stop a `.vn` site with Vietnamese content from ranking in Vietnam. What hosting location does affect is **speed**, covered in [Page experience](../page-experience/README.md).

---

## 6. Translated and duplicate content

- **Translations are not duplicate content.** `/vi/` and `/en/` versions of the same page are fine.
- **Same-language country versions** (`en-US`, `en-GB`, `en-SG`) that are nearly identical may be clustered as duplicates. hreflang helps Google show the right one, and real differences (currency, shipping, contact details) help more.
- **Machine translation**: Google's spam policies target content generated at scale without adding value, not translation as such. Unreviewed machine translation that reads badly hurts users and quality signals. Have important pages reviewed by a fluent speaker.
- **Translate the metadata too**: titles, meta descriptions, image alt text, structured data, URLs (slugs can be localised: `/vi/ao-thun-trang`).
- **Vietnamese diacritics** in URLs: either keep them (percent-encoded in the address bar, readable in many places) or strip them (`ao-thun-trang`). Stripping is more common and avoids encoding problems; pick one and be consistent.

---

## 7. Worked example: a Vietnamese shop going regional

`shop.vn` sells clothing in Vietnam and wants to add English for expats and tourists, then expand to Singapore.

| Step | Decision |
|---|---|
| English for people in Vietnam | Subfolder `shop.vn/en/`, `hreflang="en-VN"`; prices still in VND, shipping still Vietnam |
| Vietnamese stays the default | `shop.vn/` (or `/vi/`) with `hreflang="vi-VN"`; `x-default` → language picker or `/en/` |
| Singapore | Different currency, shipping and law → a real country version: `shop.com.sg` or `shop.com/en-sg/` with `hreflang="en-SG"` and prices in SGD |
| Redirects | None automatic; a banner suggests the Singapore site to visitors from Singapore |
| Hosting | One origin plus a global CDN; no need for a server in Singapore for SEO |

---

## 8. Anti-patterns

- One URL whose language changes by IP, cookie or `Accept-Language`.
- Automatic redirects by IP that hide versions from Googlebot.
- hreflang that is not reciprocal, uses invalid codes (`en-UK`, `vn`), or points to redirects or non-canonical URLs.
- Every language version canonicalised to the English page.
- Country versions that are copies with only the flag changed.
- Translating the body but leaving titles, descriptions and structured data in English.
- Hosting in the target country "for SEO" while ignoring hreflang and content.
- Language picker that only works with JavaScript and has no crawlable links.

---

## 9. Checklist

- [ ] Target defined per version: language, or language + country
- [ ] Each version has its own crawlable URL (subfolder, subdomain or ccTLD)
- [ ] hreflang generated from one source: reciprocal, self-referencing, valid codes, `x-default`
- [ ] Each version's canonical points to itself
- [ ] No automatic IP or language redirects; banner suggests alternatives
- [ ] No firewall or geo rule blocks Googlebot
- [ ] Titles, descriptions, alt text and structured data translated
- [ ] Local currency, contact details and shipping on country versions
- [ ] Translations reviewed by fluent speakers for important pages
- [ ] Search Console checked for hreflang and duplicate issues per version

---

## 10. Exercises

1. Pick an international site you use. Find its hreflang (view source, or HTTP headers, or sitemap). Is it reciprocal? Is there an `x-default`?
2. Your Vietnamese site redirects all non-Vietnamese IPs to `/en/`. Explain step by step what Googlebot sees and which pages end up indexed.
3. Choose a URL structure for a Hanoi-based SaaS selling in Vietnam, Thailand and Japan, with one shared product. Justify subfolders vs ccTLDs.
4. Write the hreflang tags for a page available in `vi-VN`, `en-VN`, `en-SG` and a global English fallback.

---

## 11. Further reading

Google Search Central documentation "Managing multi-regional and multilingual sites" and "Tell Google about localized versions of your page" (hreflang); "How Google crawls locale-adaptive pages"; Google's spam policies (scaled content abuse); ISO 639-1 and ISO 3166-1 code lists; Bing Webmaster guidelines on language and region targeting; the [Internationalization](../../UI-UX/internationalization/README.md) note in this repo for the engineering side of translations and locales.
