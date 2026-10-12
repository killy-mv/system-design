# Consistency

> **Consistency** = everyone reading the data sees values that are **correct and agree with each other**, according to the guarantees the system promises.
> "After I save something, will I (and everyone else) see it? Could two users see different answers?"

Related: [Reliability §5](../reliability/README.md#5-consistency-models) (consistency models in depth), [Availability](../availability/README.md) (the CAP trade-off), [Concurrency](../concurrency/README.md), [Scalability](../scalability/README.md) (caches and replicas create inconsistency).

---

## 1. Three meanings of "consistency"

| Meaning | Question | Example |
|---|---|---|
| **ACID consistency** | Does data obey its rules (invariants)? | Balance never negative; every order has a customer |
| **Replica consistency** | Do all copies of the data agree, and when? | A replica 2 s behind the primary |
| **Cache / derived-data consistency** | Do caches, search indexes, CDNs match the source of truth? | Product price updated in DB, old price still in search results |

This page focuses on the second and third: the ones that appear as soon as data exists in more than one place.

---

## 2. The spectrum (strongest → weakest)

| Model | What users experience |
|---|---|
| **Linearizable (strong)** | Behaves like a single copy. Once a write completes, every read sees it. |
| **Sequential** | Everyone sees operations in the same order, maybe slightly late. |
| **Causal** | Replies are never seen before the message they reply to. Unrelated events may appear in different orders. |
| **Read-your-writes** | *You* always see your own changes (others may not yet). |
| **Monotonic reads** | You never see data go back in time when refreshing. |
| **Eventual** | If writes stop, all copies converge... eventually. Meanwhile, anything goes. |

Stronger = simpler to reason about, but more coordination → more **latency** and less **availability** during network partitions.

- **CAP**: during a partition, choose consistency (reject some requests) or availability (answer, possibly stale).
- **PACELC**: even without partitions, choose **latency** or **consistency**.

Full detail: [Reliability §5](../reliability/README.md#5-consistency-models).

---

## 3. Where inconsistency comes from

- **Async replication lag**: reads from replicas are behind the primary.
- **Caches**: app cache, Redis, CDN, browser, each with its own TTL.
- **Derived data**: search indexes, analytics, materialised views updated asynchronously.
- **Dual writes**: writing to DB and cache/queue separately; a crash in between leaves them different.
- **Multi-region / multi-leader**: concurrent writes in different places that conflict.
- **Offline clients**: mobile apps with local copies.

### Worked example: "I changed my profile photo but still see the old one"

```
1. Upload new photo → saved to primary DB + object storage at the same URL /avatars/42.jpg
2. Page reloads → reads profile from a replica (lag 2 s)           → old photo URL
3. Replica catches up → app's Redis cache still has the profile (TTL 5 min) → old data
4. Cache expires → browser and CDN cached /avatars/42.jpg for 1 day       → old image
```

Four layers, four sources of staleness. Fixes, one per layer:
- Read the user's **own** profile from the primary for a short time after they write (read-your-writes).
- **Delete** the cache entry on write instead of waiting for the TTL.
- Use a **versioned URL** (`/avatars/42.v8.jpg`) so the browser and CDN fetch the new file, with no CDN purge needed.
- Other users seeing the old photo for a minute is fine: **eventual** is acceptable for them.

---

## 4. Choose consistency per data type

| Data | Needs | Why |
|---|---|---|
| Money, balances, payments | Strong | Double spending, wrong balances |
| Inventory at reservation, seat booking | Strong (at that step) | Overselling |
| Usernames, unique emails | Strong (unique constraint) | Duplicates |
| Permissions, password change, blocking a user | Strong or very fast | Security |
| User's own edits (profile, posts, settings) | Read-your-writes | "Did my save work?" confusion |
| Comments, chat messages | Causal + per-conversation order | Replies before questions |
| Feeds, likes, view counts, recommendations | Eventual | Nobody notices a few seconds |
| Analytics, reports | Eventual (minutes–hours) | Freshness SLO instead |

Pay for strong consistency only where it matters.

---

## 5. Techniques

| Goal | Technique |
|---|---|
| Strong | Single leader + read from leader, consensus (Raft), quorums (`W + R > N`), distributed SQL |
| Read-your-writes | Route the writer's reads to the primary; session tokens with WAL position (LSN); sticky replica |
| Fresh caches | Delete on write, CDC-driven invalidation, versioned keys/URLs, short TTLs |
| No dual-write gaps | Transactional outbox, CDC ([Reliability §7.3](../reliability/README.md#73-the-dual-write-problem-and-the-outbox-pattern)) |
| Merging concurrent writes | Version vectors, CRDTs, app-level merge ([Reliability §12](../reliability/README.md#12-conflict-resolution-in-replicated-data)) |
| Catching drift | Reconciliation jobs, periodic rebuild of derived data |

**Measure it:** replication lag, cache age, index freshness ("search results are at most 30 s old"), and mismatch counts from reconciliation.

---

## 6. Anti-patterns

- "Eventually consistent" used as an excuse for "sometimes permanently wrong" (no reconciliation).
- Strong consistency everywhere, paying latency for view counters.
- Reading from replicas right after a write in the same user flow.
- Updating (instead of deleting) cache entries on write: races leave stale values forever.
- Last-write-wins with wall clocks across machines.

---

## 7. Checklist

- [ ] Consistency requirement written down per data type
- [ ] Read-your-writes for every user-facing edit
- [ ] Cache invalidation strategy for every cached item; versioned URLs for static content
- [ ] No dual writes without outbox/CDC
- [ ] Replication lag and derived-data freshness monitored with targets
- [ ] Reconciliation for important derived/external data

---

## 8. Exercises

1. For a ticket-booking app, assign a consistency level to: seat map display, seat reservation, payment, confirmation email, "seats left" counter on the home page.
2. A user posts a comment, refreshes, and it's gone; refreshes again and it's back. Name the guarantee being violated and two fixes.
3. With `N = 3`, which `W` and `R` combinations give overlapping quorums? Which is fastest for reads?
4. Draw every cache layer between your database and a user's browser in a project you built. Which one would serve stale data longest after an update?

---

## 9. Further reading

*Designing Data-Intensive Applications* ch. 5 and 9; Jepsen's "Consistency Models" map (jepsen.io/consistency); Werner Vogels, "Eventually Consistent" (2008); Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (PACELC, 2012); Doug Terry, "Replicated Data Consistency Explained Through Baseball".
