# Projected lifetime

> **Projected lifetime** = how long the system is **expected to stay in service**, and whether its design, dependencies, data and team knowledge will last that long.
> "Is this a three-month experiment or a twenty-year system of record, and are we building it accordingly?"

Related: [Evolvability](../../development-evolution/evolvability/README.md) (how the architecture adapts over those years), [Maintainability](../../development-evolution/maintainability/README.md), [Time to market](../time-to-market/README.md) (the opposite pull), [Vendor independence](../vendor-independence/README.md) (will your suppliers last as long as you?), [Replaceability](../../portability-compatibility/replaceability/README.md).

Evolvability is about *how* a system changes. Projected lifetime is the planning input that tells you **how much to invest** in that, and what will run out before the system does.

---

## 1. Why it matters

The expected lifetime changes almost every design decision:

| | Short-lived (weeks to a year) | Long-lived (5–20+ years) |
|---|---|---|
| Examples | Prototype, campaign site, one-off migration tool, hackathon demo | Core banking, medical records, government registries, industrial control, a company's main product |
| Optimise for | Speed and learning | Total cost over the whole life |
| Technology choice | Whatever the team is fastest with | Mature, widely used, with long support windows |
| Tests and documentation | Minimal | Essential: the authors won't be there |
| Data | Disposable | Outlives every version of the code |
| Dependencies | Latest versions, few worries | Each one is a multi-year maintenance commitment |

Two expensive mistakes:
- **Over-building the short-lived**: months of architecture for something that's thrown away.
- **Under-building the long-lived**: more common, because **successful prototypes don't die**. "Temporary" systems regularly run for a decade.

So: state the expected lifetime explicitly, and decide in advance what happens if it's exceeded.

---

## 2. Things that expire before the system does

| What expires | Typical timescale | Consequence |
|---|---|---|
| **Runtime and framework versions** | A few years of support per major version | No security patches after end of life (EOL) |
| **Operating systems and databases** | Around 5 years of standard support for many LTS releases and major versions | Forced upgrades, sometimes with breaking changes |
| **Third-party APIs and SaaS** | Versions deprecated on the vendor's schedule; products get shut down | Rework on someone else's timetable |
| **Hardware** | Roughly 3–7 years in service | Replacement, spare parts, driver support |
| **Cryptography** | Algorithms and key sizes weaken over time (MD5, SHA-1, short RSA keys) | Re-encryption, protocol upgrades; plan for algorithm agility |
| **Certificates, domains, licences** | Months to years | Outages when nobody remembers to renew |
| **Counters and formats** | When they overflow | Two-digit years (Y2K), 32-bit timestamps (2038), 32-bit IDs ([Evolvability §4](../../development-evolution/evolvability/README.md#4-reversible-and-irreversible-decisions)) |
| **Team knowledge** | Typical tenure is a few years | The people who know why it works leave |
| **Vendors and open-source projects** | Unpredictable | Abandoned libraries, acquisitions, licence changes ([Licensing](../licensing/README.md)) |

### Worked example: a lifetime calendar

A system is built in **2026** and expected to run **10 years**. Its stack, with *illustrative* support windows:

```
Component            Version support ends     Upgrades needed before 2036
Language runtime     ~3 years after release   3
Web framework        ~3 years (LTS)           3
Database             ~5 years per major       2
Operating system     ~5 years (LTS)           2
TLS certificates     ≤ 1 year                 automated, or 10+ manual renewals
Payment provider API version deprecations     unknown: at least 2 assumed
```

That is **at least 12 planned upgrades** over the system's life, before any feature work. If each one takes two engineer-weeks on average:

```
12 upgrades × 2 weeks = 24 engineer-weeks ≈ half an engineer-year of pure upkeep
```

Skipping them doesn't remove the cost; it turns twelve small upgrades into one large, risky migration from several versions behind. Look up the real dates for your stack (vendor lifecycle pages, endoflife.date) and put them in a calendar.

---

## 3. Designing for a long life

- **Data first.** Data outlives code. Use well-understood schemas, documented meanings, open formats, and a tested way to export everything. A database migration is survivable; undocumented data isn't.
- **Boring, widely used technology** with long support windows and large communities. The **Lindy effect** is a useful rule of thumb: technology that has been around for a long time is likely to stay around.
- **Standards over products**: SQL, HTTP, POSIX, OCI images, OpenTelemetry. Standards outlive the vendors that implement them.
- **Few dependencies, kept current.** Every dependency is something that can be abandoned. Upgrade continuously in small steps (automated update PRs) instead of every few years ([Reusability §4](../../development-evolution/reusability/README.md#4-dependency-hygiene)).
- **Isolate what will change.** Put vendors, frameworks and protocols behind interfaces you own, so they can be replaced one at a time ([Replaceability](../../portability-compatibility/replaceability/README.md)).
- **Room to grow in formats**: 64-bit IDs, timezone-aware timestamps, version fields in messages and files, Unicode.
- **Reproducible builds**: pinned dependencies, a private copy of artifacts, the build defined in code. Can you rebuild the system in five years from what's in the repository?
- **Knowledge that survives people**: architecture decision records, runbooks, automated tests as executable documentation, onboarding that's actually tried on new people.
- **A maintenance budget**: a fixed share of capacity for upgrades and upkeep, every quarter, not "when there's time".

---

## 4. Designing for a short life

- Say it's temporary, **with a date and an owner**. Put the expiry in the README and in a calendar.
- Keep it **isolated**: its own repository, database and credentials, so that deleting it is easy and it can't become a hidden dependency.
- Don't let production data or other systems **depend** on it without a decision.
- Decide the **promotion rule** up front: "if this is still running in six months, or has more than N users, we rewrite or harden it". Promoting a prototype means paying for the tests, security, monitoring and documentation that were skipped.

---

## 5. End of life: plan the exit too

Every system is eventually replaced or switched off. Decommissioning is part of the design:

- **Announce and deprecate** with dates; give API consumers a migration path and a long notice ([Backward compatibility](../../portability-compatibility/backward-compatibility/README.md)).
- **Migrate or archive data** according to retention law and contracts; some records must stay readable for decades after the software is gone.
- **Find the real users first**: check logs and traffic, don't trust the documentation.
- **Remove** infrastructure, DNS records, credentials, scheduled jobs, integrations and monitoring. Forgotten remains become security holes and costs.
- **Migrate incrementally** where the replacement is large ([Evolvability §7](../../development-evolution/evolvability/README.md#7-migration-patterns)).

---

## 6. Trade-offs

| Longevity vs | Tension |
|---|---|
| **Time to market** | Durable choices take longer up front |
| **Innovation** | Mature technology means giving up newer, possibly better tools |
| **Cost now vs later** | Upkeep is a steady cost; skipping it is cheaper this quarter and much dearer later |
| **Simplicity** | Abstraction layers for future replacement add complexity today; add them where change is likely, not everywhere |
| **Features** | A maintenance budget competes directly with the roadmap |

---

## 7. Anti-patterns

- **"It's only temporary"** with no date, no owner and no promotion rule.
- **Gold-plating a throwaway**: frameworks and abstractions for a demo.
- **Upgrade debt**: running unsupported versions because upgrading "isn't a priority".
- **The planned rewrite as maintenance strategy**: assuming the system will be replaced before its problems matter.
- **Single person knowledge** (bus factor of one).
- **Data in formats only one application can read.**
- **Dependencies on one-maintainer or unmaintained libraries** for critical functions.
- **Never switching anything off**: old systems kept running because nobody knows who uses them.

---

## 8. Checklist

- [ ] Expected lifetime stated, with what happens if it's exceeded
- [ ] End-of-life dates of runtime, framework, OS, database and key APIs known and in a calendar
- [ ] Dependencies updated continuously; none unsupported
- [ ] Data documented, exportable in open formats, and readable without the application
- [ ] IDs, timestamps and counters sized for the whole lifetime
- [ ] System can be rebuilt from the repository alone
- [ ] Decisions and operations documented; more than one person understands each part
- [ ] A recurring budget for upkeep
- [ ] A decommissioning plan exists for the systems this one replaces

---

## 9. Exercises

1. Build the lifetime calendar for a project you maintain: every runtime, framework, database, OS and external API with its real end-of-life date. What expires in the next 18 months?
2. A prototype built in two weeks now has 5,000 users. List what was skipped and estimate the cost of hardening it compared with rewriting it.
3. You must store records that stay readable for 30 years. Choose formats, storage and documentation. What will certainly have changed by then?
4. Find a counter, ID or timestamp in a system you know. When does it overflow at the current growth rate, and at 10× that rate?

---

## 10. Further reading

Titus Winters, Tom Manshreck & Hyrum Wright, *Software Engineering at Google* (ch. 1: "software engineering is programming integrated over time"; Hyrum's Law); Fred Brooks, *The Mythical Man-Month*; Michael Feathers, *Working Effectively with Legacy Code*; Dan McKinley, "Choose Boring Technology"; Marianne Bellotti, *Kill It with Fire* (modernising old systems); endoflife.date (support dates for common software); Library of Congress, "Sustainability of Digital Formats"; Joel Spolsky, "Things You Should Never Do, Part I" (on rewrites).
