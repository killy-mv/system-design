# Compliance

> **Compliance** = the system and the organisation around it **meet the laws, regulations, standards and contracts that apply to them, and can prove it** with evidence.
> "An auditor asks: 'Show me that only authorized people accessed cardholder data last quarter.' Can we answer from records, without scrambling?"

Compliance and security overlap but aren't the same. A system can pass an audit and still be insecure; it can be secure and still fail an audit for lack of evidence. Aim for real security first, and collect the evidence as a by-product.

Related: [Privacy](../privacy/README.md) (data-protection law in engineering terms), [Auditability](../auditability/README.md) (the evidence), [Confidentiality](../confidentiality/README.md), [Authorization](../authorization/README.md), [Licensing](../../business-project/licensing/README.md) (open-source licence obligations), [Safety](../../safety/safety/README.md) (safety regulation), [Accessibility](../../UI-UX/accessibility/README.md) (accessibility law).

---

## 1. The landscape

| Framework | Kind | Applies when | Core idea |
|---|---|---|---|
| **GDPR** (EU), UK GDPR | Law | You process personal data of people in the EU/UK | Lawful basis, data-subject rights, breach notification to the authority within 72 hours; fines up to €20 million or 4% of global annual turnover |
| **CCPA / CPRA** (California) and other US state laws | Law | Consumer data of residents, above size thresholds | Rights to know, delete and opt out of sale/sharing |
| **HIPAA** (US) | Law | Health data handled by healthcare providers, insurers and their vendors | Safeguards for protected health information; business associate agreements |
| **PCI DSS** | Industry standard (contractual) | You store, process or transmit payment card data | Prescriptive technical requirements; scope is everything that touches card data |
| **SOC 2** | Audit report (AICPA) | B2B customers ask for it | An auditor's report on your controls against the Trust Services Criteria. *Type I*: design at a point in time. *Type II*: operation over a period |
| **ISO/IEC 27001** | Certifiable standard | International customers, enterprise sales | An information security management system (ISMS) with risk-based controls |
| **SOX** (US) | Law | Public companies | Controls over financial reporting, including change management and access |
| **NIS2, DORA** (EU) | Law | Essential/important entities; financial sector | Risk management, incident reporting, third-party and resilience requirements |
| **FedRAMP** (US) | Government programme | Selling cloud services to US federal agencies | Based on NIST SP 800-53 controls |
| **EU AI Act** | Law | Providing or deploying AI systems in the EU | Obligations scale with the system's risk category |

Which ones apply depends on your **customers, data, sector and location**. Finding out is a job for legal and compliance specialists; building systems that can satisfy them is an engineering job. This page is about the second part and is not legal advice.

---

## 2. From requirement to evidence

Every framework reduces to the same chain:

```
Requirement  →  Control             →  Implementation                →  Evidence
"Access is      "Access is granted     SSO groups; access requests     Ticket with approval; IdP export;
 restricted      by approval and        in the ticket system;           quarterly review record signed
 to need"        reviewed quarterly"    quarterly review job            by the manager
```

A control with no evidence doesn't exist as far as an audit is concerned. The frameworks overlap heavily, so map each control once to all the requirements it satisfies (a "common controls" approach) instead of running separate programmes.

Controls nearly every framework asks for:

| Area | Typical control | Where it's covered |
|---|---|---|
| Access | Unique accounts, MFA, least privilege, reviews, prompt removal | [Authentication](../authentication/README.md), [Authorization](../authorization/README.md) |
| Data protection | Encryption in transit and at rest, key management | [Confidentiality](../confidentiality/README.md) |
| Logging | Audit logs retained and reviewed | [Auditability](../auditability/README.md) |
| Change management | Reviewed, tested, approved changes; separation of duties | [Deployability](../../operations-devops/deployability/README.md) |
| Vulnerability management | Scanning, patching within set deadlines, penetration tests | [Maintainability §13](../../development-evolution/maintainability/README.md#13-dependencies-upgrades-and-security-maintenance) |
| Resilience | Backups, tested restores, disaster recovery, incident response | [Recoverability](../../runtime-operational/recoverability/README.md) |
| Vendors | Inventory, due diligence, contracts | [Vendor independence](../../business-project/vendor-independence/README.md) |
| People | Training, background checks, onboarding/offboarding | Organisational |

---

## 3. Design to reduce scope

The cheapest way to comply is to have **less in scope**.

### Worked example: card payments

A shop needs to take card payments. Two designs:

| | **A: cards pass through your servers** | **B: hosted fields / redirect** |
|---|---|---|
| Flow | Browser → your API → payment provider | Browser → provider's iframe or page; your server receives only a **token** |
| Card data touches | Web servers, load balancers, logs, anything on the same network | Nothing of yours |
| PCI DSS validation | The full assessment (hundreds of requirements; an on-site assessor at higher volumes) | The shortest self-assessment questionnaire (SAQ A) |
| Residual duties | Segmentation, key management, scans, log reviews, penetration tests across the whole card environment | Keep the payment page's integrity (scripts, CSP), manage the provider relationship |

Same product feature; design B removes most of the obligation by **never handling the data**. PCI DSS also forbids storing the card verification code (CVV) after authorisation at all, so no design should try.

The same move works elsewhere:

- **Tokenise or outsource** the regulated data (cards, identity documents) to a provider built for it.
- **Segment**: keep regulated data in one small service, account or network zone so the audit boundary is small.
- **Minimise**: don't collect health or identity data you don't need ([Privacy](../privacy/README.md)).
- **Data residency**: decide early which regions data may live in; retrofitting regional isolation is expensive.
- **Shared responsibility**: a cloud provider's certifications cover *their* layer. Your configuration and application remain yours.

---

## 4. Compliance as code

Manual, once-a-year compliance decays between audits. Automate controls and evidence the way you automate tests.

| Practice | Example |
|---|---|
| **Policy as code** | OPA/Conftest or Checkov rules on Terraform: "no public buckets", "encryption required", failing the pull request |
| **Preventive guardrails** | Cloud organisation policies that make the non-compliant state impossible |
| **Continuous detection** | AWS Config, Azure Policy, Security Hub or open-source tools checking live resources against benchmarks (CIS) |
| **The pipeline as change-management evidence** | Protected branches, required review, CI results and deploy records *are* the approval trail |
| **Automated evidence collection** | Scheduled exports of access lists, backup-restore test results, scan reports into an evidence store |
| **Software bill of materials (SBOM)** | Generated at build; used for vulnerability and licence checks ([Licensing](../../business-project/licensing/README.md)) |
| **Access reviews from data** | Review generated from the IdP and cloud IAM, not from a hand-kept spreadsheet |

---

## 5. Process

- **Risk assessment**: identify what could go wrong, how likely and how bad, and choose controls accordingly. ISO 27001 and SOC 2 are built on this.
- **Policies people can follow**: short, specific, matching what actually happens.
- **Incident and breach response**: know your notification deadlines before the incident (72 hours to the authority under GDPR; HIPAA and US state laws have their own). Rehearse.
- **Vendor management**: your processors' failures are your failures. Keep a list, agreements and their reports.
- **Internal audit and exceptions**: check yourselves between external audits; record exceptions with an owner and an end date.
- **Engineering involvement early**: compliance requirements are architecture inputs, like latency or cost. Discovering them after launch is the expensive path.

---

## 6. Anti-patterns

- **Checkbox compliance**: passing the audit is the goal, security is incidental.
- Policies that describe a process nobody follows.
- Evidence gathered by hand in a panic each year (screenshots in a shared folder).
- The whole platform in PCI or HIPAA scope because regulated data flows everywhere.
- Assuming the cloud provider's certificate makes you compliant.
- Treating compliance as a blocker team that says "no" at the end, instead of requirements known at the start.
- Regulated data in logs, analytics and non-production environments, silently widening scope.
- No owner for a control, so it quietly stops operating.
- Storing prohibited data (CVV; data kept past its retention period).

---

## 7. Checklist

- [ ] Applicable laws, standards and contractual commitments identified with specialists
- [ ] Regulated data located; scope minimised through tokenisation, segmentation and minimisation
- [ ] Controls mapped once to all the requirements they satisfy, each with an owner
- [ ] Evidence produced automatically by the systems that perform the control
- [ ] Policy-as-code checks in CI and continuous checks on live infrastructure
- [ ] Change management, access reviews and backup-restore tests leave a record
- [ ] Incident response plan with notification deadlines; rehearsed
- [ ] Vendor inventory with agreements and their assurance reports
- [ ] Exceptions recorded with owner and expiry

---

## 8. Exercises

1. For a product you know, list which frameworks from §1 would apply if it (a) sold to EU consumers, (b) took card payments directly, (c) stored patient records for US clinics, (d) sold to large enterprises.
2. Take one control ("production access requires approval and is reviewed quarterly"). Design the implementation and the evidence so that no human has to assemble anything at audit time.
3. Redesign a system where card numbers or identity documents pass through the main application so that they don't. What's left in scope?
4. You discover on Friday evening that a storage bucket with customer data was public for two weeks. Write the first 72 hours as a timeline: who decides what, and which records you'll need.

---

## 9. Further reading

PCI DSS v4.x and the PCI SSC's self-assessment questionnaire guidance; AICPA Trust Services Criteria (SOC 2); ISO/IEC 27001 and 27002; NIST Cybersecurity Framework 2.0 and NIST SP 800-53; the GDPR text and EDPB guidelines on breach notification; HHS guidance on the HIPAA Security Rule; CIS Benchmarks and CIS Controls; Cloud Security Alliance Cloud Controls Matrix; AWS, Azure and Google Cloud "shared responsibility model" documentation; Open Policy Agent and Checkov documentation; *Security Chaos Engineering* (Shortridge, Rinehart) for a critical view of checkbox compliance.
