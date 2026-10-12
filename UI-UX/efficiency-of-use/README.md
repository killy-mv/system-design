# Efficiency of use

> **Efficiency of use** = how quickly and with how little effort experienced users can perform their tasks once they've learned the product.
> "For someone who uses this all day, how many seconds, clicks and keystrokes does each task cost?"

One of Nielsen's five usability components ([Usability §2](../usability/README.md#2-the-five-components-of-usability)) and closely related to Nielsen's heuristic *Flexibility and efficiency of use* ([Usability §6.7](../usability/README.md#67-flexibility-and-efficiency-of-use)). ISO 9241-11 defines efficiency as resources spent (time, effort) relative to the results achieved.

---

## 1. Why it matters

- **Time multiplies.** A task done 200 times a day by 500 employees: saving 5 seconds saves about 140 hours of work per day. Internal tools, support consoles, warehouse apps, trading and medical systems live or die on efficiency.
- **Fatigue and errors**: unnecessary steps tire people and create more chances for slips.
- **Power users are often your most valuable users** (and your loudest advocates or critics).

---

## 2. Techniques

| Technique | Examples |
|---|---|
| **Accelerators** (invisible to novices, fast for experts) | Keyboard shortcuts, a **command palette** (Ctrl/Cmd+K), right-click menus, gestures, slash commands in editors |
| **Fewer steps** | Inline editing instead of edit pages; actions on hover/selection; one-page forms for frequent tasks |
| **Bulk actions** | Select many → act once (archive, assign, tag, export) |
| **Smart defaults and memory** | Pre-fill last used values; remember filters, sort order, column choices |
| **Templates, presets, saved views** | Reusable configurations for repeated work |
| **Autocomplete and suggestions** | Search-as-you-type, address and name completion, suggested replies |
| **Automation** | Rules, macros, scheduled actions, integrations, APIs for power users |
| **Keyboard-first data entry** | Tab order that matches the workflow, Enter to submit, no forced mouse use (also an [accessibility](../accessibility/README.md) win) |
| **Information density for experts** | Dense tables and dashboards (with options to customize) for professionals who scan lots of data |
| **Undo instead of confirmations** | Confirmation dialogs cost time on every action; undo costs time only when needed ([User error protection](../user-error-protection/README.md)) |
| **Fast system response** | Every interaction waits on the system; latency and frozen UIs directly cut efficiency ([Responsiveness](../responsiveness/README.md)) |

### 2.1 Laws that model efficiency

- **Fitts's Law**: time to hit a target depends on its distance and size. Put frequent targets close and make them big; screen edges and corners are fast targets on desktop ([Usability §4.7](../usability/README.md#47-motor-control)).
- **Hick's Law**: decision time grows with the number of choices (roughly logarithmically). Fewer, well-grouped options speed up decisions ([Usability §7](../usability/README.md#7-laws-of-ux-rules-of-thumb)).
- **Keystroke-Level Model (KLM)** from Card, Moran & Newell (*The Psychology of Human-Computer Interaction*, 1983) estimates expert task time by adding standard operator times: roughly **K** (keystroke) ≈ 0.2 s for a skilled typist, **P** (point with a mouse) ≈ 1.1 s, **H** (move hand between keyboard and mouse) ≈ 0.4 s, **M** (mental preparation) ≈ 1.35 s. Comparing designs with KLM before building them shows, for example, that a keyboard shortcut (M + K) beats a menu path (H + several P + clicks) by seconds.

---

## 3. Balancing efficiency with other qualities

| Conflict | Approach |
|---|---|
| **Efficiency vs learnability** | Layered design: discoverable defaults for novices, accelerators for experts; show shortcuts in menus and tooltips so users learn them gradually |
| **Efficiency vs error protection** | Fast actions + undo; reserve friction for irreversible, high-impact actions |
| **Density vs clarity** | Offer density settings (comfortable / compact); keep hierarchy clear even in dense views |
| **Customization vs consistency and support cost** | Let experts customize views and shortcuts, but keep sensible defaults and a reset option |

Optimize the **frequent paths** first: measure which tasks take most total time (frequency × duration), and don't over-optimize rare flows.

---

## 4. System-design aspects

- **Latency budgets** for frequent interactions (feedback < 100 ms, navigation < 1 s).
- **Bulk and batch APIs** so bulk UI actions don't send thousands of requests.
- **Fast search** (indexed, typo-tolerant, filterable) since experts navigate by search.
- **Server-side saved views, preferences and drafts**, synced across devices.
- **Public APIs, webhooks and automation hooks** for the most efficient users of all: scripts and integrations.
- **Pagination / virtualization** that keeps large data sets responsive.

---

## 5. Measuring efficiency

| Metric | How |
|---|---|
| **Time on task** (experienced users) | Usability tests with trained users; analytics on task duration |
| **Clicks, keystrokes, screens per task** | Task analysis, analytics |
| **KLM / GOMS estimates** | Model-based comparison of designs before building |
| **Throughput** | Tasks completed per hour (support tickets resolved, orders packed) |
| **Accelerator adoption** | % of power users using shortcuts, bulk actions, saved views |

---

## 6. Checklist and further reading

- [ ] Most frequent tasks identified and their step count minimized
- [ ] Keyboard shortcuts and/or command palette for frequent actions, shown in the UI
- [ ] Bulk actions, saved views, templates and remembered preferences
- [ ] Undo used instead of routine confirmations
- [ ] Interaction latency within budget
- [ ] Efficiency measured with experienced users, not only novices

**Further reading**: Card, Moran & Newell, *The Psychology of Human-Computer Interaction* (1983); Jeff Raskin, *The Humane Interface*; Nielsen Norman Group, *Flexibility and Efficiency of Use: Usability Heuristic #7*, *Accelerators*; Jon Yablonski, *Laws of UX*.
