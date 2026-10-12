# Memorability

> **Memorability** = how easily users can return to a product after a period of not using it and become productive again, without relearning it from scratch.
> "I last did this three months ago. Can I still do it?"

Memorability is one of Nielsen's five usability components ([Usability §2](../usability/README.md#2-the-five-components-of-usability)). It's different from [learnability](../learnability/README.md) (first use) and [efficiency of use](../efficiency-of-use/README.md) (frequent use): it's about **infrequent** use.

---

## 1. Where memorability matters most

Many important tasks are done rarely:

| Product / task | Typical frequency |
|---|---|
| Tax filing, annual reports, insurance renewal | Yearly |
| Admin settings, billing, user management | Monthly or less |
| Booking travel, applying for government services | A few times a year |
| Restoring a backup, disaster recovery runbooks | Rarely (and under stress) |
| Seasonal business tools (inventory counts, open enrollment) | Seasonal |
| Password reset, account recovery | Rarely |

For these, users effectively come back as semi-beginners every time. A design that's only efficient when you use it daily will fail them.

---

## 2. Design principles

| Principle | Why it helps memory | In practice |
|---|---|---|
| **Recognition over recall** | Recognizing is far easier than remembering | Visible menus and labels, recent items, autocomplete, previews ([Usability §4.3](../usability/README.md#43-recognition-over-recall)) |
| **Consistency** | One remembered pattern applies everywhere | Same pattern for the same task across the product ([Consistency](../consistency/README.md)) |
| **Follow conventions** | Users' long-term memory is mostly built from *other* products | Standard icons, locations, gestures and words |
| **Meaningful labels; icons with text** | Abstract icons are easily forgotten | "Export" + icon rather than an icon alone |
| **Stable layouts** | People rely on **spatial memory** ("it was top-right") and muscle memory | Don't move features around; avoid "adaptive" menus that reorder themselves |
| **Clear information architecture** | Logical grouping lets users re-derive where something is | Group by user tasks, not by internal team structure |
| **Search everywhere** | A fallback when memory fails | Global search, settings search, command palettes that search actions by name |
| **Remember for the user** | Reduces what users must remember | Recents, favorites, saved filters, drafts, "continue where you left off", pre-filled forms from last time |
| **Guided flows for rare complex tasks** | No need to remember the steps | Wizards, checklists, step indicators |

---

## 3. Change management: the enemy of memorability

Redesigns and moved features **destroy learned memory**, especially for occasional users who miss the announcement.

- Avoid moving frequently used features without strong reasons.
- Announce changes in context ("Settings moved here"), offer a temporary path from the old location, or let users switch back for a while.
- Keep names stable; renaming features is as disruptive as moving them.
- Use staged rollouts and watch support tickets and task success for returning users.

---

## 4. Memorability of credentials

Authentication is the most common memorability problem: users forget passwords, usernames, which email they used, and which login method ("Did I sign up with Google?").

- Support **password managers** (allow paste, proper `autocomplete` attributes).
- Offer **passkeys**, magic links and social login, and tell users which method their account uses.
- Make account recovery easy and secure.
- WCAG 2.2 SC 3.3.8 (*Accessible Authentication*) asks that login not depend on memorizing or transcribing information without an alternative ([Accessibility](../accessibility/README.md)).

---

## 5. Measuring memorability

- **Delayed retention tests**: test users on tasks, then test the same users again after weeks or months; compare success, time and errors.
- **Returning-user analytics**: task success and help usage for users returning after N days of inactivity.
- **Support tickets** spiking after redesigns, or for rarely used features ("How do I…" questions).
- **Search queries** for features that should be findable through navigation.

---

## 6. Checklist and further reading

- [ ] Rare tasks identified and designed for semi-beginners (guided, labeled, searchable)
- [ ] Icons paired with text labels for important actions
- [ ] Layouts and names stable; changes announced in context
- [ ] Recents, favorites, drafts and saved settings reduce what users must remember
- [ ] Global search covers actions and settings
- [ ] Login supports password managers, passkeys and clear recovery

**Further reading**: Nielsen Norman Group, *Recognition vs. Recall in User Interfaces*, *Memory Recognition and Recall in User Interfaces*, *Icon Usability*; Jakob Nielsen, *Usability Engineering* (memorability as a usability attribute); Don Norman, *The Design of Everyday Things* (knowledge in the world vs in the head).
