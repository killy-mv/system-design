# WordPress — Mental Model

> On every request, WordPress parses the URL into a **database query for posts**, runs it,
> renders the result as HTML through a **theme** — and at thousands of named points along
> that path it pauses and asks plugins *"anyone want to change this?"*

Everything else is detail hanging off that sentence.

---

## Origin, and why it matters

2003. Michel Valdrighi's blogging tool **b2/cafelog** was abandoned. Matt Mullenweg, 19, blogged
that they'd like to fork it; Mike Little replied in the comments *"if you're serious, I'm up for it."*
WordPress 0.7 shipped **27 May 2003**.

Two properties were *inherited, not chosen*, and both were decisive:

- **GPL license** (from b2) → anyone could build a business on top → an entire economy formed.
- **PHP + MySQL** (from b2) → matched exactly what $5/month shared hosting could run.

It won in **May 2004** when the dominant competitor, Movable Type, changed its licensing and
enraged its users. WordPress shipped an importer and caught the exodus. It didn't out-engineer
anyone; it was the free option standing there when the paid option changed the deal.

**Two constraints have dominated every design decision since: it must run on cheap shared hosting,
and it must never break a site built ten years ago.** Almost every wart is downstream of one of those.

---

## It is not a server

Between requests, WordPress is **a folder of PHP files doing nothing**. No process, no port.

```
browser → Nginx/Apache → PHP-FPM → index.php
                                     ├ load ALL of WordPress from scratch
                                     ├ connect to MySQL
                                     ├ run 20–50 queries
                                     ├ echo one big HTML string
                                     └ throw away 100% of memory
```

This is the **shared-nothing execution model**, and it explains almost everything:

- You never "restart" WordPress — edit a file, refresh, it's live.
- Anything that must persist between requests goes in the **database** (hence *transients*).
- Globals are fine, because a global lives ~100ms.
- **WP-Cron is a lie** — no daemon exists; scheduled tasks fire when a visitor loads a page.
- Two web servers pointed at the same DB are interchangeable → horizontal scaling is nearly free.
  The only stateful pieces are **the database** and **`wp-content/uploads`**.

`DB_HOST` in `wp-config.php` is one string. `localhost` or an RDS endpoint — WordPress can't tell.

---

## The data model: everything is a post

12 tables. Essentially unchanged since 2010.

```
wp_posts ──< wp_postmeta                          content + a key/value bag
    └──< wp_term_relationships >── wp_term_taxonomy ── wp_terms    classification
wp_users ──< wp_usermeta      wp_comments ──< wp_commentmeta      wp_options
```

**Insight 1:** `wp_posts` has a `post_type` column and *everything* lives there — blog posts, pages,
uploaded images, revisions, nav menu items, WooCommerce orders. A "Custom Post Type" is not a new
table, it's a new string.

**Insight 2:** `wp_postmeta` is Entity-Attribute-Value — any post can have arbitrary
`(post_id, key, value)` rows. This is *why* WordPress ate the web (no migration ever needed) and
*why* big sites die (querying by meta means self-joining 10M unindexed rows).

**Insight 3:** taxonomies are a generic many-to-many classifier. `wp_terms` = the label ("Blue");
`wp_term_taxonomy` = that label used as a *kind* of classification; `wp_term_relationships` = the join.
Categories and tags are just two built-in taxonomies.

---

## Routing = a database query

Apache/Nginx does step one: *if no real file matches, send it to `index.php`.*

Then WordPress does something unusual — it maps the URL to **a query, not a controller**.
`/blog/hello-world/` is regex-matched against rewrite rules stored in `wp_options`, becoming
`?name=hello-world`, which becomes a `SELECT`. **Every WordPress URL is secretly a query string.**
That's why "flush permalinks" fixes 404s — you regenerate that regex array.

The result populates a **global `$wp_query`**, and the template loop mutates a **global `$post`**.
Not MVC — procedural code over shared global state.

---

## Hooks: the actual architecture

No DI, no interfaces, no container. One mechanism:

```php
add_action('save_post', 'fn', 10, 2);       // "this happened, go do something"
add_filter('the_content', 'fn');            // "here's a value, hand back a modified one"
```

Mental image: **core is a script being read aloud, and plugins are hecklers with assigned seat
numbers** (priority, default 10). This is why WordPress can be extended by strangers without
forking — and why "plugin conflict" is a real bug category that barely exists in Rails or Laravel.

---

## Theme vs plugin

| | Plugin | Theme |
|---|---|---|
| Owns | behavior, data, post types | presentation |
| Survives a theme switch | yes | no |

**Rule: if switching themes shouldn't lose it, it belongs in a plugin.** Registering a `product`
post type in `functions.php` means the products vanish the day someone redesigns.
`functions.php` is best understood as *a plugin whose lifetime is tied to the theme*.

Templates resolve by a **fallback chain** — first file that exists wins:
`single-book-dune.php → single-book.php → single.php → singular.php → index.php`

---

## Where it fits now

Still ~40% of all websites. Four distinct positions:

1. **The monolith** — small business sites, publishers, WooCommerce. Most usage. Boring infrastructure.
2. **Headless** — wp-admin for editors, REST API (`/wp-json/`) or WPGraphQL out, Next/Astro renders.
3. **Managed platforms** — WP Engine, Kinsta, VIP. Real infra, caching, CI. Not $5 hosting anymore.
4. **Site builder** — Gutenberg blocks (2018) and Full Site Editing (2022), aimed at Squarespace/Wix.

You don't pick WordPress in 2026 because it's well-engineered. You pick it because the plugin
already exists, the editors already know it, or the site is already on it.

*Risk note:* the 2024 Automattic/WP Engine conflict made clear that wordpress.org — the plugin
directory and update mechanism every site depends on — is controlled by a private individual,
not a neutral foundation. That belongs in the same column as "what's the license."

---

## False models to unlearn

1. **It's not MVC.** No controllers, no models, no router. URL → global query → template file.
2. **There are no migrations.** Schema changes happen by shoving things into meta tables.
3. **There's no DI container.** Extension = global hooks + global state.
4. **`wp_posts` isn't a posts table.** It's a generic object table.
5. **Backwards compatibility outranks elegance, performance, and security ergonomics.**
   Understand that and the codebase stops looking incompetent and starts looking like a deliberate,
   extreme trade-off.

See also: [cms.md](./cms.md) — the general category this is one answer to.
