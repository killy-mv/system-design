# Internationalization & Localization (i18n / l10n)

> **Internationalization (i18n)** = designing and building a product so it *can* be adapted to any language, region and culture **without changing code**.
> **Localization (l10n)** = actually adapting it for a specific market: translating text, formatting dates and money, adjusting images, payment methods, legal content.
> "Can someone in Hanoi, Cairo, Tokyo and Berlin all use this as if it were built for them?"

**i18n** and **l10n** are numeronyms: "i" + 18 letters + "n", "l" + 10 letters + "n". You'll also see **g11n** (globalization = i18n + l10n + business readiness) and **L10n** with a capital L (to avoid confusing "l" with "1").

| Term | Who does it | When | Example |
|---|---|---|---|
| **Internationalization** | Engineers and designers | Once, built into the architecture | Strings externalized, Unicode everywhere, dates stored in UTC, layouts that grow and mirror |
| **Localization** | Translators, local market teams (plus engineers for market features) | Per market, continuously | Vietnamese translation, VND prices with no decimals, MoMo payment, local address form |
| **Translation** | Translators | Part of localization | "Save changes" → "Lưu thay đổi" |

Key point: **i18n is an engineering quality attribute; l10n is the work it enables.** Bad i18n makes every new language a code change. Good i18n makes a new language mostly a translation file and some configuration.

Related: [Usability](../usability/README.md) (users' language and conventions) · [Accessibility](../accessibility/README.md) (`lang` attribute, screen reader pronunciation) · [Responsiveness](../responsiveness/README.md) (input methods, font loading).

▶ **Interactive demo:** [index.html](index.html): open it in a browser to try 4 classic i18n bugs side by side with their fixes. It covers plurals, dates and money, RTL layout, and Unicode text handling, with switchable languages, locales and time zones.

---

## Table of contents

1. [Why internationalization matters](#1-why-internationalization-matters)
2. [Locales: the core concept](#2-locales-the-core-concept)
3. [Text and Unicode](#3-text-and-unicode)
4. [Messages, translation and plurals](#4-messages-translation-and-plurals)
5. [Formatting: numbers, money, dates, names, addresses](#5-formatting-numbers-money-dates-names-addresses)
6. [Time and time zones](#6-time-and-time-zones)
7. [Layout and visual design](#7-layout-and-visual-design)
8. [Input: keyboards, IMEs and validation](#8-input-keyboards-imes-and-validation)
9. [Choosing and detecting the locale](#9-choosing-and-detecting-the-locale)
10. [Beyond translation: culture, law, payments](#10-beyond-translation-culture-law-payments)
11. [The localization workflow](#11-the-localization-workflow)
12. [Backend / system design for i18n](#12-backend--system-design-for-i18n)
13. [Testing i18n](#13-testing-i18n)
14. [Approaches by product type](#14-approaches-by-product-type)
15. [Trade-offs](#15-trade-offs)
16. [Anti-patterns](#16-anti-patterns)
17. [Checklist](#17-checklist)
18. [Exercises](#18-exercises)
19. [Further reading](#19-further-reading)

---

## 1. Why internationalization matters

- **Most users aren't native English speakers.** English is the first language of only a minority of internet users. Most growth in internet users comes from Asia, Africa and Latin America.
- **People buy in their own language.** CSA Research's *Can't Read, Won't Buy* survey (2020, 8,700+ consumers in 29 countries) found that about 3 in 4 prefer to buy products with information in their own language, and about 40% never buy from websites in other languages.
- **It's often required.** Many countries require local-language contracts, terms, consumer information or government services (e.g. Quebec's French-language law, EU consumer rules, local labelling laws). Global B2B customers expect localized admin UIs and invoices.
- **Retrofitting is very expensive.** Adding i18n later means finding every hardcoded string, every `date.toString()`, every `price.toFixed(2)`, every `VARCHAR` column in the wrong encoding, every layout with fixed widths, every email template. It touches the whole codebase. Building it in from the start costs little.
- **It makes the product better even in one language.** Externalized strings make copy changes easy. Correct Unicode handling stops names like "Nguyễn", "O'Brien" or "Zoë" from breaking signups. Correct time zone handling fixes "my meeting moved by an hour" bugs for domestic users too.
- **Data correctness.** Many i18n bugs are data bugs: emoji turned into `????` by the database, prices off by 100× (JPY has no cents), events off by one hour after a DST change, sorting that loses customers in reports.

---

## 2. Locales: the core concept

A **locale** is a set of preferences for language and regional conventions: language, script, region, number format, calendar, first day of the week, currency, measurement system, collation (sort order).

### 2.1 Language tags (BCP 47)

Locales are identified by **BCP 47** language tags (RFC 5646):

```
language [-script] [-region] [-u-extensions]
  vi              Vietnamese
  vi-VN           Vietnamese as used in Vietnam
  en-US / en-GB   English (US) / English (UK): different spelling, dates, units
  pt-BR / pt-PT   Portuguese (Brazil) / (Portugal): noticeably different vocabulary
  zh-Hans-CN      Chinese, Simplified script, mainland China
  zh-Hant-TW      Chinese, Traditional script, Taiwan
  sr-Latn / sr-Cyrl  Serbian in Latin / Cyrillic script
  es-419          Spanish for Latin America (419 = UN region code)
  ar-EG           Arabic (Egypt)
  ja-JP-u-ca-japanese   Japanese with the Japanese imperial calendar
  th-TH-u-nu-thai       Thai with Thai digits
```

Use the standard codes: **ISO 639** for languages, **ISO 15924** for scripts, **ISO 3166-1** for regions, **ISO 4217** for currencies, **IANA** names for time zones.

### 2.2 Language ≠ country ≠ currency ≠ time zone

These are **independent** preferences. Don't derive one from another:

| Assumption | Counterexample |
|---|---|
| Country → language | Switzerland has 4 national languages; India has 22 scheduled languages; Canada has English and French |
| Language → country | Spanish is spoken in 20+ countries; English in a US user's browser in Vietnam |
| Country → currency | A Vietnamese user shopping on a US site might want USD; eurozone countries share EUR |
| Country → time zone | The US, Russia, Brazil, Australia and Indonesia have several time zones |
| IP address → language | Travelers, VPNs, expats, tourists, multilingual households |
| Language → text direction | Most languages are fixed, but some (e.g. Kurdish, Azerbaijani, Punjabi) use different scripts in different regions, sometimes with different directions |

Store them as **separate user settings**: UI language, region/format locale, currency, time zone. Pick sensible defaults (from the browser or device), and let users change each one.

### 2.3 CLDR and ICU: don't build this yourself

- **Unicode CLDR** (Common Locale Data Repository) is the world's standard database of locale data: date and number formats, plural rules, currency symbols, translated country/language names, collation, time zone names, units, list patterns, for hundreds of locales.
- **ICU** (International Components for Unicode) is the reference library that implements Unicode and CLDR algorithms. It runs inside most operating systems, browsers (via the JavaScript `Intl` APIs), Java, Android, PostgreSQL (ICU collations), and many others.
- **Rule: use `Intl` / ICU / platform APIs.** Never hand-write date patterns, plural logic or number formatting per country. The rules are huge, full of exceptions, and change over time.

---

## 3. Text and Unicode

### 3.1 Unicode and UTF-8

- **Unicode** assigns a number (a **code point**, written U+XXXX) to almost every character in every writing system, plus symbols and emoji (around 150,000 characters).
- **UTF-8** is the encoding that won: variable-length (1 byte for ASCII, 2–3 bytes for most other scripts, **4 bytes for emoji and rarer characters**), ASCII-compatible, and used by about 98% of websites.
- **Use UTF-8 everywhere**: source files, HTML (`<meta charset="UTF-8">`), HTTP headers, databases, files, message queues, logs, CSV exports.

Classic encoding bug (**mojibake**): text saved in one encoding and read as another. "Nguyễn" becomes "Nguyá»…n", "café" becomes "cafÃ©". It usually means some layer in the pipeline isn't UTF-8.

### 3.2 What is a "character"? Four different answers

Users think in **characters as they see them** (grapheme clusters). Code often counts something else:

| Text | Graphemes (what users see) | Code points | UTF-16 code units (JS `.length`, Java, C#) | UTF-8 bytes |
|---|---|---|---|---|
| `a` | 1 | 1 | 1 | 1 |
| `é` (precomposed, U+00E9) | 1 | 1 | 1 | 2 |
| `é` (e + combining accent, U+0065 U+0301) | 1 | 2 | 2 | 3 |
| `ễ` (Vietnamese, precomposed) | 1 | 1 | 1 | 3 |
| `中` | 1 | 1 | 1 | 3 |
| `😀` | 1 | 1 | **2** | 4 |
| `👍🏽` (thumbs up + skin tone) | 1 | 2 | 4 | 8 |
| `👨‍👩‍👧` (family: 3 people joined by zero-width joiners) | 1 | 5 | 8 | 18 |
| `🇻🇳` (flag = 2 regional indicator letters) | 1 | 2 | 4 | 8 |

Consequences:

- **`"😀".length === 2` in JavaScript.** Character counters, "max 280 characters" limits, and truncation based on `.length` are wrong for emoji and many scripts.
- **`str.slice(0, n)`** can cut a surrogate pair (shows `�`), a combining accent off its letter, or an emoji family in half.
- **Database limits**: `VARCHAR(50)` means 50 *characters* in some databases (PostgreSQL, MySQL with utf8mb4) but 50 *bytes* in others or in some configurations. Know which, and choose limits that fit real names and addresses.
- Use **grapheme segmentation** for anything user-facing: `Intl.Segmenter` in JavaScript, `BreakIterator` in ICU/Java, Swift's `String.count` (already grapheme-based), `\X` in regex engines that support it.

```js
const s = 'Nhà mình 👨‍👩‍👧‍👦';
s.length;                                    // 20 (UTF-16 code units)
[...s].length;                               // 16 (code points)
const seg = new Intl.Segmenter('vi', { granularity: 'grapheme' });
[...seg.segment(s)].length;                  // 10 (what a person would count)

// Safe truncation
function truncate(text, max, locale) {
  const graphemes = [...new Intl.Segmenter(locale, { granularity: 'grapheme' }).segment(text)];
  return graphemes.length <= max ? text : graphemes.slice(0, max).map(g => g.segment).join('') + '…';
}
```

### 3.3 Normalization

The same visible text can be stored as **different code point sequences**: `é` can be one code point (U+00E9, NFC) or two (`e` + U+0301, NFD). Vietnamese text typed on different keyboards or copied from different sources is a common real-world source of mixed forms. macOS file systems have historically stored names in decomposed form.

```js
'José' === 'José'                        // false, though they look identical
'José'.normalize('NFC') === 'José'.normalize('NFC')   // true
```

| Form | Meaning | Use |
|---|---|---|
| **NFC** | Composed | **Store and compare text in NFC** (the W3C recommendation for the web) |
| NFD | Decomposed | Intermediate step, e.g. to strip accents |
| NFKC / NFKD | Compatibility: also folds look-alikes (`ﬁ` → `fi`, full-width `Ａ` → `A`, `²` → `2`) | Identifiers, usernames, search keys (lossy, so don't use it for display text) |

**Normalize on input** (at the API boundary) so that comparisons, uniqueness checks (usernames, emails), and search work.

### 3.4 Case, comparison and search

- **Case mapping depends on the language.** In Turkish and Azerbaijani, `"I".toLowerCase()` is `"ı"` (dotless i) and `"i".toUpperCase()` is `"İ"`. Code that lower-cases identifiers with the *user's* locale breaks in Turkey (a famous source of bugs, the "Turkey test"). Use locale-independent case mapping for identifiers (`toLowerCase()` without locale, `Locale.ROOT` in Java) and locale-aware mapping for display.
- **Case mapping can change length**: German `"straße".toUpperCase()` → `"STRASSE"`.
- **Case-insensitive comparison** should use **case folding** (or `localeCompare` with `sensitivity: 'accent'`), not `toLowerCase()` on both sides.
- **Accent-insensitive search** is expected in many languages: a Vietnamese user typing `nguyen` (no diacritics, very common on mobile) expects to find "Nguyễn". Users of French, Spanish and German have the same expectation.

```js
// Accent- and case-insensitive "folding" for search keys
function fold(s) {
  return s.normalize('NFD')
    .replace(/\p{M}/gu, '')        // strip combining marks: ễ → e, é → e
    .replace(/đ/gi, 'd')           // đ/Đ does NOT decompose in Unicode, so map it explicitly
    .replace(/ø/gi, 'o').replace(/ł/gi, 'l').replace(/ß/g, 'ss')
    .toLowerCase();
}
fold('Nguyễn Đức') // 'nguyen duc'
```

In production, use your search engine's language analyzers and folding filters (e.g. Elasticsearch/OpenSearch `asciifolding` and ICU analysis plugins) or database features (PostgreSQL `unaccent`, ICU nondeterministic collations), not hand-written maps.

### 3.5 Sorting (collation)

Alphabetical order depends on the language:

| Language | Rule example |
|---|---|
| Swedish | `Å`, `Ä`, `Ö` are separate letters **after Z** |
| German | `Ä` sorts with `A` (in dictionary order) |
| Spanish | `Ñ` is a separate letter after `N` |
| Vietnamese | `Đ` is a separate letter after `D`; tones and vowel marks have a defined order |
| Lithuanian | `Y` sorts between `I` and `J` |
| Chinese | By pinyin, stroke count, or radical, depending on context and region |
| Japanese | By reading (kana), not by kanji code point |

**Code point order (`array.sort()` in JS, `ORDER BY` with binary collation) is wrong for every language**, including English: it puts all uppercase letters before lowercase (`"Zebra" < "apple"`) and accented letters after `z`.

```js
names.sort(new Intl.Collator(userLocale).compare);          // client
// Numeric-aware: "file2" before "file10"
new Intl.Collator('en', { numeric: true }).compare('file2', 'file10');  // -1
```

On the server: use a locale-aware collation in the database (`COLLATE "vi-VN-x-icu"` in PostgreSQL, `utf8mb4_0900_ai_ci` or language-specific collations in MySQL 8) or sort with ICU in application code.

### 3.6 Writing systems you should know exist

| Feature | Scripts | Implication |
|---|---|---|
| **Right-to-left** | Arabic, Hebrew, Persian, Urdu | Mirrored layouts, bidirectional text ([§7.3](#73-right-to-left-rtl-languages)) |
| **No spaces between words** | Chinese, Japanese, Thai, Lao, Khmer, Burmese | Line breaking and word-based search need dictionaries / segmentation (`Intl.Segmenter` with `granularity: 'word'`) |
| **Complex shaping** (letters change form by position, ligatures) | Arabic, Devanagari (Hindi), Bengali, Tamil, Thai | Never render character-by-character; don't letter-space; use real fonts and system text rendering |
| **Stacked diacritics / tall glyphs** | Vietnamese (`ặ`, `ỗ`), Thai, Devanagari, Burmese | Need generous line height; fixed-height boxes clip marks |
| **Huge character sets** | Chinese, Japanese, Korean (CJK) | Fonts are often several MB per weight: rely on system fonts or subset; input via IMEs ([§8.1](#81-input-method-editors-imes)) |
| **Vertical text** | Traditional Japanese, Chinese, Mongolian | Optional for most apps; CSS `writing-mode` supports it |
| **Native digits** | Some Arabic locales (`٠١٢٣`), Persian (`۰۱۲۳`), Hindi (optional), Thai (optional), Bengali… | Format numbers with `Intl`, and **parse** input that may contain native digits |

---

## 4. Messages, translation and plurals

### 4.1 Externalize every user-facing string

All text the user can see (labels, buttons, errors, emails, push notifications, PDFs, alt text, `aria-label`s, page titles) goes into **resource files** referenced by key, never hardcoded.

```json
// en.json
{
  "checkout.pay_button": "Pay {amount}",
  "cart.items_count": "{count, plural, one {# item} other {# items}}"
}
// vi.json
{
  "checkout.pay_button": "Thanh toán {amount}",
  "cart.items_count": "{count, plural, other {# sản phẩm}}"
}
```

Common formats and libraries:

| Platform | Typical choice |
|---|---|
| Web (JS/TS) | **ICU MessageFormat** via FormatJS / `react-intl`, `i18next` (with ICU plugin or its own plural syntax), Lingui, `vue-i18n`, Angular i18n |
| Mozilla / some web projects | **Fluent** (`.ftl`), designed for natural-sounding translations with complex grammar |
| C / Python / PHP / Linux | **gettext** (`.po` / `.mo` files) |
| Android | `strings.xml` with `<plurals>` |
| iOS / macOS | **String Catalogs** (`.xcstrings`, since Xcode 15), formerly `.strings` / `.stringsdict` |
| Java | `ResourceBundle` + ICU4J `MessageFormat` |

### 4.2 Never concatenate sentences

Word order differs between languages. Concatenation hard-codes English grammar:

```js
// ✗ Bad: assumes English word order
t('page') + ' ' + current + ' ' + t('of') + ' ' + total
// en: "Page 3 of 10"   ja: "ページ 3 の 10"  ← nonsense

// ✓ Good: one complete message with placeholders; translators control the order
t('pagination', { current, total })
// en: "Page {current} of {total}"
// ja: "{total}ページ中{current}ページ目"
```

- **Whole sentences in one message**, including punctuation.
- **Named placeholders** (`{userName}`), not positional `%s %s`, so translators can reorder them.
- **Rich text inside messages** (links, bold) with tags the library understands: `"Read our <link>privacy policy</link>."`, not three separate strings around a link.
- **Don't reuse strings across contexts.** English "Open" can be a verb (button) or an adjective (status); "Post" can be a noun or verb. Other languages need different words. One key per context.
- **Give translators context**: a description, maximum length, screenshot, and what each placeholder contains.

### 4.3 Plurals

English has two plural forms (*1 file*, *2 files*). Many languages have more, or fewer. Unicode CLDR defines up to six **plural categories**: `zero`, `one`, `two`, `few`, `many`, `other`.

| Language | Categories (cardinal numbers) | Examples |
|---|---|---|
| Japanese, Chinese, Korean, Vietnamese, Thai, Indonesian | `other` only | 1 tin nhắn, 5 tin nhắn (no plural form) |
| English, German, Spanish, Italian | `one`, `other` | 1 file, 2 files, **0 files** |
| French, Brazilian Portuguese | `one` (includes **0** and 1), `many`, `other` | **0 fichier**, 1 fichier, 2 fichiers |
| Russian, Ukrainian | `one`, `few`, `many`, `other` | 1 файл, 2 файла, 5 файлов, **21 файл**, 22 файла |
| Polish | `one`, `few`, `many`, `other` | 1 plik, 2 pliki, 5 plików, **22 pliki**, 25 plików |
| Arabic | **all six**: `zero`, `one`, `two`, `few`, `many`, `other` | Different forms for 0, 1, 2, 3–10, 11–99, 100+ |

So `count === 1 ? singular : plural` is wrong in most languages, and `"file(s)"` is wrong in all of them. Use ICU plural syntax (or your platform's equivalent), backed by CLDR rules:

```
{count, plural,
  =0    {No new messages}
  one   {# new message}
  other {# new messages}}
```

```js
new Intl.PluralRules('pl').select(22);   // 'few'  → "22 nowe wiadomości"
new Intl.PluralRules('pl').select(25);   // 'many' → "25 nowych wiadomości"
new Intl.PluralRules('ar').select(2);    // 'two'
new Intl.PluralRules('en', { type: 'ordinal' }).select(22);  // 'two' → "22nd"
```

Translators add the categories their language needs. Your code just passes the number.

### 4.4 Gender, grammatical case and other variation

- **Gender**: many languages change verbs or adjectives with the gender of a person ("Elle est partie" / "Il est parti"). ICU has `select`: `{gender, select, female {…} male {…} other {…}}`. Don't ask for gender just to translate; use neutral phrasing when you don't know it.
- **Grammatical case**: a name or noun may change form depending on its role in the sentence (Finnish, Polish, Russian, Czech). Placeholders like "Send to {name}" can come out ungrammatical. Rephrase so the placeholder sits in a neutral position ("Recipient: {name}").
- **Formality**: tu/vous (French), du/Sie (German), different politeness levels in Japanese and Korean, pronoun choice in Vietnamese (bạn / anh / chị / quý khách). Choose a voice per locale and document it in the style guide.

---

## 5. Formatting: numbers, money, dates, names, addresses

**Rule: store machine values, format at display time with the user's locale.** Use `Intl` (JS), ICU, `java.text`/`java.time` formatters, `NumberFormatter`/`DateFormatter` (Apple), `android.icu` (Android), Babel (Python).

### 5.1 Numbers

| Locale | 1234567.89 |
|---|---|
| en-US | 1,234,567.89 |
| de-DE | 1.234.567,89 |
| fr-FR | 1 234 567,89 (narrow no-break space as separator) |
| de-CH | 1'234'567.89 (apostrophe as separator) |
| en-IN / hi-IN | 12,34,567.89 (lakh / crore grouping) |
| ar-EG | ١٬٢٣٤٬٥٦٧٫٨٩ (Arabic-Indic digits) |

- **Parsing is as important as formatting.** "1.234" is one thousand two hundred thirty-four in German and one point two three four in English. Use locale-aware parsing, or locale-specific input controls, and send numbers to the server as plain machine values (`1234.5`), never as formatted strings.
- Percentages, units and compact forms also vary: `Intl.NumberFormat('de', { style: 'percent' })` → `45 %`; `{ notation: 'compact' }` → "1,2 Mio." (de), "120万" (ja).

### 5.2 Money

```js
new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(1234.5);  // "$1,234.50"
new Intl.NumberFormat('de-DE', { style: 'currency', currency: 'EUR' }).format(1234.5);  // "1.234,50 €"
new Intl.NumberFormat('ja-JP', { style: 'currency', currency: 'JPY' }).format(1234);    // "￥1,234"
new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(32500000); // "32.500.000 ₫"
```

- **Locale and currency are independent**: a German user viewing a USD price sees "1.234,50 $"; a US user viewing EUR sees "€1,234.50".
- **Decimal places vary by currency** (ISO 4217 minor units): USD/EUR 2, **JPY/KRW/VND 0**, **KWD/BHD/OMR 3**. Hardcoding `toFixed(2)` is wrong.
- **Store money as integer minor units or decimal types**, plus the ISO 4217 code: `{ amount_minor: 123450, currency: "USD" }` or `DECIMAL(19,4)`. **Never use floating-point** for money.
- **Ambiguous symbols**: `$` is used by USD, CAD, AUD, MXN, SGD and others; `¥` by JPY and CNY. Show the ISO code or a qualified symbol (`US$`, `CA$`) when currencies can be mixed.
- **Prices aren't just converted**: markets have their own price points (€9.99, not €9.37), tax rules (EU prices include VAT; US prices usually add sales tax at checkout), and rounding rules (some countries round cash to 5 or 10 units).

### 5.3 Dates and times

| Locale | 4 March 2026 | Notes |
|---|---|---|
| en-US | 3/4/2026 | Month first |
| en-GB | 04/03/2026 | Day first; "03/04" is ambiguous across the Atlantic |
| de-DE | 4.3.2026 | |
| ja-JP | 2026/3/4 | Year first |
| ISO 8601 | 2026-03-04 | Unambiguous: use for data, APIs, file names |
| th-TH | 4/3/2569 | Thai Buddhist calendar by default (year + 543) |
| ja-JP-u-ca-japanese | R8/3/4 | Japanese imperial era (Reiwa 8) |

- **Don't write date patterns by hand** (`MM/DD/YYYY`). Use styles (`dateStyle: 'medium'`) or skeletons/components (`{ month: 'short', day: 'numeric' }`) and let CLDR choose the order and separators.
- **Calendars**: Gregorian is most common, but Thai Buddhist, Japanese imperial, Islamic (Hijri), Hebrew, Persian (Solar Hijri), Chinese lunisolar (for holidays) and others are in official or daily use.
- **First day of the week**: Sunday (US, Canada, Japan), Monday (most of Europe, ISO 8601, Vietnam), Saturday (some Middle Eastern countries). Date pickers and weekly reports must respect it.
- **12-hour vs 24-hour clock** depends on locale (and user override).
- **Relative times**: `Intl.RelativeTimeFormat` → "in 3 days", "sau 3 ngày nữa", "dans 3 jours".
- **Lists**: `Intl.ListFormat` → "A, B, and C" (en), "A, B et C" (fr), "A、B、C" (ja).

### 5.4 Personal names

From W3C's *Personal names around the world* and the classic *Falsehoods Programmers Believe About Names*:

- **Order varies.** Vietnamese, Chinese, Japanese, Korean and Hungarian names put the **family name first**: in "Nguyễn Văn An", *Nguyễn* is the family name and *An* is the given name people use. "Dear Mr. Nguyen" is awkward, and since a large share of Vietnamese people (often cited around 40%) have the surname Nguyễn, it doesn't identify anyone.
- **Not everyone has two names.** Some people have one name (mononyms, common in Indonesia), some have several family names (Spanish and Portuguese), some have patronymics (Icelandic, Russian middle names).
- **Characters**: names contain spaces, hyphens, apostrophes (O'Brien), accents (Zoë, José), non-Latin scripts (李小龙, محمد), and can be very short (Ng, Li) or very long.
- **Design**: prefer a single **"Full name"** field plus an optional **"What should we call you?"** field. If you need given/family separately (e.g. for legal or payment reasons), label them "Given name" / "Family name", not "First" / "Last". Don't force capitalization or "fix" names.

### 5.5 Addresses and phone numbers

- **Address formats vary hugely**: order of fields (Japan goes from postal code and prefecture down to the building; the US goes from street up to state), which fields exist (states, provinces, prefectures, districts, wards), and whether postal codes exist at all (Hong Kong has none; Ireland introduced Eircode only in 2015).
- Use a **country-first** address form that adapts its fields, labels and validation to the selected country. Google's open **libaddressinput** data and CLDR provide formats.
- **Phone numbers**: store in **E.164** format (`+84901234567`), accept any input format (spaces, dashes, local format), and parse/format with **libphonenumber** (or its ports).

### 5.6 Units and other measurements

- Metric vs US customary (and UK mixed: miles on roads, stones for body weight). Format with `Intl.NumberFormat(locale, { style: 'unit', unit: 'kilometer' })`, but **conversion** is your job: store canonical units (meters, grams, Celsius), convert for display based on user preference or region.
- Paper sizes (A4 vs Letter), clothing and shoe sizes, temperature (Celsius vs Fahrenheit).

---

## 6. Time and time zones

Time is one of the most bug-prone parts of i18n, and it affects even single-country products.

### 6.1 Three kinds of time values

| Kind | Example | Store as |
|---|---|---|
| **Instant** (a point on the global timeline) | When an order was placed, log timestamps, message sent | **UTC** (`2026-03-04T09:30:00Z`, or epoch ms) |
| **Local date-time + zone** (a future event in someone's local time) | "Dentist at 9:00 in Hanoi on 15 June", recurring "every Monday 08:00 Berlin" | **Local date-time + IANA zone ID** (`2026-06-15T09:00`, `Asia/Ho_Chi_Minh`), computing the instant when needed |
| **Floating date / time** (no zone at all) | Birthday, public holiday, opening hours "9:00–17:00", contract date | **Plain date** (`2026-03-04`) or plain time, no zone |

Why not store future events as UTC? Because **time zone rules change**. Countries change DST rules or offsets with little notice, and the IANA tz database is updated several times a year. A meeting stored as a UTC instant before a rule change will show at the wrong local time afterwards; a meeting stored as "09:00 Europe/Berlin" stays at 09:00.

### 6.2 Time zone facts that break assumptions

- Use **IANA zone IDs** (`Asia/Ho_Chi_Minh`, `America/New_York`), not offsets (`+07:00`) or abbreviations (`EST`, `IST` is used for India, Ireland and Israel). An offset is a fact about one instant; a zone is a set of rules over time.
- **Not all offsets are whole hours**: India +05:30, Nepal +05:45, parts of Australia +09:30 / +10:30, Chatham Islands +12:45.
- **DST transitions** create local times that **don't exist** (clocks jump from 02:00 to 03:00) and times that **happen twice** (01:30 occurs twice when clocks go back). A "daily at 02:30" job needs a defined behavior for both.
- **A day isn't always 24 hours** (23 or 25 on DST days). Adding "1 day" and adding "24 hours" are different operations.
- **"Today" depends on the user's zone.** At 23:00 in New York it's already tomorrow in Hanoi. "Orders today", daily streaks, "posted yesterday" and daily limits must be computed in the user's (or the business's) time zone.
- **DST dates differ by country**: the US and EU switch on different weekends, and the southern hemisphere switches in the opposite season.

### 6.3 Practical rules

- Servers, databases and logs run in **UTC**. Convert to local time only at the edges (display, user input).
- APIs exchange **ISO 8601 / RFC 3339** strings with offset (`2026-03-04T16:30:00+07:00`) or `Z`, plus IANA zone IDs where local intent matters.
- Store the **user's time zone** in their profile (detect from the device: `Intl.DateTimeFormat().resolvedOptions().timeZone`, let them change it).
- **Background jobs have no browser**: emails, push notifications, PDFs and reports generated on the server must use the *recipient's* stored locale and zone ("Your appointment is at 9:00" in the right zone and format).
- **Schedule per user**: "send the digest at 08:00 local time" means grouping users by zone, not one cron job at 08:00 UTC.
- Keep **tzdata up to date** on servers, runtimes and devices.
- Use a real date-time library: `java.time`, Python `zoneinfo`, Noda Time (.NET), Luxon / date-fns-tz / Day.js plugins (JS), or the new JavaScript **Temporal** API, which is arriving in browsers (polyfills available) and finally models instants, plain dates and zoned date-times as separate types.
- When showing times to people in different zones (meetings, live events, deadlines), **show the zone** ("14:00 GMT+7") or show both zones.

---

## 7. Layout and visual design

### 7.1 Text expansion and contraction

Translations are often **longer** than English, sometimes much longer for short strings:

| English length | Typical expansion (IBM guideline, roughly) |
|---|---|
| Up to 10 characters | 100–200% (a 5-letter button can double or triple) |
| 11–20 | 80–100% |
| 21–30 | 60–80% |
| 31–50 | 40–60% |
| 51–70 | 30–40% |
| Over 70 | ~30% |

- German, Finnish, Russian and French are often 20–35% longer than English; German compound words are long and hard to break ("Kontoeinstellungen", "Datenschutzerklärung").
- Chinese, Japanese and Korean are usually **shorter** in characters but may need a **larger font size** and more line height to stay readable.

Design rules:
- **No fixed widths or heights on text containers**: buttons, tabs, labels, table headers, badges, menu items. Let them grow, wrap, or truncate *gracefully* (with the full text available, e.g. in a tooltip).
- Leave room in layouts: avoid designs that only work with exactly the English copy.
- Prefer layouts that stack vertically on overflow (labels above fields, not beside them).
- **Test with pseudo-localization** ([§13.1](#131-pseudo-localization)) during design and development, not after translation.

### 7.2 Fonts and line height

- **Font coverage**: a beautiful Latin brand font may have no Vietnamese diacritics, no Cyrillic, no Arabic. Missing glyphs fall back to a different font mid-word (ugly) or render as boxes ("tofu"). Check coverage for every target language (Google Fonts lists subsets like `vietnamese`, `cyrillic-ext`, `greek`).
- **CJK and other large fonts**: rely on high-quality system fonts, or subset web fonts with `unicode-range` so users only download what they need.
- **Line height**: Vietnamese stacked marks (`ặ`, `ẫ`), Thai, Devanagari and Burmese need more vertical space. `line-height: 1` and fixed-height boxes with `overflow: hidden` clip them. Use ~1.4–1.6 for body text.
- **Don't fake bold or italic** for scripts that don't use them; italic is rarely used in CJK or Arabic typography.
- Set the **`lang` attribute** correctly: it controls font selection (the same Han character has different preferred glyph shapes in Japanese, Simplified and Traditional Chinese), hyphenation, line breaking, quotation marks, and screen reader pronunciation.

### 7.3 Right-to-left (RTL) languages

Arabic, Hebrew, Persian and Urdu are written right to left. Hundreds of millions of people use RTL scripts daily.

**Set direction in markup**, not just CSS:

```html
<html lang="ar" dir="rtl">
```

**Mirror the layout**: the whole interface flips horizontally, because reading starts on the right.

| Mirror | Don't mirror |
|---|---|
| Overall layout: navigation, sidebars, text alignment, form label positions | **Media playback controls** (play, fast-forward represent tape direction) |
| Back / forward arrows, "next" chevrons, breadcrumbs separators | **Clocks** and circular "refresh" icons that represent clockwise motion |
| Progress bars, sliders, carousels (start on the right) | Logos, brand marks, photos |
| Icons with a direction (send, reply, undo/redo, list bullets, text alignment icons) | Numbers and phone numbers (always read left to right) |
| Order of buttons in a row, table columns | Checkmarks, universal symbols; most charts' numeric content |

**Use CSS logical properties**, which automatically follow the text direction:

| Physical (breaks in RTL) | Logical (works in both) |
|---|---|
| `margin-left`, `padding-right` | `margin-inline-start`, `padding-inline-end` |
| `left: 0`, `right: 0` | `inset-inline-start: 0`, `inset-inline-end: 0` |
| `text-align: left` | `text-align: start` |
| `border-left` | `border-inline-start` |
| `float: left` | `float: inline-start` |
| `width` / `height` | `inline-size` / `block-size` (also adapts to vertical text) |

Flexbox and grid follow `direction` automatically. Icons that need mirroring can use `:dir(rtl)` or `[dir=rtl] .icon { transform: scaleX(-1); }`.

**Bidirectional (bidi) text**: RTL text often contains LTR pieces (brand names, URLs, code, numbers, user names). The Unicode Bidirectional Algorithm handles most cases, but **neutral characters** (punctuation, spaces, symbols like `+`, `-`, `(`, `)`) at the boundaries can be placed on the wrong side. Classic symptom: "C++" displayed as "++C" inside Arabic text, or a parenthesis jumping to the other end.

- Wrap **user-generated or variable content** in `<bdi>` (bidi isolate), or use `dir="auto"` on elements containing user input.
- In plain-text contexts (notifications, logs, `title` attributes), use Unicode isolate characters (FSI U+2068 … PDI U+2069). Most i18n libraries can add them around placeholders.
- Input fields for user content: `dir="auto"` so typing Arabic in an English UI (or vice versa) aligns correctly.

### 7.4 Images, icons, colors and symbols

- **No text inside images.** It can't be translated, resized or read by screen readers. Use real text over images.
- **Culturally specific imagery**: hand gestures (thumbs up, OK sign) are offensive in some regions; mailbox, piggy bank, owl, checkmark and dollar-sign icons don't mean the same thing everywhere (in some countries a check mark ✓ can mean "wrong" in school contexts, and ○ means "correct" in Japan).
- **Color meanings vary**: red means danger/loss in Western finance but luck and prosperity in China, and rising stock prices are shown in **red** (and falling in green) in mainland China, Taiwan, Japan and Korea, the opposite of US conventions. White is associated with mourning in parts of East Asia.
- **Maps and borders** are politically sensitive and in some countries legally regulated. Region and country names in drop-downs can be contentious; use CLDR names and follow local requirements.
- **Photos of people**: diverse and appropriate for the market (clothing, settings).

---

## 8. Input: keyboards, IMEs and validation

### 8.1 Input Method Editors (IMEs)

Users of Chinese, Japanese and Korean (and some others) type through an **IME**: they type phonetic keys, see candidate characters in a temporary "composition", then confirm with **Enter** or Space. Vietnamese is often typed with Telex/VNI input methods that combine keystrokes into accented letters.

Common bugs:
- **Enter submits or sends while the user is still composing**: a Japanese user pressing Enter to confirm a conversion sends a half-written chat message. Check `event.isComposing` (and `keyCode === 229` for older browsers).
- Search-as-you-type firing requests on every intermediate composition state.
- Character counters and validation running on uncommitted composition text.
- Custom input components (rich text editors, canvas-based inputs) that don't support composition at all.

```js
input.addEventListener('keydown', e => {
  if (e.key === 'Enter' && !e.isComposing && e.keyCode !== 229) sendMessage();
});
```

### 8.2 Validation that respects real people

- **Don't validate names with `[A-Za-z]`.** It rejects most of the world: "Nguyễn", "José", "O'Brien", "李", "محمد". If you must restrict anything, block only control characters and enforce a generous length limit in graphemes.
- Be careful with `\p{C}` (all "other" characters): it includes zero-width joiners and non-joiners, which are needed in emoji and in scripts like Persian and Hindi. Block `\p{Cc}` (control characters) instead.
- **Emails**: internationalized domain names (`.vn`, `пример.рф`) and Unicode local parts exist; overly strict regexes reject valid addresses. Validate loosely, confirm by sending an email.
- **Postal codes and phone numbers**: validate per country (libphonenumber, libaddressinput), or not at all.
- **Accept native digits and full-width characters** in numeric inputs (Japanese users often have full-width `１２３` on), and normalize (NFKC) before parsing.
- Use `inputmode="numeric"` / `type="tel"` and `autocomplete` attributes to get the right keyboard on mobile.

---

## 9. Choosing and detecting the locale

### 9.1 Detection order

1. **Explicit user choice** (saved in their profile or a cookie) always wins.
2. **The URL** (`/vi/`, `vi.example.com`, `example.vn`), if the user arrived through a localized link.
3. **Browser / OS preferences**: the `Accept-Language` header on the server, `navigator.languages` on the client. These list the user's preferred languages in order with weights (`vi-VN,vi;q=0.9,en-US;q=0.8,en;q=0.7`).
4. **Default** locale.

**Don't choose the language from the IP address.** Geolocation is fine for suggesting a region, currency or store ("Looks like you're in Vietnam. Go to the Vietnam store?"), but language should come from the user's language preferences.

### 9.2 Fallback chains

Translations are never 100% complete. Define fallback order: `pt-BR` → `pt` → default (`en`). Missing strings should fall back gracefully (and be reported), never show raw keys like `checkout.pay_button` to users.

Fallbacks between languages can also be wrong: falling back from Simplified to Traditional Chinese, or from Serbian Latin to Serbian Cyrillic, can be worse than English for some users. Configure fallbacks deliberately.

### 9.3 URLs and SEO

| Strategy | Example | Notes |
|---|---|---|
| **Subdirectory** | `example.com/vi/` | Simplest to run; shares domain authority. Most common choice. |
| Subdomain | `vi.example.com` | Can be hosted separately |
| Country-code domain | `example.vn` | Strong local signal; separate domains to manage; may require local presence |
| Query parameter | `example.com?lang=vi` | Not recommended for SEO |

- Add **`hreflang`** annotations linking each language version to the others (plus `x-default`), so search engines show the right version.
- Each language version should be crawlable at its **own URL**, not swapped client-side based on the browser.

### 9.4 The language switcher

- Show each language **in its own language**: "Tiếng Việt", "Deutsch", "日本語", "العربية", not "Vietnamese", "German"… (a user who doesn't read the current language can't find their own).
- **Don't use flags for languages.** Flags represent countries: which flag is "English", "Spanish" or "Arabic"? Use flags only for country/store selection.
- Make it easy to find (header or footer, with a globe icon or "Language" label in the current language) and keep the user on the same page after switching.

---

## 10. Beyond translation: culture, law, payments

Localization is more than strings. Entering a new market usually requires product and business changes:

| Area | Examples |
|---|---|
| **Payment methods** | Cards dominate in some markets but not others: **Pix** and **Boleto** (Brazil), **iDEAL** (Netherlands), **UPI** (India), **Alipay / WeChat Pay** (China), **konbini** convenience-store payments (Japan), **MoMo / ZaloPay / VNPay** (Vietnam), SEPA Direct Debit (EU), cash on delivery in many markets |
| **Taxes and invoices** | VAT/GST rules, tax-inclusive price display, e-invoicing mandates, local invoice numbering formats |
| **Legal and privacy** | GDPR (EU), data localization / residency requirements in some countries, local consumer protection, cookie consent rules, age requirements, local-language terms of service |
| **Content and regulation** | Content restrictions, licensing (e.g. an ICP license to host websites in mainland China), local app store rules |
| **Identity** | National ID formats, local login methods (e.g. Zalo in Vietnam, LINE in Japan/Thailand, Kakao in Korea), phone-number-first sign-up in many Asian markets |
| **Communication channels** | Email vs SMS vs WhatsApp/Zalo/LINE/WeChat for notifications and support |
| **Calendar and seasons** | Local holidays (Tết, Diwali, Ramadan, Golden Week) affect promotions, delivery estimates, support hours and traffic peaks; southern-hemisphere seasons are reversed |
| **Network and devices** | Lower-end Android devices and expensive mobile data in some markets: page weight and responsiveness matter more |
| **Support** | Local-language support, local business hours, local phone numbers |

---

## 11. The localization workflow

### 11.1 Continuous localization

```
Developer adds/changes a string key in code
        │  (CI extracts new/changed strings automatically)
        ▼
Translation Management System (TMS): Crowdin, Lokalise, Phrase, Transifex, Smartling…
        │  translation memory + glossary + screenshots/context
        │  machine translation draft → human translator/reviewer
        ▼
Translations synced back to the repo (PR) or delivered over the air (OTA)
        │
        ▼
CI checks: missing keys, placeholder mismatches, ICU syntax errors, length limits
        │
        ▼
Release (missing strings fall back to the default language and get reported)
```

Key pieces:
- **Translation memory (TM)**: reuses previous translations of identical or similar strings (consistency, lower cost).
- **Glossary / termbase**: how product terms are translated (or deliberately not translated, like brand names).
- **Style guide per language**: tone, formality, punctuation conventions.
- **Context for translators**: screenshots, descriptions, character limits, placeholder meanings. Lack of context is the #1 cause of bad translations.
- **In-context review**: reviewers see translations in the running app (staging builds, in-context editors).
- **Over-the-air (OTA) string updates** for mobile apps, so typo fixes don't need an app store release (with care for placeholder safety).

### 11.2 Machine translation and LLMs

- Neural machine translation and LLMs produce good first drafts for many language pairs and dramatically cut cost and time.
- **Human review still matters** for UI strings (short, ambiguous, context-dependent), marketing, legal, medical and safety content, and for languages where models are weaker.
- A common pipeline: machine translation → post-editing by a human (MTPE) → in-context review for key flows.
- LLMs can help generate context descriptions, check placeholder consistency, and flag strings that are too long or inconsistent with the glossary.
- **Runtime machine translation of user-generated content** (reviews, chat messages) should be labeled as translated, with an option to see the original.

### 11.3 What engineers own

- String externalization, keys, and context metadata.
- Message format correctness (ICU/plurals), placeholder and markup safety (translations are user input to your templates: escape them).
- Build and delivery pipeline (extraction, sync, bundling per locale, lazy-loading locale data).
- Locale data size: load only the user's locale bundle and only the `Intl` polyfills needed.
- Locale-aware features: formatting, sorting, search, RTL, fonts.

---

## 12. Backend / system design for i18n

Many i18n problems are decided in the data model and APIs, not the UI.

| System decision | Why it matters for i18n |
|---|---|
| **UTF-8 end to end** (and `utf8mb4` in MySQL) | MySQL's legacy `utf8` (= `utf8mb3`) stores at most 3 bytes per character, so **emoji and some CJK characters fail or become `????`**. MySQL 8 defaults to `utf8mb4`; older schemas often don't. Check every DB, column, connection setting, queue, and file export. |
| **Normalization at the boundary** (NFC; NFKC + case folding for identifiers) | Usernames, emails, slugs and search keys compare correctly; no duplicate accounts like "José" vs "José" |
| **Locale-aware collation** for sorting and uniqueness | `ORDER BY` with ICU collations; decide whether unique constraints are case- and accent-insensitive |
| **Search per language** | Language analyzers (stemming, stop words), ASCII/accent folding, CJK and Thai tokenization (n-grams or dictionary segmenters), synonyms per locale |
| **APIs exchange machine values, not formatted strings** | ISO 8601 timestamps, ISO 4217 currency codes, amounts in minor units or decimal strings, ISO 3166 country codes, BCP 47 locale tags. The client (or a server-side formatting layer) formats for the user. |
| **Errors as codes + parameters, not English sentences** | `{ "code": "INSUFFICIENT_FUNDS", "params": { "available": 1250, "currency": "USD" } }` so each client can show a localized message. If the server returns human messages, it must localize them using `Accept-Language` or the user's profile. |
| **User profile fields**: UI language, format locale, time zone, currency, measurement preference | Every channel (web, mobile, email, push, SMS, PDF, support tools) uses the same preferences |
| **Server-generated content uses the recipient's locale** | Emails, push notifications, invoices, PDFs and exports are generated in background jobs without a browser. The job needs the recipient's stored locale and time zone, not the server's, and not the locale of whoever triggered the job. |
| **Translatable content model** | For user-facing data (product names, descriptions, categories, CMS pages): a translations table (`product_translations(product_id, locale, name, description)`) or per-locale JSON fields, with fallback rules and a "translation status" |
| **Caching keyed by locale** | CDN and application caches must vary by locale (URL per locale, or `Vary: Accept-Language` with care, since it fragments caches). A cached Vietnamese page served to a German user is a classic bug. |
| **Time handling** | UTC storage for instants, local + zone for future events, plain dates for birthdays; per-user scheduling; tzdata updates ([§6](#6-time-and-time-zones)) |
| **Money and pricing** | Integer minor units / decimals with currency code; price lists per currency (not just conversion); tax-inclusive/exclusive rules per market; rounding rules |
| **Per-market configuration** | Feature flags, payment providers, legal texts, shipping rules, supported languages per country, configurable without code changes |
| **Data residency** | Some markets require personal data to be stored in-country, which affects database and region architecture |
| **Validation libraries shared by client and server** | Unicode-aware name/email/phone/postal validation that agrees on both sides |
| **Slugs and URLs** | Allow Unicode slugs (`/vi/san-pham/áo-thun`) or transliterate consistently; percent-encode correctly; IDN domains |
| **Logs and analytics** | Store the locale and zone with events so you can analyze by market; never log only localized text (search logs, error messages) without codes |

---

## 13. Testing i18n

### 13.1 Pseudo-localization

Automatically transform every English string into a fake "translation" that keeps it readable for developers but exposes problems:

```
"Save changes"  →  "[Šåvé çĥåñĝéš ~~~~~]"
```

- **Accented characters** reveal encoding and font problems.
- **Expansion padding** (~30–40%) reveals truncation, overflow and fixed-width layouts.
- **Brackets** at both ends reveal clipping and **concatenation**: a sentence built from pieces shows several bracketed fragments, and any text *without* brackets is **hardcoded**.
- Add an **RTL pseudo-locale** (e.g. wrapping strings in right-to-left marks, or rendering a mirrored pseudo language) to test RTL layouts without knowing Arabic or Hebrew.

Run the app in a pseudo-locale during development and in screenshot tests. It's the cheapest i18n test there is.

### 13.2 Other tests

| Test | How |
|---|---|
| **Hardcoded strings** | Lint rules (e.g. `eslint-plugin-i18next`, FormatJS ESLint rules, Android Lint "HardcodedText") |
| **Missing / unused keys, placeholder mismatches, ICU syntax** | CI checks in the TMS sync or with i18n tooling |
| **Formatting across locales** | Unit tests with several locales (en-US, de-DE, ar-EG, ja-JP, hi-IN) for numbers, currency, dates |
| **Time zones** | Run tests with different `TZ` environment values; test DST transition days, non-hour offsets (Asia/Kathmandu), dates near midnight, leap years (29 Feb) |
| **Turkish locale** | Run the server and tests in `tr-TR` to catch locale-sensitive case mapping of identifiers |
| **Unicode data** | Test fixtures with emoji (including ZWJ sequences and flags), combining marks (NFD text), CJK, Arabic, very long and very short names, apostrophes, mixed scripts |
| **Database round-trip** | Insert and read back 4-byte characters (`😀`, `𠜎`) through every storage layer |
| **RTL** | Screenshot tests in Arabic/Hebrew (or RTL pseudo-locale); check mirroring, logical properties, bidi isolation |
| **Visual regression per locale** | Screenshot key screens in the longest language (often German or Finnish), a CJK language and an RTL language |
| **Linguistic QA** | Native speakers review the running app, especially key flows (signup, checkout, errors) |
| **IME** | Manually test text inputs with a Japanese or Chinese IME and a Vietnamese input method |

---

## 14. Approaches by product type

| Product type | i18n priorities |
|---|---|
| **E-commerce / marketplaces** | Currencies and local price points, tax display, local payment methods, address formats, shipping rules, translated catalog content (translation tables in the data model), SEO per locale (`hreflang`) |
| **SaaS / B2B** | Admin UIs in key languages; per-tenant default locale and time zone with per-user overrides; localized emails, invoices and exports; time zone-correct reports and scheduling; data residency |
| **Social / messaging / UGC** | Unicode everywhere (emoji!), bidi-safe rendering of user content (`dir="auto"`, `<bdi>`), language-aware search and moderation, optional machine translation of posts, IME-safe inputs |
| **Calendar / scheduling / booking** | Time zones are the product: local + zone storage, DST handling, recurring events, showing times across zones, first day of week, local holidays |
| **Fintech / banking** | Money correctness (minor units, rounding, currency codes), local regulations and disclosures, number parsing, local payment rails, legally reviewed translations |
| **Mobile apps** | Platform string resources (String Catalogs, `strings.xml`), OTA string updates, per-app language settings (iOS and Android support choosing an app language separately from the OS), app store listings per locale |
| **Games** | Large volumes of text and voice-over, subtitles, font coverage, culturalization (content that's acceptable in one market but not another), ratings boards per region |
| **Developer tools / APIs** | Often English-only UI, but must handle Unicode data correctly; API docs may be localized for big markets (Japan, China, Brazil); error codes rather than English-only messages |
| **Government / public services** | Legally required languages (e.g. official and minority languages), plain language, accessibility, and often RTL or multiple scripts |

---

## 15. Trade-offs

| Tension | Notes |
|---|---|
| **Upfront cost vs retrofit cost** | Externalizing strings and using `Intl` from day one costs little; retrofitting a large codebase is a multi-month project. Even if you launch in one language, do the i18n groundwork. |
| **Number of languages vs quality** | Each language adds translation, QA, support and legal costs forever. Fewer, well-localized languages often beat many poorly localized ones. Prioritize by market size and strategy. |
| **Machine vs human translation** | MT/LLMs are fast and cheap but can be wrong in context-sensitive UI text. Use humans for key flows, legal and marketing; MT for long-tail content with review. |
| **Speed of release vs complete translations** | Waiting for all translations slows releases; shipping with fallbacks shows mixed languages. Common compromise: ship with fallback for low-traffic strings, block release on key flows. |
| **Consistency vs local adaptation** | One global design is cheaper; local adaptation (colors, imagery, payment flows, features) converts better. Decide per market what's allowed to differ. |
| **Cache efficiency vs personalization** | Per-locale (and per-currency, per-region) pages fragment CDN caches. Use locale in the URL, limit dimensions, and personalize small parts client-side. |
| **Bundle size vs coverage** | Shipping all locales and fonts makes apps heavy; lazy-load locale data and subset fonts. |
| **Strict validation vs inclusiveness** | Strict rules catch some garbage input but reject real names, addresses and phone numbers. Prefer permissive validation plus verification. |
| **Accent-insensitive search vs precision** | Folding helps users find things but can merge different words (in Vietnamese, "ma", "má", "mà", "mã" are different words). Rank exact matches higher than folded matches. |

---

## 16. Anti-patterns

| Anti-pattern | Why it's harmful | Fix |
|---|---|---|
| **Hardcoded user-facing strings** | Every new language becomes a code change | Externalize to resource files; lint for hardcoded text |
| **String concatenation for sentences** | Wrong word order in other languages | Whole messages with named placeholders |
| **`n === 1 ? "item" : "items"`, or `"item(s)"`** | Wrong in most languages (French 0, Polish 22, Arabic 2…) | ICU plurals / `Intl.PluralRules` |
| **Reusing one string in different contexts** | Translators can't pick the right word | One key per context, with descriptions |
| **Hand-written date and number formats** (`MM/DD/YYYY`, `toFixed(2)`, `"$" + x`) | Ambiguous dates; wrong separators; wrong decimals for JPY/VND/KWD | `Intl.DateTimeFormat`, `Intl.NumberFormat` with currency |
| **Floating-point money** | Rounding errors | Integer minor units or decimal types + ISO 4217 code |
| **Storing local times without zones / server-local timestamps** | Off-by-hours bugs, DST bugs, wrong "today" | UTC for instants; local + IANA zone for future events |
| **Using offsets or abbreviations as zones** (`+07:00`, `EST`) | Wrong after DST/rule changes; abbreviations are ambiguous | IANA zone IDs |
| **Language from IP address** | Travelers, expats, VPN users get the wrong language | `Accept-Language` + explicit user choice |
| **Flags for languages** | Countries ≠ languages | Language names in their own language |
| **Fixed-width buttons, tabs and labels** | Truncation and overflow in longer languages | Flexible layouts; pseudo-localization tests |
| **Physical CSS (`margin-left`, `text-align: left`)** | RTL layouts break | Logical properties; `dir` attribute |
| **Text in images** | Untranslatable, inaccessible | Real text over images |
| **`[A-Za-z]` name validation, mandatory "Last name"** | Rejects real people | Permissive Unicode validation; full name field |
| **`.length` / `slice()` for user-visible limits and truncation** | Wrong counts; broken emoji and accents | Grapheme segmentation (`Intl.Segmenter`) |
| **`array.sort()` / binary collation for display lists** | Wrong order in every language | `Intl.Collator`, ICU collations |
| **Comparing un-normalized strings** | Duplicate usernames; failed matches | NFC (or NFKC + case fold for identifiers) at input |
| **MySQL `utf8` (utf8mb3)** | Emoji and some characters lost or rejected | `utf8mb4` everywhere, including the connection |
| **Locale-sensitive lower-casing of identifiers** | The Turkish "i" bug | Locale-independent case mapping for identifiers |
| **Enter-to-send without checking IME composition** | Messages sent mid-word by CJK users | Check `isComposing` |
| **Server returns English error messages only** | Clients can't localize errors | Error codes + parameters |
| **Emails and PDFs rendered in the server's locale/zone** | Wrong language and wrong times for recipients | Use the recipient's stored preferences in jobs |
| **Caches not keyed by locale** | Users get other languages' pages | Locale in the URL / cache key |
| **Raw keys shown when translations are missing** | `checkout.pay_button` on screen | Fallback chains + missing-string reporting |
| **Translating at the last minute, without context** | Bad translations, delayed launches | Continuous localization, screenshots, glossary |

---

## 17. Checklist

**Foundations**
- [ ] UTF-8 everywhere: source, HTML, HTTP, DB (`utf8mb4`), connections, files, queues, exports
- [ ] Every user-facing string externalized (including emails, push, PDFs, alt text, `aria-label`s, titles)
- [ ] ICU MessageFormat (or platform equivalent) for plurals, select and placeholders; no concatenation
- [ ] Translator context: descriptions, screenshots, max lengths
- [ ] Locale, language, currency and time zone stored as separate user preferences
- [ ] Locale detection: user choice → URL → `Accept-Language` → default; fallback chains defined

**Formatting**
- [ ] Numbers, currency, percent, units, dates, times, relative times and lists formatted with `Intl` / ICU
- [ ] Locale-aware parsing of numeric and date input
- [ ] Money as integer minor units / decimal + ISO 4217 code; currency decimals respected
- [ ] Names: full-name field, no Latin-only validation, family-name-first cultures respected
- [ ] Addresses adapt to country; phones stored in E.164 via libphonenumber

**Time**
- [ ] Instants stored in UTC; future local events as local time + IANA zone; plain dates for birthdays
- [ ] User time zone stored; "today" and schedules computed per user/business zone
- [ ] DST gaps/overlaps handled; tzdata kept up to date

**Text handling**
- [ ] Input normalized (NFC; NFKC + case fold for identifiers)
- [ ] Grapheme-aware length limits and truncation
- [ ] Locale-aware sorting (`Intl.Collator`, ICU collations) and accent-insensitive search where expected
- [ ] Locale-independent case mapping for identifiers (passes the Turkey test)
- [ ] IME-safe inputs (no submit on Enter during composition)

**Layout**
- [ ] No fixed widths/heights on text; layouts survive +40% expansion
- [ ] `lang` and `dir` set; CSS logical properties; directional icons mirrored, media/clock icons not
- [ ] `<bdi>` / `dir="auto"` around user-generated and variable content
- [ ] Fonts cover all target scripts; generous line height
- [ ] No text in images; culturally appropriate icons, colors and imagery

**System**
- [ ] APIs exchange machine values; errors as codes + params
- [ ] Background jobs (email, push, PDF, reports) use the recipient's locale and zone
- [ ] Translatable data model for catalog/CMS content, with fallback
- [ ] Caches keyed by locale; `hreflang` and per-locale URLs
- [ ] Per-market config: payments, taxes, legal texts, features

**Process & testing**
- [ ] Continuous localization pipeline (extraction, TMS, sync, CI checks)
- [ ] Pseudo-localization (incl. RTL) in development and screenshot tests
- [ ] Lint for hardcoded strings; CI for missing keys and placeholder mismatches
- [ ] Tests across locales and time zones (incl. DST days, `tr-TR`, `ar`, `ja`, `hi-IN`)
- [ ] Unicode fixtures (emoji ZWJ sequences, flags, NFD text, CJK, RTL) round-tripped through storage
- [ ] Native-speaker review of key flows

---

## 18. Exercises

1. **Audit a codebase.** Search a project for `toFixed(`, `toLocaleString()` without arguments, `.sort()` on display strings, `+ " "` near user-facing text, hardcoded date patterns (`MM/DD`, `dd/MM`), `[A-Za-z]` in validation, and `margin-left`. Count how many places would break for Vietnamese, German, Arabic and Japanese users.
2. **Plural table.** Using `Intl.PluralRules`, print the plural category for 0, 1, 2, 3, 5, 11, 21, 22, 100, 101 and 1.5 in English, French, Polish, Russian, Arabic and Vietnamese. Write the ICU message "You have {count} unread messages" for two of them.
3. **Pseudo-localize.** Write a pseudo-localization function (accents + ~40% expansion + brackets) and run your app in it. List every truncation, overflow and hardcoded string you find.
4. **RTL conversion.** Take one page of a project and make it work in `dir="rtl"`: replace physical CSS with logical properties, mirror the right icons, and isolate user-generated text. What broke first?
5. **Time zone puzzle.** A user in `America/New_York` creates a weekly meeting every Monday at 09:00 with a colleague in `Asia/Ho_Chi_Minh`. Design the storage model. What time is it for the colleague in January vs July? What happens if New York changes its DST rules next year?
6. **Unicode round trip.** Create a MySQL table with `utf8` (utf8mb3) and another with `utf8mb4`. Insert "Nguyễn 😀 𠜎". What happens in each? Then compare `'José'` stored in NFC and NFD with `=` and with a unique index.
7. **Name form redesign.** Redesign a sign-up form with "First name / Last name" so it works for Nguyễn Văn An, Björk, María José García López and 李小龙, while still letting the email say a natural greeting.
8. **System-design tie-in.** Design the i18n architecture for an e-commerce platform launching in Vietnam, Japan and Germany: locale detection and URLs, product translation data model, pricing per currency with tax rules, search per language (including accent-insensitive Vietnamese), localized order emails sent by background workers, and CDN caching.

---

## 19. Further reading

**Standards and references**
- **W3C Internationalization** (w3.org/International): articles on language tags, `lang`/`dir`, bidi, personal names, character encodings; the *Internationalization Best Practices* and *i18n checker*.
- **Unicode CLDR** (cldr.unicode.org): plural rules tables, date/number patterns, collation charts.
- **ICU** user guide (unicode-org.github.io/icu): MessageFormat, collation, formatting, BreakIterator.
- Unicode Standard Annexes: **UAX #9** (Bidirectional Algorithm), **UAX #15** (Normalization), **UAX #29** (Text Segmentation: graphemes, words, sentences).
- BCP 47 / RFC 5646 (language tags); ISO 8601 / RFC 3339 (dates and times); ISO 4217 (currencies); E.164 (phone numbers).
- IANA Time Zone Database (tzdata).

**Developer documentation**
- MDN: the `Intl` namespace (`DateTimeFormat`, `NumberFormat`, `PluralRules`, `Collator`, `Segmenter`, `RelativeTimeFormat`, `ListFormat`, `DisplayNames`), CSS logical properties, `<bdi>`.
- FormatJS / react-intl, i18next, Lingui, Fluent (projectfluent.org) docs.
- Android: *Localize your app*, *Support different languages and cultures*, RTL support, per-app language preferences. Apple: *Localization* guides, String Catalogs, *Right to left* in the Human Interface Guidelines.
- Material Design: *Bidirectionality* (what to mirror and what not to).
- TC39 **Temporal** proposal docs and cookbook.
- Google's libphonenumber and libaddressinput.

**Articles and books**
- Joel Spolsky, *The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets* (2003): still the best starting point.
- Patrick McKenzie, *Falsehoods Programmers Believe About Names* (2010), and the follow-up "falsehoods" lists about time, addresses and phone numbers.
- W3C, *Personal names around the world*.
- Jukka Korpela, *Unicode Explained* (O'Reilly).
- John Yunker, *Think Outside the Country* (web globalization strategy).
- Engineering blogs from Airbnb, Shopify, Netflix, Spotify and Figma on scaling localization.
- CSA Research, *Can't Read, Won't Buy* (2020).
- Related notes: [Usability](../usability/README.md) · [Accessibility](../accessibility/README.md) · [Responsiveness](../responsiveness/README.md).
