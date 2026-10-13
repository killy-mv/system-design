# Local SEO: hands-on practice notes

Planned labs for learning local SEO by **auditing real local listings, then building consistent location pages and structured data for a multi-branch business**. Theory is in [README.md](README.md).

These labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)), extended with three fictional pickup stores in Hanoi and Ho Chi Minh City. Keep store data (name, address, phone, hours, coordinates) in one table and generate everything from it. Labs 1 and 2 use real businesses as research subjects only: don't edit listings you don't own.

Never create Google Business Profiles for fictional locations; that violates Google's guidelines. Practise profile work only on a business you actually run or manage, with permission.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** JSON-LD and the Rich Results Test ([Structured data labs](../structured-data/example-notes.md)), URL design for branch pages ([Site architecture labs](../site-architecture/example-notes.md)).

---

## Labs

### 1. Watch distance at work ⭐
- **Do:** search "phở", "cà phê" and "nha khoa" on your phone in two places a few kilometres apart (or use a location-emulating tool). Screenshot the local pack each time.
- **Write:** which businesses appear in both, which change, and guess whether relevance, distance or prominence explains each.
- **See:** README §2.

### 2. NAP audit of a real business
- **Do:** pick a local business with several branches. For each branch, collect name, address, phone and hours from Google Maps, its website, its Facebook page and one delivery or review platform.
- **Write:** a table of inconsistencies and which one you'd fix first.
- **See:** README §4.

### 3. Location pages from one table ⭐
- **Build:** `/chi-nhanh/` index and one page per store, generated from the store table: text NAP, hours, embedded map, directions, and a section of store-specific content.
- **Check:** view source: is the address in the HTML as text (not only an image or JavaScript map)? Are the pages linked from the index and the footer?
- **See:** README §6.

### 4. LocalBusiness structured data
- **Build:** JSON-LD `Store` (a `LocalBusiness` subtype) on each location page from the same table: `name`, `address`, `geo`, `telephone`, `openingHoursSpecification`, `url`.
- **Check:** Rich Results Test and the Schema Markup Validator pass; change a phone number in the table and confirm the page, footer and JSON-LD all update together.
- **See:** README §6, [Structured data](../structured-data/README.md).

### 5. Holiday hours and closures
- **Build:** special hours for Tết in the store table, shown on the location page and in `openingHoursSpecification` with `validFrom` and `validThrough`. Add a "closed permanently" state that keeps the page with a notice and links to the nearest store.
- **Then:** write the checklist you'd follow to update every external listing when a store moves.
- **See:** README §4 and §7.

---

## Progress

- [ ] 1. Watch distance at work
- [ ] 2. NAP audit of a real business
- [ ] 3. Location pages from one table
- [ ] 4. LocalBusiness structured data
- [ ] 5. Holiday hours and closures
