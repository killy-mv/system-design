# Compliance: hands-on practice notes

Planned labs for learning compliance by **turning written requirements into automated controls and evidence, and by designing regulated data out of scope**. Theory is in [README.md](README.md).

They use the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)) with **Terraform** against **LocalStack** (or a cloud sandbox account), **Checkov** or **OPA/Conftest**, **GitHub Actions** (or any CI), **Syft** and **Grype**/**Trivy**, and **Stripe test mode** (or any payment provider's sandbox).

These labs teach the engineering side. They don't make anything certified, and they aren't legal advice.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** a real CI/CD pipeline (deployability lab 1), rebuild everything from code and drift detection (manageability labs 1–2), audit events (auditability lab 1), export and erase (privacy lab 3), least privilege in the cloud (authorization lab 8).

---

## Labs

### 1. Shrink the scope ⭐
- **Break:** build checkout the naive way: a card form that posts to messy-shop's API, which forwards to the provider's sandbox. List every component and log that sees the card number (use the provider's test card numbers only).
- **Fix:** switch to the provider's hosted fields or redirect, so the server only receives a token.
- **Check:** search requests, logs and the database for the test card number. Draw the before/after diagram of what would be in PCI DSS scope.
- **See:** README §3 worked example.

### 2. Policy as code ⭐
- **Build:** Terraform for messy-shop's storage, database and network. Add Checkov or Conftest rules in CI: no public buckets, encryption at rest required, no `0.0.0.0/0` on admin ports, required tags (owner, data class).
- **Break:** open a pull request that makes a bucket public and confirm CI blocks it.
- **Then:** write one custom rule of your own for a requirement the built-in checks don't cover.
- **See:** README §4.

### 3. A control with automatic evidence
- **Do:** pick three controls: "changes are reviewed", "access is reviewed quarterly", "backups are restored and tested monthly".
- **Build:** for each, a script or scheduled job that produces the evidence: merged PRs with reviewers from the Git host's API, an access list export with a sign-off file, a restore test that writes a dated result.
- **Check:** hand the evidence folder to a friend acting as auditor. Can they verify each control without asking you anything?
- **See:** README §2.

### 4. SBOM and vulnerability deadlines
- **Build:** generate an SBOM for the messy-shop image with Syft in CI; scan it with Grype or Trivy; fail the build on critical findings with a fix available.
- **Then:** define patch deadlines (for example critical 7 days, high 30 days) and write a report listing findings older than their deadline.
- **Also:** list the licences in the SBOM and flag any your policy wouldn't allow.

### 5. Breach tabletop
- **Do:** run a 60-minute exercise on paper: a bucket with customer exports was public for two weeks. Assign roles (incident lead, engineer, legal, communications).
- **Produce:** a timeline, the list of records you needed (access logs, data inventory, who was affected), a draft notification, and the gaps you found.
- **See:** README §5.

---

## Progress

- [ ] 1. Shrink the scope
- [ ] 2. Policy as code
- [ ] 3. A control with automatic evidence
- [ ] 4. SBOM and vulnerability deadlines
- [ ] 5. Breach tabletop
