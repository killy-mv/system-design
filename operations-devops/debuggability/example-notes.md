# Debuggability: hands-on practice notes

Planned labs for learning debuggability by **planting bugs and timing how long it takes to find them**, before and after adding debuggability features. Theory is in [README.md](README.md).

They reuse the **demo system** from the observability notes. The best way to practise: have someone else (or a script with a random seed) plant the bug, so you don't know the answer.

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** find the bug with high-cardinality slicing (observability lab 7), CPU profiling (performance lab 4), mini game day (availability bonus).

---

## Labs

### 1. The reference ID ⭐
- **Build:** return the trace ID in every error response and show it on the error page ("Reference: 4bf92f35"). Make it searchable in your trace UI and logs.
- **Practice:** recreate the README §3 worked example ("payment failed at 14:03"). Time the investigation with and without the reference ID.

### 2. Error tracking with releases
- **Build:** add **Sentry** (free tier or self-hosted) or GlitchTip to the backend and the frontend. Tag events with the release version; upload frontend source maps.
- **Break:** ship a release with a bug in one code path. Find it in the error tracker, see "first seen in release X", and the stack trace with real file names.

### 3. Dynamic debug logging
- **Build:** a way to raise the log level for one user or one request (a header like `X-Debug: 1` accepted only from internal/authenticated callers, or a flag keyed by user ID with an expiry).
- **Check:** debug logs appear only for that user, and the setting expires automatically.

### 4. Debug endpoints and live profiling
- **Build:** `/version` (commit, build time, config version) and, for a Go service, `net/http/pprof` on an **internal-only** port. For other languages, the equivalent (Spring Boot Actuator, py-spy dump on a running process).
- **Practice:** a planted goroutine/thread leak or a deadlock: diagnose it live with a goroutine/thread dump, without restarting.
- **Check:** confirm the debug port isn't reachable from outside.

---

## Progress

- [ ] 1. The reference ID
- [ ] 2. Error tracking with releases
- [ ] 3. Dynamic debug logging
- [ ] 4. Debug endpoints and live profiling
