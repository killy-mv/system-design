# Installability: hands-on practice notes

Planned labs for learning installability by **installing from nothing, timing it, and then making it one command**. Theory is in [README.md](README.md).

They need **Docker Compose**, a clean VM or container to install into, and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)). Lab 4 needs a local Kubernetes (**kind** or **k3d**) and **Helm**.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** the clean-machine test (portability lab 1), validate config and fail fast (configurability lab 1), zero-downtime schema change (deployability lab 6).

---

## Labs

### 1. Measure time to first success ⭐
- **Do:** on a clean machine, follow the current setup instructions with a stopwatch. Better: watch someone else do it and don't help.
- **Record:** total time, number of manual steps, every question asked, every error.
- **See:** README §1, §4.

### 2. One-command setup with a doctor ⭐
- **Build:** `docker compose up` starts the app and its dependencies, runs migrations and seed data idempotently. Add a `doctor` command that checks runtime version, database connectivity and required environment variables, and prints the fix for each failure.
- **Break:** stop the database, unset a variable, use a wrong version. Check each message names the problem.
- **Then:** run the setup in CI so it can't rot. Repeat lab 1 and compare.

### 3. Idempotent, unattended installer
- **Build:** an install script for a small CLI or daemon: non-interactive flags, checks prerequisites first, safe to run twice, verifies itself at the end, and has an uninstall.
- **Break:** kill it halfway (`kill -9`) and run it again. It must finish cleanly.
- **Check:** after uninstall, compare the file system before and after (`find` + `diff`).
- **See:** README §2.

### 4. Package it for others
- **Build:** a Helm chart for the app with sensible defaults in `values.yaml`, a readiness probe and a `helm test`. Install on kind.
- **Then:** sign the image with cosign and verify the signature before install.
- **See:** README §3, §6.

### 5. Upgrade with a way back
- **Build:** v1 and v2 of the app, where v2 has a schema migration. Upgrade a running v1 install with data in it.
- **Break:** make the migration fail halfway. Does the upgrade resume? Can you return to v1?
- **Check:** a user-edited config file survives the upgrade.
- **See:** README §5.

---

## Progress

- [ ] 1. Measure time to first success
- [ ] 2. One-command setup with a doctor
- [ ] 3. Idempotent, unattended installer
- [ ] 4. Package it for others
- [ ] 5. Upgrade with a way back
