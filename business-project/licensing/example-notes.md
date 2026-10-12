# Licensing: hands-on practice notes

Planned labs for learning licensing by **finding out what's really inside a build, and making CI refuse what policy doesn't allow**. Theory is in [README.md](README.md). None of this is legal advice; the labs teach the tooling and the questions to ask.

They need **Docker**, **Syft** and **Trivy** (or the OSS Review Toolkit / ScanCode for a deeper scan), your language's licence tool (`pip-licenses`, `license-checker`, `cargo-deny`, `go-licenses`), **GitHub Actions**, and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)).

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** dependency audit (reusability lab 2), a real CI/CD pipeline (deployability lab 1), dependency and image updates (manageability lab 4).

---

## Labs

### 1. What's in the box? ⭐
- **Do:** generate an SBOM for messy-shop twice: from the source repository, and from its built container image (`syft <image> -o spdx-json`, and again as CycloneDX).
- **Measure:** number of components, distinct licences, and the count per family (permissive, weak copyleft, strong copyleft, network copyleft, source-available, unknown). Compare repository vs image: how many components come from the base image?
- **Then:** rebuild on a minimal base image (distroless or Alpine) and compare.
- **See:** README §2, §4.

### 2. Licence policy in CI ⭐
- **Build:** a policy file with allow / review / deny lists of SPDX identifiers. A CI job that scans dependencies (including transitive ones) and fails on denied or unknown licences.
- **Break:** add a dependency with an AGPL licence and one with no licence metadata. Confirm the build fails with a message a developer can act on.
- **Fix:** add a documented exception mechanism (component, reason, approver, expiry date) and use it once.
- **See:** README §4.

### 3. Third-party notices
- **Build:** a script that turns the SBOM into a `THIRD-PARTY-NOTICES` file with each component's name, version, licence and licence text, bundled into the image and shown on an "open-source licences" page.
- **Check:** pick 5 components at random and verify the notice against the licence file in the package. Do Apache-2.0 components with a `NOTICE` file have it included?
- **See:** README §2, §4.

### 4. Trace the copied code
- **Do:** search messy-shop (or your own project) for code that came from somewhere else: vendored folders, snippets with a URL in a comment, bundled fonts, icons and images.
- **Build:** an inventory with origin, licence and whether attribution is in place. Add `SPDX-License-Identifier` headers and check the repository with the REUSE tool.
- **Fix:** replace or remove anything whose licence you can't establish.
- **See:** README §2, §8.

### 5. A licence changes under you
- **Tabletop, no code.** Pick a real dependency that changed licence (Elasticsearch, Terraform or Redis) and assume messy-shop uses it in three ways: internally in CI, as a backing service for a SaaS product, and bundled into an on-premises edition.
- **Do:** read the new licence's actual terms. For each of the three uses, write down whether it's still clearly fine, needs legal review, or needs a change, and list the options (stay on the old version, move to the fork, buy a commercial licence, replace).
- **Check:** what would have told you earlier? Add the check: licence diff on dependency upgrades.
- **See:** README §3, §7, [Vendor independence](../vendor-independence/README.md).

---

## Progress

- [ ] 1. What's in the box?
- [ ] 2. Licence policy in CI
- [ ] 3. Third-party notices
- [ ] 4. Trace the copied code
- [ ] 5. A licence changes under you
