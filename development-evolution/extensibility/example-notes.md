# Extensibility: hands-on practice notes

Planned labs for learning extensibility by **building extension points, then letting "outside" code use them safely**. Theory is in [README.md](README.md).

They use **WordPress** (Docker) and the shared **`messy-shop`** codebase (described in the [modularity notes](../modularity/example-notes.md)).

**Suggested order:** 1 → 2 → 3 → 4.

**Related labs elsewhere:** domain events inside the monolith (modularity lab 6), shotgun surgery → registry (modifiability lab 1).

---

## Labs

### 1. A WordPress plugin ⭐
- **Do:** write a small plugin using an action (`save_post`: log or notify when a post is published) and a filter (`the_content`: add a reading-time line). Run WordPress in Docker.
- **Explore:** find which hooks a popular plugin uses. What would break for plugins if WordPress renamed `the_content`?
- **See:** README §1 worked example.

### 2. A plugin system from scratch
- **Do:** add plugin support to messy-shop for **shipping carriers**: a versioned interface (`quote(order)`, `create_label(order)`), discovery (entry points / a plugins folder), registration, and a manifest with the API version each plugin supports.
- **Then:** write 2 carrier plugins in separate packages without touching the core.

### 3. Isolate a misbehaving plugin
- **Break:** a plugin that hangs forever, one that throws, one that uses lots of memory.
- **Fix:** call plugins with timeouts, catch failures, track per-plugin errors and latency. Optional: run plugins in a separate process or as WebAssembly modules (e.g. Extism) to sandbox them.
- **See:** README §2.

### 4. Outgoing webhooks
- **Do:** messy-shop sends `order.paid` and `order.shipped` webhooks to subscriber URLs: HMAC signatures, retries with backoff, an event ID for deduplication, a payload version, and a "replay" button.
- **Test:** a receiver that's down for 10 minutes, then recovers; a receiver that rejects bad signatures.

---

## Progress

- [ ] 1. A WordPress plugin
- [ ] 2. A plugin system from scratch
- [ ] 3. Isolate a misbehaving plugin
- [ ] 4. Outgoing webhooks
