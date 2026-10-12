# Reusability

> **Reusability** = parts of the system can be **used again** in other places or projects, saving effort without creating harmful coupling.
> "Should these three services share a library, or should each have its own copy?"

In **ISO/IEC 25010**, reusability is a sub-characteristic of maintainability: the degree to which an asset can be used in more than one system or in building other assets.

Related: [Modularity](../modularity/README.md) (good modules are easier to reuse), [Simplicity](../simplicity/README.md) (the wrong abstraction is worse than duplication), [Evolvability](../evolvability/README.md) (shared code is harder to change).

---

## 1. Levels of reuse

| Level | Example | Coupling it creates |
|---|---|---|
| **Copy (duplication)** | Copy a function into another project | None (but fixes don't propagate) |
| **Shared function / module** | A `Money` type inside one codebase | Low, same repo and release |
| **Internal library** | Company-wide auth or logging library | Version coupling across teams |
| **Open-source dependency** | `requests`, `lodash`, a web framework | Coupling to external maintainers, security risk |
| **Shared service** | One authentication service used by all products | Runtime coupling (availability, latency) |
| **Platform / golden path** | Templates, CI pipelines, a design system | Organisational coupling, but huge leverage |

---

## 2. The cost of reuse

Reuse looks free but isn't:
- **Building for reuse is harder**: more general API, documentation, versioning, support. Fred Brooks estimated a generalised, tested, documented "programming product" costs about **3×** a program that works only for its author.
- **Every user is coupled** to the shared thing: a change must work for all of them, so it changes slowly.
- **The wrong abstraction** is worse than duplication: code forced to serve cases that only look similar fills up with flags and special cases (Sandi Metz).
- **Dependencies are a supply chain**: you inherit their bugs, vulnerabilities and maintainers' decisions.

**Rule of three**: write it, duplicate it once if needed, and extract a shared version the **third** time, when you can see what's really common.

### Worked example: the shared `common` library

30 services depend on `company-common` v2, which has HTTP clients, date helpers, auth, logging and some order logic.

```
A breaking change in one helper → 30 services must upgrade
If each upgrade takes 2 hours of one team's time: 60 engineer-hours
Teams that can't upgrade stay on v2 → you now maintain v2 and v3
```

Better:
- **Split** it into small, focused libraries (`logging`, `auth-client`), each versioned on its own.
- **Remove domain logic** from shared libraries (order logic belongs to the orders service).
- Keep shared libraries **stable and small**; use semantic versioning and automated upgrade PRs (Renovate).
- For small helpers, **duplication is fine**.

---

## 3. When reuse pays off

- **Stable, well-understood problems**: logging, auth clients, HTTP retries, money and date handling, UI components.
- **Cross-cutting standards** you *want* to be identical everywhere: telemetry setup, security defaults, error formats.
- **Design systems** for UI: consistent look and accessibility for free.
- **Platforms and templates**: new services start with CI, observability and deployment built in.
- **Mature open source** instead of writing your own (with dependency hygiene).

---

## 4. Dependency hygiene

- Know what you depend on: lockfiles, SBOMs, `npm ls` / `pip tree`.
- Scan for vulnerabilities (OSV-Scanner, Dependabot, Trivy) and update continuously.
- Prefer few, well-maintained dependencies over many tiny ones.
- Pin versions, use a private mirror/proxy for critical builds.

---

## 5. Anti-patterns

- A `common`/`utils` library that everything depends on and everyone changes.
- Extracting a shared abstraction from the first or second use case.
- Shared libraries containing business logic of one domain.
- Reusing a service's database instead of its API.
- Hundreds of tiny transitive dependencies nobody reviews.
- "Not invented here": rewriting solid open-source tools.

---

## 6. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **left-pad (Mar 2016)** | The author unpublished an 11-line npm package that many popular packages depended on (indirectly). Builds across the JavaScript ecosystem broke until it was restored. npm changed its unpublish policy afterwards. | Tiny dependencies are still dependencies; know your transitive tree. |
| **Log4Shell (Dec 2021)** | A critical remote code execution flaw in Log4j, a logging library reused almost everywhere in the Java ecosystem, affected countless systems at once. | Reuse spreads vulnerabilities as efficiently as features. Inventory and patch fast. |
| **Ariane 5 (1996)** | Software reused from Ariane 4 failed under Ariane 5's different flight conditions ([Testability §12](../testability/README.md#12-real-world-stories)). | Reused components carry assumptions from their original context. |

---

## 7. Checklist

- [ ] Shared code extracted only after the third real use
- [ ] Shared libraries small, focused, versioned, without domain logic
- [ ] Automated upgrade PRs for internal and external dependencies
- [ ] Dependency inventory (SBOM) and vulnerability scanning
- [ ] Design system / templates for things that should be consistent

---

## 8. Exercises

1. Find a shared library or `utils` module in a codebase you know. Who depends on it, and how often does it change? Should it be split, slimmed or copied?
2. Count your project's direct and transitive dependencies. How many have had a release in the last year? Any known vulnerabilities?
3. Find an abstraction with several boolean flags for different callers. Inline it back into the callers and see whether separate code is clearer.
4. Decide what belongs in a company-wide "service template" (shared) and what each team should own.

---

## 9. Further reading

Sandi Metz, "The Wrong Abstraction"; Fred Brooks, *The Mythical Man-Month* ch. 1 (program vs programming systems product); Russ Cox, "Our Software Dependency Problem" (2019); *Software Engineering at Google* ch. 21 "Dependency Management"; OpenSSF Scorecard; Brad Frost, *Atomic Design* (design systems).
