# Reusability: hands-on practice notes

Planned labs for learning reusability by **extracting, versioning and upgrading shared code, and auditing what you reuse from others**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)) plus a second small service, a local package registry (**Verdaccio** for npm, **devpi** or **pypiserver** for Python), and **OSV-Scanner**.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** dependency upgrade drill (evolvability lab 8), supply chain basics (deployability lab 11).

---

## Labs

### 1. Extract and version a shared library ⭐
- **Do:** messy-shop and a second service both need the same `Money` type and an HTTP client with retries and tracing. Extract them into a library, publish it to a local registry with semantic versioning.
- **Then:** make a breaking change (v2). Upgrade one service, leave the other on v1. Note what it costs to maintain both versions.
- **See:** README §2 worked example.

### 2. Dependency audit
- **Do:** for messy-shop, list direct and transitive dependencies (`npm ls --all`, `pipdeptree`), generate an SBOM (**Syft**), and scan with **OSV-Scanner**.
- **Find:** the deepest dependency chain, unmaintained packages, known vulnerabilities. Remove or replace at least one dependency.

### 3. Wrong abstraction → duplication → right abstraction
- **Do:** find (or write) a shared function with flags for different callers (`format_price(amount, for_email=False, short=False, admin=False)`). Inline it back into each caller, then look for what's *really* common and extract only that.
- **Compare:** lines, branches and readability before and after.

### 4. A tiny design system (optional, frontend)
- **Do:** build 5 shared UI components (button, input, alert, modal, table) in **Storybook**, used by two small apps. Include accessibility checks (Storybook a11y addon).
- **Then:** change the button's API and handle the upgrade in both apps.

---

## Progress

- [ ] 1. Extract and version a shared library
- [ ] 2. Dependency audit
- [ ] 3. Wrong abstraction → duplication → right abstraction
- [ ] 4. A tiny design system
