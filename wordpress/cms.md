# CMS — Mental Model

**CMS (Content Management System) is a program that lets a non-programmer edit a website.**

That's the whole definition, and the important word is *non-programmer*. Every design decision
in every CMS is downstream of it. If only developers ever touched the content, you wouldn't need
a CMS — you'd use a text editor and git.

---

## The problem it solves

Without a CMS, changing "Tuesday special: $12" means: open an editor → find the file → edit →
commit → push → wait for a deploy. That's five skills and repo access.

A CMS replaces that chain with: **log in → type → save.**

So a CMS is really three things stapled together:

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  A DATABASE  │ + │  AN ADMIN UI │ + │  A RENDERER  │
│  stores      │   │  humans put  │   │  turns rows  │
│  content     │   │  things in   │   │  into pages  │
└──────────────┘   └──────────────┘   └──────────────┘
```

Different CMSs keep all three, or drop the renderer, or swap the database for git files.
That's the entire taxonomy.

---

## Three inputs, not one

People say "I provide the content and it does the rest." Actually you provide three things:

| Input | Controls | Who supplies it |
|---|---|---|
| **Content** | what the site says | the editor, daily |
| **Structure** | what kinds of things exist (post, product, event) | the developer, once |
| **Presentation** | what it looks like | the theme / front-end |

Most CMS confusion comes from mixing these up — putting structure decisions in the presentation
layer, or letting editors change presentation when they only needed to change content.

---

## The key axis: *when* does the HTML get made?

This is the single most useful distinction in the whole space.

| | **Request time** | **Build time** |
|---|---|---|
| HTML is made | on every visit | once, in CI |
| Examples | WordPress, Drupal, Rails | Astro, Hugo, Next SSG |
| Needs at runtime | PHP/Node + a database, 24/7 | nothing — static files on a CDN |
| Content goes live | instantly | after a rebuild (1–3 min) |
| Speed | slow by default, needs caching | fast for free |
| Security surface | large — patch forever | nearly zero |

Everything else follows from this one choice. Request-time rendering buys **immediacy and
non-technical editing**; build-time rendering buys **speed, cheapness, and safety**.

Modern systems blur it: static pages with dynamic holes (Astro server islands, ISR), or
request-time CMSs sitting behind a full-page cache. But the axis is still how to reason about them.

---

## The CMS spectrum

```
  COUPLED ─────────────── HEADLESS ─────────────── GIT-BASED ────────── NONE
  (monolith)              (content API)            (files in repo)      (dev edits)

  WordPress               Sanity, Storyblok        Sveltia, Keystatic   markdown
  Drupal                  Contentful, Payload      TinaCMS              in the repo

  DB + admin + renderer   DB + admin, no renderer  git + admin UI       no admin
  one box, one deploy     you build the front end  commits trigger CI   full control
  easiest for editors     best editing UX, $/mo    free, no vendor      zero for editors
```

Moving right: more developer control, less editor autonomy, fewer moving parts at runtime.
Moving left: less code to write, more surface to maintain.

---

## The trade nobody warns you about

**Schema rigidity vs. editor autonomy.**

- **Rigid schema** (typed fields, a fixed content model) → design stays consistent forever,
  and every unanticipated request *("can I put a photo next to the special?")* becomes a
  phone call to you.
- **Flexible schema** (WordPress blocks, page builders) → editors can do almost anything,
  never call you, and the site slowly gets uglier.

There is no setting that gives you both. The practical move is to **negotiate the content model
up front**: find what genuinely changes (prices, hours, menu items, one hero image), make exactly
those editable, freeze everything else.

---

## How to choose

One question: **how often does the content change, and how fast must it appear?**

| Change frequency | Answer |
|---|---|
| A few times a year | No CMS. Edit it yourself and bill for it. |
| Weekly, minutes of lag is fine | Git-based CMS, or headless + webhook rebuild |
| Daily, must be instant | Don't make that part static — fetch it at request time |
| Constantly, many editors, everything | Use a real CMS. This is where WordPress wins. |

---

## The law underneath all of it

> **Complexity is conserved. You choose where to put it — build time, runtime, or someone's lap.**

- WordPress puts it at **runtime and in ops** (servers, patches, caching) → the editor's life is easy.
- Astro puts it at **build time and in composition** → the runtime is easy, the editor must be technical.
- Squarespace puts it in **a vendor's lap** → you pay rent and lose the keys.

Nobody escapes it. Picking a CMS is picking who carries the weight.

See also: [wordpress.md](./wordpress.md) — one specific, very consequential answer to all of this.
