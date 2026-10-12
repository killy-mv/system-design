# Manageability: hands-on practice notes

Planned labs for learning manageability by **turning manual operations into code and automation**. Theory is in [README.md](README.md).

They need **Terraform/OpenTofu** (with the Docker or kind provider so no cloud account is needed), **kind/k3d**, **Argo CD**, **cert-manager**, and a GitHub repo.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** GitOps with Argo CD (deployability lab 7), secrets (configurability lab 3), backups (reliability lab 12, recoverability lab 1).

---

## Labs

### 1. Rebuild everything from code ⭐
- **Build:** the whole demo environment (cluster, namespaces, databases, monitoring stack) from **OpenTofu/Terraform** + Helm/Kustomize, with one command.
- **Test:** destroy it completely and recreate it. Time it. Anything you had to do by hand is a gap to fix.

### 2. Drift detection
- **Build:** run `terraform plan` on a schedule in CI, and Argo CD for in-cluster state.
- **Break:** change something by hand (a replica count, a resource tag, a config map).
- **See:** both tools report the drift. Decide: revert automatically or alert?

### 3. Certificates on autopilot
- **Build:** **cert-manager** in kind with a self-signed or local ACME issuer (e.g. step-ca or Pebble), issuing short-lived certificates (1 day) for an Ingress.
- **See:** automatic renewal. Add an alert for "certificate expires in < 7 days" and test it by breaking renewal.

### 4. Dependency and image updates
- **Build:** enable **Renovate** or **Dependabot** on your repo for app dependencies, base images and Helm charts, with CI running tests on each PR.
- **See:** how many updates arrive per week, and how much is safely auto-mergeable.

### 5. Measure and remove toil
- **Do:** for two weeks, log every manual ops action you take on your lab systems (what, how long). Pick the most frequent and automate it (script → Make target → CI job → automatic trigger).
- **See:** README §1 worked example.

---

## Progress

- [ ] 1. Rebuild everything from code
- [ ] 2. Drift detection
- [ ] 3. Certificates on autopilot
- [ ] 4. Dependency and image updates
- [ ] 5. Measure and remove toil
