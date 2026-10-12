# Operability

> **Operability** (in UI/UX) = the degree to which a product has attributes that make it **easy to operate and control**: users can start, steer, pause, stop and adjust what the system does, with input methods that suit them, and the system behaves as they expect. (ISO/IEC 25010 sub-characteristic of usability / interaction capability.)
> "Am I in control of this system, or is it in control of me?"

⚠ **Two meanings.** In operations and DevOps, "operability" means how easy a system is to **run in production** (deploy, monitor, debug, scale, recover). That meaning lives in the [operations-devops](../../operations-devops/) notes (e.g. [configurability](../../operations-devops/configurability/README.md), [debuggability](../../operations-devops/debuggability/README.md)). This note is about the **end-user** meaning.

Operability overlaps with most other UI/UX attributes. It's the umbrella for **"can the user actually drive it"**, while [learnability](../learnability/README.md) is about learning, [efficiency](../efficiency-of-use/README.md) about speed, and [user error protection](../user-error-protection/README.md) about mistakes.

---

## 1. ISO 9241-110 interaction principles

ISO 9241-110 (2020) lists seven principles for interactive systems. Together they describe what "easy to operate and control" means:

| Principle | Meaning | Examples |
|---|---|---|
| **Suitability for the user's tasks** | Supports the task without unnecessary steps | Fields in the order the paper form uses; defaults that match the common case |
| **Self-descriptiveness** | It's clear what's happening, where you are, what you can do | Clear labels, visible state, progress indicators, breadcrumbs |
| **Conformity with user expectations** | Behaves according to conventions and the user's knowledge | Standard controls, consistent behavior ([Consistency](../consistency/README.md)) |
| **Learnability** | Supports learning to use it | See [Learnability](../learnability/README.md) |
| **Controllability** | The user can start, pace, direct, interrupt and undo interactions | Pause/cancel, undo, back, adjustable speed, no forced flows |
| **Use error robustness** | Prevents errors, tolerates them, helps recovery | See [User error protection](../user-error-protection/README.md) |
| **User engagement** | Motivating, trustworthy, pleasant to use | See [Satisfaction](../satisfaction/README.md), [Aesthetics](../aesthetics/README.md) |

---

## 2. Controllability in practice

The heart of operability, and of Nielsen's heuristic *User control and freedom*:

- **Exits everywhere**: Cancel, Back, Close, Esc, "Skip for now". No dead ends or forced flows.
- **Interruptible long operations**: pause, resume and cancel uploads, exports, imports, installs, playback ([Responsiveness §10](../responsiveness/README.md#10-long-running-operations)).
- **Undo and redo** for user actions.
- **User-controlled pacing**: no auto-advancing carousels or timed steps without pause; adjustable or extendable timeouts (WCAG 2.2.1, 2.2.2).
- **No surprises**: the system doesn't act on its own in unexpected ways (auto-play, auto-submit, focus jumps, layout shifts, unrequested changes). Automation and AI agents should show what they're about to do and let users approve, adjust or stop it.
- **Control over automation level**: users can turn off auto-correct, auto-formatting, smart suggestions, auto-renewals.
- **Settings and preferences**: notification controls, defaults, display density, language, units, keyboard shortcuts.

---

## 3. Operable with every input method

A product is only operable if people can control it with the input they actually use:

| Input | Requirements |
|---|---|
| **Mouse / trackpad** | Adequate target sizes; hover isn't the only way to reveal actions |
| **Touch** | Targets ≥ 44pt / 48dp (WCAG minimum 24 CSS px); no precision-dependent gestures; thumb-reachable primary actions |
| **Keyboard** | Everything reachable and operable; visible focus; logical order; no traps |
| **Screen readers, switch access, voice control** | Correct names, roles and states; label in name for voice commands |
| **Gestures** | Every multi-finger, path-based or drag gesture has a simple alternative (WCAG 2.5.1, 2.5.7) |
| **Game controllers, TV remotes, car controls** | D-pad-friendly focus navigation; large targets; minimal text entry |

WCAG's whole **Operable** principle (keyboard, enough time, seizures, navigation, input modalities) is a checklist for this ([Accessibility §5](../accessibility/README.md#5-wcag-the-standard)).

---

## 4. Visible system status

Users can only control what they can see:

- Current state of toggles, filters, modes and selections is always visible (and not by color alone).
- Background activity is visible: syncing, uploading, scheduled jobs, automation running.
- Mode indicators for anything that changes how input is interpreted (editing vs viewing, test vs live, recording).
- Clear feedback after every action ([Usability §8.4](../usability/README.md#84-feedback-and-ui-states)).

---

## 5. System-design aspects

- **Cancellable operations end to end**: cancellation propagates from UI to API to workers (job cancel endpoints, `AbortController`, context deadlines).
- **Resumable operations**: resumable uploads, checkpointed jobs, drafts.
- **Status and progress APIs** for long-running work.
- **Preferences stored server-side** and applied across devices and channels (email, push, SMS).
- **Explicit consent and controls for automation** (auto-renewal, auto-pay, AI actions), with audit trails users can see.
- **Rate limits and timeouts** designed so clients can show clear status rather than hanging.

---

## 6. Measuring operability

- **Task success and errors** in usability tests, including keyboard-only, touch-only and screen-reader sessions.
- **Abandonment** of long or forced flows; use of Cancel/Back/Undo.
- **Accessibility audits** of the Operable principle.
- **Support tickets** like "how do I stop/cancel/turn off…".

---

## 7. Checklist and further reading

- [ ] Every flow has a clear way out (Cancel / Back / Esc / Skip)
- [ ] Long operations can be paused, cancelled and resumed
- [ ] Undo for user actions; no unexpected automatic actions
- [ ] Users control timing, automation and notifications
- [ ] Fully operable by keyboard, touch, screen reader and voice; gesture alternatives
- [ ] System status and modes always visible

**Further reading**: ISO 9241-110:2020 *Interaction principles*; ISO/IEC 25010; Nielsen Norman Group, *User Control and Freedom: Usability Heuristic #3*; W3C WCAG 2.2, Principle 2 *Operable*.
