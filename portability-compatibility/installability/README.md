# Installability

> **Installability** = how easily and reliably a system can be **installed, upgraded and removed** in a target environment.
> "A new user (or a new engineer, or a new server) starts from nothing. How long until it works, and how many things can go wrong on the way?"

Related: [Portability](../portability/README.md) (which environments it can run in at all), [Adaptability](../adaptability/README.md), [Deployability](../../operations-devops/deployability/README.md) (releasing your own service), [Configurability](../../operations-devops/configurability/README.md), [Learnability](../../UI-UX/learnability/README.md) (the first-run experience after install).

---

## 1. What "install" means by product type

| Product | "Install" is | Good looks like |
|---|---|---|
| **CLI tool** | Getting one binary on the `PATH` | `brew install x` / `apt install x` / download one static binary |
| **Library / SDK** | Adding a dependency | One package-manager command; few transitive dependencies; no native build step |
| **Desktop app** | Installer or app store | Signed installer, no admin rights needed, auto-update |
| **Mobile app** | Store download + first launch | Small download, permissions asked when needed, not all up front |
| **Self-hosted server software** | Running it on the customer's infrastructure | Container image, Helm chart or package; documented requirements; an upgrade path |
| **SaaS** | Sign-up and onboarding | Time to first value in minutes; nothing to install |
| **Internal service** | A new engineer's development environment | Clone, one command, running tests within the first hour |

The useful metric is the same in every row: **time to first success**, and the share of people who get there without help.

---

## 2. Properties of a good installation

| Property | Meaning |
|---|---|
| **Few steps** | Each manual step loses some users and is a chance for error |
| **Prerequisites checked** | The installer says "needs PostgreSQL ≥ 14, found 12" instead of failing later with a stack trace |
| **Idempotent** | Running it twice is safe; a failed install can be re-run |
| **Unattended** | A silent/non-interactive mode with flags or an answer file, so it can be automated |
| **Sensible defaults** | Works out of the box; configuration is for people who need it |
| **Verifiable** | A `doctor`, `--version` or health endpoint that confirms the install works |
| **Upgradable** | In-place upgrades with data migrations, a documented supported upgrade path, and a way back |
| **Removable** | Uninstall leaves no services, files or registry entries behind (and says what it does with user data) |
| **Offline-capable** | Can be installed without internet access where customers need that (air-gapped environments) |
| **Trustworthy** | Signed artifacts, checksums, least privilege |

---

## 3. Distribution mechanisms

| Mechanism | Good for | Watch out for |
|---|---|---|
| **OS package managers** (apt, dnf, Homebrew, winget) | CLIs, daemons; upgrades and removal for free | One package format per platform to maintain |
| **Language package managers** (pip, npm, cargo, Maven) | Libraries, developer tools | Needs the right runtime already installed; dependency conflicts |
| **Single static binary** | CLIs and small servers (common with Go and Rust) | You handle updates yourself |
| **Container image** | Servers; carries its dependencies | Needs a container runtime; configuration and data volumes must be documented |
| **Helm chart / Kubernetes operator** | Multi-component systems on Kubernetes | Chart complexity; upgrades of stateful parts |
| **Native installers** (MSI, pkg, AppImage, Flatpak) and app stores | Desktop and mobile | Signing, notarisation, store review |
| **`curl … \| sh` script** | A quick start | Runs unreviewed code; must be HTTPS, short, readable and idempotent |

---

## 4. Worked example: from 14 steps to one command

messy-shop's original setup guide for a new engineer:

```
1. Install Python 3.11 (not 3.12, a dependency breaks)
2. Install PostgreSQL 15 and create a user "shop" with password "shop"
3. Install Redis
4. brew install libpq imagemagick        (on Linux: ask Sam)
5. cp .env.example .env and fill in 9 values (ask in chat for the payment sandbox key)
6. pip install -r requirements.txt
7. python manage.py migrate
8. python manage.py loaddata seed.json   (fails unless step 7 ran twice)
...
14. If the port is taken, edit settings.py
```

Measured on three new joiners: between half a day and two days until the tests ran, each with at least one question to a teammate.

After the rewrite:

```bash
git clone … && cd messy-shop
docker compose up        # app + Postgres + Redis, migrations and seed data run on start
make doctor              # checks versions, connectivity, required env vars
```

| Change | Why it helps |
|---|---|
| Runtime and services defined in `compose.yaml`, versions pinned | No "which Python?" and no local database setup |
| Migrations and seed run on start, idempotently | No ordering mistakes; re-running is safe |
| `.env` has working defaults for local use; secrets fetched by one documented command | Nothing to ask a person for |
| `make doctor` | Problems are reported by name, with the fix |
| The setup runs in CI on every commit | The instructions can't silently rot |

Time to first passing test: about 15 minutes, mostly image downloads.

---

## 5. Upgrades and removal

Installation is done once; **upgrading is done for years**.

- State the **supported upgrade paths** ("upgrade to 4.2 before going to 5.x").
- Run **data migrations** automatically, and make them resumable ([Deployability §9](../../operations-devops/deployability/README.md#9-database-and-schema-changes)).
- **Back up before upgrading**, or tell the user to, and document the rollback.
- **Keep the user's configuration**: never overwrite edited config files; merge or write a `.new` file next to them.
- Keep old config keys working for a while ([Backward compatibility](../backward-compatibility/README.md)).
- Uninstall removes the software; removing user data is a separate, explicit choice.

---

## 6. Security of the install path

- Sign artifacts and publish checksums (Sigstore cosign for images, GPG-signed package repositories, code signing on Windows and macOS).
- Install with the **least privilege** that works; don't ask for root/admin unless needed.
- No default passwords. Generate one at install time or force a change on first login.
- Pin dependencies so an install today and one next month get the same code.

---

## 7. Anti-patterns

- Setup instructions that only work on the author's machine.
- An installer that fails halfway and can't be re-run.
- Interactive prompts with no unattended mode.
- Default credentials (`admin` / `admin`) left in place.
- Upgrades that silently reset configuration.
- "Just pull `latest`."
- No uninstall, or one that leaves services running.

---

## 8. Checklist

- [ ] Time to first success measured on a clean machine by someone new
- [ ] Prerequisites checked up front with clear messages
- [ ] Install is idempotent and has an unattended mode
- [ ] A verification command or health check exists
- [ ] Upgrade paths documented and tested from the previous supported versions
- [ ] Uninstall is clean; user data is handled explicitly
- [ ] Artifacts are signed and versions pinned
- [ ] The install is exercised in CI

---

## 9. Exercises

1. Time a clean install of a project you built, following only the README. Write down every step that was missing.
2. Compare how three tools you use are installed (for example a database, a CLI and an editor). Which properties from §2 does each have?
3. Design the upgrade from v1 to v2 for self-hosted software with a database: what happens if the migration fails halfway?
4. A customer's servers have no internet access. What do you ship, and how do they verify it?

---

## 10. Further reading

*The Twelve-Factor App* (factors II "Dependencies" and V "Build, release, run"); Homebrew, Debian and Helm packaging documentation; Sigstore documentation (cosign); the Command Line Interface Guidelines (clig.dev); Docker docs on Compose; GitLab's and Sentry's self-hosted upgrade guides as examples of documented upgrade paths; ISO/IEC 25010 (installability).
