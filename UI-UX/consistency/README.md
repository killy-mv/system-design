# Consistency

> **Consistency** = the same things look the same, are called the same, and behave the same, everywhere in the product, and in line with the conventions of the platform and the wider world.
> "Once I've learned how one part works, can I trust that the rest works the same way?"

Consistency is one of Nielsen's 10 heuristics ([Usability §6.4](../usability/README.md#64-consistency-and-standards)) and one of the biggest drivers of [learnability](../learnability/README.md), [memorability](../memorability/README.md) and [user error protection](../user-error-protection/README.md). Every inconsistency forces users to stop and think, and creates opportunities for slips.

---

## 1. Kinds of consistency

| Kind | Meaning | Example of a violation |
|---|---|---|
| **Visual** | Same styles for the same roles: colors, typography, spacing, icons, components | Three different button styles for "primary action" across screens |
| **Functional / behavioral** | Same controls behave the same way | Clicking a row opens details on one page but selects it on another; Esc closes some dialogs but not others |
| **Terminology** | Same words for the same concepts, and different words for different concepts | "Delete", "Remove", "Discard" and "Trash" used interchangeably, or "Workspace" vs "Team" vs "Organization" for the same thing |
| **Internal** | Consistency within your own product (or product family) | The mobile app and web app put "Settings" in different places and name things differently |
| **External** | Consistency with platform and industry conventions | A custom scrollbar that scrolls backwards; a logo that doesn't link home; "Cancel" on the right on Windows dialogs |

**Jakob's Law**: users spend most of their time on *other* products, so they expect yours to work like the ones they already know. External consistency is usually worth more than originality ([Usability §7](../usability/README.md#7-laws-of-ux-rules-of-thumb)).

---

## 2. Why it matters

- **Faster learning**: learn a pattern once, reuse it everywhere.
- **Fewer errors**: predictable behavior prevents slips (e.g. destructive buttons always styled and placed the same way).
- **Trust and quality perception**: inconsistent UIs feel unfinished and unreliable.
- **Faster development**: teams reuse components instead of rebuilding them; design reviews focus on new problems.
- **Accessibility**: consistent navigation and identification are WCAG requirements (SC 3.2.3 Consistent navigation, 3.2.4 Consistent identification, 3.2.6 Consistent help).

---

## 3. How teams achieve consistency

### 3.1 Design systems

A **design system** is the single source of truth for how the product looks and behaves:

| Layer | Contents |
|---|---|
| **Design tokens** | Named values for color, typography, spacing, radius, shadow, motion (e.g. `color.action.primary`, `space.4`). One change propagates everywhere, including dark mode and brand themes. Tokens are usually stored in a platform-neutral format (JSON) and compiled to CSS variables, iOS and Android resources; a W3C community group is standardizing the format. |
| **Components** | Buttons, inputs, dialogs, tables, toasts, with all states (hover, focus, disabled, loading, error) and accessibility built in |
| **Patterns** | Solutions for recurring problems: forms, empty states, error handling, confirm vs undo, navigation, search, onboarding |
| **Content guidelines** | Voice and tone, terminology glossary, capitalization, error message formula, date/number formats |
| **Documentation and governance** | Usage guidance ("when to use a dialog vs a page"), contribution process, versioning, ownership |

Well-known public examples: Google **Material Design**, Apple **Human Interface Guidelines**, Microsoft **Fluent**, IBM **Carbon**, Shopify **Polaris**, Atlassian Design System, **GOV.UK Design System**.

### 3.2 Engineering support

- **Shared component libraries** (one implementation per platform), published with semantic versioning so teams can upgrade safely.
- **Lint rules** that forbid raw colors, ad-hoc spacing or one-off components (e.g. stylelint rules requiring tokens).
- **Visual regression tests** (screenshot diffs) and Storybook-style component catalogs.
- **Consistent backend contracts**: consistent API naming, error formats and status models make consistent UIs much easier (the same error object → the same error component everywhere).
- **Cross-channel consistency**: emails, notifications, PDFs and support tools use the same terms and tokens as the app.

### 3.3 Content consistency

Keep a **glossary** of product terms (and terms you deliberately *don't* use), a style guide for capitalization ("Sentence case for buttons"), and a standard formula for messages. Inconsistent words cause as much confusion as inconsistent visuals, and they multiply in translation ([Internationalization §4](../internationalization/README.md#4-messages-translation-and-plurals)).

---

## 4. When to break consistency

Consistency is a means, not the goal. "A foolish consistency is the hobgoblin of little minds."

- **Different things should look different.** Making a destructive action look like every other button is "consistent" but dangerous.
- **Platform conventions vs brand consistency**: follow iOS and Android conventions for navigation, gestures and system controls, even if the result differs from your web app. Users are more consistent with their *platform* than with your *brand*.
- **Better patterns**: an old pattern that's consistently bad should be replaced, ideally everywhere at once, with a migration plan, rather than kept forever for consistency's sake.
- **Innovation where it matters**: be conventional in 95% of the product and original where it creates real value.

---

## 5. Measuring consistency

- **UI audits / interface inventories**: screenshot every button, form, modal and table; count the variants.
- **Design system adoption**: % of screens or code using system components and tokens; number of one-off components.
- **Usability tests**: hesitation and errors at places where patterns differ.
- **Heuristic evaluation** against your own pattern library and platform guidelines.

---

## 6. Trade-offs and anti-patterns

| Issue | Notes |
|---|---|
| **Design system as bottleneck** | If contributing is slow, teams build one-offs. Make contribution easy and keep a small core team. |
| **Over-generic components** | Components with 40 props become hard to use consistently. Prefer composition and clear variants. |
| **Inconsistent by acquisition / team structure** | Products built by different teams or companies drift apart (Conway's Law). Shared systems and cross-team reviews counter it. |
| **Visual consistency without behavioral consistency** | Looks the same but works differently: worse than looking different. |
| **Redesigning one part only** | Half-migrated UIs are inconsistent for years. Plan migrations. |

**Checklist**
- [ ] Design tokens and a shared component library exist and are used
- [ ] Each common action has one pattern (one way to delete, confirm, search, filter, paginate)
- [ ] A terminology glossary exists and is enforced in reviews
- [ ] Platform conventions followed on each platform
- [ ] Navigation, help and key actions stay in the same place across screens
- [ ] Destructive and primary actions are consistently distinct
- [ ] Visual regression tests and lint rules protect consistency

**Further reading**: Nielsen Norman Group, *Consistency and Standards: Usability Heuristic #4*; Brad Frost, *Atomic Design*; Alla Kholmatova, *Design Systems*; Nathan Curtis's articles on design systems; Material, HIG, Carbon, Polaris and GOV.UK documentation.
