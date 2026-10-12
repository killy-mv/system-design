# Projected lifetime: hands-on practice notes

Planned labs for learning projected lifetime by **finding out what expires first, rebuilding an old project, and switching something off properly**. Theory is in [README.md](README.md).

They need a **project that's at least two years old** (your own, or an open-source one checked out at an old tag), **Docker**, a **spreadsheet or calendar**, and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)).

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** the 32-bit ID wall and the major dependency upgrade drill (evolvability labs 6 and 8), write ADRs (evolvability lab 1), dependency audit (reusability lab 2), certificates on autopilot and dependency updates (manageability labs 3 and 4).

---

## Labs

### 1. Lifetime calendar ⭐
- **Do:** list every runtime, framework, database, base image, OS and external API version that messy-shop (or your own project) depends on. Look up each real end-of-life date (vendor lifecycle pages, endoflife.date).
- **Build:** a calendar or table sorted by date, with an owner and an estimated effort for each upgrade.
- **Measure:** how many are already past end of life? How many upgrades over a 10-year life?
- **See:** README §2 worked example.

### 2. Rebuild a two-year-old project ⭐
- **Do:** check out a commit from two or more years ago and try to build and run it from scratch on a clean machine or container. Note every failure: missing package versions, removed base images, dead download URLs, expired keys.
- **Fix:** make today's version reproducible: lock files, pinned base image digests, a documented toolchain version, dependencies mirrored or vendored.
- **Check:** can someone else build it using only the repository?
- **See:** README §3.

### 3. Data that outlives the application
- **Build:** an export of all messy-shop data to open formats (CSV or Parquet, plus JSON for nested data) with a data dictionary describing every field, unit, enum value and timezone.
- **Check:** delete the application code from your head: using only the export and the dictionary, answer "what was order 1234, what did it cost, and what happened to it?" with a different tool (DuckDB, a spreadsheet).
- **See:** README §3, §5.

### 4. Find what overflows
- **Do:** list every bounded thing in messy-shop: integer primary keys, sequence sizes, timestamp types, fixed-width columns, 2-digit anything, counters, partition schemes.
- **Measure:** at the current insert rate and at 10×, when does each run out? Insert rows in bulk to confirm the failure mode for one of them.
- **See:** README §2, [Evolvability §4](../../development-evolution/evolvability/README.md#4-reversible-and-irreversible-decisions).

### 5. Decommission something
- **Do:** pick a small feature, endpoint or service (the legacy "admin reports" in messy-shop works). Write a decommissioning plan: how you prove who still uses it, the notice period, where its data goes, and the removal checklist.
- **Then:** carry it out: log callers for a week (or simulate), return a deprecation header, then remove the code, tables, config, credentials, scheduled jobs and dashboards.
- **Check:** search the repository and infrastructure for leftovers a month later.
- **See:** README §5.

---

## Progress

- [ ] 1. Lifetime calendar
- [ ] 2. Rebuild a two-year-old project
- [ ] 3. Data that outlives the application
- [ ] 4. Find what overflows
- [ ] 5. Decommission something
