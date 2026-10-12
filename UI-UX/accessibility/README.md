# Accessibility (a11y)

> **Accessibility** = the degree to which a product can be used by people with the **widest range of abilities** (vision, hearing, motor, cognitive, speech) to achieve their goals, in the way they actually interact with technology: screen readers, keyboards, voice, magnifiers, switches, captions.
> "Can *everyone* use it, including people who can't see the screen, can't use a mouse, can't hear the audio, or can't process a cluttered page?"

**a11y** is a *numeronym*: "a", then 11 letters, then "y" (like i18n).

Accessibility is a sub-part of [Usability](../usability/README.md): it's usability for people who are excluded by default designs. In **ISO/IEC 25010:2023** the old "Accessibility" sub-characteristic was split into **Inclusivity** (usable by people of diverse backgrounds, abilities and ages) and **User assistance** (usable by people with specific needs, including through assistive technology).

▶ **Interactive demo:** [index.html](index.html): open it in a browser to try 4 inaccessible designs side by side with their fixes. It includes a simulated screen reader, a color-blindness filter and a contrast checker.

---

## Table of contents

1. [What accessibility is (and isn't)](#1-what-accessibility-is-and-isnt)
2. [Who it's for: types of disability](#2-who-its-for-types-of-disability)
3. [Why accessibility matters](#3-why-accessibility-matters)
4. [Assistive technologies: how people actually use your UI](#4-assistive-technologies-how-people-actually-use-your-ui)
5. [WCAG: the standard](#5-wcag-the-standard)
6. [Laws and regulations](#6-laws-and-regulations)
7. [Designing accessibly (design side)](#7-designing-accessibly-design-side)
8. [Building accessibly (engineering side)](#8-building-accessibly-engineering-side)
9. [Testing accessibility](#9-testing-accessibility)
10. [Accessibility as a system / organization concern](#10-accessibility-as-a-system--organization-concern)
11. [Approaches by product type](#11-approaches-by-product-type)
12. [Trade-offs and misconceptions](#12-trade-offs-and-misconceptions)
13. [Anti-patterns](#13-anti-patterns)
14. [Checklist](#14-checklist)
15. [Exercises](#15-exercises)
16. [Further reading](#16-further-reading)

---

## 1. What accessibility is (and isn't)

### 1.1 Disability is a mismatch

The **medical model** says disability is a property of the person. The **social model** (used by the WHO and by most modern design practice) says disability happens when a person's abilities **don't match the environment they're in**. A wheelchair user isn't disabled by their legs at a ramp. They're disabled by stairs.

For software, this means **we create the disability** when we build a UI that only works with a mouse, only works with perfect vision, or only works if you read fast. That's also why it's fixable.

### 1.2 Permanent, temporary, situational

Microsoft's *Inclusive Design* toolkit popularized this framing: almost every disability has a temporary and a situational equivalent, so designing for the permanent case helps far more people.

| Ability | Permanent | Temporary | Situational |
|---|---|---|---|
| **Touch / motor** | One arm, paralysis, tremor | Broken wrist, RSI | Holding a baby, on a bumpy bus |
| **See** | Blind, low vision | Eye surgery, dilated pupils | Bright sunlight on a phone, cracked screen |
| **Hear** | Deaf, hard of hearing | Ear infection | Noisy café, sleeping baby, no headphones |
| **Speak** | Non-verbal | Laryngitis | Heavy accent, quiet library |
| **Cognition** | Learning disability, dementia | Concussion, medication | Stressed, tired, distracted, second language |

### 1.3 The curb-cut effect

Curb cuts (sloped sidewalk edges) were fought for by wheelchair users. Now they're used constantly by people with strollers, suitcases, bikes and delivery carts. The same thing happens in software:

| Built for | Used by everyone as |
|---|---|
| Captions for Deaf users | Watching videos on mute in public (a large share of social video is watched without sound) |
| Keyboard navigation for motor disabilities | Power-user shortcuts |
| High contrast for low vision | Reading outdoors in sunlight |
| Voice control | Hands-free cooking, driving |
| Plain language for cognitive disabilities | Non-native speakers, people skimming |
| Text-to-speech for blind users | Listening to articles while commuting |
| Semantic structure for screen readers | SEO, AI agents and scrapers, reader mode |

### 1.4 Accessibility vs usability vs inclusive design

| Term | Focus |
|---|---|
| **Accessibility** | Removing barriers for people with disabilities. Often measured against a standard (WCAG), and often legally required. |
| **Usability** | How effectively, efficiently and pleasantly the *intended* users complete tasks. |
| **Inclusive design** | A *process*: design with diverse people (including excluded groups) from the start, instead of fixing things later. |

Key point: **something can be WCAG-conformant and still unusable** for disabled people (every image has alt text, but all of it says "image"). Conformance is the floor, not the goal.

---

## 2. Who it's for: types of disability

The WHO estimates about **1.3 billion people (around 1 in 6)** live with a significant disability. Add ageing populations (most people get disabilities as they age) and temporary or situational cases, and it covers almost everyone at some point.

| Category | Examples | How they use tech | What typically breaks for them |
|---|---|---|---|
| **Blind** | No usable vision | Screen reader (speech or braille), keyboard or touch gestures | Images without alt text, unlabeled buttons, `div` buttons, no headings, focus lost in modals, CAPTCHAs, canvas-only UIs |
| **Low vision** | Blurry vision, tunnel vision, light sensitivity | Zoom 200–400%, screen magnifier, high contrast mode, large fonts, dark mode | Low contrast, text in images, layouts that break or hide content on zoom, fixed font sizes, content far from where the magnifier is |
| **Color vision deficiency** | Red-green (most common), blue-yellow, total (rare) | Normal tools; may use filters | Information conveyed **only by color**: red/green status, chart legends, error borders. Affects roughly 1 in 12 men and 1 in 200 women of Northern European descent. |
| **Deaf / hard of hearing** | Partial or total hearing loss | Captions, transcripts, sign language, visual alerts | Videos without captions, audio-only alerts, voice-only support lines, phone-only 2FA |
| **Motor / mobility** | Paralysis, tremor, arthritis, missing limbs, RSI, cerebral palsy | Keyboard only, switch devices, voice control, eye tracking, head pointers, mouth sticks | Mouse-only interactions, tiny targets, drag-only actions, complex gestures, timeouts, hover-only menus, no visible focus |
| **Cognitive / learning / neurodivergent** | Dyslexia, ADHD, autism, intellectual disability, memory loss, brain injury | Text-to-speech, reading aids, simplified views, extra time | Dense jargon, inconsistent navigation, time limits, distracting animation, memory-dependent tasks (remember this code), complex auth puzzles |
| **Seizure / vestibular** | Photosensitive epilepsy, vestibular disorders (motion sickness) | `prefers-reduced-motion`, avoiding triggers | Flashing content (> 3 flashes/second can trigger seizures), parallax, zooming transitions, autoplaying video |
| **Speech** | Non-verbal, stutter | Text input, AAC devices | Voice-only interfaces, phone-only support |

---

## 3. Why accessibility matters

- **It's a human right.** The UN Convention on the Rights of Persons with Disabilities (Article 9) treats access to information and communication technology as part of equal participation. Banking, healthcare, government services, education and jobs are all online now. An inaccessible site can lock someone out of their own bank account.
- **It's often the law.** See [§6](#6-laws-and-regulations). In the US, web accessibility lawsuits are filed in the thousands every year, and the European Accessibility Act has applied to many private-sector products since June 2025.
- **The web is mostly broken.** WebAIM's yearly scan of the top 1,000,000 home pages ("The WebAIM Million") finds detectable WCAG failures on **around 95%** of them, every year. The most common: low-contrast text, missing alt text, missing form labels, empty links, empty buttons and missing page language. These are all easy to fix.
- **Market size.** People with disabilities, plus their families, control a lot of spending. If your checkout doesn't work with a screen reader, those users go to a competitor and usually don't come back.
- **Ageing users.** Vision, hearing, dexterity and memory decline with age. Older adults are a large and growing share of users.
- **Better product for everyone.** The curb-cut effect ([§1.3](#13-the-curb-cut-effect)). Accessible apps are usually also clearer, more keyboard-friendly, and easier to use on mobile.
- **SEO and machine readability.** Search engines, AI agents, voice assistants and automation tools read the same semantic structure (headings, alt text, labels, landmarks) that screen readers do.
- **Engineering quality.** Accessible components are built on semantic HTML, which tends to be simpler, more robust and easier to test. Testing tools like Playwright and Testing Library recommend querying by **role and accessible name** (`getByRole('button', { name: 'Save' })`). If your UI isn't accessible, it's also harder to test.
- **Cost of change.** Like any quality attribute, it's cheap to build in and expensive to retrofit. Fixing a color palette in the design system takes a day. Fixing the same contrast issue across 400 screens after a lawsuit takes months.

---

## 4. Assistive technologies: how people actually use your UI

You can't design for something you've never seen. **Spend 30 minutes using a screen reader on your own product.** It's the single most eye-opening accessibility exercise.

### 4.1 Screen readers

Software that reads the screen aloud (or outputs to a refreshable braille display). It doesn't read pixels. It reads the **accessibility tree**, which the browser builds from your HTML, ARIA and CSS. Each node in the tree has:

- a **role** (button, link, heading level 2, checkbox, textbox, dialog…)
- an **accessible name** ("Save", "Search products")
- **states and properties** (checked, expanded, disabled, invalid, required, current page)
- optionally a **description** (hint or error text)

Example announcement for `<button aria-expanded="false">Filters</button>`: *"Filters, button, collapsed."*

| Screen reader | Platform | Notes |
|---|---|---|
| **NVDA** | Windows | Free, open source, very widely used. Use with Firefox or Chrome. |
| **JAWS** | Windows | Commercial, common in enterprise and government |
| **VoiceOver** | macOS, iOS, iPadOS | Built in. On iPhone it's the dominant mobile screen reader. |
| **TalkBack** | Android | Built in |
| **Narrator** | Windows | Built in |
| **Orca** | Linux (GNOME) | Free |

**How screen reader users navigate.** Most don't read a page top to bottom. They skim, using shortcuts to jump between:

- **Headings** (by far the most common way to find content on a page, according to WebAIM's screen reader user surveys). Press `H` in NVDA/JAWS to jump to the next heading, `1`–`6` for a level.
- **Landmarks** (`<header>`, `<nav>`, `<main>`, `<footer>`, `<aside>`, `<form>` with a name, `<section>` with a name)
- **Links list / form controls list** (VoiceOver's "rotor"): a list of every link on the page, out of context. This is why "click here" and "read more" links are useless.
- **Tab** for interactive elements, arrow keys for reading line by line
- **Swipe gestures** on mobile (swipe right = next element, double-tap = activate)

So: if your headings, landmarks, and link text are bad, the page is basically unnavigable even if every element is technically "readable".

### 4.2 Other assistive technology and settings

| Technology / setting | Who uses it | What it needs from you |
|---|---|---|
| **Keyboard only** | Motor disabilities, screen reader users, power users | Everything reachable and operable by keyboard; visible focus; logical focus order; no keyboard traps |
| **Switch access** (one or two buttons, scanning through items) | Severe motor disabilities | Same as keyboard; generous timeouts; few steps |
| **Voice control** (Voice Control on macOS/iOS, Voice Access on Android/Windows, Dragon) | Motor disabilities, RSI | The **visible label must be part of the accessible name**, so users can say "click Send". If the button shows "Send" but its `aria-label` is "Submit form", the voice command fails (WCAG 2.5.3). |
| **Screen magnifier** (ZoomText, built-in OS zoom) | Low vision | Related content close together; no important info far off to the side; tooltips and errors near their trigger |
| **Browser zoom / text size** | Low vision, older adults | Layout reflows at 400% zoom; text scales; nothing gets clipped or overlaps |
| **High contrast / forced colors** (Windows Contrast Themes) | Low vision, light sensitivity, migraines | Don't rely on background images or box-shadows to convey meaning; test with `forced-colors: active` |
| **Dark mode / inverted colors** | Light sensitivity, low vision | Support `prefers-color-scheme`; make sure contrast holds in both themes |
| **Reduced motion** | Vestibular disorders, ADHD, migraines | Respect `prefers-reduced-motion` |
| **Captions, transcripts** | Deaf / hard of hearing, non-native speakers, everyone in a noisy place | Accurate captions (not just raw auto-generated ones), transcripts for audio |
| **Reading tools / text-to-speech** | Dyslexia, cognitive disabilities, low vision | Real text (not images of text), clear structure, plain language |
| **Eye tracking, head pointers** | Severe motor disabilities | Large targets with spacing; dwell-friendly; no precision-required actions |

---

## 5. WCAG: the standard

**WCAG** (Web Content Accessibility Guidelines) is published by the W3C's Web Accessibility Initiative (WAI). It's the reference standard behind almost every accessibility law worldwide. It applies to web content but is widely used for mobile apps, desktop software and documents too (W3C publishes guidance on applying it to non-web ICT).

### 5.1 Versions

| Version | Year | Notes |
|---|---|---|
| WCAG 2.0 | 2008 | Still referenced by some laws (US Section 508, Ontario AODA). Also ISO/IEC 40500. |
| WCAG 2.1 | 2018 | Added mobile, low-vision and cognitive criteria (reflow, non-text contrast, orientation, target size AAA…). Referenced by EN 301 549 in the EU and by the US DOJ's 2024 ADA Title II rule. |
| **WCAG 2.2** | 2023 | **Current recommendation.** Added 9 criteria (focus not obscured, target size minimum, dragging alternatives, accessible authentication, redundant entry, consistent help…). Removed 4.1.1 Parsing. Backward compatible: meeting 2.2 also meets 2.1 and 2.0. |
| WCAG 3.0 | In development | A full restructure with a different scoring model. A W3C draft, not a standard yet. Don't target it for compliance today. |

**Practical target: WCAG 2.2 Level AA.** It's what most laws, procurement contracts and audits ask for (or 2.1 AA, which 2.2 AA covers).

### 5.2 The four principles: POUR

| Principle | Meaning | Examples |
|---|---|---|
| **Perceivable** | Users can perceive the information with at least one of their senses | Alt text, captions, contrast, don't rely on color alone, content works at 400% zoom |
| **Operable** | Users can operate the interface | Keyboard access, enough time, no seizure triggers, ways to navigate and find content, target size |
| **Understandable** | Users can understand the information and the UI | Readable language, predictable behavior, help with input errors |
| **Robust** | Works with current and future technologies, including assistive tech | Valid semantics, correct name/role/value, status messages exposed to AT |

Under the principles are **guidelines**, and under those are testable **success criteria (SC)**, numbered like `1.4.3`.

### 5.3 Conformance levels

| Level | Meaning |
|---|---|
| **A** | The minimum. Without these, some groups simply can't use the content. |
| **AA** | The standard target. Removes the most significant and common barriers. Includes all of A. |
| **AAA** | The highest. W3C itself says it's not recommended as a general policy for entire sites, because some content can't meet all AAA criteria. Use individual AAA criteria where they make sense (e.g. 7:1 contrast for body text). |

### 5.4 Success criteria you'll hit most often (Level A and AA)

Not the full list (WCAG 2.2 has 86 criteria). These are the ones that cause most real-world failures.

| SC | Name | Level | What it means in practice |
|---|---|---|---|
| 1.1.1 | Non-text content | A | Images, icons and controls have a text alternative. Decorative images are hidden (`alt=""`). |
| 1.2.2 / 1.2.4 | Captions (prerecorded / live) | A / AA | Videos with audio have captions |
| 1.2.5 | Audio description | AA | Important visual info in videos is described in audio |
| 1.3.1 | Info and relationships | A | Structure shown visually also exists in code: headings, lists, tables, labels tied to inputs, grouped radio buttons |
| 1.3.2 | Meaningful sequence | A | DOM order matches the reading order |
| 1.3.4 | Orientation | AA | Works in both portrait and landscape (unless essential) |
| 1.3.5 | Identify input purpose | AA | Use `autocomplete` for personal data fields (name, email, address) |
| 1.4.1 | Use of color | A | Color isn't the **only** way information is conveyed |
| 1.4.3 | Contrast (minimum) | AA | Text contrast ≥ **4.5:1**, large text ≥ **3:1** |
| 1.4.4 | Resize text | AA | Text can be zoomed to 200% without losing content or function |
| 1.4.10 | Reflow | AA | No horizontal scrolling at **320 CSS px** width (equivalent to 1280 px at 400% zoom), except for things like data tables and maps |
| 1.4.11 | Non-text contrast | AA | UI component boundaries, states and meaningful graphics ≥ **3:1** |
| 1.4.12 | Text spacing | AA | Nothing breaks when users increase line, letter and word spacing |
| 1.4.13 | Content on hover or focus | AA | Tooltips/popovers can be dismissed (Esc), hovered over, and stay until dismissed |
| 2.1.1 | Keyboard | A | **Everything** works with a keyboard |
| 2.1.2 | No keyboard trap | A | Focus can always move away from a component |
| 2.1.4 | Character key shortcuts | A | Single-letter shortcuts can be turned off or remapped (they collide with voice control and screen readers) |
| 2.2.1 | Timing adjustable | A | Users can turn off, adjust or extend time limits |
| 2.2.2 | Pause, stop, hide | A | Auto-moving, blinking or auto-updating content can be paused |
| 2.3.1 | Three flashes or below threshold | A | Nothing flashes more than 3 times per second |
| 2.4.1 | Bypass blocks | A | A way to skip repeated content (skip link, landmarks, headings) |
| 2.4.2 | Page titled | A | Each page has a descriptive `<title>` (also on SPA route changes) |
| 2.4.3 | Focus order | A | Tab order is logical and preserves meaning |
| 2.4.4 | Link purpose (in context) | A | Link text (plus its context) says where it goes. No bare "click here". |
| 2.4.6 | Headings and labels | AA | Headings and labels describe the topic or purpose |
| 2.4.7 | Focus visible | AA | The keyboard focus indicator is visible |
| 2.4.11 | Focus not obscured (minimum) *(new in 2.2)* | AA | The focused element isn't fully hidden behind sticky headers, cookie banners or chat widgets |
| 2.5.1 | Pointer gestures | A | Multi-finger or path-based gestures (pinch, swipe patterns) have a single-pointer alternative |
| 2.5.3 | Label in name | A | The accessible name contains the visible label text (for voice control) |
| 2.5.7 | Dragging movements *(new in 2.2)* | AA | Anything done by dragging can also be done with simple clicks (e.g. "Move up / Move down" buttons for a sortable list) |
| 2.5.8 | Target size (minimum) *(new in 2.2)* | AA | Targets are at least **24×24 CSS px**, or have enough spacing |
| 3.1.1 / 3.1.2 | Language of page / of parts | A / AA | `<html lang="en">`, and `lang` on passages in other languages, so the screen reader uses the right voice and pronunciation |
| 3.2.1 / 3.2.2 | On focus / On input | A | Focusing or changing a field doesn't cause unexpected context changes (e.g. a select that navigates away immediately) |
| 3.2.3 | Consistent navigation | AA | Repeated navigation stays in the same order across pages |
| 3.2.6 | Consistent help *(new in 2.2)* | A | Help mechanisms (contact, chat, FAQ) appear in the same place across pages |
| 3.3.1 | Error identification | A | Errors are identified and described in text |
| 3.3.2 | Labels or instructions | A | Inputs have labels or instructions |
| 3.3.3 | Error suggestion | AA | If you know how to fix the error, say so |
| 3.3.4 | Error prevention (legal, financial, data) | AA | Submissions are reversible, checked, or confirmed |
| 3.3.7 | Redundant entry *(new in 2.2)* | A | Don't make users re-enter information they already gave in the same process (auto-fill or let them select it) |
| 3.3.8 | Accessible authentication (minimum) *(new in 2.2)* | AA | Login doesn't require a cognitive test (memorizing, transcribing, puzzles) unless there's an alternative. **Allow paste and password managers.** Support passkeys, email links, etc. |
| 4.1.2 | Name, role, value | A | Custom components expose correct name, role and state to assistive tech |
| 4.1.3 | Status messages | AA | Status messages ("3 results", "Saved", "Item added to cart") are announced without moving focus (live regions) |

---

## 6. Laws and regulations

*Not legal advice. Laws change; check the current rules for your market.* In general: **if you sell to the public or to governments, assume you have legal accessibility obligations.**

| Region | Law / standard | Applies to | Technical standard |
|---|---|---|---|
| **International** | UN CRPD, Article 9 | Signatory states (most countries) | Principle, implemented through national laws |
| **USA** | **ADA** (Americans with Disabilities Act) | Title III: "public accommodations" (courts have applied it to many websites and apps). Title II: state and local governments. | The DOJ's 2024 Title II rule requires **WCAG 2.1 AA**. Title III has no codified standard; lawsuits and settlements typically use WCAG 2.x AA. |
| USA | **Section 508** (Rehabilitation Act) | Federal agencies and what they buy | WCAG 2.0 AA (via the 508 refresh) |
| **EU** | **European Accessibility Act** (Directive 2019/882) | Many private-sector products and services: e-commerce, banking, e-books, transport ticketing, telecom, computers and phones, OS. Applies since **28 June 2025**; micro-enterprises providing services are exempt. | Harmonized standard **EN 301 549** (currently references WCAG 2.1 AA) |
| EU | Web Accessibility Directive (2016/2102) | Public sector websites and apps | EN 301 549 |
| **UK** | Equality Act 2010; Public Sector Bodies Accessibility Regulations 2018 | All service providers; public sector websites and apps | WCAG 2.2 AA for the public sector |
| **Canada** | Accessible Canada Act (federal); AODA (Ontario) | Federally regulated organizations; Ontario organizations | WCAG 2.0/2.1 AA |
| **Japan** | Act for Eliminating Discrimination against Persons with Disabilities (reasonable accommodation mandatory for businesses since April 2024) | Public and private sector | JIS X 8341-3 (harmonized with WCAG) |
| **Australia** | Disability Discrimination Act 1992 | Public and private sector | WCAG 2.x AA expected |
| **Vietnam** | Law on Persons with Disabilities (2010) | Includes provisions on accessible information and communication technology | — |

**Procurement documents:**

- **VPAT** (Voluntary Product Accessibility Template) is a template a vendor fills in to produce an **ACR** (Accessibility Conformance Report), stating how a product meets WCAG / Section 508 / EN 301 549, criterion by criterion. B2B and government buyers often require one before purchase.
- **Accessibility statement**: a public page stating your conformance level, known issues, and how users can report problems. Required for EU public sector sites and expected under the EAA.

---

## 7. Designing accessibly (design side)

Most accessibility problems are **decided in design**, before any code is written. Contrast, target size, color usage, focus styles, content structure and interaction patterns all come from the design file.

### 7.1 Color and contrast

**Contrast ratio** compares the relative luminance of two colors, from 1:1 (same color) to 21:1 (black on white).

| Element | Minimum (AA) | Enhanced (AAA) |
|---|---|---|
| Normal text (< 24px, or < 18.66px bold) | **4.5:1** | 7:1 |
| Large text (≥ 24px, or ≥ 18.66px bold; i.e. 18pt / 14pt bold) | **3:1** | 4.5:1 |
| UI component boundaries (input borders, toggle tracks), focus indicators, icons, chart lines | **3:1** against adjacent colors | — |
| Disabled controls, logos, pure decoration | No requirement | — |

Common traps:

- **Light gray text on white.** `#999` on white is only about 2.8:1, which fails. `#767676` is about the lightest gray that passes 4.5:1 on white.
- **Placeholder text** is usually too light, and it disappears when you type. Don't use it as a label.
- **Brand colors.** Many brand blues, oranges and greens fail with white text. Define accessible variants in the design system once.
- **Text on images / gradients.** Contrast varies across the image. Add a solid or semi-opaque overlay behind the text.
- **Input borders.** Light gray input borders (e.g. `#ddd`) fail 1.4.11, so low-vision users can't find the field.
- **Dark mode.** Re-check everything. Pure white on pure black can cause halation (glow) for people with astigmatism; slightly off-white on dark gray is often more comfortable.

**Don't use color alone** (SC 1.4.1). About 1 in 12 men have a color vision deficiency, and everyone loses color on a monochrome screen or in bright sunlight.

| Bad | Good |
|---|---|
| Red/green dots for down/up | Icon + text: "✕ Down", "✓ Up" |
| Error = red border only | Red border **+** error icon **+** text message |
| Links = blue text in a paragraph, no underline | Underline links in body text (or ≥ 3:1 contrast with surrounding text **plus** a non-color cue on hover and focus) |
| Chart lines distinguished only by hue | Different line styles or markers, direct labels on the lines |
| Required fields shown in red | "(required)" text, or an asterisk explained at the top of the form |

Test by viewing the design in **grayscale** and with a color-blindness simulator (Figma plugins, Chrome DevTools → Rendering → "Emulate vision deficiencies").

### 7.2 Typography and readability

- Body text **16px or larger** on the web (and use `rem`, not `px`, for font sizes so user font-size settings work).
- Line height around **1.5** for body text; paragraphs no wider than about **80 characters** (WCAG 1.4.8 AAA suggests ≤ 80).
- Left-aligned text (for left-to-right languages). Avoid **justified** text: uneven word spacing creates "rivers" that are hard for dyslexic readers.
- Avoid ALL CAPS for long text, and avoid long italic passages.
- **Real text, not images of text** (SC 1.4.5). Images of text can't be resized, recolored, translated or read by text-to-speech.
- Support user **text spacing** overrides (SC 1.4.12) and OS-level font scaling (iOS Dynamic Type, Android font size). Don't put text in fixed-height containers.

### 7.3 Layout, zoom and reflow

- Design for **320 CSS px** wide (SC 1.4.10). If your responsive mobile layout works at 320px, it will mostly work at 400% desktop zoom too.
- Test at **200% and 400% browser zoom**. Common breakages: sticky headers that eat half the screen, modals taller than the viewport with no scroll, truncated text, overlapping elements.
- Keep **related things close together** (label next to field, error next to field, price next to "Buy"). A magnifier user sees only a small part of the screen at a time.
- Don't lock orientation (SC 1.3.4). Some people have a device mounted on a wheelchair in a fixed orientation.

### 7.4 Focus indicators

Keyboard users need to see where they are at all times, like a mouse cursor.

- **Never remove the focus outline** (`outline: none`) without providing a better one.
- Design a focus style deliberately: at least **2px** thick, with **≥ 3:1 contrast** against both the component and the background. A two-tone ring (e.g. dark outline + white inner ring) works on any background.
- Use `:focus-visible` in CSS: it shows the ring for keyboard focus but not after a mouse click, which removes the main reason designers ask to hide it.
- Make sure focused elements aren't hidden under **sticky headers, cookie banners or chat widgets** (SC 2.4.11). CSS `scroll-padding-top` helps.

### 7.5 Target size and spacing

| Guideline | Size |
|---|---|
| WCAG 2.5.8 (AA, minimum) | **24×24 CSS px**, or enough spacing that a 24px circle around each target doesn't overlap another |
| WCAG 2.5.5 (AAA, enhanced) | 44×44 CSS px |
| Apple HIG | 44×44 pt |
| Material Design | 48×48 dp |

- The target can be bigger than the visible icon (use padding).
- Keep destructive actions away from frequent ones.
- **Don't require precise or complex gestures.** Every drag, swipe, pinch or long-press needs a simple tap or click alternative (SC 2.5.1, 2.5.7).

### 7.6 Motion and animation

- **Nothing flashes more than 3 times per second** (SC 2.3.1). This is a seizure risk, not a style preference.
- Respect **`prefers-reduced-motion`**: replace large movement (parallax, zooming page transitions, sliding panels) with a fade or no animation.
- Anything that moves, blinks, scrolls or auto-updates for more than 5 seconds needs a **pause/stop** control (SC 2.2.2): carousels, tickers, background videos.
- Avoid autoplaying video and audio. If audio plays automatically for more than 3 seconds, provide a way to stop it (SC 1.4.2); it also drowns out the screen reader.

### 7.7 Forms

Forms are where accessibility and [usability](../usability/README.md) overlap the most.

- **Visible labels** on every field, placed above or next to it. Not placeholder-only.
- **Instructions before the field** ("At least 8 characters"), not only after an error.
- **Group related inputs** (radio buttons, a date split into three fields) with a group label (`<fieldset>` + `<legend>`).
- **Errors in text**, next to the field, saying how to fix it. Not only a red border.
- On submit with errors: **move focus** to an error summary at the top (with links to each field) or to the first invalid field.
- **Don't clear user input** on error.
- **Give enough time.** Warn before a session timeout and offer to extend it (SC 2.2.1). Save drafts.
- **Accessible authentication** (SC 3.3.8): allow paste into password and code fields, support password managers, offer alternatives to puzzle CAPTCHAs (passkeys, email magic links, invisible risk scoring).
- **Don't ask twice** (SC 3.3.7): "Billing address same as shipping" checkbox.
- **Confirm or allow undo** for legal, financial and data-deleting submissions (SC 3.3.4).

### 7.8 Content and cognitive accessibility

- **Plain language.** Short sentences, common words, active voice. Explain jargon and abbreviations. Aim for roughly a lower-secondary-school reading level for general audiences.
- **Clear headings** that describe the content below them, and a logical heading hierarchy (one `h1`, then `h2`s, then `h3`s; don't skip levels for styling).
- **Descriptive link and button text.** "Download the 2026 annual report (PDF, 2 MB)", not "Click here". "Delete invoice #1042", not just "Delete" when there are many.
- **Consistency**: same navigation in the same place (SC 3.2.3), same names for the same functions (SC 3.2.4), help in the same place (SC 3.2.6).
- **Predictability**: no surprise context changes, no new windows without warning.
- **Reduce memory load**: don't make users remember a code from one screen to type on another; show progress in multi-step flows; summarize before submit.
- **Support focus and attention**: minimize distractions, avoid autoplay, let users dismiss popups easily.
- **Errors are forgivable**: undo, confirmation, clear recovery paths.

W3C's *Making Content Usable for People with Cognitive and Learning Disabilities* (COGA) covers this area in much more depth than WCAG's testable criteria.

### 7.9 Images, icons and alt text

**Alt text decision guide:**

| The image is… | Do this | Example |
|---|---|---|
| **Decorative** (adds nothing, or the same info is in nearby text) | `alt=""` (empty, but present), or a CSS background | A divider swirl; an icon next to the word "Settings" |
| **Informative** (conveys simple information) | Short description of the *information*, not the pixels | `alt="Wi-Fi signal: weak"`, not `alt="three curved bars, one filled"` |
| **Functional** (inside a link or button) | Describe the **action or destination** | Logo link: `alt="Acme home"`. Magnifier icon button: "Search". |
| **Text in an image** | The alt is the text itself (better: use real text) | — |
| **Complex** (chart, diagram, map) | Short alt + a longer description nearby, or a data table | `alt="Sales by quarter, 2026. Details in the table below."` |
| **Photo in content** | Describe what matters *in this context* | On a news article vs. on a product page, the same photo needs different alt |

Rules of thumb:

- **Never omit the `alt` attribute.** With no `alt` at all, many screen readers read the **file name** ("IMG underscore 4 0 2 7 dot jpeg").
- Don't start with "Image of…" or "Picture of…": the screen reader already says "image".
- Keep it short (often under ~125 characters). Use a longer description for complex images.
- **Icon-only buttons need an accessible name** (`aria-label` or visually hidden text). Even better: show a visible text label, because icons are ambiguous for everyone.

### 7.10 Audio and video

| Content | Needs |
|---|---|
| Video with speech | **Captions** (synchronized; include speaker names and meaningful sounds like "[door slams]") |
| Video where visuals carry meaning | **Audio description** (narration of important visual information during pauses) or a descriptive transcript |
| Audio only (podcast) | **Transcript** |
| Live video / streams | Live captions (CART or good real-time speech recognition, ideally reviewed) |
| Any media player | Keyboard-operable controls with labels, no autoplay with sound |

Auto-generated captions are a starting point, not a finished product. They often get names, technical terms and accents wrong.

### 7.11 Annotate your designs for developers

Visual mockups don't show the things assistive technology depends on. Add **accessibility annotations** to design handoffs:

- Heading levels and landmark regions
- **Focus order** (numbered) and where focus goes when dialogs open/close or content changes
- Accessible names for icon buttons and images (alt text written by the designer or content writer, not invented by the developer)
- Component roles and states (e.g. "this is a tab list, not buttons")
- Keyboard behavior for custom interactions
- What's announced for status changes ("Added to cart" toast → live region)
- Reflow / zoom behavior and reduced-motion alternatives

---

## 8. Building accessibly (engineering side)

### 8.1 Semantic HTML first

The single most effective technique: **use the native HTML element for the job.** Native elements give you keyboard behavior, focus, role, name and state for free, and they work with every assistive technology.

```html
<!-- ✗ Bad: looks like a button, isn't one -->
<div class="btn" onclick="save()">Save</div>
```

What the `div` is missing compared to a `<button>`:

| Feature | `<button>` | `<div onclick>` |
|---|---|---|
| Reachable with Tab | ✓ | ✗ (needs `tabindex="0"`) |
| Activates on Enter **and** Space | ✓ | ✗ (needs keydown handler for both, with Space on keyup) |
| Announced as "button" | ✓ | ✗ (needs `role="button"`) |
| Disabled state | ✓ (`disabled`) | ✗ (needs `aria-disabled` + logic) |
| Submits forms | ✓ | ✗ |
| Works in Windows high contrast mode | ✓ | Often not |
| Voice control "click Save" | ✓ | Unreliable |

```html
<!-- ✓ Good -->
<button type="button" onclick="save()">Save</button>
```

Element cheat sheet:

| Need | Use | Not |
|---|---|---|
| Do an action | `<button>` | `<div>`, `<span>`, `<a href="#">` |
| Go somewhere | `<a href="…">` | `<button onclick="location=…">`, `<div>` |
| Page structure | `<header> <nav> <main> <aside> <footer>` | `<div id="header">` |
| Headings | `<h1>`–`<h6>` in order | `<div class="title-large">` |
| Lists | `<ul> <ol> <li>` | Stacked `<div>`s |
| Tabular data | `<table>` with `<th scope>` and `<caption>` | CSS grid of `<div>`s |
| Form field label | `<label for="id">` | Text in a `<div>` next to the input |
| Group of controls | `<fieldset>` + `<legend>` | `<div>` with a bold title |
| Modal | `<dialog>` + `showModal()` | Custom `<div>` overlay (unless you handle everything in [§8.5](#85-focus-management)) |
| Show/hide section | `<details>` + `<summary>` | Custom `div` accordion |
| On/off | `<input type="checkbox">` (style it) or `<button aria-pressed>` | `<div class="toggle on">` |

**Buttons vs links**: links *navigate* (change the URL, can open in a new tab, appear in the links list); buttons *do things* (submit, open a menu, toggle). Users and screen readers expect different behavior from each: links activate on Enter only, buttons on Enter and Space.

### 8.2 Accessible name, role, state

Every interactive element must expose all three (SC 4.1.2). Inspect them in Chrome/Firefox DevTools → **Accessibility** panel (or the "accessibility tree" view).

How the **accessible name** is computed, in priority order (simplified):

1. `aria-labelledby` (text of the referenced element(s))
2. `aria-label`
3. Native labelling: `<label for>`, `alt` on images, `<caption>`, `<legend>`, button/link text content
4. `title` attribute (fallback; also shown as a tooltip, unreliable)
5. `placeholder` (fallback for inputs in some browsers; don't rely on it)

```html
<!-- Icon-only button: name via aria-label -->
<button type="button" aria-label="Close dialog">
  <svg aria-hidden="true" focusable="false">…</svg>
</button>

<!-- Or visually hidden text (also gets translated by browser translation tools, unlike aria-label in some cases) -->
<button type="button">
  <svg aria-hidden="true" focusable="false">…</svg>
  <span class="visually-hidden">Close dialog</span>
</button>

<!-- Input with label, hint and error -->
<label for="email">Email</label>
<p id="email-hint">We'll send the receipt here.</p>
<input id="email" type="email" autocomplete="email"
       aria-describedby="email-hint email-error" aria-invalid="true" required>
<p id="email-error">Enter an email address like name@example.com.</p>
```

A screen reader announces the last input as something like: *"Email, edit text, required, invalid entry. We'll send the receipt here. Enter an email address like name@example.com."*

The standard `visually-hidden` class (hidden visually, still read by screen readers):

```css
.visually-hidden {
  position: absolute !important;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

How different hiding methods behave:

| Technique | Visible | In accessibility tree (read by SR) | Focusable |
|---|---|---|---|
| `display: none` / `hidden` attribute | ✗ | ✗ | ✗ |
| `visibility: hidden` | ✗ | ✗ | ✗ |
| `.visually-hidden` | ✗ | ✓ | ✓ if interactive |
| `aria-hidden="true"` | ✓ | ✗ | **Still ✓, which is a bug if it's interactive.** Never put `aria-hidden` on focusable elements. |
| `inert` attribute | ✓ | ✗ | ✗ (great for content behind a modal) |
| `opacity: 0` | ✗ | ✓ | ✓ (often a bug: invisible but focusable) |

### 8.3 ARIA: the rules

**ARIA** (Accessible Rich Internet Applications) is a set of HTML attributes (`role`, `aria-*`) that change what's exposed in the accessibility tree. It **only changes semantics. It adds no behavior.** `role="button"` doesn't make an element focusable or respond to Enter.

The W3C's rules of ARIA use (paraphrased):

1. **If a native HTML element exists, use it instead of ARIA.** ("The first rule of ARIA is don't use ARIA.")
2. Don't change native semantics unless you really must (`<h2 role="button">` is wrong; put a `<button>` inside the `<h2>`).
3. All interactive ARIA controls must be **keyboard operable**.
4. Don't use `role="presentation"` or `aria-hidden="true"` on focusable elements.
5. All interactive elements must have an **accessible name**.

**No ARIA is better than bad ARIA.** WebAIM Million data consistently shows that pages using ARIA have *more* detected errors on average than pages without it, because ARIA is easy to misuse.

Most useful ARIA attributes:

| Attribute | Use for |
|---|---|
| `aria-label`, `aria-labelledby` | Accessible name when there's no visible text label (icon buttons, a nav region) |
| `aria-describedby` | Hints and error messages attached to a field |
| `aria-expanded` | Disclosure buttons, menus, accordions ("Filters, button, collapsed") |
| `aria-controls` | Which element a control affects (support is limited, but harmless) |
| `aria-pressed` | Toggle buttons ("Bold, toggle button, pressed") |
| `aria-checked` + `role="switch"` | On/off switches |
| `aria-current="page"` | Current item in navigation or breadcrumbs |
| `aria-invalid`, `aria-required` | Form validation state (prefer native `required`) |
| `aria-live`, `role="status"`, `role="alert"` | Announcing dynamic updates ([§8.6](#86-announcing-dynamic-changes-live-regions)) |
| `aria-hidden="true"` | Hiding decorative icons and duplicate content from AT |
| `aria-modal="true"` | Telling AT that content outside a custom dialog is inert |

### 8.4 Keyboard interaction

Standard keys users expect:

| Key | Expected behavior |
|---|---|
| `Tab` / `Shift+Tab` | Move to next / previous interactive element |
| `Enter` | Activate links and buttons; submit forms |
| `Space` | Activate buttons; toggle checkboxes; scroll page if nothing is focused |
| Arrow keys | Move **within** a composite widget: radio group, tabs, menu, listbox, grid, slider |
| `Esc` | Close dialogs, menus, popovers, tooltips |
| `Home` / `End` | First / last item in a list, menu or slider |

**Composite widgets use one tab stop.** A tab list, toolbar, menu or radio group should be **one** Tab stop, with arrow keys moving inside it (the "roving tabindex" or `aria-activedescendant` technique). Otherwise a toolbar with 20 buttons takes 20 Tabs to get past.

`tabindex` values:

| Value | Meaning |
|---|---|
| (none) on native interactive elements | Focusable in DOM order. Usually what you want. |
| `tabindex="0"` | Makes a non-interactive element focusable in DOM order (only for custom widgets with a role and keyboard handling) |
| `tabindex="-1"` | Focusable by script (`el.focus()`), not by Tab. Use for headings or containers you move focus to, and inactive items in a roving tabindex. |
| `tabindex="1"` or higher | **Don't.** It overrides the natural order for the whole page and is almost always a bug. |

**Focus order = DOM order.** If you reorder things visually with CSS (`order`, `flex-direction: row-reverse`, absolute positioning, grid placement), the Tab order and screen reader reading order no longer match what's on screen (SC 1.3.2, 2.4.3). Change the DOM order instead.

For complex widgets (combobox, tree, grid, menu bar, tabs), follow the **WAI-ARIA Authoring Practices Guide (APG)** patterns, which specify the roles, states and keyboard behavior for each. Better still, use a well-tested library (React Aria, Radix UI, Headless UI, Angular CDK, Reach-style primitives) instead of writing them yourself.

### 8.5 Focus management

When the UI changes, decide **where keyboard focus goes**. If you don't, focus is lost (it goes back to `<body>`, and the screen reader user is back at the top of the page with no idea what happened).

| Situation | Where focus should go |
|---|---|
| **Open a modal dialog** | Into the dialog: the first focusable element, the safest action (e.g. "Cancel" for destructive dialogs), or the dialog heading |
| **While the modal is open** | Trapped inside: Tab cycles within the dialog; the rest of the page is `inert` |
| **Close the dialog** | **Back to the element that opened it** |
| **SPA route change** | To the new page's `<h1>` (with `tabindex="-1"`) or main region; update `document.title`. Without this, screen reader users get **no signal** that the page changed. |
| **Delete an item from a list** | To the next item, the previous item, or the list heading. Not lost to `<body>`. |
| **Form submitted with errors** | To the error summary or the first invalid field |
| **Expand an inline section** | Usually stays on the toggle button (with `aria-expanded="true"`) |
| **Infinite scroll / "Load more"** | To the first newly loaded item (or announce "20 more results loaded") |

The native `<dialog>` element opened with `showModal()` handles a lot of this for you: it makes the rest of the page inert, closes on `Esc`, moves focus into the dialog, and in modern browsers returns focus to the previously focused element on close.

```html
<button type="button" id="delete-btn">Delete account</button>

<dialog id="confirm" aria-labelledby="confirm-title">
  <h2 id="confirm-title">Delete your account?</h2>
  <p>This permanently deletes your account and all 34 projects.</p>
  <form method="dialog">
    <button value="cancel" autofocus>Cancel</button>
    <button value="delete">Delete account</button>
  </form>
</dialog>

<script>
  const dialog = document.getElementById('confirm');
  document.getElementById('delete-btn').addEventListener('click', () => dialog.showModal());
  dialog.addEventListener('close', () => {
    if (dialog.returnValue === 'delete') deleteAccount();
  });
</script>
```

### 8.6 Announcing dynamic changes (live regions)

Sighted users notice when a toast appears or a counter changes. Screen reader users don't, unless you announce it (SC 4.1.3).

```html
<!-- Put the live region in the DOM on page load (empty), then change its text -->
<div role="status" aria-live="polite" class="visually-hidden" id="announcer"></div>

<script>
  function announce(message) {
    const el = document.getElementById('announcer');
    el.textContent = '';                       // clear so repeated messages are re-announced
    setTimeout(() => { el.textContent = message; }, 50);
  }
  announce('Item added to cart. Cart has 3 items.');
</script>
```

| Mechanism | Behavior | Use for |
|---|---|---|
| `aria-live="polite"` / `role="status"` | Announced when the screen reader is idle | "Saved", "12 results", "Item added to cart" |
| `aria-live="assertive"` / `role="alert"` | Interrupts whatever is being read | Urgent errors only: "Connection lost", "Session expires in 1 minute" |

Gotchas:

- The live region must **already exist** in the DOM before you change its content. Inserting a new element that has `aria-live` on it is often not announced.
- Don't overuse `assertive`. Interrupting the user constantly is like a sighted user getting a popup every 5 seconds.
- Don't announce every keystroke (e.g. on a search box, debounce "15 results" until the user pauses typing).

### 8.7 Landmarks, headings, skip links, titles, language

```html
<html lang="en">
<head><title>Order #1042 – Orders – Acme</title></head>   <!-- specific → general -->
<body>
  <a class="skip-link" href="#main">Skip to main content</a>
  <header>…</header>
  <nav aria-label="Main">…</nav>
  <main id="main" tabindex="-1">
    <h1>Order #1042</h1>
    <section aria-labelledby="items-heading">
      <h2 id="items-heading">Items</h2> …
    </section>
  </main>
  <footer>…</footer>
</body>
</html>
```

- **Skip link**: the first focusable element; visually hidden until focused. Lets keyboard users jump past a 40-link navigation bar (SC 2.4.1).
- **One `<main>`**; label multiple `<nav>`s ("Main", "Breadcrumb", "Footer").
- **Heading outline** should make sense on its own, like a table of contents. Use heading level for structure and CSS for size.
- **Unique, descriptive page titles**, updated on SPA navigation. The title is the first thing a screen reader announces on page load.
- **`lang`** on `<html>`, and on any passage in a different language (`<span lang="vi">Xin chào</span>`), so the right pronunciation rules are used.

### 8.8 CSS and media query gotchas

```css
/* Visible keyboard focus, without showing it on mouse click */
:focus-visible {
  outline: 3px solid #1a5fd0;
  outline-offset: 2px;
}

/* Reduce motion for users who ask for it */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* Windows high contrast / forced colors: restore borders lost with backgrounds */
@media (forced-colors: active) {
  .btn { border: 1px solid ButtonText; }
}

/* Keep focused elements visible below a sticky header */
html { scroll-padding-top: 5rem; }
```

- `outline: none` without a replacement is one of the most damaging single lines of CSS for keyboard users.
- Use `rem`/`em` for font sizes and avoid fixed heights on text containers, so zoom and user font settings work.
- Content added with CSS `::before`/`::after` is read by some screen readers and not others. Don't put essential information there.
- Hover-only UI (menus, actions that only appear on hover) must also appear on focus, and needs a touch alternative.
- Custom-styled checkboxes, radios and selects: style the **native** element (`appearance: none` + CSS), or keep the native input visually hidden but focusable. Don't replace it with a `div`.

### 8.9 Component quick reference

| Component | Key requirements |
|---|---|
| **Modal dialog** | `<dialog>` + `showModal()`; labelled by its heading; focus in on open, back to trigger on close; Esc closes; background inert |
| **Disclosure / accordion** | `<button aria-expanded>` inside a heading, or `<details>/<summary>` |
| **Tabs** | `role="tablist"`, `tab`, `tabpanel`; `aria-selected`; arrow keys between tabs; one tab stop |
| **Menu button / dropdown** | Button with `aria-expanded` + `aria-haspopup`; Esc closes and returns focus; arrow keys inside. Note: for site navigation, a simple list of links in a disclosure is usually better than `role="menu"`. |
| **Combobox / autocomplete** | Follow the APG combobox pattern exactly (or use a library); announce result count; arrow keys move through options |
| **Toggle / switch** | `<button aria-pressed>` or `role="switch"` + `aria-checked`, or a styled native checkbox |
| **Tooltip** | Shows on hover **and** focus; dismissible with Esc; hoverable; supplementary info only. Never put essential info or interactive content in a tooltip. |
| **Toast / snackbar** | Text in a live region; doesn't steal focus; stays long enough to read (or until dismissed); any action (Undo) is also reachable another way |
| **Carousel** | Pause control; previous/next buttons with names; no autoplay (or pause on hover/focus); slides announced as "Slide 2 of 5" |
| **Data table** | `<table>`, `<caption>`, `<th scope="col|row">`; sortable headers are buttons with `aria-sort` |
| **Icon button** | Accessible name; 24px+ target; visible label preferred |
| **Pagination** | `<nav aria-label="Pagination">`; current page with `aria-current="page"`; links named "Page 3", not just "3" (or with context) |
| **Infinite scroll** | Provide "Load more" or pagination as an alternative; don't put the footer behind it; manage focus or announce new content |
| **Drag and drop** | Keyboard and click alternative (e.g. "Move up / Move down" buttons, or a "Move to…" menu) |
| **CAPTCHA** | Avoid puzzles; use invisible risk scoring, passkeys, email verification; if unavoidable, provide audio *and* non-cognitive alternatives |
| **Charts** | Text summary of the insight, accessible data table alternative, patterns or labels in addition to color |

### 8.10 Native mobile and cross-platform

The concepts are the same; the APIs differ.

| Platform | Name / label | Role / traits | Hide from AT | Group |
|---|---|---|---|---|
| **iOS (UIKit / SwiftUI)** | `accessibilityLabel`, `.accessibilityLabel()` | `accessibilityTraits` (`.button`, `.header`…) | `isAccessibilityElement = false`, `.accessibilityHidden(true)` | `.accessibilityElement(children: .combine)` |
| **Android (Views / Compose)** | `contentDescription`, `Modifier.semantics { contentDescription = … }` | `Role.Button`, `heading()` | `importantForAccessibility="no"`, `clearAndSetSemantics {}` | `mergeDescendants = true` |
| **React Native** | `accessibilityLabel` | `accessibilityRole` | `accessibilityElementsHidden` / `importantForAccessibility` | `accessible={true}` on a container |
| **Flutter** | `Semantics(label: …)` | `Semantics(button: true, header: true)` | `ExcludeSemantics` | `MergeSemantics` |

Mobile-specific:

- Support **Dynamic Type / font scaling** up to the largest sizes; test layouts with it on.
- Touch targets **44pt (iOS) / 48dp (Android)**.
- Every **custom gesture** needs an alternative (custom accessibility actions, visible buttons).
- Test with **VoiceOver** and **TalkBack** on real devices: swipe through every screen.
- Respect system settings: Reduce Motion, Bold Text, Increase Contrast, Dark Mode.

---

## 9. Testing accessibility

No single method is enough. Combine them:

| Method | Catches | Misses | Cost |
|---|---|---|---|
| **Automated scanners** (axe, Lighthouse, WAVE) | Missing alt, missing labels, contrast, invalid ARIA, missing `lang`, empty buttons | Whether alt text is *meaningful*, focus order, keyboard traps, usability, focus management, most cognitive issues. Typically only around a third to a half of issues are automatically detectable. | Very low |
| **Linting** (eslint-plugin-jsx-a11y, angular-eslint template a11y rules, Android Lint, SwiftLint) | Obvious code-level issues while typing | Runtime behavior | Very low |
| **Manual keyboard test** | Unreachable controls, traps, invisible focus, focus loss, wrong order | Screen-reader-specific issues | Low |
| **Screen reader test** | Missing names, wrong roles, unannounced updates, confusing structure | — (but requires practice) | Medium |
| **Zoom / reflow / contrast / reduced-motion checks** | Low-vision and vestibular issues | — | Low |
| **Expert audit** (against WCAG) | Comprehensive conformance report, ACR/VPAT | Real-world usability | High |
| **Usability testing with disabled users** | Real problems, real workarounds, whether it's actually *usable* | — | Medium–high (most valuable) |

### 9.1 The 5-minute keyboard test (do this on every feature)

1. Click in the browser address bar, then put the mouse away.
2. Press `Tab` repeatedly through the whole page. **Can you always see where focus is?**
3. Is the order logical (follows the visual layout)?
4. Can you reach and **operate** every interactive thing (Enter, Space, arrows, Esc)?
5. Open a menu, dialog or popover. Does focus go into it? Can you close it with Esc? Does focus come back to where you were?
6. Is there any place you get **stuck** (keyboard trap)?
7. Is any focused element **hidden** behind a sticky header or banner?
8. Is there a skip link at the start?

### 9.2 Screen reader smoke test

| Action | NVDA (Windows) | VoiceOver (macOS) | VoiceOver (iOS) | TalkBack (Android) |
|---|---|---|---|---|
| Turn on/off | `Ctrl+Alt+N` | `Cmd+F5` | Settings → Accessibility, or triple-click side button (if set) | Hold both volume keys (if set) |
| Read next item | `↓` | `Ctrl+Option+→` | Swipe right | Swipe right |
| Activate | `Enter` | `Ctrl+Option+Space` | Double-tap | Double-tap |
| Next heading | `H` | `Ctrl+Option+Cmd+H` | Rotor → Headings, swipe down | Reading controls → Headings |
| List links / headings / landmarks | `Insert+F7` | `Ctrl+Option+U` (rotor) | Rotate two fingers (rotor) | Swipe up/down to change granularity |
| Stop speaking | `Ctrl` | `Ctrl` | Two-finger tap | — |

Check:
- Does the page title make sense? Does the heading list read like a table of contents?
- Does every button, link and field announce a sensible **name + role + state**?
- Fill in a form, including an error. Are errors announced?
- Trigger dynamic updates (add to cart, save, filter). Are they announced?
- Do images have meaningful (or empty, if decorative) alt text?

### 9.3 Low-vision and motion checks

- Zoom to **200%** and **400%** (Ctrl/Cmd +). Anything clipped, overlapping, or requiring horizontal scroll?
- Set the browser default font size to 24px or larger. Does text grow?
- Check **contrast** with DevTools (the color picker shows the ratio) or a contrast checker.
- Emulate **color vision deficiencies** and **grayscale** (DevTools → Rendering).
- Turn on **reduced motion** in the OS. Do big animations stop?
- Turn on **Windows Contrast Themes** / emulate `forced-colors: active`. Are buttons, inputs, focus rings and icons still visible?

### 9.4 Automating in CI

```js
// Playwright + axe-core: fail the build on WCAG A/AA violations
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('checkout page has no detectable a11y violations', async ({ page }) => {
  await page.goto('/checkout');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});

// Role-based queries test accessibility for free:
// if this button has no accessible name, the test can't find it.
await page.getByRole('button', { name: 'Place order' }).click();
await expect(page.getByRole('status')).toHaveText(/Order confirmed/);
```

Other tools: `jest-axe` / `vitest-axe` (component tests), Storybook a11y addon, Pa11y CI, Lighthouse CI, Accessibility Insights for Web (guided manual assessment), Android Accessibility Scanner, Xcode Accessibility Inspector.

**Important:** "axe reports 0 violations" means "no *automatically detectable* violations". It doesn't mean accessible.

---

## 10. Accessibility as a system / organization concern

Accessibility isn't only a front-end task. Many barriers come from architecture, content pipelines, and process.

### 10.1 Process: shift left

| Phase | Accessibility work |
|---|---|
| **Requirements** | State the target (e.g. "WCAG 2.2 AA") as a non-functional requirement. Include disabled users in personas and research. |
| **Design** | Accessible design system tokens (colors, focus, type scale, spacing); annotations ([§7.11](#711-annotate-your-designs-for-developers)); design review checklist |
| **Build** | Semantic HTML; accessible component library; linting |
| **Test** | Automated checks in CI; keyboard and screen reader checks in the **definition of done**; periodic expert audits; usability tests including disabled participants |
| **Release** | Accessibility statement; feedback channel; ACR/VPAT for B2B |
| **Operate** | Track accessibility bugs with severity like any other defect; regression tests; monitor new content (CMS) |

### 10.2 The design system is your biggest lever

If the shared `Button`, `Dialog`, `Select`, `Tabs`, `Toast` and `FormField` components are accessible, every team that uses them gets accessibility for free. If they're broken, every screen is broken, and fixing it means touching every product. Invest in the design system first: accessible color tokens, focus styles, components following APG patterns, documented usage ("icon buttons require `label`"), and enforced through types (`label: string` as a required prop).

### 10.3 Backend and architecture decisions that affect accessibility

| System decision | Accessibility impact |
|---|---|
| **CMS / content model** | Make `alt` a **field** on every image asset (with a "decorative" checkbox), required at upload. Store captions and transcripts alongside media. Without this, editors *can't* make content accessible. |
| **Media pipeline** | Generate captions (ASR) and route them for human review; store multiple caption tracks (languages) and audio-description tracks; support WebVTT. |
| **User-generated content** | Prompt users for alt text (as Mastodon, X and LinkedIn do); AI-suggested alt as a starting point for the user to edit. |
| **API error format** | Return **field-level**, human-readable errors (`{ "field": "email", "message": "…" }`) so the UI can attach each error to its field with `aria-describedby`. A generic `400 Bad Request` forces a vague "Something went wrong". |
| **Session timeouts** | Warn and allow extending (SC 2.2.1); preserve form state across re-authentication. Screen reader and switch users take longer. |
| **Authentication** | Allow paste and password managers; support passkeys/WebAuthn and magic links; avoid puzzle CAPTCHAs (SC 3.3.8). |
| **Rate limits and anti-bot rules** | Assistive tech and slower interaction patterns can look "unusual" to bot detection. Don't block keyboard-only or slow users. |
| **Notifications** | Offer multiple channels (email, SMS, push, in-app); 2FA not only by voice call or only by SMS. |
| **Generated documents** (PDF invoices, reports, emails) | Tagged PDFs with reading order and alt text, or an HTML alternative; accessible HTML emails (real text, `lang`, alt, sufficient contrast, semantic tables only for data). |
| **Performance** | Assistive tech adds overhead, and many disabled users use older, cheaper devices. Huge DOMs (thousands of nodes) make screen readers slow. Virtualized lists need care (keep the accessibility tree consistent; consider `aria-rowcount`/`aria-rowindex`). |
| **Server-side rendering / progressive enhancement** | Semantic HTML from the server works before (and without) JavaScript; also improves SEO and slow-network users. |
| **i18n** | Set `lang` per locale; translations must include `aria-label`s and alt text; support right-to-left layouts (`dir="rtl"`). |
| **Third-party widgets** (chat, payments, maps, cookie banners, video players) | You're responsible for them on your site. Check vendors' ACR/VPAT during procurement; test them like your own code. Cookie banners and chat widgets are common causes of focus traps and obscured focus. |
| **Feature flags / experiments** | A/B test variants must meet the same standard; include accessibility in experiment review. |

### 10.4 AI and accessibility

- **Helpful:** AI-generated alt text drafts, auto-captioning, summarizing complex content into plain language, voice interfaces, screen-reader-friendly chat assistants, apps like Be My Eyes that describe surroundings.
- **Risks:** AI alt text can be confidently wrong or miss the *purpose* of an image in context. Auto-captions mangle names and terminology. Treat AI output as a draft that a human reviews for important content.
- **Chat / AI UIs themselves:** streaming responses must be announced sensibly (don't announce every token; announce when the response is complete, or in chunks), and focus shouldn't jump around while text streams in.

### 10.5 Accessibility overlays

"Overlays" are third-party scripts (a toolbar or widget) that claim to make a site WCAG-compliant automatically with one line of JavaScript. Disability organizations and accessibility professionals widely oppose them (see the *Overlay Fact Sheet*, signed by hundreds of practitioners):

- They can't fix the underlying code (missing semantics, broken keyboard behavior, focus management).
- They often **conflict with the assistive technology** users already have.
- Sites using them are still sued, and some lawsuits cite the overlay itself as a barrier.
- In 2025 the US FTC finalized an order against one major overlay vendor (accessiBe) over deceptive claims that its product made websites compliant, including a $1 million penalty.

Fix the code instead.

---

## 11. Approaches by product type

| Product type | Priorities |
|---|---|
| **Government, public services** | Legally required, often at a strict level (WCAG 2.1/2.2 AA); must serve *everyone*, including low digital literacy; plain language; accessibility statement. Model: GOV.UK design system. |
| **Banking, insurance, healthcare** | Legally required in many regions (EAA covers banking); accessible authentication; long forms with timeouts and error recovery; accessible PDFs and statements. |
| **E-commerce** | EAA covers it in the EU; product images need good alt; filters and carousels need keyboard support; accessible checkout is the money path; avoid CAPTCHA at checkout. |
| **Education** | Captions and transcripts for lectures; accessible documents and math (MathML); extra time options for assessments; assistive tech in exam modes. |
| **Media, streaming, news** | Captions, audio description, accessible media players, reduced motion, no autoplay with sound. |
| **Enterprise / B2B SaaS** | ACR/VPAT required by procurement; complex widgets (data grids, dashboards, drag-and-drop) need keyboard and screen reader support; employees with disabilities must be able to do their jobs. |
| **Developer tools, CLIs** | Don't encode meaning only in color (use symbols/text too); respect `NO_COLOR`; avoid spinners that spam screen readers; offer plain/non-interactive output modes; accessible documentation sites. |
| **Mobile apps** | Dynamic Type, VoiceOver/TalkBack, touch targets, gesture alternatives, orientation. |
| **Games** | Remappable controls, subtitles with speaker labels, colorblind modes, difficulty and assist options, hold-vs-toggle settings. The Xbox Adaptive Controller and *The Last of Us Part II* (60+ accessibility options) are reference examples. |
| **Kiosks, ATMs, TVs** | Headphone jack with speech output, tactile keys, reachable height, high contrast, TV remote (D-pad) navigation = keyboard-like focus management. |
| **Voice and conversational UIs** | Alternatives for people who can't speak or hear; text input and output; don't time out quickly; transcripts. |

---

## 12. Trade-offs and misconceptions

### 12.1 Real trade-offs

| Tension | How to balance |
|---|---|
| **Brand aesthetics vs contrast** | Brand palettes often fail contrast. Keep the brand color for large elements and backgrounds, and define darker accessible variants for text and small UI. Good designers treat contrast as a constraint, like a grid. |
| **Minimal UI vs discoverability** | Icon-only, hover-reveal and gesture-only UIs look clean but hide functionality from everyone. Use visible labels for important actions. |
| **Security vs accessibility** | CAPTCHAs, short timeouts, disabled paste and complex auth hurt disabled users most. Use risk-based authentication, passkeys and warnings before timeouts instead. |
| **Custom components vs native** | Custom selects and date pickers give full styling control but take a lot of work to make accessible. Prefer styled native elements or a proven accessible library. |
| **Rich animation vs comfort** | Provide reduced-motion alternatives; animation should support understanding, not decorate. |
| **Time to market vs accessibility** | Retrofitting costs far more. Building on accessible components keeps the extra cost small. |
| **Conformance vs real usability** | Passing WCAG doesn't guarantee a good experience. Test with real users. |
| **Different needs can conflict** | Low-vision users may want large text; some cognitive disabilities benefit from less on screen; some users need high contrast while others with light sensitivity need low glare. Respect user settings (OS font size, contrast, dark mode, reduced motion) instead of choosing one "right" design. |

### 12.2 Common misconceptions

| Myth | Reality |
|---|---|
| "Accessibility is only for blind people." | It covers vision, hearing, motor, cognitive, speech, seizure and vestibular disabilities, plus temporary and situational ones. |
| "Our users don't have disabilities." | You don't know: most disabilities are invisible and users don't tell you. And people who can't use your product aren't in your analytics at all. |
| "It makes design ugly." | Many beautiful products are highly accessible. Constraints like contrast and visible focus are design problems, not design limits. |
| "It's expensive." | Built in from the start, it's a small cost. Retrofitting, or losing a lawsuit, is expensive. |
| "A plugin / overlay will fix it." | It won't ([§10.5](#105-accessibility-overlays)). |
| "Lighthouse says 100, so we're done." | Automated tools detect a fraction of issues. |
| "We'll do it later / in a hardening sprint." | Like security, it's much harder to add later. |
| "Add ARIA everywhere to be safe." | Wrong ARIA is worse than no ARIA. Use semantic HTML first. |
| "Screen readers don't run JavaScript." | They do. Modern screen readers work with JavaScript apps fine, *if* the JavaScript produces accessible markup and manages focus. |

---

## 13. Anti-patterns

| Anti-pattern | Why it's harmful | Fix |
|---|---|---|
| `<div onclick>` / `<span onclick>` as buttons | Not focusable, no role, no keyboard activation | `<button>` |
| `<a href="#" onclick>` for actions | Announced as a link, adds `#` to URL, wrong expectations | `<button type="button">` |
| `outline: none` with no replacement | Keyboard users can't see where they are | Custom `:focus-visible` style |
| Placeholder as the only label | Disappears on typing, low contrast, unreliable as accessible name | Visible `<label>` |
| Icon buttons with no accessible name | Screen reader says just "button" | `aria-label` or visually hidden text |
| Images without `alt` | File name read aloud | Meaningful `alt` or `alt=""` |
| "Click here" / "Read more" links | Meaningless in a links list | Descriptive link text, or visually hidden context ("Read more <span class="visually-hidden">about pricing</span>") |
| Color-only meaning | Invisible for color-blind users and in grayscale | Add text, icons or patterns |
| Low-contrast text (light gray on white) | Unreadable for low vision, older users, sunlight | ≥ 4.5:1 |
| Custom modals without focus management | Focus stays behind the modal; Tab goes to hidden page; no Esc | `<dialog>` + `showModal()` |
| Skipping heading levels or using headings for styling | Broken document outline for navigation | Semantic levels; style with CSS |
| `tabindex="1+"` | Breaks the natural Tab order of the whole page | Fix DOM order; use `0` or `-1` |
| Autoplaying carousels and videos with no pause | Distracting; drowns out screen readers; can trigger vestibular issues | No autoplay, or a visible pause button |
| Disabling zoom (`user-scalable=no`, `maximum-scale=1`) | Low-vision users can't zoom on mobile | Remove it |
| Hover-only menus and actions | Unreachable for keyboard and touch users | Also show on focus; tap/click to open |
| Timeouts without warning | Slower users lose their work | Warn, allow extension, autosave |
| Blocking paste in password / OTP fields | Breaks password managers; cognitive barrier (SC 3.3.8) | Allow paste |
| Inaccessible CAPTCHA | Blocks blind and cognitively disabled users entirely | Invisible checks, passkeys, alternatives |
| SPA navigation with no focus or title change | Screen reader users don't know the page changed | Move focus to `<h1>`, update `document.title` |
| Toast messages not announced, or disappearing too fast | Updates are invisible to screen reader users; slow readers miss them | Live region; longer duration or persist until dismissed |
| `aria-hidden="true"` on focusable elements | Focus lands on something the screen reader can't describe ("blank") | Use `inert`, or remove from tab order too |
| Accessibility overlays | Don't fix the code; interfere with AT | Fix the code |
| Disabled buttons with no explanation | Users don't know why they can't continue; disabled elements are also often unfocusable, so screen reader users may not even find them | Keep the button enabled and show validation errors on click, or explain what's missing |

---

## 14. Checklist

**Strategy**
- [ ] Target standard defined (e.g. WCAG 2.2 AA) as a non-functional requirement
- [ ] Accessibility in the definition of done for every feature
- [ ] Design system components are accessible and documented
- [ ] Accessibility statement published, with a way to report problems
- [ ] Third-party components and vendors checked (ACR/VPAT)

**Design**
- [ ] Text contrast ≥ 4.5:1 (large text ≥ 3:1); UI components and icons ≥ 3:1
- [ ] Color isn't the only way any information is conveyed
- [ ] Visible, high-contrast focus style designed for every interactive component
- [ ] Targets ≥ 24×24 CSS px (aim for 44×44)
- [ ] Layout works at 320px width / 400% zoom; text scales
- [ ] Visible labels on all form fields; instructions before input; errors in text next to fields
- [ ] Drag, swipe and gestures all have simple alternatives
- [ ] Reduced-motion version for big animations; nothing flashes > 3 times/second
- [ ] Headings, landmarks, focus order, alt text and announcements annotated in handoff
- [ ] Plain language; consistent navigation and help location

**Build**
- [ ] Semantic HTML: real buttons, links, headings, lists, tables, labels, landmarks
- [ ] `<html lang>` set; unique page titles (updated on SPA navigation)
- [ ] Skip link to main content
- [ ] All functionality works with keyboard only; no traps; logical order; no `tabindex` > 0
- [ ] Every interactive element has an accessible name, correct role and state
- [ ] Images: meaningful `alt` or `alt=""`; icon buttons labelled; SVG icons `aria-hidden`
- [ ] Modals: focus in, trapped, Esc closes, focus returns to trigger
- [ ] Dynamic updates announced via live regions
- [ ] Focus managed on route change, deletion, form errors, loaded content
- [ ] `:focus-visible` styles present; no `outline: none` without replacement
- [ ] `prefers-reduced-motion` and `forced-colors` handled
- [ ] Zoom not disabled on mobile; paste allowed in password/code fields
- [ ] Videos captioned; audio transcribed; no autoplay with sound
- [ ] Mobile: Dynamic Type/font scale, VoiceOver/TalkBack labels, 44pt/48dp targets

**Test**
- [ ] Automated checks (axe/Lighthouse) in CI; a11y lint rules on
- [ ] Keyboard-only walkthrough of every new flow
- [ ] Screen reader smoke test (at least NVDA or VoiceOver, plus mobile SR for apps)
- [ ] 200%/400% zoom, grayscale/color-blind, high contrast and reduced-motion checks
- [ ] Periodic expert audit against WCAG
- [ ] Usability tests include participants with disabilities

---

## 15. Exercises

1. **Screen reader day.** Turn on VoiceOver (Mac/iPhone) or NVDA (Windows). Close your eyes or turn off the screen. Try to: find a product on a shopping site, add it to the cart, and reach checkout. Then do the same with your own project. Write down every point where you got lost.
2. **Unplug the mouse.** Complete 3 key tasks in an app you built using only the keyboard. Count the unreachable controls, invisible focus states and traps.
3. **Contrast audit.** Pick 10 text/background pairs from a site you like (DevTools color picker shows contrast). How many pass 4.5:1? For the failing ones, find the closest passing color in the same hue.
4. **Grayscale test.** View a dashboard or chart in grayscale (DevTools → Rendering → Emulate vision deficiencies → Achromatopsia). Is any information lost? Redesign it so nothing depends on color.
5. **Alt text practice.** Write alt text for the same photo (a person holding a coffee cup in an office) for three contexts: a coffee shop's product page, a news article about remote work, and a decorative hero banner. Why are they different?
6. **Fix the div soup.** Take this and make it accessible with the least code:
   `<div class="card" onclick="open(42)"><img src="p42.jpg"><div class="title">Blue Sneakers</div><div class="heart" onclick="fav(42)">♥</div></div>`
   (Hint: what should be a link, what should be a button, what needs a name, and can a link contain a button?)
7. **Build an accessible modal from scratch** without `<dialog>`: focus trap, Esc, `inert` background, focus return, labelled heading. Then replace it with `<dialog>` and compare the amount of code.
8. **Run axe in CI** on one project ([§9.4](#94-automating-in-ci)). Fix all violations. Then do a manual keyboard + screen reader pass and list the issues axe *didn't* find.
9. **System-design tie-in.** Design the data model and API for an image upload feature in a CMS so that every published image has correct alt text or is explicitly marked decorative. What happens to existing images? Who reviews AI-generated alt text?
10. **Read the WebAIM Million report** for the latest year. Which 3 errors are most common? Check your own project for each.

---

## 16. Further reading

**Standards and official guidance**
- WCAG 2.2: w3.org/TR/WCAG22, and the **"How to Meet WCAG" quick reference** (filterable by level and technology). Start with the quick reference rather than the spec.
- *Understanding WCAG 2.2*: explains the intent behind each success criterion, with examples.
- **WAI-ARIA Authoring Practices Guide (APG)**: w3.org/WAI/ARIA/apg: patterns and keyboard behavior for widgets.
- W3C WAI *Easy Checks* (a first review in about an hour), *Tips for Designing / Developing / Writing*, and *Making Content Usable for People with Cognitive and Learning Disabilities* (COGA).
- ARIA in HTML / "Using ARIA" (W3C): which roles and attributes are allowed on which elements.

**Books**
- *Inclusive Design Patterns* (Heydon Pickering): accessible component patterns. Also his site *Inclusive Components*.
- *Accessibility for Everyone* (Laura Kalbag): short, practical introduction.
- *A Web for Everyone* (Sarah Horton & Whitney Quesenbery): accessible UX design.
- *Mismatch: How Inclusion Shapes Design* (Kat Holmes): the "disability as mismatch" idea and inclusive design.
- *Form Design Patterns* (Adam Silver): accessible forms.

**Practical resources**
- **WebAIM** (webaim.org): articles, contrast checker, WAVE tool, The WebAIM Million, screen reader user surveys.
- **The A11Y Project** (a11yproject.com): checklist and articles.
- **MDN Web Docs** accessibility section, including the ARIA reference.
- **GOV.UK Design System** and Service Manual: some of the best-researched accessible components and patterns.
- **Deque University** and **axe** docs; **Accessibility Insights** (Microsoft).
- **Microsoft Inclusive Design** toolkit: the permanent/temporary/situational framework.
- Apple Human Interface Guidelines (Accessibility); Material Design (Accessibility); Android and iOS accessibility developer docs.
- Sara Soueidan's articles and *Practical Accessibility* course; Adrian Roselli's and Léonie Watson's blogs (real screen reader behavior and testing).
- *Overlay Fact Sheet* (overlayfactsheet.com).
- Game Accessibility Guidelines (gameaccessibilityguidelines.com).

**Standards (formal)**
- WCAG 2.2 = W3C Recommendation; WCAG 2.0 = ISO/IEC 40500.
- EN 301 549 (EU ICT accessibility requirements).
- ISO/IEC 25010:2023 (Interaction capability → inclusivity, user assistance).
- ISO 9241-171 (software accessibility), ISO 9241-11 (usability).
