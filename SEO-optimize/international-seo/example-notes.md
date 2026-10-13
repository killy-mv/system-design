# International SEO: hands-on practice notes

Planned labs for learning international SEO by **adding an English version to a Vietnamese site, wiring up hreflang from one source of truth, and proving Googlebot can see every version**. Theory is in [README.md](README.md).

These labs use the shared `seo-shop` site (see [how-search-works notes](../how-search-works/example-notes.md)). Its relevant problems: an IP-based redirect of foreign visitors to an English page, a firewall geo-block of non-Vietnam IPs, and language chosen by cookie on a single URL. You need a US VPS or VPN exit (to see what Googlebot sees), `curl`, and a hreflang checker (a crawler such as Screaming Frog's free tier, or a small script).

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** translation pipeline and locale formatting ([Internationalization](../../UI-UX/internationalization/README.md)), canonical tags ([Indexability labs](../indexability/example-notes.md)), Googlebot and the firewall ([Page experience lab 5](../page-experience/example-notes.md)).

---

## Labs

### 1. What does a foreign crawler see? ⭐
- **Do:** from a US VPS, `curl -A "Googlebot" -I` and `curl -I` the home page and a product page, with no `Accept-Language` header. Follow redirects (`-L`). Repeat from a Vietnamese connection.
- **Write:** which URLs each location can reach. Which Vietnamese pages are invisible from the US?
- **See:** README §4.

### 2. Give every version its own URL
- **Fix:** move from cookie-based language on one URL to subfolders `/vi/` and `/en/`. Localise slugs (`/vi/ao-thun-trang`, `/en/white-t-shirt`) from a translation table.
- **Fix:** remove the IP redirect; add a dismissible banner suggesting the other language; remember an explicit choice in a cookie.
- **Check:** both versions return 200 from both locations, with and without the cookie.
- **See:** README §2 and §4.

### 3. Generate hreflang from one source ⭐
- **Build:** a function that, given a page, emits `<link rel="alternate" hreflang>` tags for all its versions plus `x-default`, using the translation table. Then emit the same mapping into the XML sitemap instead, and compare.
- **Check:** write a script that crawls all pages and verifies reciprocity, self-reference, valid codes, 200 status and self-canonical on every target. Break one translation and see the check fail.
- **See:** README §3.

### 4. Translate the metadata
- **Do:** list everything on a product page that is still in the wrong language: title, meta description, alt text, structured data, breadcrumb labels, price format.
- **Fix:** move them into the translation pipeline; format prices with `Intl.NumberFormat` (`vi-VN` → `₫`).
- **See:** README §6, [Internationalization](../../UI-UX/internationalization/README.md).

### 5. Add a country version
- **Do:** add an `en-SG` version with SGD prices and Singapore shipping text. Update hreflang to `vi-VN`, `en-VN`, `en-SG`, `x-default`.
- **Check:** the reciprocity script from lab 3 still passes. Decide which pages differ enough to deserve a Singapore version and which should not have one.
- **See:** README §1 and §7.

---

## Progress

- [ ] 1. What does a foreign crawler see?
- [ ] 2. Give every version its own URL
- [ ] 3. Generate hreflang from one source
- [ ] 4. Translate the metadata
- [ ] 5. Add a country version
