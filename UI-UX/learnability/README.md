# Learnability

> **Learnability** = how easily new users can accomplish basic tasks the first time they use a product, and how quickly they become proficient.
> "Can someone get value from this in the first few minutes, without training or a manual?"

Learnability is the first of Nielsen's five usability components ([Usability §2](../usability/README.md#2-the-five-components-of-usability)) and a sub-characteristic of usability in ISO/IEC 25010. For most consumer and self-serve products it decides whether users ever reach the point where other qualities matter.

---

## 1. Why it matters

- **First impressions decide adoption.** Users who don't succeed early often never come back. In self-serve and product-led SaaS, "activation" (reaching first value) is a key growth metric.
- **Users don't read manuals.** The *paradox of the active user* (Carroll & Rosson, 1987): people start using software immediately to get their task done, skipping tutorials and documentation, even when learning first would save them time. Design for people who are learning *while doing*.
- **Training and support cost**: enterprise software with poor learnability needs courses, certifications, and large support teams.
- **Changing users**: new employees, occasional users, and users switching from a competitor are all "new users" again.

---

## 2. What makes a product learnable

| Principle | In practice |
|---|---|
| **Follow conventions** | Standard patterns, icons, gestures and terms (Jakob's Law). The less there is to learn, the faster learning is. See [Consistency](../consistency/README.md). |
| **Visible signifiers** | It's obvious what's clickable, draggable, editable. Flat, hidden or hover-only controls slow learning. |
| **Recognition over recall** | Show options and examples instead of requiring memorized commands ([Usability §4.3](../usability/README.md#43-recognition-over-recall)). |
| **Match the user's language and mental model** | Use users' words, not internal jargon or database terms. |
| **Progressive disclosure** | Show the essentials first; reveal advanced options when needed. Beginners aren't overwhelmed, experts can still find everything. |
| **Immediate feedback** | Every action shows its result, so users learn by trying. |
| **Safe exploration** | Undo, drafts, previews and sandboxes let users experiment without fear ([User error protection](../user-error-protection/README.md)). Fear of breaking things is a major barrier to learning. |
| **Good defaults and templates** | A working starting point teaches by example. |
| **Consistency** | One learned pattern transfers to the whole product. |

---

## 3. Onboarding patterns

| Pattern | Good for | Watch out for |
|---|---|---|
| **Empty states that teach** ("No projects yet. Create your first project" + a short explanation) | Almost every product | Blank screens with no guidance |
| **Sample data / templates** | Dashboards, design tools, project tools | Users confusing sample data with real data |
| **Contextual tips** shown at the moment a feature becomes relevant | Feature discovery | Too many interruptions |
| **Checklists** ("3 of 5 steps done") | Setup flows, activation | Long lists that feel like homework |
| **Interactive walkthroughs** (do a real task with guidance) | Complex products | Forced, non-skippable flows |
| **Product tours** (a sequence of tooltips pointing at UI parts) | Rarely the best choice | Most users skip or forget them; if the UI needs a tour to be understood, fix the UI first |
| **Progressive onboarding** (teach one thing at a time, over days) | Feature-rich products | Losing track of what the user already knows |
| **In-context help** ("?" links, inline hints, searchable help) | Rare or complex tasks | Help that's generic and far from the task |

Principles: **teach by doing, at the moment of need, with the user's real goal.** Let users skip and come back.

---

## 4. Designing for different user levels

Alan Cooper (*About Face*) points out that most users become **"perpetual intermediates"**: they move past beginner quickly but never become experts. Design mainly for intermediates, make beginners successful fast, and give experts accelerators ([Efficiency of use](../efficiency-of-use/README.md)).

**Learnability vs efficiency** is a classic trade-off: Vim and professional tools (Bloomberg terminal, CAD, DAWs) are very efficient but hard to learn; simple consumer apps are easy to learn but can feel slow to experts. Layered interfaces (simple by default, advanced on demand) reduce the conflict.

---

## 5. Measuring learnability

| Metric | How |
|---|---|
| **First-time task success rate** | Usability tests with new users, no help |
| **Time to first success / time to value** | Analytics: time from sign-up to first key action (first project, first invoice, first message) |
| **Activation rate** | % of new users who reach the "aha" action within N days |
| **Learning curve** | Repeat the same tasks several times; plot time per trial. A steep early drop means fast learning; a plateau shows proficiency. |
| **Help and support usage** by new users | Tickets, help-center searches, tutorial views |
| **Onboarding funnel drop-off** | Where new users abandon setup |

---

## 6. System-design aspects

- **Fast, reliable first experience**: sign-up, verification emails and initial data loading must be quick and robust; failures in the first session are disproportionately costly.
- **Sandbox / test modes** (e.g. payment test keys, demo workspaces) let users learn without real consequences.
- **Feature flags and progressive rollout** for onboarding experiments; track per-cohort activation.
- **Per-user state** for onboarding progress (dismissed tips, completed steps) synced across devices.
- **Good error messages and API errors** (developer products: clear docs, quickstarts, copy-paste examples, helpful error responses).

---

## 7. Checklist and further reading

- [ ] New users can complete the core task without help in usability tests
- [ ] Conventions followed; signifiers visible; jargon removed
- [ ] Empty states explain and offer the next action
- [ ] Onboarding is contextual, skippable and based on real tasks
- [ ] Safe exploration: undo, drafts, previews, sample data
- [ ] Advanced features progressively disclosed
- [ ] Activation and time-to-value tracked

**Further reading**: Nielsen Norman Group, *User Onboarding*, *Progressive Disclosure*, *Empty States*; John Carroll & Mary Beth Rosson, *Paradox of the Active User* (1987); Samuel Hulick, *The Elements of User Onboarding*; Alan Cooper et al., *About Face* (perpetual intermediates); Steve Krug, *Don't Make Me Think*.
