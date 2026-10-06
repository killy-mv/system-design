# Usability

> **Usability** = how well specific users can use a product to achieve specific goals with **effectiveness**, **efficiency**, and **satisfaction** in a specific context of use. (ISO 9241-11)
> "Can people actually get their job done with this, without confusion, mistakes, or frustration?"

Usability is the **umbrella** for most other UI/UX qualities:
[Learnability](../learnability/README.md) · [Efficiency of use](../efficiency-of-use/README.md) · [Memorability](../memorability/README.md) · [User error protection](../user-error-protection/README.md) · [Satisfaction](../satisfaction/README.md) · [Operability](../operability/README.md) · [Consistency](../consistency/README.md) · [Accessibility](../accessibility/README.md) · [Responsiveness](../responsiveness/README.md)

In **ISO/IEC 25010:2023**, the old "Usability" characteristic was renamed **Interaction capability**, with sub-characteristics: appropriateness recognizability, learnability, operability, user error protection, user engagement (formerly "UI aesthetics"), inclusivity, user assistance (formerly part of accessibility), and self-descriptiveness.

▶ **Interactive demo:** [index.html](index.html): open it in a browser to try 3 bad designs side by side with their fixes.

---

## Table of contents

1. [Definitions: usability vs UX vs UI](#1-definitions-usability-vs-ux-vs-ui)
2. [The five components of usability](#2-the-five-components-of-usability)
3. [Why usability matters](#3-why-usability-matters)
4. [Human factors: how people actually perceive and think](#4-human-factors-how-people-actually-perceive-and-think)
5. [Norman's design principles](#5-normans-design-principles)
6. [Nielsen's 10 usability heuristics](#6-nielsens-10-usability-heuristics)
7. [Laws of UX (rules of thumb)](#7-laws-of-ux-rules-of-thumb)
8. [Practical patterns](#8-practical-patterns)
9. [Measuring usability](#9-measuring-usability)
10. [Evaluation methods](#10-evaluation-methods)
11. [Running a usability test](#11-running-a-usability-test)
12. [Usability in the design process](#12-usability-in-the-design-process)
13. [How backend / system design affects usability](#13-how-backend--system-design-affects-usability)
14. [Approaches by product type](#14-approaches-by-product-type)
15. [Trade-offs](#15-trade-offs)
16. [Anti-patterns and dark patterns](#16-anti-patterns-and-dark-patterns)
17. [Checklist](#17-checklist)
18. [Exercises](#18-exercises)
19. [Further reading](#19-further-reading)

---

## 1. Definitions: usability vs UX vs UI

| Term | Scope | Question it answers |
|---|---|---|
| **UI** (User Interface) | The surface: screens, controls, layout, typography, color | "What does it look like and what can I touch?" |
| **Usability** | Quality of the interaction for a task | "Can I get my task done easily and correctly?" |
| **UX** (User Experience) | The whole experience before, during, after use — incl. emotion, brand, support, onboarding, pricing, delivery | "How does it feel to be this product's user?" |
| **Accessibility** | Usability for people with disabilities / in constrained situations | "Can *everyone* use it?" |

- A beautiful UI can have terrible usability (pretty but confusing).
- Good usability is **necessary but not sufficient** for good UX (an easy-to-use app that solves the wrong problem is still bad UX).
- Usability is always relative to **who** (users), **what** (goals/tasks), and **where** (context: device, environment, time pressure, expertise). "Easy to use" without these is meaningless.

The ISO 9241-11 definition breaks down into three measurable parts:

| Part | Meaning | Example metric |
|---|---|---|
| **Effectiveness** | Accuracy and completeness of achieving goals | Task success rate, error rate |
| **Efficiency** | Resources spent (time, effort, clicks) to achieve goals | Time on task, number of steps |
| **Satisfaction** | Comfort and attitude toward use | SUS score, post-task rating |

---

## 2. The five components of usability

Jakob Nielsen's widely used breakdown:

| Component | Question | Detailed note |
|---|---|---|
| **Learnability** | How easy is it for first-time users to accomplish basic tasks? | [learnability](../learnability/README.md) |
| **Efficiency** | Once learned, how quickly can users perform tasks? | [efficiency-of-use](../efficiency-of-use/README.md) |
| **Memorability** | After a period away, how easily do users get proficient again? | [memorability](../memorability/README.md) |
| **Errors** | How many errors do users make, how severe, and how easily do they recover? | [user-error-protection](../user-error-protection/README.md) |
| **Satisfaction** | How pleasant is it to use? | [satisfaction](../satisfaction/README.md) |

Plus **utility** — does it do what users *need*? Nielsen: **Useful = Utility + Usability**. A perfectly usable feature nobody needs is waste.

These components often conflict — e.g. a keyboard-shortcut-heavy tool (Vim) is extremely *efficient* but has poor *learnability*. Know which matters most for your users (see [§14](#14-approaches-by-product-type)).

---

## 3. Why usability matters

- **Users don't read manuals and won't wait.** On the web, if a user can't figure it out quickly, they leave — the competitor is one tab away.
- **Conversion & revenue.** Checkout friction (forced account creation, confusing forms, surprise costs) is one of the top documented reasons for cart abandonment (Baymard Institute research).
- **Support cost.** Every confusing screen generates tickets, calls, and docs.
- **Errors cost real money / safety.** Bad UI in medical devices, aviation, and finance has caused deaths and multi-million losses (e.g. the 2018 Hawaii false missile alert was partly a UI issue: the real and test options were nearly identical items in a list).
- **Cost of change curve.** Fixing a usability problem found with a paper prototype costs minutes; fixing it after launch costs a redesign, migration, retraining, and lost users.
- **Productivity of internal tools.** A 10-second slowdown on a task done 200×/day by 500 employees is ~280 hours/day.

---

## 4. Human factors: how people actually perceive and think

Usability rests on how human perception, memory, and attention really work — not how we wish they did.

### 4.1 Mental models

A **mental model** is what the user *believes* about how the system works. The **conceptual model** is how the designer intended it to work. The **implementation model** is how it actually works.

- Usability problems happen when the UI exposes the *implementation model* instead of matching the user's *mental model*.
  - Bad: "Error: FK constraint violation on orders.customer_id"
  - Good: "This customer has orders, so it can't be deleted. Archive it instead?"
- Users form mental models from **other products they use** (see Jakob's Law, [§7](#7-laws-of-ux-rules-of-thumb)). Match conventions.

### 4.2 Cognitive load

Working memory is tiny — modern research suggests **about 4 ± 1 chunks** (Miller's famous "7 ± 2" is often misquoted and overstated).

Three types of load:
- **Intrinsic** — complexity of the task itself (doing taxes is hard).
- **Extraneous** — complexity added by the UI (clutter, jargon, inconsistent layout). **This is what designers must minimize.**
- **Germane** — effort spent actually learning/building understanding (good effort).

Reduce extraneous load by: chunking information, showing only what's needed (progressive disclosure), using defaults, recognition over recall, and consistent patterns.

### 4.3 Recognition over recall

Recognizing something is far easier than remembering it from scratch.
- Menus > command lines (for novices).
- Show recent items, autocomplete, previews, visible options.
- Don't make users remember info from screen 1 to use on screen 3.

### 4.4 Attention and scanning

- People **scan, they don't read** — especially on the web. Eye-tracking shows **F-shaped** and **layer-cake** scanning patterns on text-heavy pages.
- **Banner blindness**: users ignore things that look like ads, even important content.
- **Change blindness**: users miss changes outside their focus area (e.g. an error message appearing at the top of a long form when the user is looking at the button at the bottom).
- **Inattentional blindness**: if users are focused on a task, they won't notice unrelated things.

### 4.5 Gestalt principles (visual grouping)

How the brain groups visual elements — use them to communicate structure *without* adding lines and boxes.

| Principle | Meaning | UI use |
|---|---|---|
| **Proximity** | Things close together are related | Label sits close to its field, far from the next field |
| **Similarity** | Things that look alike are related | All links the same color; all destructive buttons red |
| **Common region** | Things in the same container are related | Cards, grouped settings panels |
| **Continuity** | Eyes follow lines and curves | Aligned layouts, step indicators |
| **Closure** | The brain completes incomplete shapes | Icons, partially visible carousel item → "there's more" |
| **Figure–ground** | We separate foreground from background | Modals with dimmed backdrop |
| **Common fate** | Things moving together are related | Items animating together after a filter |

### 4.6 Response time limits

Classic limits (Miller 1968, Card et al. 1991, Nielsen 1993) — still valid:

| Delay | Perception | What to show |
|---|---|---|
| **~0.1 s** | Instant; user feels they directly caused it | Nothing needed |
| **~1 s** | Noticeable, but flow of thought is kept | Subtle indicator is OK |
| **~10 s** | Limit of keeping attention on the task | Progress indicator, ideally with % / estimate |
| **> 10 s** | User switches tasks | Progress + allow background work + notify on completion |

The **Doherty threshold** (~400 ms): productivity rises sharply when system response is under ~400 ms. See [Responsiveness](../responsiveness/README.md) and [Performance](../../runtime-operational/performance/README.md).

### 4.7 Motor control

- **Fitts's Law**: time to hit a target depends on **distance** and **size**. Big, close targets are fast. Screen edges and corners are "infinitely large" targets on desktop (that's why macOS menu bar is at the top edge).
- **Touch**: fingers are imprecise. Minimum touch targets: Apple HIG **44×44 pt**, Material **48×48 dp**, WCAG 2.2 AA minimum **24×24 CSS px** (AAA: 44×44).
- **Thumb zones**: on large phones, the top corners are hard to reach one-handed → primary actions go to the bottom (bottom nav bars, bottom sheets).

---

## 5. Norman's design principles

From Don Norman's *The Design of Everyday Things* — the foundation of interaction design.

| Principle | Meaning | Good example | Bad example |
|---|---|---|---|
| **Affordance** | What actions an object *allows* | A button can be pressed | — |
| **Signifier** | A perceivable cue showing *where/how* to act | Raised button with shadow, underlined link | Flat text that's secretly clickable; "Norman doors" (pull handle on a push door) |
| **Mapping** | Relationship between controls and their effects | Stove knobs laid out like the burners; scroll down = content moves up | Volume slider where left = louder |
| **Feedback** | Immediate, clear result of an action | Button pressed state, "Saved ✓", progress bar | Click → nothing visibly happens → user clicks 5 more times |
| **Constraints** | Limit possible actions to prevent errors | Date picker instead of free text; disabled "Next" until valid | Free-text field for a phone number with no format hint |
| **Conceptual model** | A simple, coherent explanation of how it works | Files & folders metaphor | Exposing DB tables as the UI |
| **Consistency** | Same thing works the same way everywhere | Same "Save" location on every form | Save top-right here, bottom-left there |

### The two gulfs

- **Gulf of execution** — "How do I do what I want?" (Can the user figure out *what* actions are possible and *how* to perform them?) → bridged by signifiers, constraints, mapping.
- **Gulf of evaluation** — "Did it work? What state is the system in?" → bridged by feedback and a clear conceptual model.

Every usability problem can be classified as widening one of these two gulfs.

---

## 6. Nielsen's 10 usability heuristics

The most widely used checklist for evaluating interfaces (Nielsen, 1994; still current).

### 6.1 Visibility of system status
Always keep users informed about what's going on, through appropriate feedback within reasonable time.
- Loading indicators, progress bars, "Saving…" → "Saved", upload percentages, "You are here" in navigation, unread counts.
- ❌ Clicking "Submit" with no visible change for 3 seconds.

### 6.2 Match between the system and the real world
Speak the user's language — words, phrases, concepts familiar to them, not system-oriented jargon. Follow real-world conventions.
- "Trash" / "Shopping cart" metaphors; dates in local format; "Send money to Alice" not "Create transfer entity".
- ❌ "Error 0x80070005", "Invalid payload", "NullPointerException".

### 6.3 User control and freedom
Users often act by mistake and need a clearly marked "emergency exit" — **undo, redo, cancel, back**.
- Gmail's "Undo send"; Ctrl+Z; cancel buttons on long operations; back button works as expected.
- ❌ Multi-step wizard where Back loses all entered data.

### 6.4 Consistency and standards
Users shouldn't wonder whether different words, situations, or actions mean the same thing.
- **Internal consistency** (within your product) and **external consistency** (platform conventions — iOS, Android, web).
- See [Consistency](../consistency/README.md).
- ❌ "Delete", "Remove", "Discard", "Trash" used interchangeably for the same action.

### 6.5 Error prevention
Better than good error messages is a design that prevents the problem in the first place.
- **Slips** (right intention, wrong action — e.g. a mis-click) → constraints, spacing, undo.
- **Mistakes** (wrong intention due to a wrong mental model) → clear labels, previews, good defaults.
- Inline validation, disabling invalid options, confirmation for destructive actions, "Are you sure you want to delete 1,204 files?".
- See [User error protection](../user-error-protection/README.md).

### 6.6 Recognition rather than recall
Minimize memory load by making elements, actions, and options visible.
- Recently used files, autocomplete, visible menu options, showing the selected filter.

### 6.7 Flexibility and efficiency of use
Accelerators — unseen by novices — speed up interaction for experts.
- Keyboard shortcuts, command palettes (Ctrl+K / Cmd+K), bulk actions, saved filters, templates, customizable toolbars.
- See [Efficiency of use](../efficiency-of-use/README.md).

### 6.8 Aesthetic and minimalist design
Every extra unit of information competes with the relevant units and lowers their visibility.
- Not "make it pretty" — it's about **signal-to-noise**. Remove or hide what isn't needed for the current task.
- See [Aesthetics](../aesthetics/README.md).

### 6.9 Help users recognize, diagnose, and recover from errors
Error messages should be in plain language, precisely indicate the problem, and constructively suggest a solution.
- Good error message = **what happened** + **why** + **how to fix it**, shown **near the problem**.
- ❌ "Invalid input." ✅ "Password must be at least 12 characters — you entered 9."

### 6.10 Help and documentation
Best if the system needs none, but help may be necessary. Make it searchable, task-focused, concise, and contextual.
- Tooltips, inline hints, empty-state guidance, onboarding checklists, contextual "?" links.

---

## 7. Laws of UX (rules of thumb)

Collected at lawsofux.com. Useful vocabulary, but they're heuristics, not physics.

| Law | Statement | Implication |
|---|---|---|
| **Jakob's Law** | Users spend most of their time on *other* sites, so they expect yours to work the same way | Follow conventions (logo top-left → home, cart top-right). Innovate only where it adds real value |
| **Hick's Law** | Decision time increases with the number and complexity of choices | Fewer options; group and categorize; highlight a recommended option |
| **Fitts's Law** | Time to acquire a target depends on distance and size | Make primary actions large and close to where the user's attention/cursor is |
| **Miller's Law** | Working memory holds a small number of chunks | Chunk info (phone numbers 090-1234-5678, card numbers in groups of 4) |
| **Tesler's Law** (conservation of complexity) | Every system has irreducible complexity — someone must deal with it | Move complexity from the user to the system (e.g. auto-detect card type instead of asking) |
| **Postel's Law** | Be liberal in what you accept, conservative in what you send | Accept "0901234567", "090-123-4567", "+84 90 123 4567" — normalize it yourself |
| **Doherty Threshold** | Productivity soars when response < ~400 ms | Optimize perceived performance, optimistic UI |
| **Peak–End Rule** | People judge an experience by its peak moment and its end | Invest in key moments and the finish (success screens, smooth checkout end) |
| **Serial Position Effect** | First and last items in a list are remembered best | Put key nav items at the start and end |
| **Von Restorff Effect** | The item that differs is remembered | Make the primary CTA visually distinct — but only one per screen |
| **Zeigarnik Effect** | Incomplete tasks are remembered better | Progress bars, "profile 70% complete" |
| **Goal-Gradient Effect** | Motivation increases as people get closer to a goal | Show progress in multi-step flows |
| **Aesthetic–Usability Effect** | Attractive designs are *perceived* as more usable | Visual polish helps — but it can mask real problems in testing |
| **Pareto Principle** | ~80% of effects come from ~20% of causes | Optimize the few most-used flows first |
| **Parkinson's Law** | Tasks expand to fill the time available | Autofill and smart defaults shorten tasks |

---

## 8. Practical patterns

### 8.1 Visual hierarchy

Users should be able to tell what's most important within ~1 second.
- Tools: **size, weight, color/contrast, position, whitespace**.
- **One primary action per screen** (filled button); secondary actions are outlined/text; destructive actions are visually distinct and *not* the default.
- Squint test: blur your eyes at the screen — the most important element should still stand out.

### 8.2 Navigation & information architecture (IA)

- Users need to answer: **Where am I? Where can I go? How do I get back?**
- Use labels from **user vocabulary** (validate with card sorting / tree testing, [§10](#10-evaluation-methods)).
- Prefer **broad and shallow** over deep hierarchies (but not 40 items in one menu).
- Highlight current location; breadcrumbs for deep hierarchies.
- Search for large content sets — and make search forgiving (typos, synonyms).
- Deep links: every meaningful state should have a URL (shareable, bookmarkable, back-button-safe).

### 8.3 Forms (where most usability problems live)

- **Ask less.** Every field is a cost. Remove, defer, or infer (city from postal code, card type from number).
- **Labels above fields**, always visible — placeholders are not labels (they disappear when typing).
- **One column** layout; group related fields.
- Use the **right input type** (`type="email"`, `inputmode="numeric"`, date pickers) → right mobile keyboard.
- Support **autofill** (`autocomplete="email"`, `"cc-number"`, …) and **paste** (never block paste in password fields).
- **Inline validation** after the user leaves a field (on blur), not on every keystroke while they're still typing.
- Mark **optional** fields (or required ones — be consistent); explain format *before* the user makes an error.
- On submit error: keep all entered data, scroll/focus the first error, show a summary for long forms.
- Primary button text describes the result: "Create account", "Pay $42.00" — not "Submit".

### 8.4 Feedback and UI states

Every screen/component has more states than the "happy path" mockup. Design all of them:

| State | What to show |
|---|---|
| **Empty** (no data yet) | Explain what goes here + a primary action ("No projects yet — Create your first project") |
| **Loading** | Skeleton screens for content; spinners for short actions; progress for long ones |
| **Partial** | Some data loaded, some failed — show what you have, mark what's missing |
| **Error** | What happened + how to recover (Retry button), without losing user input |
| **Success** | Confirm the result ("Invoice sent to alice@example.com") |
| **Ideal** | The normal populated state |
| **Too much data** | Pagination, virtualization, truncation with "show more", filters |
| **Offline / stale** | Indicate staleness; queue actions for later |
| **No permission** | Explain why and how to get access — don't just hide or show a 403 |

### 8.5 Destructive actions: confirm vs undo

| Approach | When |
|---|---|
| **Undo** (do it immediately, offer "Undo" for a few seconds) | Frequent, reversible actions (archive email, delete a list item). Faster and less annoying |
| **Confirmation dialog** | Rare, irreversible, high-impact actions. Make the dialog specific: "Delete project 'Apollo' and its 312 files?" with a button labeled "Delete project", not "OK" |
| **Type-to-confirm** | Catastrophic actions (delete a production database, GitHub repo deletion) |
| **Soft delete / trash** | Most user data — recoverable for N days |

Overusing confirmations trains users to click "OK" without reading — they stop protecting anyone.

### 8.6 Defaults

- Most users never change defaults → **defaults are design decisions with huge impact**.
- Choose the safest and most common option; pre-fill what you know (country from locale, last-used values).
- Privacy/security defaults should be protective (opt-in for sharing, not opt-out).

### 8.7 Progressive disclosure

Show the basics first; reveal advanced options on demand ("Advanced settings", expandable sections, "More filters").
- Serves both novices (learnability) and experts (power is still available).
- Danger: hiding something users *frequently* need. Base it on usage data.

### 8.8 Content and microcopy

- Words are UI. Most usability problems are partly **wording** problems.
- Front-load key words in headings and links ("Billing settings", not "Settings for your billing").
- Link text describes the destination — never "click here".
- Use the same verb throughout a flow: button "Publish post" → toast "Post published".
- Write for scanning: short sentences, bullet points, meaningful headings.

---

## 9. Measuring usability

"Easy to use" must become numbers, otherwise it can't be tracked or compared.

### 9.1 Behavioral (performance) metrics

| Metric | How | Notes |
|---|---|---|
| **Task success rate** | % of participants who complete the task correctly | The single most important metric. Can be binary or partial credit |
| **Time on task** | Time from start to completion | Compare against an expert baseline; use geometric mean (times are skewed) |
| **Error rate** | Number of errors per task / per user | Classify by severity |
| **Efficiency** | Steps, clicks, or screens to complete vs optimal path | "Lostness" metric for navigation |
| **Learnability curve** | Time on task across repeated trials | How fast it flattens = how learnable |

### 9.2 Attitudinal (self-reported) metrics

**SUS — System Usability Scale** (Brooke, 1986): the de-facto standard questionnaire. 10 statements, each rated 1 (strongly disagree) to 5 (strongly agree):

1. I think that I would like to use this system frequently.
2. I found the system unnecessarily complex.
3. I thought the system was easy to use.
4. I think that I would need the support of a technical person to be able to use this system.
5. I found the various functions in this system were well integrated.
6. I thought there was too much inconsistency in this system.
7. I would imagine that most people would learn to use this system very quickly.
8. I found the system very cumbersome to use.
9. I felt very confident using the system.
10. I needed to learn a lot of things before I could get going with this system.

**Scoring:**
```
odd items  (1,3,5,7,9):  contribution = score − 1
even items (2,4,6,8,10): contribution = 5 − score
SUS = (sum of contributions) × 2.5          → range 0–100
```

| SUS score | Interpretation (approx.) |
|---|---|
| < 51 | Poor (F) |
| ~68 | **Average** (50th percentile) |
| > 80 | Good (top ~10–15%, A-) |
| > 85 | Excellent |

SUS is **not a percentage** — 68 is average, not "68% usable".

Other questionnaires:
- **SEQ** (Single Ease Question): after each task, "Overall, how difficult or easy was this task?" 1–7. Cheap and surprisingly reliable.
- **UMUX-Lite**: 2-item short version correlating well with SUS.
- **NPS** (Net Promoter Score): "How likely are you to recommend…?" — a loyalty metric, *not* a usability metric. Often misused for usability.
- **CSAT**: satisfaction with a specific interaction.

### 9.3 Product analytics (in production)

- **Funnel conversion** & drop-off per step (where do users give up?).
- **Rage clicks**, dead clicks, repeated back-navigation, form field abandonment, error message frequency.
- **Search with no results**; search refinements (a sign navigation failed).
- **Support tickets** tagged by feature.
- Session recordings / heatmaps (Hotjar, FullStory, PostHog, Microsoft Clarity) — respect privacy and mask PII.

### 9.4 Google's HEART framework

For defining UX metrics at product scale:

| | Category | Example signal |
|---|---|---|
| **H** | Happiness | Satisfaction surveys, SUS |
| **E** | Engagement | Sessions/week, actions per session |
| **A** | Adoption | New users of a feature |
| **R** | Retention | % returning after 7/30 days |
| **T** | Task success | Completion rate, time, errors |

Use with **Goals → Signals → Metrics**: define the goal first, then what behavior indicates it, then how to measure it.

**Quantitative tells you *what* and *where*. Qualitative (watching users) tells you *why*.** You need both.

---

## 10. Evaluation methods

| Method | Type | When | Users needed | Finds |
|---|---|---|---|---|
| **Heuristic evaluation** | Expert inspection | Any stage, fast & cheap | 0 (3–5 evaluators) | Violations of known principles |
| **Cognitive walkthrough** | Expert inspection | Early, for learnability | 0 | Where a new user would get stuck |
| **Moderated usability test** | Qualitative | Prototype → production | ~5 per round | *Why* users struggle |
| **Unmoderated remote test** | Qual/quant | When you need scale or speed | 10–50+ | Task success, time, recordings |
| **Guerrilla testing** | Qualitative | Very early, low budget | 3–5 strangers (cafés, colleagues from other teams) | Obvious problems |
| **Card sorting** | Generative | Designing IA/navigation | 15–30 | How users group and name things |
| **Tree testing** | Evaluative | Validating IA | 50+ | Whether users can find items in your menu structure |
| **First-click testing** | Evaluative | Validating layouts | 30+ | Whether the first click is right (strongly predicts success) |
| **5-second test** | Evaluative | Landing pages | 20+ | What people remember / understand at a glance |
| **A/B testing** | Quantitative | Production, with traffic | Thousands | Which variant performs better (not *why*) |
| **Surveys** | Attitudinal | Production | Many | Satisfaction, priorities |
| **Analytics review** | Quantitative | Production | All users | Where problems are |
| **Diary studies / field studies** | Generative | Understanding context | 5–15 | Real-world usage over time |
| **Accessibility audit** | Expert + tools | Always | — | WCAG issues (see [Accessibility](../accessibility/README.md)) |

### Why "5 users"?

Nielsen & Landauer (1993): with ~5 users you find ~85% of the usability problems *that are common* in that user group, for that task set. Caveats:
- It's **per round, per user segment** — test 5, fix, test 5 again. Iterating beats one big study.
- Rare problems and quantitative metrics need **many more** users (20+ for metrics with reasonable confidence intervals).
- If you have very different user types (buyers vs sellers), test ~3–5 of each.

---

## 11. Running a usability test

### 11.1 Steps

1. **Define goals** — what decisions will this test inform? ("Can new users create and share a project without help?")
2. **Pick participants** — representative of real users (not your teammates, unless guerrilla). Screen for the right segment.
3. **Write realistic tasks** — scenarios, not instructions.
4. **Pilot** the test with one person to fix broken tasks/prototype.
5. **Run sessions** (~30–60 min): intro → background questions → tasks with think-aloud → post-test questionnaire (SUS) → debrief.
6. **Observe, don't help.** Take notes on behavior, quotes, errors, hesitations.
7. **Synthesize** — affinity map observations, count how many participants hit each issue.
8. **Rate severity** and prioritize.
9. **Fix and re-test.**

### 11.2 Writing tasks

| ❌ Bad (leading / instructional) | ✅ Good (scenario-based) |
|---|---|
| "Click on Settings, then Notifications, and turn off email alerts." | "You're getting too many emails from this app. Make it stop." |
| "Use the search to find a red shirt." | "You need a red shirt for a party this weekend, budget $30. Find one you'd buy." |
| "Test the export feature." | "Your manager wants last month's sales in a spreadsheet. Get it to them." |

Don't use the words that appear in the UI — otherwise you're testing word-matching, not understanding.

### 11.3 Think-aloud & moderation tips

- Ask participants to say what they're thinking, looking for, expecting.
- Remind them: **"We're testing the design, not you. You can't do anything wrong."**
- When asked "What should I do?", reflect it back: "What would you do if I weren't here?"
- Neutral prompts only: "What are you thinking?", "What did you expect to happen?" — never "Did you see the button at the top?"
- **Watch what they do, not what they say.** People say "it's easy" right after failing a task. Behavior > opinion.

### 11.4 Severity rating (Nielsen)

| Rating | Meaning |
|---|---|
| 0 | Not a usability problem |
| 1 | Cosmetic — fix if time permits |
| 2 | Minor — low priority |
| 3 | Major — important to fix, high priority |
| 4 | Catastrophe — must fix before release |

Severity combines **frequency** (how many users hit it), **impact** (how hard to overcome), and **persistence** (one-time vs every time).

---

## 12. Usability in the design process

### 12.1 User-centered design (ISO 9241-210)

An iterative loop:
```
Understand context of use → Specify user requirements → Produce design solutions → Evaluate against requirements
        ↑______________________________________________________________________________________|
```

### 12.2 Double Diamond (UK Design Council)

```
   Discover      Define        Develop       Deliver
    /\            /\            /\            /\
   /  \          /  \          /  \          /  \
  /    \  ──▶   /    \  ──▶   /    \  ──▶   /    \
  diverge   converge      diverge   converge
  (research) (problem)    (ideas)   (solution)
  ───── right problem ─────  ──── problem solved right ────
```

### 12.3 Where usability work fits

| Stage | Usability activities |
|---|---|
| Research | Interviews, contextual inquiry, analytics, competitor usability review |
| Define | Personas, user journeys, Jobs-to-be-Done, **usability goals** ("90% of new users complete signup in < 2 min, SEQ ≥ 5.5") |
| Design | Sketches, wireframes, IA (card sort), design system usage, heuristics as you go |
| Prototype | Paper/Figma prototypes → moderated tests, tree tests, first-click tests |
| Build | Design QA, accessibility checks, test all states (empty/error/loading) |
| Launch | Beta testing, analytics instrumentation, funnels |
| Operate | Monitor metrics, A/B tests, support-ticket analysis, periodic SUS benchmarks |

Key ideas:
- **Test early with low fidelity.** Paper prototypes get honest feedback; polished designs get "it looks nice" and resistance to change (aesthetic–usability effect).
- **Usability goals should be written like SLOs** — specific, measurable, tied to user tasks.
- **Design systems** (Material, Apple HIG, Fluent, your company's own) bake usability and consistency into reusable components.

---

## 13. How backend / system design affects usability

Usability isn't only the frontend's job. Many UX problems come from architecture decisions.

| System design decision | Usability impact | Approach |
|---|---|---|
| **Latency** | Slow = feels broken; users double-click, abandon | Budget latency per interaction (< 100 ms for direct manipulation); see [Performance](../../runtime-operational/performance/README.md) |
| **Non-idempotent APIs** | Double-submit → double charge / duplicate posts | Idempotency keys; disable button after click *and* dedupe on server |
| **Synchronous long operations** | UI freezes, timeouts, lost work | Async jobs + progress API + notification on completion |
| **Optimistic UI** | Feels instant | Requires API that can reconcile/rollback; show failures clearly |
| **Eventual consistency** | "I saved it but it's not in the list!" → user re-creates it | Read-your-writes consistency for the acting user; local cache insertion |
| **Error responses** | Raw `500 Internal Server Error` shown to users | Structured errors (code + user-safe message + field), e.g. RFC 9457 Problem Details |
| **Validation only on server** | User finds errors only after submit | Share validation rules client & server; field-level error responses |
| **No autosave / drafts** | Lost work on crash, timeout, or session expiry | Periodic draft persistence; preserve form state across re-login |
| **Session expiry** | Losing a long form because the token expired | Silent refresh; warn before expiry; re-auth without losing state |
| **Hard deletes** | No undo possible | Soft delete + trash + restore endpoint |
| **Pagination design** | Offset pagination → duplicates/skips while data changes | Cursor-based pagination; stable sort |
| **Search quality** | Users can't find what exists | Typo tolerance, synonyms, relevance tuning (Elasticsearch/OpenSearch/Typesense…) |
| **Offline / poor network** | App useless on the subway | Local-first storage, sync queue, conflict resolution |
| **Rate limits** | Silent failures | Return `429` with `Retry-After`; UI explains and retries |
| **Availability / degradation** | Whole page fails because one widget failed | Graceful degradation; render what's available; see [Availability](../../runtime-operational/availability/README.md) |
| **Data model mismatches mental model** | UI forced to expose internal concepts | Model the domain in user terms (DDD's ubiquitous language helps) |

Rule: **the API shape often becomes the UI shape.** Design APIs around user tasks, not around database tables.

---

## 14. Approaches by product type

### 14.1 Consumer web / mobile apps
- **Learnability is king** — zero training, short attention, many alternatives.
- Follow platform conventions strictly (iOS HIG, Material). Onboarding should be minimal; teach in context.
- Measure with funnels, retention, A/B tests.

### 14.2 E-commerce
- Findability (search, filters, categories), product page clarity, frictionless checkout (guest checkout, autofill, wallet payments, no surprise costs).
- Trust signals: reviews, clear return policy, secure payment indicators.

### 14.3 Enterprise / B2B SaaS / internal tools
- **Efficiency and error prevention** over first-use learnability — people use it 8 hours a day.
- Dense information displays, keyboard shortcuts, bulk actions, saved views, tables with sort/filter/export.
- Buyers ≠ users: still test with the actual end-users.

### 14.4 Developer tools / CLIs / APIs
- Usability matters here too ("Developer Experience", DX).
- CLIs: `--help` everywhere, clear error messages with suggested fixes ("Did you mean `git status`?"), sane defaults, `--dry-run`, consistent flags. See clig.dev.
- APIs: consistent naming, predictable errors, good docs with runnable examples, quick time-to-first-successful-call.

### 14.5 Safety-critical (medical, aviation, industrial, automotive)
- **Error prevention dominates.** Strong regulation (IEC 62366 for medical device usability, FDA human factors guidance).
- Formal use-error analysis, summative usability validation, alarm fatigue management.
- Distinct, hard-to-confuse controls for dangerous actions.

### 14.6 Games
- Intentional difficulty is fine in *gameplay* — but menus, controls, and onboarding must be highly usable.
- Playtesting is the usability test.

### 14.7 Content / media / news
- Readability (typography, line length ~50–75 characters, contrast), scannability, fast loading, unobtrusive ads.

### 14.8 Kiosks / public terminals / TVs
- Walk-up-and-use: no learning curve allowed. Large targets, very few options per screen, timeouts that reset state.
- TV (10-foot UI): D-pad navigation, clear focus states, large text.

### 14.9 Voice and conversational UIs / AI assistants
- No visible affordances → discoverability is the hard problem. Tell users what they can say.
- Confirm understanding for consequential actions; allow correction ("No, I meant…").
- AI features: show sources/uncertainty, let users edit and undo AI actions, keep the human in control for consequential operations.

### Summary table

| Product type | Top usability priority | Key techniques |
|---|---|---|
| Consumer app | Learnability, satisfaction | Conventions, onboarding-in-context, A/B tests |
| E-commerce | Findability, low-friction checkout | Search/filters, guest checkout, autofill |
| Enterprise / internal | Efficiency, error prevention | Shortcuts, bulk actions, dense views |
| Dev tools / APIs | Error clarity, consistency | Good errors, docs, defaults, `--help` |
| Safety-critical | Error prevention | Formal human-factors process, validation testing |
| Kiosk / TV | Walk-up usability | Huge targets, minimal choices, focus states |
| Voice / AI | Discoverability, control | Explicit prompts, confirmations, undo |

---

## 15. Trade-offs

| Tension | Description |
|---|---|
| **Learnability ↔ Efficiency** | Simple guided UI for novices slows experts; dense shortcut-heavy UI scares novices. → Progressive disclosure, accelerators (§6.7) |
| **Simplicity ↔ Power / flexibility** | Every option added makes the common case harder. → Good defaults, advanced sections |
| **Error prevention ↔ Speed** | Confirmations and constraints slow users down. → Prefer undo for reversible actions |
| **Security ↔ Usability** | MFA, short sessions, complex passwords add friction. → Passkeys, risk-based auth, remember trusted devices |
| **Consistency ↔ Optimal local design** | The best design for one screen may break the pattern used elsewhere. → Usually choose consistency |
| **Platform conventions ↔ Brand uniqueness** | Custom UI controls look distinctive but users don't recognize them. → Innovate in content, not in basic controls |
| **Business goals ↔ User goals** | Upsells, data collection, retention tricks add friction. → Watch out for dark patterns (§16) |
| **Minimalism ↔ Discoverability** | Hiding everything behind icons/gestures looks clean but users can't find features |
| **Personalization ↔ Predictability** | Adaptive menus that reorder themselves break muscle memory |
| **Aesthetics ↔ Readability** | Low-contrast light-gray text looks elegant but is hard to read (and fails WCAG) |

---

## 16. Anti-patterns and dark patterns

### 16.1 Usability anti-patterns

- Designing for yourself ("I find it obvious") — **you are not your user**.
- Asking users what they want instead of observing what they do.
- Testing only the happy path; no empty/error/loading states.
- Placeholder text used as labels.
- Mystery-meat navigation: unlabeled icons with no tooltip.
- Clickable things that don't look clickable; non-clickable things that do.
- Error messages that blame the user or show stack traces.
- Clearing the form after a validation error.
- Disabled buttons with no explanation of *why* they're disabled.
- Confirmation dialogs for everything (users stop reading them).
- Modal on top of modal; pop-ups on page load.
- Infinite scroll on pages where users need the footer or need to return to a position.
- Carousels that auto-rotate (users rarely see slides beyond the first, and moving content hurts accessibility).
- Breaking the browser Back button.
- Inconsistent terminology across screens.
- Tiny touch targets packed close together.
- Doing one big usability study at the end instead of small studies throughout.

### 16.2 Dark patterns (deceptive design)

Designs that *intentionally* trick users against their own interest. Increasingly **illegal** (EU Digital Services Act, GDPR consent rules, US FTC enforcement, California CPRA).

| Pattern | Example |
|---|---|
| **Roach motel** | Easy to subscribe online; cancellation requires a phone call |
| **Confirmshaming** | "No thanks, I don't like saving money" |
| **Sneak into basket** | Insurance pre-added to cart |
| **Hidden costs** | Fees appear only at the final checkout step |
| **Forced continuity** | Free trial silently converts to paid without reminder |
| **Disguised ads** | Ads styled as content or download buttons |
| **Trick questions** | Double negatives in opt-out checkboxes |
| **Privacy zuckering** | Default settings that over-share data |
| **Fake urgency / scarcity** | Fake countdown timers, "Only 2 left!" when untrue |
| **Nagging** | Repeatedly asking the same permission after the user said no |

Good usability makes the **user's** goal easy. Dark patterns make the **business's** goal easy at the user's expense — they may raise short-term metrics but damage trust, retention, and legal standing.

---

## 17. Checklist

**Define**
- [ ] Target users, key tasks, and context of use are written down
- [ ] Measurable usability goals exist for top tasks (success rate, time, SEQ/SUS)
- [ ] The top ~20% most-used flows are identified and prioritized

**Design**
- [ ] UI language matches user vocabulary (no internal jargon / DB terms)
- [ ] Follows platform conventions and the design system
- [ ] One clear primary action per screen; clear visual hierarchy
- [ ] Navigation answers: where am I, where can I go, how do I go back
- [ ] All states designed: empty, loading, partial, error, success, no permission, offline
- [ ] Forms: visible labels, correct input types, autofill, inline validation, data kept on error
- [ ] Destructive actions: undo or specific confirmation; soft delete where possible
- [ ] Sensible, safe defaults
- [ ] Error messages: what happened + why + how to fix, near the problem
- [ ] Accelerators for frequent users (shortcuts, bulk actions)
- [ ] Touch targets ≥ 44pt/48dp (≥ 24px minimum); enough spacing
- [ ] Accessibility basics: contrast, keyboard navigation, focus states, screen reader labels

**System / engineering**
- [ ] Interaction latency budgets defined; feedback within 100 ms
- [ ] Idempotent submits; no double charges / duplicates
- [ ] Long operations are async with progress
- [ ] Read-your-own-writes for the acting user
- [ ] Structured, user-safe API errors with field-level detail
- [ ] Autosave / drafts for long inputs; no data loss on session expiry
- [ ] Graceful degradation when a dependency fails

**Evaluate**
- [ ] Heuristic evaluation done on new designs
- [ ] Usability tested with ~5 representative users per round, iterated
- [ ] IA validated (card sort / tree test) for navigation-heavy products
- [ ] Analytics: funnels, drop-offs, error frequency, no-result searches
- [ ] Periodic SUS benchmark to track trends
- [ ] No dark patterns

---

## 18. Exercises

1. **Heuristic evaluation.** Pick an app you use daily (bank app, food delivery, your company's internal tool). Walk through one key task and list every violation of Nielsen's 10 heuristics. Rate each 0–4 severity.
2. **Norman doors hunt.** Find 5 real-world or digital examples of bad signifiers or mappings. For each, say which gulf (execution or evaluation) it widens and propose a fix.
3. **Error message rewrite.** Find 5 bad error messages (search your own codebase for `"Invalid"`, `"Error"`, `"failed"`). Rewrite each as *what happened + why + how to fix*.
4. **State inventory.** Pick one screen from a project you've built. Sketch every state from [§8.4](#84-feedback-and-ui-states). How many did the original design handle?
5. **Run a mini usability test.** Write 3 scenario-based tasks for a product. Test with 3 people (friends who've never used it). Don't help. Note where each struggled. Have each fill in the SUS and compute the score.
6. **Form audit.** Take a sign-up or checkout form and count the fields. Which can be removed, inferred, or deferred? Check labels, input types, `autocomplete` attributes, and error behavior.
7. **System-design tie-in.** For a "Place order" button, list every backend design decision needed so that the user never gets double-charged, always knows the order status, and never loses their cart.

---

## 19. Further reading

- *The Design of Everyday Things* (Don Norman) — affordances, signifiers, mapping, feedback, the two gulfs. **Start here.**
- *Don't Make Me Think, Revisited* (Steve Krug) — short, practical web usability.
- *Rocket Surgery Made Easy* (Steve Krug) — how to run DIY usability tests.
- *Usability Engineering* (Jakob Nielsen) — the classic foundation.
- Nielsen Norman Group articles (nngroup.com) — "10 Usability Heuristics", "Why You Only Need to Test with 5 Users", "Response Times: The 3 Important Limits", form and error message guidelines.
- *Laws of UX* (Jon Yablonski) and lawsofux.com.
- *Measuring the User Experience* (Tullis & Albert) — metrics, SUS, statistics for UX.
- *Quantifying the User Experience* (Sauro & Lewis) — statistics for usability; measuringu.com for SUS benchmarks.
- *Form Design Patterns* (Adam Silver) and *Web Form Design* (Luke Wroblewski).
- *About Face* (Cooper et al.) — goal-directed interaction design.
- Apple Human Interface Guidelines; Google Material Design; Microsoft Fluent.
- Baymard Institute — large-scale e-commerce usability research.
- deceptive.design — catalogue of dark patterns.
- clig.dev — Command Line Interface Guidelines.
- Standards: ISO 9241-11 (usability definitions), ISO 9241-110 (interaction principles), ISO 9241-210 (human-centred design), ISO/IEC 25010:2023 (interaction capability), IEC 62366 (medical device usability).
