# Modifiability: hands-on practice notes

Planned labs for learning modifiability by **measuring how far changes spread, and refactoring so they stay local**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)).

**Suggested order:** 1 → 2 → 3.

**Related labs elsewhere:** package by feature (modularity lab 3), change coupling (modularity lab 2), booleans → state machine (simplicity lab 2).

---

## Labs

### 1. Shotgun surgery → registry ⭐
- **Do:** add "bank transfer" to messy-shop as-is. Count files touched and note anything you forgot (check receipts and reports).
- **Refactor:** the payment-method interface + registry from README §2.
- **Then:** add "buy now, pay later" and count again.

### 2. Measure change effort from Git
- **Do:** a script over `git log --numstat` for messy-shop (and a real repo you work on): files per commit, the most frequently changed files, and hotspots (churn × lines or complexity, e.g. with **radon** or **lizard**).
- **Then:** refactor the top hotspot and watch its change rate and size over the next weeks.

### 3. Make the change easy, then make the easy change
- **Do:** pick a feature that's awkward in messy-shop (e.g. "discounts can apply per item, not just per order"). Do it in two separate PRs: first a pure refactor with no behaviour change (tests stay green), then the feature.
- **Compare:** the review experience and risk with doing both in one PR.

---

## Progress

- [ ] 1. Shotgun surgery → registry
- [ ] 2. Measure change effort from Git
- [ ] 3. Make the change easy, then make the easy change
