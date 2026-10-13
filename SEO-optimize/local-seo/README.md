# Local SEO

> **Local SEO** = making a business with physical locations or a service area show up when people nearby search for what it offers, especially in the map results ("local pack") and on Google Maps.
> "When someone near our branch searches 'phở gần đây' or 'pho near me', do they find us, with the right address, hours and phone number?"

Local results work differently from ordinary web results. For a query with local intent, Google shows a map with a few businesses (the **local pack**) above or among the web results, and those listings come mainly from **Google Business Profile** data, not from your website alone. A small restaurant with no website at all can appear there; a large brand with a great website but a wrong address may not.

Related: [How search works](../how-search-works/README.md), [Structured data](../structured-data/README.md) (`LocalBusiness` markup), [Site architecture](../site-architecture/README.md) (location pages), [International SEO](../international-seo/README.md) (country-level targeting), [Authority](../authority/README.md) (links and mentions), [SEO monitoring](../seo-monitoring/README.md).

---

## 1. Where local results appear

| Surface | Triggered by | Data source |
|---|---|---|
| **Local pack** (map + a few listings in web results) | Queries with local intent: "phở Hà Nội", "dentist near me", "bánh mì" from a phone | Business profiles, ranked per searcher location |
| **Google Maps** search | Any search in Maps | Business profiles |
| **Knowledge panel** for a business | Brand searches ("Phở Thìn Lò Đúc") | Business profile plus web information |
| **Ordinary web results** | Same query | Your website, location pages, directories |

Local intent is often **implicit**. "Phở" typed on a phone in Hanoi is treated as local even without "near me". The searcher's location (GPS on mobile, IP address, or the location in the query) changes results street by street.

---

## 2. How local ranking works

Google states three main factors:

| Factor | Meaning | What you control |
|---|---|---|
| **Relevance** | How well the profile matches the search | Correct primary category, complete profile, services and menu, website content |
| **Distance** | How far the business is from the searcher or the location in the query | Nothing (except accurate address and service area) |
| **Prominence** | How well known the business is | Reviews (count and rating), links and mentions on the web, press, overall web presence |

**Distance is not under your control.** A branch 3 km away will rarely beat one 300 m away for a "near me" search, however well optimised. Opening a branch in the right place is a business decision, not an SEO trick. Fake addresses to appear "closer" violate Google's guidelines and get profiles suspended.

---

## 3. Google Business Profile

The single most important local asset. Claim and verify a profile for **each physical location** customers visit.

| Field | Guidance |
|---|---|
| **Name** | The real business name as on the sign. Adding keywords ("Phở Ngon - Best Pho Hanoi Cheap") violates the guidelines |
| **Primary category** | The most specific accurate one ("Pho restaurant" rather than "Restaurant"); add secondary categories sparingly |
| **Address or service area** | Storefronts show an address; service-area businesses (plumbers, delivery-only) hide it and list areas served |
| **Phone** | A local number that reaches that location |
| **Hours** | Including special hours for Tết and public holidays; wrong hours generate angry reviews |
| **Website** | Link to that location's page, not the home page, for multi-location businesses |
| **Photos, menu, products, services, attributes** | Real and current; they help relevance and conversions |
| **Posts and Q&A** | Optional; useful for events and common questions |

---

## 4. NAP consistency

**NAP** = Name, Address, Phone. It should be identical (or trivially different) everywhere the business appears:

- Google Business Profile, Apple Maps (Apple Business Connect), Bing Places
- Your website's location pages and footer
- Directories and platforms relevant to the market. In Vietnam that can include food and delivery platforms, Foody and similar review sites, and Facebook pages
- Social profiles

Inconsistent details (old address after a move, a different phone on each platform) confuse both search engines and customers. When a branch moves or closes, update every listing, and mark closed locations as permanently closed rather than leaving them.

---

## 5. Reviews

Reviews influence prominence and, more directly, whether people choose you.

Do:
- Ask every customer, the same way (a QR code on the table or receipt, a link in the order confirmation).
- Reply to reviews, especially negative ones, politely and specifically.
- Fix the problems reviews reveal.

Don't:
- **Buy reviews**, offer incentives for reviews, or write them yourself. This violates Google's policies and can get reviews removed or the profile restricted.
- **Gate reviews**: asking happy customers for a Google review while sending unhappy ones to a private form is against Google's policies.
- Report legitimate negative reviews as spam.

---

## 6. Location pages on your website

For multi-location businesses, one page per location:

```
/chi-nhanh/                      ← list of branches with a map
/chi-nhanh/hoan-kiem/            ← one branch
/chi-nhanh/cau-giay/
/chi-nhanh/quan-1-tp-hcm/
```

Each location page should have:

- The exact NAP, matching the business profile
- Hours, embedded map, directions, parking, nearby landmarks
- **Real, location-specific content**: photos of that branch, its menu differences, staff, events. Not the same paragraph with the district name swapped
- `LocalBusiness` (or a more specific type such as `Restaurant`) structured data with address, geo coordinates, opening hours and phone ([Structured data](../structured-data/README.md))
- Crawlable links from a branch index page ([Site architecture](../site-architecture/README.md))

Thousands of "doorway pages" for cities where you have no presence ("pho delivery in Đà Nẵng" from a Hanoi-only shop) are treated as spam.

---

## 7. Worked example: a phở chain with five branches in Hanoi

| Problem found | Fix |
|---|---|
| One Google Business Profile for the whole chain, pointing at the head office | One verified profile per branch, each with its own phone and hours |
| Profile names "Phở X - Phở Ngon Nhất Hà Nội" | Real name "Phở X Hoàn Kiếm" as on the sign |
| Website has one "Contact" page listing all five addresses as an image | `/chi-nhanh/` index plus five location pages with text NAP, map, photos and `Restaurant` structured data |
| Old Cầu Giấy address still on delivery apps after a move | Update every listing; mark the old location closed |
| Tết hours never set; customers arrive to closed shutters | Special hours added before every holiday |
| Reviews only answered when negative, often defensively | Reply to all reviews within a few days; QR code on tables asking every customer |

Expected result: each branch competes in its own neighbourhood. None of it changes the distance factor: a customer in Long Biên will still mostly see the Long Biên competitors.

---

## 8. Anti-patterns

- Keyword-stuffed business names.
- Fake or virtual-office addresses to appear in more areas.
- One profile for many locations, or several duplicate profiles for one location.
- Different phone numbers and addresses on every platform.
- Bought, incentivised or gated reviews.
- Location pages that are copies with the city name swapped; doorway pages for places you don't serve.
- Address shown only as an image, or the map only in JavaScript without text.
- Holiday hours never updated.

---

## 9. Checklist

- [ ] One verified Google Business Profile per physical location; also Bing Places and Apple Business Connect
- [ ] Real name, most specific primary category, correct address or service area, local phone, hours including holidays
- [ ] NAP identical on the website, profiles, directories and platforms
- [ ] One location page per branch with unique content, text NAP, map and `LocalBusiness` structured data
- [ ] Profile links to the matching location page
- [ ] A routine to ask for reviews and reply to them; no incentives or gating
- [ ] Closed and moved locations updated everywhere
- [ ] Local results checked from each branch's area, not only from the office

---

## 10. Exercises

1. Search "phở" (or another local business type) on your phone at two places 2 km apart. How much do the local pack results change? What does that tell you about the distance factor?
2. Pick a local business you know. Compare its name, address, phone and hours on Google Maps, its website, Facebook and one delivery or review platform. List the inconsistencies.
3. Design the location pages and structured data for a dental clinic with three branches in Ho Chi Minh City. What content makes each page genuinely different?
4. A client asks you to create 60 city pages to "rank everywhere in Vietnam" while having two branches. Explain the risk and propose an alternative.

---

## 11. Further reading

Google Business Profile Help: "Tips to improve your local ranking on Google" and "Guidelines for representing your business on Google"; Google's policies on fake engagement and reviews; schema.org `LocalBusiness` and Google Search Central documentation on local business structured data; Bing Places and Apple Business Connect documentation; Whitespark's annual Local Search Ranking Factors survey (expert opinion, not Google data).
