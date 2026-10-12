# User error protection

> **User error protection** = the degree to which a system **prevents users from making errors**, **catches** the errors they do make before they cause harm, and lets them **recover** easily when something goes wrong. (ISO/IEC 25010 sub-characteristic of usability / interaction capability.)
> "When a tired, distracted, hurried human clicks the wrong thing, what happens next?"

The core design assumption: **people will make errors, no matter how careful or well-trained they are.** Experts make errors too (often different ones). If a single slip can delete a customer's data, send money to the wrong account, or take down production, that's a design flaw, not a user flaw. Don Norman's point in *The Design of Everyday Things* is that most of what gets labeled "human error" is really a system that made the error easy and the recovery hard.

Related: [Usability](../usability/README.md) (§6.5 error prevention, §6.9 error messages, §8.3 forms, §8.5 confirm vs undo, §8.6 defaults) · [Reliability](../../runtime-operational/reliability/README.md) (§4.3 backups, §11 idempotency) · [Accessibility](../accessibility/README.md) (accessible error identification, SC 3.3.x) · [Responsiveness](../responsiveness/README.md) (double submits, optimistic UI rollback).

▶ **Interactive demo:** [index.html](index.html): open it in a browser to try 4 error-prone designs side by side with their fixes. They cover bulk deletion, a money transfer, two people editing the same record, and an ops console.

---

## Table of contents

1. [Why it matters](#1-why-it-matters)
2. [Understanding human error](#2-understanding-human-error)
3. [Defense in depth: five layers of protection](#3-defense-in-depth-five-layers-of-protection)
4. [Layer 1: Prevent](#4-layer-1-prevent)
5. [Layer 2: Detect](#5-layer-2-detect)
6. [Layer 3: Confirm (only when it helps)](#6-layer-3-confirm-only-when-it-helps)
7. [Layer 4: Recover](#7-layer-4-recover)
8. [Layer 5: Limit the damage](#8-layer-5-limit-the-damage)
9. [Concurrency: errors caused by other people's changes](#9-concurrency-errors-caused-by-other-peoples-changes)
10. [Operator and admin error](#10-operator-and-admin-error)
11. [Backend / system design for error protection](#11-backend--system-design-for-error-protection)
12. [Approaches by domain](#12-approaches-by-domain)
13. [Measuring error protection](#13-measuring-error-protection)
14. [Trade-offs](#14-trade-offs)
15. [Anti-patterns](#15-anti-patterns)
16. [Checklist](#16-checklist)
17. [Exercises](#17-exercises)
18. [Further reading](#18-further-reading)

---

## 1. Why it matters

User errors are some of the most expensive and most visible failures a product can have. Real cases:

| Incident | What happened | Design lesson |
|---|---|---|
| **Hawaii false missile alert** (Jan 2018) | An emergency worker sent a real "BALLISTIC MISSILE THREAT INBOUND… THIS IS NOT A DRILL" alert during a drill. Correcting it took **38 minutes**, partly because there was no prepared "false alarm" message. | Test and real actions must look clearly different; high-impact actions need strong confirmation; **plan the recovery path in advance** |
| **Citibank / Revlon** (Aug 2020) | Citibank meant to send about $7.8M in interest to Revlon's lenders, but its loan software's confusing screen (checkboxes that had to be set in a particular way) caused it to send about **$900M**, the full loan principal. A court first let some lenders keep about $500M; the decision was reversed on appeal in 2022. | Complex forms with non-obvious options; the "six-eyes" review process approved the error. Reviews only work if reviewers can **see what will actually happen**. |
| **GitLab.com database deletion** (Jan 2017) | During incident response late at night, an engineer ran a delete command on the **primary** database server instead of the replica. Several backup mechanisms turned out not to work; about 6 hours of production data was lost. | Environment confusion (similar terminals, similar host names); destructive commands without guardrails; **untested backups** |
| **AWS S3 outage, us-east-1** (Feb 2017) | An engineer ran an established playbook command to remove a small number of servers, but one input was entered incorrectly and a much larger set was removed, taking down S3 in the region and many services built on it for hours. AWS then changed the tool to remove capacity more slowly and to refuse to go below minimum capacity. | **Blast radius limits** and safety checks in tools; typos must not scale to outages |
| **Everyday product failures** | Form data lost on error or session timeout, double charges from repeated clicks, permanent deletion with one click, sending an email to the wrong recipient, overwriting a colleague's edits | The same principles at smaller scale, multiplied by millions of users |

Business impact:
- **Support cost**: "I deleted it by accident, can you restore it?" is one of the most common support requests in many products.
- **Trust and churn**: users who lose data or money due to a UI often leave and tell others.
- **Legal and financial liability**: misdirected payments, privacy breaches (emailing the wrong person), regulatory penalties.
- **Productivity**: fear of making errors makes people slow and hesitant. Good recovery (undo) makes people **confident**, which makes them faster.

---

## 2. Understanding human error

### 2.1 Types of error

Based on James Reason's *Human Error* (1990) and Don Norman's work:

| Type | What happens | Example | Typical cause | Best defenses |
|---|---|---|---|---|
| **Slip** (action-based) | Right intention, wrong action | Click "Delete" instead of "Edit" next to it; type 30 instead of 3 | Similar or nearby controls, automatic behavior, distraction | Spacing, distinct design, constraints, undo, confirmation of unusual values |
| **Lapse** (memory-based) | Forget a step or lose track | Forget to attach the file; forget to save; leave the stove on | Interruptions, long multi-step tasks, memory load | Reminders ("You mentioned an attachment"), autosave, checklists, progress indicators |
| **Mistake** (rule- or knowledge-based) | Wrong intention: the plan itself is wrong | "Archive" thinking it deletes; choosing "Disburse principal" thinking it pays interest | Wrong mental model, confusing labels, missing information | Clear language, previews, showing consequences, good defaults, help at the point of decision |
| **Violation** (deliberate deviation) | Knowingly skipping a rule | Clicking through warnings; sharing passwords; disabling a safety check "to get it done" | Rules that are too costly, frequent false alarms, time pressure | Make the safe path the easy path; reduce false alarms; understand *why* people work around it |

Common **slip** subtypes (Norman):
- **Capture slip**: a frequent action "captures" a similar, rarer one (you meant to go to the store but drove to work on autopilot; you meant "Save as" but hit "Save").
- **Description-similarity slip**: acting on the wrong object because it looks like the right one (two identical-looking buttons, environments, or files).
- **Mode error**: the same action means different things in different modes (Caps Lock, vim's insert vs normal mode, editing production while thinking it's staging, a car's cruise control state).
- **Memory-lapse slip**: forgetting where you were after an interruption.

The distinction matters because **the fixes are different**: you don't fix a slip with more training or a better explanation (the user already knew what to do), and you don't fix a mistake with more spacing between buttons.

### 2.2 Conditions that increase errors

- **Time pressure and stress** (incidents, deadlines, checkout timers).
- **Fatigue** (late at night, long shifts).
- **Interruptions and multitasking** (very common in real work).
- **Similarity**: look-alike options, names, environments, drug names, account numbers.
- **Modes**, especially invisible ones.
- **Habituation**: dialogs and warnings seen too often get dismissed automatically.
- **Complexity**: many options, unclear consequences, unusual workflows done rarely.
- **Unfamiliarity**: new users, rare tasks, changed UIs ("the button moved").

### 2.3 The Swiss cheese model and just culture

James Reason's **Swiss cheese model**: every layer of defense (UI design, validation, review, permissions, backups) has holes. An accident happens when the holes in several layers **line up**. The person at the end ("the operator who clicked") is usually the last hole, not the root cause.

Implications:
- Add **multiple independent layers** ([§3](#3-defense-in-depth-five-layers-of-protection)). Don't rely on "users should be careful".
- After an incident, ask **"how did the system allow this?"**, not "who did it?". **Blameless postmortems** (standard in SRE practice) surface the real causes; blaming individuals makes people hide errors and near misses.
- Treat **near misses** (errors caught just in time) as free lessons.

---

## 3. Defense in depth: five layers of protection

| Layer | Goal | Examples |
|---|---|---|
| **1. Prevent** | Make the error impossible or unlikely | Constraints (date pickers, dropdowns), good defaults, forcing functions, distinct dangerous actions, no modes |
| **2. Detect** | Catch the error before it takes effect | Validation, warnings for unusual values, previews, dry runs, check digits, review screens |
| **3. Confirm** | Make the user consciously verify a high-impact action | Specific confirmation dialogs, type-to-confirm, re-authentication, second approver |
| **4. Recover** | Undo the effect after it happened | Undo, soft delete/trash, version history, drafts, delayed execution, reversals |
| **5. Limit damage** | Reduce the impact if everything else fails | Blast radius limits, permissions, rate limits, deletion protection, backups, audit logs |

Rules of thumb:
- **Prevent > Recover > Confirm.** Prevention and recovery protect users without slowing them down. Confirmation adds friction to every use and loses effectiveness with habituation.
- **Match protection to risk**: frequency × severity × reversibility. A daily, reversible action needs undo, not a dialog. A rare, irreversible, catastrophic action needs several layers.
- **Recovery is the most robust layer**, because it works for every error type, including ones you didn't predict.

| Action | Frequency | Reversible? | Severity | Appropriate protection |
|---|---|---|---|---|
| Archive email | Very high | Yes | Low | Just do it + undo toast |
| Delete a document | Medium | With trash | Medium | Soft delete + undo + trash for 30 days |
| Bulk delete 1,200 records | Low | With trash | High | Preview with count + specific confirmation + trash |
| Send money to a new payee | Low | Hard | High | Validation + payee name check + review screen + delayed execution/cancel window |
| Delete the whole account / workspace | Very low | Grace period only | Very high | Type-to-confirm + re-auth + grace period + email notification |
| Drop a production database | Very rare | Only from backups | Catastrophic | Deletion protection flag + second approver + delayed deletion + tested backups |

---

## 4. Layer 1: Prevent

### 4.1 Constraints: make invalid input impossible

| Instead of | Use |
|---|---|
| Free-text date ("03/04/26") | Date picker, or three clear fields, with the format shown |
| Free-text country, state, currency | Select / combobox with search |
| Free-text quantity | Stepper with min/max; `type="number"` with `min`/`max`/`step` |
| Free-text phone, card number | Inputs that accept any formatting (spaces, dashes) and normalize, with `inputmode` and `autocomplete` |
| Re-typing known data | Autofill, "Same as shipping address", pick from saved items (WCAG 3.3.7 Redundant entry) |
| Choosing impossible combinations | Only offer valid options (e.g. return dates after the departure date) |

Two cautions:
- **Disabled controls need explanation.** A grayed-out "Continue" button with no reason is a dead end. Either explain what's missing next to it, or keep it enabled and show validation errors on click.
- **Don't over-constrain real data.** Name and address fields restricted to `[A-Za-z]` *prevent* valid input; that's an error the system makes ([Internationalization §8.2](../internationalization/README.md#82-validation-that-respects-real-people)).

### 4.2 Forcing functions

Norman's term for physical or logical constraints that **force** the correct sequence:

| Type | Meaning | Examples |
|---|---|---|
| **Interlock** | Operations must happen in the right order | Microwave won't run with the door open; car won't shift out of Park without the brake; can't submit an order before choosing a shipping method; ATM gives back the card **before** the cash (so you don't walk away without your card) |
| **Lock-in** | Prevent stopping an operation prematurely | "You have unsaved changes. Leave anyway?"; the computer won't shut down while saving; autosave instead of a warning, where possible |
| **Lockout** | Prevent entering a dangerous state | Stair gates; "Delete" not available while a job is running; can't delete a project that still has active subscriptions; production deploy blocked during a freeze |

The Japanese term **poka-yoke** ("mistake-proofing", from Shigeo Shingo's work at Toyota) describes the same idea in manufacturing: parts that only fit one way (like USB-C, unlike older USB-A plugs, or SIM cards with a cut corner).

### 4.3 Good defaults

- Most users keep defaults, so **the default should be the safest common choice** (e.g. "Visibility: Private", "Notify customers: Off" for test orders, "Dry run" checked in admin tools).
- Pre-fill from context (last used values, the user's locale) to avoid re-typing errors.
- **Never default to the destructive option** in a dialog. Keyboard focus and the Enter key should land on the safe choice.

### 4.4 Make dangerous actions distinct and distant

- **Different appearance**: destructive buttons styled differently (danger color + icon + clear label), never visually identical to the safe action. Remember color alone isn't enough ([Accessibility §7.1](../accessibility/README.md#71-color-and-contrast)).
- **Physical distance**: don't put "Delete" right next to "Edit" or "Save"; put rarely used destructive actions in an overflow menu or a separate "Danger zone".
- **Specific labels**: "Delete project", "Send $500 to Lan", not "OK" / "Yes" / "Submit".
- **Avoid look-alike items in lists**: the Hawaii alert system reportedly presented drill and real alerts as similar entries in one menu. Separate test and real actions physically and visually.
- **Tall-man lettering** in healthcare (hydrOXYzine vs hydrALAzine) highlights the different parts of look-alike names. The same trick helps for similar product names, account names or host names.

### 4.5 Avoid modes, or make them unmistakable

- Prefer **modeless** designs (actions that mean the same thing everywhere).
- If modes are necessary, make the current mode **always visible and hard to ignore**: a red "PRODUCTION" banner on admin consoles, "Test mode" banners on payment dashboards (Stripe shows a clear test-mode indicator), "Recording" indicators, Caps Lock warnings on password fields.
- **Spring-loaded / quasi-modes** (active only while a key is held, like Shift) are safer than toggled modes.
- Reset dangerous modes automatically (e.g. elevated "sudo mode" in an admin tool expires after 15 minutes).

### 4.6 Reduce the load that causes errors

- **Recognition over recall**: show options, recent items, autocomplete; don't make users remember IDs or codes.
- **Plain language** and consequences shown at the point of decision ("Archiving hides the project from your team. You can restore it anytime.").
- **Checklists** for rare, complex procedures (aviation, surgery, and production operations all use them; see Atul Gawande's *The Checklist Manifesto*).
- **Fewer steps** and less typing mean fewer chances for slips.
- **Protect against interruptions**: autosave, drafts, clear progress in multi-step flows, and remember where the user was.

---

## 5. Layer 2: Detect

### 5.1 Validation

| Timing | Use for |
|---|---|
| **While typing** | Formatting help and positive feedback (password strength meter, character count). Not for errors on half-typed values. |
| **On blur** (leaving the field) | Most field-level errors: format, required, range |
| **On submit** | Cross-field and server-side rules; focus the first error, keep all input |
| **Server-side, always** | Client validation is a UX convenience; the **server is the authority** (security and integrity) |

Good error messages: **what happened + why + how to fix it**, next to the field, in plain language, without blaming the user ([Usability §6.9](../usability/README.md#69-help-users-recognize-diagnose-and-recover-from-errors)). Make errors accessible: text (not only color), linked with `aria-describedby`, announced or focused (WCAG 3.3.1, 3.3.3).

### 5.2 Soft validation: warn on valid-but-unusual input

Many costly errors are **technically valid**: a transfer of 10,000 instead of 1,000, an email to `gmial.com`, a price of 0, a meeting at 3 AM, a discount of 100%. Hard validation can't block them, but the system can **warn**:

- "This amount is **12× larger** than your usual transfers to this payee."
- "Did you mean **gmail.com**?" (email typo suggestion)
- "You're about to email **all 4,812 customers**, not your test list."
- "This price is 90% below the current price. Is that intended?"
- "Your message mentions 'attached' but there's no attachment." (Gmail does this)
- "You're replying all to 230 people."

Make the warning specific, show the numbers, and let the user proceed with one deliberate action.

### 5.3 Check digits and verification

- **Check digits** detect typos in identifiers: credit cards (Luhn algorithm), IBANs (mod 97), ISBNs, many national ID and account formats. Validate them on the client for instant feedback.
- **Payee name verification**: the UK's **Confirmation of Payee** (rolled out from 2020) and the EU's **Verification of Payee** (required for euro transfers since October 2025) check that the account holder's name matches the name the sender entered, and show "match", "close match: did you mean…?", or "no match" before the payment is sent. This targets both typos and scams.
- **Email and phone verification** codes confirm contact details before they're used for important messages.

### 5.4 Previews, dry runs and diffs

Show **what will happen** before it happens:

- **Preview** the result: "This will move **1,204 files** to Trash, including **3 starred** files." Show a sample of affected items.
- **Dry run** for bulk and admin operations: `terraform plan`, `kubectl diff`, `rsync --dry-run`, database migration previews, "Send test email to me".
- **Diffs** before saving or applying changes (code review, config changes, document versions).
- **Review step** for multi-step transactions (checkout, transfers, bookings): a summary of all choices in plain language, with links to edit each one.

The Citibank case shows that a review step only works if it shows **the effect** ("You are about to pay out $900M principal"), not just the raw form fields that confused the original user.

---

## 6. Layer 3: Confirm (only when it helps)

### 6.1 The habituation problem

Confirmation dialogs are the most common error-protection technique and the most overused. When people see "Are you sure?" after every action, they learn to click "OK" **automatically**, without reading. Research on security warnings has shown attention dropping sharply after repeated exposure. An over-used confirmation protects no one and slows everyone.

**Use a confirmation only when the action is both high-impact and hard to reverse.** If you can offer undo instead, do that.

### 6.2 Designing a confirmation that works

| Bad | Good |
|---|---|
| "Are you sure?" | "Delete project **Apollo**? Its **312 files** and **4 integrations** will be moved to Trash for 30 days." |
| Buttons "OK" / "Cancel" | Buttons "**Delete project**" / "Keep project" (the action is named) |
| Focus on the destructive button (Enter = delete) | Focus on the **safe** button |
| Same dialog for deleting 1 item and 10,000 | Friction scales with impact |
| Dialog appears for every routine action | Only for high-impact, low-frequency actions |
| Destructive button looks the same as the safe one | Destructive button visually distinct (danger style + clear label) |

### 6.3 Friction proportional to risk

| Risk level | Technique |
|---|---|
| Low, reversible | No confirmation; undo |
| Medium | Specific confirmation dialog |
| High | **Type-to-confirm**: type the resource name ("apollo-prod") or the count ("1204"). This forces the user to read *what* they're acting on. GitHub requires typing the repository name to delete a repo. |
| High, security-sensitive | **Re-authentication** ("sudo mode": enter password or passkey again) for changing email, disabling 2FA, deleting the account, viewing secrets |
| Very high / regulated | **Second person approval** ("four-eyes principle"): another authorized person reviews and approves (large payments, production changes, permission grants). The reviewer must see the effect, not just the request. |

### 6.4 Don't weaponize confirmations

- **Confirmshaming** ("No thanks, I don't like saving money") is a dark pattern.
- Don't use confirmation dialogs or extra friction to stop users from **cancelling subscriptions** or deleting their data. Regulators increasingly treat that as deceptive (e.g. "click to cancel" rules).
- Friction should protect the user's interests, not the company's metrics.

---

## 7. Layer 4: Recover

### 7.1 Undo

Undo is the single most valuable error-protection feature: it protects against **every** kind of error, including ones you didn't anticipate, without slowing down correct use.

| Pattern | Example |
|---|---|
| **Undo toast** (action happens immediately, "Undo" available for a few seconds) | Gmail archive/delete, Slack, most mobile apps |
| **Multi-level undo / redo** (Ctrl+Z / Ctrl+Shift+Z) | Editors, design tools, spreadsheets, form builders |
| **History / version list** (restore any earlier version) | Google Docs version history, Figma, Notion page history, Git |
| **Selective undo** (undo one specific change, not everything since) | Collaborative tools, audit-log-based "revert this change" |
| **Delayed execution** (wait N seconds before really doing it; cancel = undo) | Gmail "Undo Send" (delays sending by 5–30 seconds), scheduled payments with a cancel window |

Implementation approaches:
- **Command pattern**: each action is an object with `do()` and `undo()` (inverse operation). Keep a stack.
- **Snapshots**: store the previous state and restore it (simple, costs memory/storage).
- **Event sourcing**: store every change as an event; any past state can be rebuilt, and "undo" is a compensating event.
- **Soft delete** and **versioning** on the server, so undo works across devices and sessions, not only in one browser tab.

Undo toasts must be **accessible**: announced via a live region, long enough to act on (or persistent until dismissed), and the action reachable another way (e.g. via Trash).

### 7.2 Soft delete and trash

Instead of deleting data immediately, **mark it as deleted** and purge it later.

```sql
ALTER TABLE documents ADD COLUMN deleted_at TIMESTAMPTZ NULL;
ALTER TABLE documents ADD COLUMN deleted_by BIGINT NULL;

-- "Delete"
UPDATE documents SET deleted_at = now(), deleted_by = :user WHERE id = :id;
-- "Restore"
UPDATE documents SET deleted_at = NULL, deleted_by = NULL WHERE id = :id;
-- Normal queries exclude deleted rows (or use a view / default scope)
SELECT * FROM documents WHERE workspace_id = :ws AND deleted_at IS NULL;
-- Purge job: hard-delete after the retention period
DELETE FROM documents WHERE deleted_at < now() - interval '30 days';
```

Design details that matter:
- **Retention period** visible to users ("Items in Trash are deleted forever after 30 days").
- **Cascades**: deleting a project soft-deletes its files; restoring the project restores them, but not files that were deleted individually before.
- **Unique constraints**: a soft-deleted record may still "own" a name or email. Use partial unique indexes (`UNIQUE (workspace_id, name) WHERE deleted_at IS NULL`) and decide what happens if a restored item conflicts.
- **Every query must exclude deleted rows**: use views, ORM default scopes, or row-level security. Forgetting the filter leaks "deleted" data.
- **Privacy law**: soft delete is not erasure. For GDPR "right to erasure" and similar, you need a real purge (including derived data, search indexes and, within a defined period, backups).
- **Admin/support restore tools**, so support can help users who notice too late (with audit logging).

### 7.3 Versioning, drafts and autosave

- **Version history** for documents, configurations, and important records. Let users compare and restore.
- **Object storage versioning** (e.g. S3 Versioning) keeps previous versions of overwritten or deleted objects; **object lock / WORM** prevents deletion for a retention period (protection against mistakes and ransomware).
- **Drafts and autosave**: long forms, messages and documents save continuously. Crashes, closed tabs, expired sessions and network errors shouldn't lose work.
- **Session expiry** should never destroy unsaved input: save locally, re-authenticate in place, then continue.
- **"Unsaved changes" warnings** (lock-ins) as a last resort when autosave isn't possible.

### 7.4 Grace periods for big irreversible actions

- **Account and workspace deletion**: deactivate immediately, delete permanently after e.g. 14–30 days, with an email that includes a "Cancel deletion" link.
- **Cloud resources**: many providers build in waiting periods: AWS KMS keys can only be scheduled for deletion with a 7–30 day waiting period; Google Cloud projects stay recoverable for about 30 days after shutdown.
- **Domain names, usernames**: hold released names for a period before others can take them.

### 7.5 Reversal processes

Some actions can't be undone technically, but can be **compensated**: refunds, payment recalls, "unsend" requests to banks, correction notices. Design these flows in advance (the Hawaii alert lacked a prepared correction message), make them discoverable, and make support tooling able to execute them quickly.

---

## 8. Layer 5: Limit the damage

When prevention, detection, confirmation and recovery all fail, the system should still limit the **blast radius**:

- **Least privilege**: users and staff only get the permissions they need. Most people should not be *able* to bulk-delete all customer data.
- **Scope limits**: bulk operations limited to a maximum size per action, or processed in stages with a pause/abort option.
- **Rate limits** on destructive and outbound actions (mass emails, deletions, transfers).
- **Hard safety limits** in tools: refuse to reduce capacity below a minimum; refuse to delete more than N% of resources at once; refuse to deploy to all regions at once.
- **Deletion protection flags** that must be explicitly turned off first (AWS RDS deletion protection, Terraform `prevent_destroy`, protected Git branches).
- **Staged rollouts** for changes affecting many users or systems: canary → 1% → 10% → 100%, with automatic rollback.
- **Backups and point-in-time recovery**, **tested regularly by actually restoring** (GitLab's 2017 incident showed backups that silently didn't work). See [Reliability §4.3](../../runtime-operational/reliability/README.md).
- **Audit logs**: who did what, when, from where. Essential for diagnosing and reversing errors (and for accountability).
- **Alerts on unusual activity**: a spike in deletions or a sudden bulk export can mean an error (or an attack).

---

## 9. Concurrency: errors caused by other people's changes

Some user errors aren't caused by the user at all, but by the system letting them act on **stale information**.

### 9.1 The lost update problem

```
Alice opens record (version 7): price = 100
Bob opens record (version 7):   price = 100
Bob saves price = 120          → version 8
Alice saves description change → her form still has price = 100 → overwrites Bob's change silently
```

Bob's change is gone and nobody knows. This is extremely common in admin panels, CRMs, CMSs and settings pages.

### 9.2 Optimistic concurrency control

Every record has a **version** (or an `updated_at`, or an HTTP **ETag**). Updates succeed only if the version hasn't changed since the user loaded it:

```sql
UPDATE products SET price = :price, description = :desc, version = version + 1
WHERE id = :id AND version = :version_user_loaded;
-- 0 rows updated → someone else changed it → conflict
```

```http
GET /products/42           → 200 OK, ETag: "v7"
PUT /products/42           If-Match: "v7"
                           → 412 Precondition Failed   (if the current version is "v8")
```

The UI then **shows the conflict** instead of silently overwriting:
- Show what changed and who changed it ("Bob changed **price** from 100 to 120, 2 minutes ago").
- Offer: keep both changes when they touch different fields (automatic merge), or let the user choose per field.
- Never discard the user's own unsaved input.

### 9.3 Other approaches

| Approach | When |
|---|---|
| **Optimistic concurrency** (versions/ETags + conflict UI) | Most CRUD apps; conflicts are rare |
| **Field-level updates** (PATCH only changed fields) | Reduces conflicts: Alice's description edit doesn't touch the price at all |
| **Pessimistic locking / check-out** ("Bob is editing this record") | Long edits where conflicts are expensive (legal documents, CAD files) |
| **Presence indicators** ("Bob is viewing this page") | Warn before conflicts happen |
| **Real-time collaboration** (OT or CRDTs: Google Docs, Figma, Notion) | Many people editing simultaneously; merges continuously |
| **Revalidation before critical actions** | "The price changed from $40 to $45 since you added this to your cart." "This seat is no longer available." |

### 9.4 Double submission

Repeated clicks, retries and flaky networks create duplicates (two orders, two payments). The UI should show instant feedback and block repeated submits, but the real protection is **idempotency** on the server: see [Reliability §11](../../runtime-operational/reliability/README.md#11-idempotency) and the payment example in the [usability demo](../usability/index.html).

---

## 10. Operator and admin error

Internal tools, admin panels, CLIs and infrastructure consoles often have **the most powerful actions with the least protection**, used under stress (during incidents) by tired people. Many of the biggest outages in tech history were operator errors made through tools that allowed them.

| Problem | Protection |
|---|---|
| **Environment confusion** (prod vs staging look identical) | Different colors/banners per environment (red for production), environment name in the shell prompt and page title, separate credentials and accounts for production |
| **Typos that scale** (`30` instead of `3`, wrong host name) | Show the effect before running ("This will remove 30 of 32 servers"); minimum-capacity and max-percentage guardrails; type-to-confirm the target |
| **Destructive commands without preview** | `--dry-run` by default for bulk tools; `plan` before `apply`; diff before deploy |
| **Manual changes in production** | Infrastructure as code, changes via pull requests with review, automated pipelines; manual access via "break-glass" with audit and expiry |
| **Too much access** | Least privilege, just-in-time elevated access, separate admin accounts |
| **Running the wrong runbook step** | Runbooks as tested scripts, not copy-paste commands; checklists |
| **Irreversible infrastructure deletion** | Deletion protection, delayed deletion (KMS waiting periods, soft-deleted buckets/projects), backups in separate accounts, object lock |
| **Errors during incidents** | Two people for risky actions during incidents (one acts, one verifies), slow down: "pause before you press enter" |

**CLI conventions** that protect users:
- Destructive commands ask for confirmation in interactive mode and require an explicit flag (`--force`, `--yes`) in scripts.
- Support `--dry-run` and print what will happen.
- Show the target clearly (`Deleting 3 pods in namespace "production" on cluster "eu-prod-1"`).
- Make operations reversible where possible (Git's reflog keeps references to "lost" commits; `git push --force-with-lease` refuses to overwrite others' work, unlike `--force`).

---

## 11. Backend / system design for error protection

UI protections only work if the backend supports them. Many of the most effective protections are architectural:

| System design decision | Error it protects against |
|---|---|
| **Server-side validation** as the authority (shared schema with the client) | Invalid data from buggy clients, old app versions, scripts, malicious requests |
| **Idempotency keys** for mutations | Double submits, retries creating duplicate orders/payments |
| **Optimistic concurrency** (version columns, ETags + `If-Match`) | Lost updates, acting on stale data |
| **State machines** for business objects (order: draft → placed → shipped → delivered) | Invalid transitions (refunding an unpaid order, shipping a cancelled one) |
| **Soft delete + retention + purge jobs** | Accidental deletion, with a path to real erasure |
| **Versioning / history tables / event sourcing** | Overwrites; enables undo, restore and audit |
| **Delayed execution queues** (scheduled jobs with a cancel window) | "Undo send", cancellable transfers and bulk operations |
| **Preview / dry-run endpoints** (`POST /bulk-delete?dry_run=true` returns counts and samples) | Bulk operations with unexpected scope; lets the UI show consequences |
| **Limits on bulk operations** and rate limits | Small errors becoming huge ones |
| **Fine-grained authorization** and least privilege | Users acting outside their role, staff errors with too much power |
| **Re-authentication and approval workflows** (step-up auth, four-eyes) | High-impact errors and account takeovers |
| **Audit logs** (immutable, with before/after values) | Diagnosing and reversing errors; accountability |
| **Backups, point-in-time recovery, restore drills** | Logical errors (deletes, bad migrations) that replication faithfully copies |
| **Structured, field-level error responses** | Lets the UI show precise, fixable error messages next to the right field |
| **Draft storage / autosave endpoints** | Lost work on crashes, timeouts and closed tabs |
| **Name/account verification services** (payee checks, address validation) | Misdirected payments and deliveries |
| **Feature flags and staged rollouts** | A bad change (an operator error) affecting all users at once |

Note: **replication is not protection against user error.** If a user (or an operator, or a bad migration) deletes data, replicas delete it too, within milliseconds. Only backups, delayed replicas, versioning and soft delete protect against logical errors.

---

## 12. Approaches by domain

| Domain | Key protections |
|---|---|
| **Banking / payments** | Payee name verification, check digits, review screen with amount written clearly, unusual-amount warnings, new-payee cooling-off periods and limits, re-authentication, cancel window for scheduled transfers, idempotent payment APIs, dual approval for large business payments |
| **E-commerce** | Clear cart review, address validation, revalidation of price/stock before payment, idempotent checkout, easy order cancellation within a window, returns |
| **Email / messaging** | Undo send, warnings for external recipients / reply-all to large groups / missing attachments, edit and delete messages, recall for internal email |
| **Productivity / documents** | Autosave, version history, undo/redo, trash, conflict-free collaboration, "restore this version" |
| **SaaS admin / B2B** | Role-based access, bulk operation previews, audit logs, soft delete with admin restore, type-to-confirm for workspace deletion, approval workflows |
| **Developer tools / infrastructure** | Dry runs, plan/apply, deletion protection, protected branches, `--force-with-lease`, environment indicators, staged rollouts, automatic rollback |
| **Healthcare** | Barcode scanning of patient and medication, tall-man lettering, dose range limits (smart infusion pumps), hard stops vs soft warnings carefully balanced to avoid alert fatigue, checklists |
| **Safety-critical (aviation, industrial, automotive)** | Interlocks, two-person rules, checklists, clear modes, fail-safe defaults; see the [Safety](../../safety/) notes |
| **Consumer mobile apps** | Undo toasts, trash, generous touch targets with spacing, confirmation only for irreversible actions, recovery via cloud sync |

---

## 13. Measuring error protection

| Metric | Source |
|---|---|
| **Form error rate** per field (how often each field fails validation) | Analytics on validation events |
| **Task error rate and recovery rate** | Usability tests (errors per task, % recovered without help) |
| **Abandonment after errors** | Funnels: users who hit an error and then left |
| **Undo / restore usage** | Product analytics: how often users undo, restore from trash, restore versions. High usage of undo isn't bad: it means the safety net works. Look for *what* gets undone most, since that may reveal a confusing control. |
| **"Accidental" support tickets** | Tag tickets: accidental deletion, wrong recipient, duplicate charge, lost work |
| **Duplicate transactions / conflicts detected** | Backend metrics: idempotency key replays, 412 conflicts, payee name mismatches |
| **Incidents caused by human/operator error** and near misses | Postmortems (blameless), with action items that change the system |
| **Time to recover** from user errors | How long until data was restored or the error corrected |

---

## 14. Trade-offs

| Tension | Notes |
|---|---|
| **Protection vs efficiency** | Every confirmation, review step and warning costs time for the 99% who didn't make an error. Prefer invisible protections (constraints, undo, soft delete) and scale friction with risk. |
| **Warnings vs alert fatigue** | Too many warnings get ignored, including the important one. Tune thresholds, measure how often warnings are overridden, and remove low-value ones. |
| **Undo vs storage and complexity** | Versioning and soft delete cost storage and query complexity (filters, unique constraints, cascades). Usually worth it for user-created data. |
| **Soft delete vs privacy and compliance** | Keeping "deleted" data is a liability. Define retention, purge reliably, handle backups and erasure requests. |
| **Delayed execution vs speed** | A 30-second cancel window on transfers or emails protects users but delays the action. Make it optional or show it clearly. |
| **Strict validation vs real-world data** | Over-strict rules create errors (rejecting valid names, addresses, phone numbers). Prefer soft warnings for "unusual" data. |
| **Locking vs collaboration** | Pessimistic locks prevent conflicts but block colleagues; optimistic concurrency allows parallel work but needs a conflict UI. |
| **Guardrails vs emergency speed** | Safety limits in ops tools can slow down incident response. Provide an audited "break-glass" path rather than removing guardrails. |
| **Protecting users vs protecting the business** | Friction should protect users; using "protection" to block cancellations or data deletion is a dark pattern. |

---

## 15. Anti-patterns

| Anti-pattern | Why it's harmful | Fix |
|---|---|---|
| **"Are you sure?" on every action** | Habituation: nobody reads it, so it protects no one | Undo for frequent actions; confirmations only for rare, irreversible ones |
| **"OK / Cancel" buttons** | Users must read the question to know what OK does | Name the action: "Delete 3 files" / "Keep files" |
| **Default focus on the destructive button** | Enter or a quick click destroys data | Focus the safe option |
| **Permanent delete with no trash or undo** | One slip = permanent loss | Soft delete, trash, undo |
| **Destructive button next to a frequent one, same style** | Slips | Distance, distinct style, overflow menu |
| **Generic confirmation for bulk actions** ("Delete selected items?") | User doesn't see that "selected" means 1,204 items | Show the count and a sample; type-to-confirm above a threshold |
| **Clearing the form on error** | Users lose everything they typed | Keep input; mark only the problem fields |
| **Vague errors** ("Invalid input", "Something went wrong") | Users can't fix the problem | What + why + how to fix, next to the field |
| **Errors shown only in red or only at the top** | Missed by color-blind and screen reader users and by people looking at the field | Text near the field + summary + focus management |
| **Silent "last write wins"** | Lost updates nobody notices | Optimistic concurrency + conflict UI |
| **Client-side-only validation** | Bad data from other clients and scripts | Server-side validation as the authority |
| **No idempotency on payments and orders** | Duplicates from retries and double clicks | Idempotency keys |
| **Look-alike environments, accounts, test vs real actions** | Description-similarity and mode errors | Distinct colors, banners, names; separate test and real actions |
| **Invisible modes** | Same action, different effect | Visible mode indicators; avoid modes |
| **Tools without blast radius limits** | One typo takes down production | Minimum-capacity checks, max-percentage limits, staged execution |
| **Bulk actions with no preview or dry run** | Unexpected scope | Preview counts and samples; dry-run endpoints |
| **Untested backups** | You find out they don't work when you need them | Regular restore drills |
| **Relying on replication as a backup** | Deletes replicate instantly | Backups, versioning, delayed replicas |
| **Session timeout destroys unsaved work** | Lost effort, especially for slower users | Autosave, local drafts, re-auth in place |
| **Disabled submit button with no explanation** | Users are stuck and don't know why | Explain what's missing, or validate on click |
| **Blaming the user** in messages or postmortems | Hides real causes; errors recur | Fix the system; blameless postmortems |
| **Confirmshaming and friction to cancel** | Deceptive; legal risk | Neutral language; easy cancellation |

---

## 16. Checklist

**Prevent**
- [ ] Constrained inputs (pickers, selects, steppers) where values come from a known set or range
- [ ] Inputs accept any reasonable formatting and normalize it; autofill and "same as" options reduce re-typing
- [ ] Safe defaults; destructive options never default or pre-focused
- [ ] Destructive actions visually distinct, clearly labeled, and physically separated from frequent ones
- [ ] Test vs real / staging vs production clearly differentiated; modes always visible
- [ ] Forcing functions for required sequences (interlocks, lockouts)
- [ ] Disabled controls explain why

**Detect**
- [ ] Field validation on blur, cross-field on submit, server-side always
- [ ] Error messages: what + why + how to fix, next to the field, accessible, input kept
- [ ] Soft warnings for valid-but-unusual input (large amounts, typo domains, mass recipients)
- [ ] Check digits validated; payee/account name verification for payments
- [ ] Previews / dry runs / diffs for bulk, admin and configuration changes
- [ ] Review step for multi-step transactions, showing effects in plain language

**Confirm**
- [ ] Confirmations only for high-impact, hard-to-reverse actions
- [ ] Specific text (object, count, consequence) and action-named buttons
- [ ] Type-to-confirm, re-authentication or second approval for the highest-risk actions

**Recover**
- [ ] Undo for frequent actions (accessible toasts, Ctrl+Z where relevant)
- [ ] Soft delete + trash with a visible retention period; restore for users and support
- [ ] Version history for documents and important records
- [ ] Autosave and drafts; session expiry doesn't lose input
- [ ] Grace period + cancellation email for account/workspace deletion
- [ ] Delayed execution / cancel window for sends, transfers and bulk jobs where appropriate
- [ ] Planned reversal processes (refunds, corrections, recall)

**Limit damage & system**
- [ ] Idempotency keys on mutations; optimistic concurrency with conflict UI
- [ ] State machines guard business object transitions
- [ ] Least privilege; bulk operation limits; rate limits on destructive/outbound actions
- [ ] Guardrails in ops tools (minimum capacity, max % change, environment confirmation)
- [ ] Deletion protection on critical resources; staged rollouts with rollback
- [ ] Backups and point-in-time recovery with regular restore drills
- [ ] Audit logs with before/after values
- [ ] Purge jobs meet retention and erasure requirements

**Learn**
- [ ] Track field error rates, undo/restore usage and "accidental" support tickets
- [ ] Blameless postmortems for user- and operator-error incidents, including near misses

---

## 17. Exercises

1. **Classify errors.** Recall 5 errors you made with software this month. Classify each as slip, lapse, mistake or violation. For each, which layer (prevent, detect, confirm, recover, limit) would have helped most?
2. **Confirmation audit.** Count the confirmation dialogs in an app you build. For each, decide: keep (specific, high-impact), replace with undo, or remove. Rewrite the ones you keep with specific text and action-named buttons.
3. **Soft delete design.** Add soft delete to a `projects` table that has child `files` and a unique project name per workspace. Handle: cascades, restore, name conflicts on restore, queries that must exclude deleted rows, a 30-day purge, and a GDPR erasure request.
4. **Lost update.** Build a small edit form backed by an API. Open it in two tabs and reproduce a lost update. Then add a version column / ETag with `If-Match`, return `412`, and design the conflict UI.
5. **Soft validation.** For a money transfer form, define three "unusual but valid" conditions worth warning about, write the warning text, and decide what the user must do to proceed.
6. **Blast radius.** Design guardrails for an internal tool command `remove-servers --count N --pool P`. What checks, previews and limits would have prevented the 2017 S3 outage? What's the break-glass path?
7. **Undo architecture.** Design "Undo send" for a messaging product: where does the delay live (client or server)? What happens if the user closes the app during the window? How do notifications and read receipts behave?
8. **Postmortem.** Read the public GitLab (2017) database incident postmortem. List every "hole in the cheese" that lined up, and which system changes (not "be more careful") address each.

---

## 18. Further reading

- Don Norman, *The Design of Everyday Things* (revised 2013): slips vs mistakes, forcing functions, modes, "human error? No, bad design." **Start here.**
- James Reason, *Human Error* (1990) and *Managing the Risks of Organizational Accidents* (1997): error types and the Swiss cheese model.
- Sidney Dekker, *The Field Guide to Understanding 'Human Error'*: the "new view" of human error and blameless investigation.
- Atul Gawande, *The Checklist Manifesto* (2009).
- Nielsen Norman Group: *Preventing User Errors: Avoiding Unconscious Slips*, *Preventing User Errors: Avoiding Conscious Mistakes*, *Confirmation Dialogs Can Prevent User Errors (If Not Overused)*, *Error-Message Guidelines*, *User Control and Freedom*.
- Alan Cooper et al., *About Face*: undo and the case against confirmation dialogs.
- Google, *Site Reliability Engineering* (free online): postmortem culture, safe change management, canarying.
- Public incident reports: GitLab.com database incident postmortem (Jan 2017); AWS *Summary of the Amazon S3 Service Disruption in the Northern Virginia Region* (Feb 2017); FCC report on the Hawaii false ballistic missile alert (2018).
- ISMP (Institute for Safe Medication Practices): tall-man lettering, error-prone abbreviations; a rich source of error-prevention design in healthcare.
- Shigeo Shingo, *Zero Quality Control: Source Inspection and the Poka-Yoke System*.
- deceptive.design: catalogue of dark patterns, including confirmshaming and obstruction.
- Related notes: [Usability](../usability/README.md) · [Reliability](../../runtime-operational/reliability/README.md) · [Accessibility](../accessibility/README.md) · [Responsiveness](../responsiveness/README.md).
