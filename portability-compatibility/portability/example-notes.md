# Portability: hands-on practice notes

Planned labs for learning portability by **moving one app between environments and writing down everything that breaks**. Theory is in [README.md](README.md).

They need **Docker** (with `buildx`), **GitHub Actions** (or any CI with a build matrix), and the shared **`messy-shop`** codebase (described in the [modularity notes](../../development-evolution/modularity/example-notes.md)).

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** rebuild everything from code (manageability lab 1), validate config and fail fast (configurability lab 1), swapping a component behind a port (replaceability lab 1).

---

## Labs

### 1. The clean-machine test ⭐
- **Do:** clone messy-shop into a fresh container or VM with nothing installed (`docker run -it debian:stable-slim`). Follow only the README until the app answers a request.
- **Record:** every step that wasn't written down, every implicit tool, path, environment variable and version.
- **Fix:** a `Dockerfile` and `compose.yaml` so `docker compose up` is the whole setup. Pin the runtime and lock dependencies.
- **See:** README §2, §3.

### 2. Multi-architecture image ⭐
- **Build:** `docker buildx build --platform linux/amd64,linux/arm64` and push to a local registry. Inspect the manifest list.
- **Break:** add a dependency with a native extension and switch the base image to Alpine. Note the error on each platform.
- **Fix:** multi-stage build; compare build time under QEMU emulation with cross-compiling in the build stage.
- **See:** README §3 worked example.

### 3. A CI matrix
- **Build:** a pipeline that runs the tests on Linux, macOS and Windows, and on two runtime versions.
- **Expect to find:** path separators, a wrong-case import, line endings, a Bash-only script, a test that depends on the local time zone.
- **Fix** each one with the portable API, not with an `if windows:` branch where avoidable.

### 4. Dev/prod parity
- **Break:** run messy-shop's tests against SQLite, then against Postgres. Find a query or constraint that behaves differently (types, `ON CONFLICT`, case sensitivity, transaction behaviour).
- **Fix:** tests run against the real engine in a container (Testcontainers or a Compose service).
- **See:** README §3 (Twelve-Factor X).

### 5. Move the storage
- **Do:** messy-shop stores product images on local disk. Put it behind a small storage interface and run it against **MinIO** using the S3 API, changing only configuration.
- **Then:** list what a real move to a cloud object store would still need (IAM, lifecycle rules, URLs already stored in the database).
- **See:** README §4.

---

## Progress

- [ ] 1. The clean-machine test
- [ ] 2. Multi-architecture image
- [ ] 3. A CI matrix
- [ ] 4. Dev/prod parity
- [ ] 5. Move the storage
