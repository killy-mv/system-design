# Analyzability: hands-on practice notes

Planned labs for learning analyzability by **measuring how hard code is to understand, and making the worst parts clear**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)) and one unfamiliar open-source repository.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** change coupling (modularity lab 2), change effort from Git (modifiability lab 2), the "new engineer" test (simplicity lab 6).

---

## Labs

### 1. Find the worst functions ⭐
- **Do:** run a complexity tool on messy-shop: **SonarQube Community** (Docker) or **eslint-plugin-sonarjs** for cognitive complexity; **radon** / **lizard** for cyclomatic complexity.
- **Refactor:** the top 3 functions with guard clauses, named predicates and extracted functions (README §2 worked example). Tests must stay green.
- **Add:** a CI rule failing on new functions above a complexity threshold.

### 2. Hotspot map
- **Do:** combine Git churn with complexity (a script, or **code-maat**) for messy-shop and a real repo you know. Plot or list the top 10 hotspots.
- **Discuss:** which hotspot would you clean up first, and why that one?
- **See:** README §3.

### 3. Architecture docs that answer questions
- **Do:** write C4 context and container diagrams (Structurizr DSL or Mermaid) and a one-page README per module for messy-shop.
- **Test:** write 5 questions ("where are refunds calculated?", "what happens after an order is paid?"). Have someone answer them with and without the docs. Time both.

### 4. Reading an unfamiliar codebase
- **Do:** pick an open-source project in your language. Give yourself 3 tasks: find where a feature lives, explain a request's path through the code, predict the impact of changing a function.
- **Use:** IDE "find usages", call hierarchies, tests as documentation, traces if it runs. Note which tools and which properties of the code helped most.

---

## Progress

- [ ] 1. Find the worst functions
- [ ] 2. Hotspot map
- [ ] 3. Architecture docs that answer questions
- [ ] 4. Reading an unfamiliar codebase
