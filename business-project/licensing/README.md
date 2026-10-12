# Licensing

> **Licensing** (legal / licence compliance) = the system only uses software, data and services **in ways their licences allow**, and the organisation can prove it: it knows what it ships, under which terms, and which obligations follow.
> "A customer's lawyer asks for the list of every open-source component in our product and its licence. Can we produce it today, and is anything on it a problem?"

Related: [Compliance](../../security/compliance/README.md) (regulatory rules in general), [Reusability §4](../../development-evolution/reusability/README.md#4-dependency-hygiene) (dependency hygiene), [Deployability §12](../../operations-devops/deployability/README.md#12-artifacts-and-supply-chain) (artifacts and supply chain), [Vendor independence](../vendor-independence/README.md) (licence changes as a vendor risk), [Cost efficiency](../cost-efficiency/README.md) (commercial licence cost).

**These notes are an engineering overview, not legal advice.** Licence interpretation depends on the exact text, how components are combined, and jurisdiction. For real decisions, ask a lawyer who works with software licences.

---

## 1. Why it matters

- A modern application is mostly **other people's code**: hundreds or thousands of direct and transitive dependencies, each with a licence.
- **No licence means no permission.** Code published without a licence is "all rights reserved" by default, however public the repository.
- **Violations have real consequences**: being required to stop distributing, to release source, or to pay; failed due diligence in an acquisition or funding round; lost enterprise deals.
- **Licences shape architecture**: whether a component may be linked in, must stay a separate process, can be offered as a service, or can't be used at all.
- **Customers and regulators ask for it**: software bills of materials are increasingly required in contracts and by law (for example the EU Cyber Resilience Act and US federal procurement rules).

---

## 2. Licence families

| Family | Examples (SPDX IDs) | Main obligations | Typical company policy |
|---|---|---|---|
| **Permissive** | `MIT`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `Apache-2.0` | Keep copyright and licence notices. Apache-2.0 also: state changes, pass on the `NOTICE` file, and it includes an explicit patent grant | Allowed |
| **Weak copyleft** | `LGPL-2.1-or-later`, `LGPL-3.0-or-later`, `MPL-2.0`, `EPL-2.0` | Changes to the *library itself* (LGPL) or to the *licensed files* (MPL) must be shared under the same licence; your own code can stay closed. LGPL also requires that users can replace the library | Allowed with conditions |
| **Strong copyleft** | `GPL-2.0-only`, `GPL-3.0-or-later` | If you **distribute** a work that includes or derives from it, the whole work must be offered under the GPL with source | Review; usually not inside distributed proprietary products |
| **Network copyleft** | `AGPL-3.0-or-later` | As GPL, and users who interact with a modified version **over a network** must also be offered its source | Often prohibited, or isolated and reviewed |
| **Source-available** (not open source) | `BUSL-1.1` (Business Source License), `SSPL-1.0`, `Elastic-2.0` | Source is readable, but use is restricted: commonly no competing hosted service, or limits on production use. BUSL converts to an open-source licence after a set date | Review each one |
| **Public-domain-like** | `CC0-1.0`, `Unlicense`, `0BSD` | None, or almost none | Usually allowed |
| **Proprietary / commercial** | Vendor EULAs, SaaS terms | Whatever the contract says: seats, cores, instances, field of use, audits | Procurement process |

"Open source" formally means a licence approved by the **Open Source Initiative**; SSPL, BUSL and the Elastic License are not.

Things that are licensed too, and often forgotten: **fonts, icons, images, datasets, ML models and their weights, API terms of service, documentation, and code snippets copied from the web**. Creative Commons licences (`CC-BY-4.0`, `CC-BY-SA-4.0`, `-NC` variants) are for content, not code, and the non-commercial ones generally can't be used in a product.

---

## 3. What triggers obligations

The same licence can mean nothing or a lot, depending on what you do:

| What you do | Permissive | GPL | AGPL |
|---|---|---|---|
| Use a tool internally (compiler, editor, CI tool) | Nothing | Nothing | Nothing |
| Run it unmodified as a separate server (a database, a cache) | Nothing | Nothing | Generally nothing, but many policies still require review |
| Link it into a backend you run as SaaS, never distributed | Notices | Not triggered: no distribution | **Triggered** if you modified it; whether your own code is covered depends on how they're combined |
| Ship it inside something customers receive | Notices | **Triggered** | **Triggered** |

"Distribution" is wider than it sounds. These all count as shipping software to someone else:
- **Mobile and desktop apps**, firmware and devices.
- **Front-end JavaScript** bundles sent to browsers.
- **Container images** and VM images given to customers or published: you're distributing everything in the image, including the base OS packages.
- **On-premises** or self-hosted editions, SDKs and CLIs.

**Compatibility** matters when licences are combined in one work: for example, Apache-2.0 code can go into a GPL-3.0 project but is generally considered incompatible with GPL-2.0-only. Check before mixing copyleft licences.

---

## 4. Knowing what you ship

- **SBOM (software bill of materials)**: a machine-readable list of every component, version and licence in an artifact. Two standard formats: **SPDX** (ISO/IEC 5962) and **CycloneDX** (OWASP). Generate one per build and store it with the artifact.
- **SPDX licence identifiers**: short standard names (`MIT`, `Apache-2.0`, `GPL-3.0-only`) and expressions (`MIT OR Apache-2.0`). Use them in package metadata and file headers (`SPDX-License-Identifier:`).
- **Scanners**: Syft and Trivy (SBOMs and licence detection for images and repositories), ScanCode Toolkit and the OSS Review Toolkit (deep licence scanning and policy), ecosystem tools such as `pip-licenses`, `license-checker`, `cargo-deny`, `go-licenses`; commercial services exist too.
- **Policy in CI**: an allow list, a review list and a deny list. The build fails on a denied or unknown licence, the same way it fails on a failing test.
- **Attribution**: generate the third-party notices file from the SBOM; ship it with the product ("open-source licences" screen, `NOTICE` file).
- **Process standard**: **OpenChain** (ISO/IEC 5230) describes what a working open-source compliance programme needs.

### Worked example: a dependency audit

A team ships an on-premises edition of its product as a container image. A scan finds 412 components:

```
MIT / BSD / ISC / Apache-2.0      389   allowed
MPL-2.0                             6   allowed (no files modified)
LGPL-2.1-or-later                   9   allowed (dynamically linked, unmodified)
GPL-2.0-only                        5   review
AGPL-3.0-only                       1   review
BUSL-1.1                            1   review
no licence detected                 1   review
```

So 404 pass policy and **8 need a decision**:

| Finding | Decision |
|---|---|
| 5 × GPL: shell tools in the base OS image, not linked to the product | Allowed, but the image is distributed, so provide the corresponding source offer for those packages (or switch to a minimal base image without them) |
| 1 × AGPL: a PDF library imported into the backend | Replace with a permissively licensed library, or buy the vendor's commercial licence |
| 1 × BUSL: a database the product bundles | Read the additional use grant; bundling in a commercial product may not be permitted. Ask legal; consider an alternative |
| 1 × no licence: a small utility copied from a gist | No permission to use. Rewrite it or get a licence from the author |

Without the scan, the first anyone hears of these is a customer's due-diligence questionnaire.

---

## 5. Commercial licences

Proprietary software and SaaS are metered by contract: per user, per core, per instance, per host, per GB ingested. The licence metric interacts with architecture:

```
A database licensed per core at an illustrative $2,000 per core per year:
  1 server × 8 cores              = $16,000 / year
  scale to 3 replicas × 16 cores  = 48 cores = $96,000 / year
```

Scaling out multiplied the licence cost by 6, and autoscaling or a standby replica may need licences too. Know the metric before designing around a commercial component, track usage against entitlements, and expect audits. See [Cost efficiency](../cost-efficiency/README.md).

---

## 6. Licensing your own software

- **Outbound licence**: choose deliberately. Permissive for maximum adoption; copyleft to keep derivatives open; source-available or dual licensing to stop competitors hosting it. Each choice affects who will use and contribute.
- **Licence changes hurt trust**: see the stories below. A project controlled by one company can relicense future versions; a project with many copyright holders and no contributor agreement practically can't.
- **Contributions**: a **DCO** (developer sign-off) or a **CLA** (contributor licence agreement) clarifies that contributors have the right to submit their code.
- **Inside a company**: check who owns what employees and contractors write, and have a policy for contributing to open source.
- **AI-generated code** and code from assistants: the legal position is still unsettled in many jurisdictions. Follow your organisation's policy, and treat verbatim reproduction of existing code like any other copied code.

---

## 7. Real-world stories

| Story | What happened | Lesson |
|---|---|---|
| **Linksys WRT54G (2003)** | The router's firmware was found to contain GPL-licensed code, including Linux. The source was released, and community firmware projects such as OpenWrt grew from it. | Firmware and devices are distribution; copyleft obligations apply. |
| **MongoDB (2018)** | MongoDB moved from AGPL to its own Server Side Public License, aimed at providers offering the database as a service. The OSI did not approve SSPL as open source, and some Linux distributions dropped the package. | A licence can change under you; "open source" has a specific meaning. |
| **Elasticsearch (2021)** | Elastic moved Elasticsearch and Kibana from Apache-2.0 to SSPL / Elastic License. AWS forked the last Apache-licensed version as OpenSearch. In 2024 Elastic added AGPL as a licence option. | Forks follow licence changes; users end up choosing between ecosystems. |
| **HashiCorp (2023), Redis (2024)** | Both moved widely used projects to source-available licences; community forks (OpenTofu, Valkey) followed under the Linux Foundation. | Check who controls a dependency, not only its current licence ([Vendor independence](../vendor-independence/README.md)). |

---

## 8. Anti-patterns

- **Nobody knows what's in the product**: no SBOM, no inventory.
- **"It's on GitHub, so it's free to use."**
- **Checking only direct dependencies**; the problem is usually transitive.
- **Copy-pasting code** from blogs, Q&A sites and other repositories without recording its origin and licence.
- **Licence review at release time** (or during an acquisition) instead of when the dependency is added.
- **Treating SaaS as a loophole** without checking for AGPL and source-available terms.
- **Forgetting non-code assets**: fonts, images, datasets, models.
- **Removing licence headers and notices** from vendored code.
- **Ignoring licence changes on upgrade**: the new major version may have different terms from the one you approved.
- **Banning all open source** out of fear, then rebuilding it badly.

---

## 9. Checklist

- [ ] A written licence policy: allowed, needs review, prohibited
- [ ] SBOM (SPDX or CycloneDX) generated for every release artifact, including container base images
- [ ] Licence check in CI that fails on denied or unknown licences, covering transitive dependencies
- [ ] Third-party notices generated and shipped with every distributed product
- [ ] Copied code and non-code assets recorded with source and licence
- [ ] Copyleft and source-available components reviewed against how they're used and distributed
- [ ] Commercial licence metrics understood and usage tracked
- [ ] Dependency upgrades checked for licence changes
- [ ] Own repositories have a `LICENSE` file and contribution terms
- [ ] A named person or team to ask, with access to legal advice

---

## 10. Exercises

1. Generate an SBOM for a project you work on. How many components are there, how many distinct licences, and how many of each family in §2? Is anything unlicensed?
2. For each case, say whether GPL obligations are likely triggered and why: (a) a GPL command-line tool used in CI, (b) a GPL library linked into a mobile app, (c) a GPL library linked into a backend that's only run as SaaS, (d) the same backend shipped to customers as a Docker image.
3. Your product depends on a database whose vendor announces a move from Apache-2.0 to a source-available licence. List your options and what each costs.
4. Write a one-page licence policy for a 20-person company that ships a SaaS product and a mobile app. Which licences go in each of the three lists, and who decides the "review" cases?

---

## 11. Further reading

Open Source Initiative (opensource.org): the Open Source Definition and approved licences; choosealicense.com; the SPDX licence list and specification (spdx.dev); CycloneDX (cyclonedx.org); OpenChain (ISO/IEC 5230); the FSF's GPL FAQ and licence compatibility list; FSFE's REUSE specification; Heather Meeker, *Open (Source) for Business*; tl;drLegal for plain-language summaries (a starting point, not a substitute for the licence text); OSS Review Toolkit, ScanCode, Syft and Trivy documentation; Google's open-source licence classification (opensource.google) as an example of a company policy.
