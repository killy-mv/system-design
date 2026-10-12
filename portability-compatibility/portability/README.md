# Portability

> **Portability** = how easily a system can be **moved from one environment to another** (hardware, operating system, runtime, cloud, database) and still work, with little or no change.
> "We built it on x86 Linux in one cloud. What does it cost to run it on ARM, on a customer's servers, or in another cloud?"

Related: [Adaptability](../adaptability/README.md) (adjusting to an environment once you're there), [Installability](../installability/README.md) (getting it set up), [Replaceability](../replaceability/README.md) (swapping one component), [Vendor independence](../../business-project/vendor-independence/README.md) (the business side of lock-in), [Deployability](../../operations-devops/deployability/README.md).

In ISO/IEC 25010:2011 portability is a top-level characteristic with three parts: adaptability, installability, replaceability. The 2023 revision renames it **flexibility** and adds scalability. The ideas are the same.

---

## 1. Kinds of portability

| Kind | Moving between | Example |
|---|---|---|
| **Hardware / CPU architecture** | x86-64 ↔ arm64, 64-bit ↔ 32-bit | Laptops on Apple Silicon, servers on AWS Graviton |
| **Operating system** | Linux ↔ Windows ↔ macOS; distro ↔ distro | A CLI tool shipped for three OSes |
| **Runtime / language version** | Python 3.10 → 3.13, Node 18 → 22, JVM 11 → 21 | Framework end of life forces an upgrade |
| **Infrastructure provider** | Cloud ↔ cloud, cloud ↔ on-premises | A customer demands their own data centre |
| **Data store** | Postgres ↔ MySQL, S3 ↔ another object store | Changing database vendor |
| **Data** | Getting your data *out* in a usable format | Export to open formats, GDPR data portability |
| **Environment** | Laptop ↔ CI ↔ staging ↔ production | "Works on my machine" |

Environment portability is the one you need every day. The others matter when the business needs them.

---

## 2. What makes software non-portable

| Cause | Example |
|---|---|
| **Paths and file systems** | Hard-coded `/home/app/data`, `\` vs `/`, case-insensitive file systems on macOS and Windows hiding a wrong-case `import` that fails on Linux |
| **Text and bytes** | `\r\n` vs `\n`, default encodings that differ per OS, byte order (endianness) in binary formats |
| **Native code** | A dependency with a compiled extension that has no build for arm64 |
| **C library** | A binary built against glibc doesn't run on musl (Alpine Linux), and the other way round |
| **Implicit environment** | Assumes a tool, locale, time zone, or environment variable that only exists on the build machine |
| **Shell and OS tools** | Bash-only scripts run by `sh`; GNU `sed -i` vs BSD `sed -i ''` |
| **Provider services** | Code written directly against one cloud's queue, IAM, or proprietary database API |
| **SQL dialects** | `ON CONFLICT` vs `ON DUPLICATE KEY`, different JSON functions, different default isolation levels |
| **Undefined behaviour** | Relying on hash-map ordering, integer sizes, or float behaviour that differs by platform |

---

## 3. Techniques

- **Standards first**: POSIX, SQL, HTTP, OCI container images, the S3 API, OpenTelemetry. A standard has more than one implementation, so you can move.
- **Containers**: an OCI image carries the OS userland and dependencies, so the same artifact runs on a laptop, in CI and in production. It does **not** hide the CPU architecture or the kernel.
- **Config in the environment** ([Twelve-Factor](https://12factor.net/) III): hostnames, credentials and paths come from environment variables, not from code ([Configurability](../../operations-devops/configurability/README.md)).
- **Dev/prod parity** (Twelve-Factor X): same database engine and versions locally as in production, not SQLite locally and Postgres in production.
- **Isolate platform-specific code** behind a small interface (ports and adapters). Everything else stays platform-neutral.
- **Use the language's portable APIs**: `pathlib` / `path.join`, explicit encodings (`UTF-8`), explicit time zones.
- **Pin and lock**: runtime version, dependency lock files, base image digests. A build that depends on "whatever is installed" is not portable.
- **Infrastructure as code** with a tool that isn't tied to one provider (Terraform/OpenTofu, Pulumi), plus Kubernetes as a common deployment target. This lowers the cost of moving; the provider-specific resources still need rewriting.
- **Test on the targets**: a CI matrix across OSes, architectures and runtime versions. Portability that isn't tested doesn't exist.

### Worked example: one image, two CPU architectures

A team builds images on x86 CI runners. A developer with an Apple Silicon laptop and a new arm64 node pool both report the same thing:

```
exec /app/server: exec format error
```

The image only contains an `amd64` binary. Fix: build a **multi-platform image**, one tag pointing at a manifest list with a variant per architecture.

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.example.com/shop/api:1.8.0 \
  --push .

docker buildx imagetools inspect registry.example.com/shop/api:1.8.0
# Manifests:
#   Platform: linux/amd64
#   Platform: linux/arm64
```

Each node pulls the variant that matches its CPU. What usually breaks on the way:

| Problem | Fix |
|---|---|
| A Python/Node dependency has no arm64 wheel/prebuilt binary | Upgrade it, or install a compiler in the build stage and build from source |
| Switching the base image to Alpine breaks a prebuilt binary (glibc vs musl) | Stay on a glibc base (Debian slim, distroless) or build for musl |
| Emulated builds (QEMU) are very slow | Cross-compile (`GOARCH=arm64`, `--platform=$BUILDPLATFORM` in the build stage) or use native arm64 runners |
| Tests only ran on amd64 | Add arm64 to the CI matrix |

---

## 4. How much portability?

Portability costs something: you give up provider-specific features, add abstraction layers, and test more combinations. Aim for the **lowest cost of moving that the business actually needs**, not for "runs anywhere".

| Need | Reasonable target |
|---|---|
| Any team | Laptop ↔ CI ↔ production portability (containers, config in env, pinned versions) |
| Self-hosted product sold to customers | Several OSes/distros, air-gapped installs, more than one database version |
| SaaS on one cloud | Standard interfaces where cheap (Postgres, S3 API, Kubernetes, OpenTelemetry); accept managed services where they save real work |
| Regulated or very large | A tested exit plan per critical provider ([Vendor independence](../../business-project/vendor-independence/README.md)) |

**The lowest-common-denominator trap**: using only features every provider has means using none of them well. Prefer knowing *where* you're tied in and what leaving would cost.

---

## 5. Anti-patterns

- "Works on my machine": setup lives in one person's head or shell history.
- Different database engines in development and production.
- Hard-coded paths, hostnames, regions or account IDs.
- Building on `latest` base images and unpinned dependencies.
- Claiming multi-cloud portability that has never been exercised.
- An abstraction layer over every cloud service "in case we move", paid for every day and never used.
- Data that can only be read by the application that wrote it.

---

## 6. Checklist

- [ ] The same build artifact (image) runs in every environment; only configuration differs
- [ ] Runtime, dependencies and base images are pinned
- [ ] No hard-coded paths, hosts or provider identifiers in code
- [ ] Platform- and provider-specific code is isolated behind interfaces
- [ ] CI runs on every OS/architecture/runtime you claim to support
- [ ] Data can be exported in an open, documented format
- [ ] The places where you are tied to a provider are known and written down

---

## 7. Exercises

1. Take a project you built and list everything it assumes about its environment (OS, CPU, paths, tools, services). Which assumptions are written down?
2. Your service uses a cloud-specific queue, object storage and a managed Postgres. Rank the three by how hard they'd be to move, and explain why.
3. A Node.js image works on `node:22` but crashes on `node:22-alpine` with a missing symbol error. What's the likely cause, and what are two fixes?
4. A customer wants your SaaS product installed in their own data centre with no internet access. List what has to change.

---

## 8. Further reading

*The Twelve-Factor App* (12factor.net), especially factors II, III and X; Open Container Initiative image and runtime specifications; Docker docs, "Multi-platform builds"; ISO/IEC 25010 (portability / flexibility); *Cloud Native Patterns* (Cornelia Davis); Gregor Hohpe, "Don't get locked up into avoiding lock-in" (martinfowler.com, 2019); the POSIX standard (IEEE 1003.1) as a reference for what is actually portable in shell scripts.
