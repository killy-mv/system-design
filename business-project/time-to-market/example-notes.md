# Time to market: hands-on practice notes

Planned labs for learning time to market by **measuring where delivery time really goes, then shortening the slowest step**. Theory is in [README.md](README.md).

They need a **Git repository with real history** (your own project or an open-source one), a **spreadsheet**, **GitHub Actions**, and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)).

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** a real CI/CD pipeline and feature flags (deployability labs 1 and 5), make CI fast (testability lab 8), measure change effort from Git (modifiability lab 2).

---

## Labs

### 1. Value stream map ⭐
- **Do:** pick three recent changes in a project you know. For each, reconstruct the timeline from issue, pull request and deploy timestamps: created → started → PR opened → first review → merged → in production.
- **Measure:** working vs waiting time per step, total lead time, flow efficiency.
- **Check:** name the single largest queue and one change that would shrink it.
- **See:** README §2 worked example.

### 2. DORA metrics from Git and CI ⭐
- **Build:** a script that reads Git tags or deployment records and computes deployment frequency and lead time for changes (commit time → deploy time) per week. Add change fail rate if you can mark failed deploys (reverts, hotfix tags).
- **Measure:** plot 3 months. Is lead time stable, improving or getting worse?
- **Optional:** compare with an existing tool such as the open-source DORA "Four Keys" project or your Git host's built-in metrics.
- **See:** README §2.

### 3. Thin vertical slice in a day
- **Build:** add "gift message on an order" to messy-shop as the thinnest possible end-to-end slice: one field, one API change, one screen or endpoint, one test, deployed behind a feature flag.
- **Measure:** time from first commit to running in your environment. Write down everything you deliberately left out.
- **Then:** widen the slice in two more small releases instead of one big one.
- **See:** README §4.

### 4. Build vs buy sprint
- **Do:** add login to messy-shop twice, with a time box of half a day each: once hand-written (sessions, password hashing, reset flow), once with a library or hosted identity provider (e.g. Keycloak in Docker, or an auth library for your framework).
- **Measure:** time to a working login, and the list of things still missing in each (password reset, MFA, lockout, audit log).
- **Check:** what would it take to switch away from the option you bought?
- **See:** README §4, [Vendor independence](../vendor-independence/README.md).

### 5. Cost of delay ordering
- **No code.** Take a real or invented backlog of 6–8 items. Estimate value per week and duration for each, then compute the total delay cost for three orderings: biggest value first, shortest first, cost of delay ÷ duration.
- **Then:** halve the duration of the largest item by cutting its scope and recompute.
- **See:** README §5 worked example.

---

## Progress

- [ ] 1. Value stream map
- [ ] 2. DORA metrics from Git and CI
- [ ] 3. Thin vertical slice in a day
- [ ] 4. Build vs buy sprint
- [ ] 5. Cost of delay ordering
