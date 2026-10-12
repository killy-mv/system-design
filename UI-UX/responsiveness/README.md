# Responsiveness (perceived performance)

> **Responsiveness** = how quickly and clearly a system **reacts to the user**: it acknowledges every input immediately, shows what's happening while it works, and never freezes, even when the actual work takes time.
> "Does it *feel* fast and alive, or does it feel stuck?"

Two related but different things:

| | [Performance](../../runtime-operational/performance/README.md) | **Responsiveness** |
|---|---|---|
| Measures | How long the work actually takes (latency, throughput) | How long until the user **sees a reaction**, and how the wait **feels** |
| Example | The search API takes 800 ms | The input shows the typed letter in 10 ms, results area dims at once, results arrive at 800 ms |
| Fixed by | Caching, indexing, better algorithms, more hardware | Instant feedback, optimistic UI, skeletons, progress, streaming, keeping the main thread free, prefetching |

A fast system can feel slow (no feedback, frozen UI, content jumping around), and a slow system can feel fast (instant acknowledgement, progressive results). You need both, but responsiveness is what users actually experience.

> ⚠ Not the same as **responsive design** (layouts that adapt to screen size). Responsive design is about layout; responsiveness is about time.

Related: [Usability](../usability/README.md) (§4.6 response time limits, §8.4 UI states) · [Performance](../../runtime-operational/performance/README.md) (§13 frontend performance, §5 caching) · [Accessibility](../accessibility/README.md) (announcing loading states, reduced motion).

▶ **Interactive demo:** [index.html](index.html): open it in a browser to try 4 sluggish designs side by side with their fixes. Every example uses the **same simulated server latency** on both sides; only the UI design differs.

---

## Table of contents

1. [Why responsiveness matters](#1-why-responsiveness-matters)
2. [How humans perceive time](#2-how-humans-perceive-time)
3. [Metrics and budgets](#3-metrics-and-budgets)
4. [The main thread: why UIs freeze](#4-the-main-thread-why-uis-freeze)
5. [Feedback patterns: what to show while waiting](#5-feedback-patterns-what-to-show-while-waiting)
6. [Optimistic UI](#6-optimistic-ui)
7. [Visual stability: no layout shift](#7-visual-stability-no-layout-shift)
8. [Stale responses and race conditions](#8-stale-responses-and-race-conditions)
9. [Getting there sooner: prefetching, caching, streaming](#9-getting-there-sooner-prefetching-caching-streaming)
10. [Long-running operations](#10-long-running-operations)
11. [Offline and unreliable networks](#11-offline-and-unreliable-networks)
12. [Backend / system design for responsiveness](#12-backend--system-design-for-responsiveness)
13. [Approaches by platform and product type](#13-approaches-by-platform-and-product-type)
14. [Measuring and monitoring](#14-measuring-and-monitoring)
15. [Trade-offs](#15-trade-offs)
16. [Anti-patterns](#16-anti-patterns)
17. [Checklist](#17-checklist)
18. [Exercises](#18-exercises)
19. [Further reading](#19-further-reading)

---

## 1. Why responsiveness matters

- **Users judge quality by feel.** People can't measure milliseconds, but they notice instantly when a UI lags behind their fingers or freezes. A laggy UI feels broken and untrustworthy, even if every request succeeds.
- **Conversion and revenue.** Deloitte's *Milliseconds Make Millions* study (2020, with Google) found that a 0.1 s improvement in mobile site speed increased retail conversions by about 8% and travel conversions by about 10%. Google has reported that a large share of mobile visits are abandoned when a page takes more than about 3 seconds to load.
- **Errors and duplicates.** No feedback → users click again → duplicate orders, double payments, duplicate messages. Responsiveness bugs become data bugs (see the payment example in the [usability demo](../usability/index.html)).
- **Flow and productivity.** Below roughly 400 ms (the *Doherty threshold*), users and computers keep a rapid back-and-forth and people stay focused. Above it, attention drifts. For internal tools used all day, every sluggish interaction multiplies across thousands of uses.
- **Search ranking.** Google uses Core Web Vitals (including INP, a responsiveness metric) as a ranking signal.
- **Mobile and the "next billion users".** Many users are on mid- or low-end Android phones over slow or patchy networks. A UI that feels fine on a developer's laptop can freeze for seconds on those devices.

---

## 2. How humans perceive time

### 2.1 Response time limits

Covered in [Usability §4.6](../usability/README.md#46-response-time-limits). In short:

| Delay | Feels like | Design response |
|---|---|---|
| **< 100 ms** | Instant: "I did that" | Just do it, no indicator |
| **100 ms – 1 s** | Noticeable delay, flow kept | Immediate acknowledgement (pressed state, inline change) |
| **1 – 10 s** | Waiting; attention starts to wander | Skeleton or spinner; keep the rest of the UI usable |
| **> 10 s** | Task switch | Determinate progress, cancel, run in background, notify on completion |

### 2.2 Frames and jank

Screens refresh 60 times per second (many phones and laptops: 90–120). To feel smooth, every frame must be ready in time:

| Refresh rate | Time per frame |
|---|---|
| 60 Hz | **16.7 ms** (realistically ~10 ms for your code after browser overhead) |
| 120 Hz | 8.3 ms |

Missed frames show up as **jank**: stuttering scrolls, animations that hitch, typing that lags behind your fingers. Humans are very sensitive to this during direct manipulation (dragging, scrolling, drawing, typing). Touch-drag latency is noticeable even at a few tens of milliseconds.

### 2.3 The psychology of waiting

David Maister's *The Psychology of Waiting Lines* (1985) was about physical queues but applies directly to software:

| Principle | Software application |
|---|---|
| **Unoccupied time feels longer than occupied time** | Show partial content, progress steps, or something useful to read; let users keep working |
| **Uncertain waits feel longer than known, finite waits** | Determinate progress bars; time estimates; "Step 2 of 3" |
| **Unexplained waits feel longer than explained waits** | Say what's happening: "Checking availability…", "Uploading 3 of 12 photos" |
| **Anxiety makes waits feel longer** | Reassure on high-stakes actions: "Your payment is processing. Don't close this page." Better: make it safe to close. |
| **Pre-process waits feel longer than in-process waits** | Acknowledge the request immediately so the user knows they're "in the queue" |

Other perception effects:

- **Peak-end rule.** People remember the worst moment and the end. One 5-second freeze can define the experience more than a hundred fast interactions.
- **The ~20% rule.** As a rule of thumb (from Steven Seow's work on time perception), users often don't notice speed differences smaller than about 20%. Shaving 50 ms off a 2 s load won't be felt; cutting it to 1.5 s will. Conversely, regressions above ~20% get noticed.
- **Progress bar shape matters.** Research on progress bars (Harrison et al., 2007/2010) found that bars that speed up toward the end feel faster than bars that slow down, even when total time is identical. Stalls near the end ("stuck at 99%") feel worst.
- **The labor illusion.** Showing the work being done ("Searching 300 airlines…") can increase perceived value (Buell & Norton, 2011). It's a legitimate technique when the work is real; adding *fake* delays to look busy is a dark pattern.

---

## 3. Metrics and budgets

### 3.1 Core Web Vitals (web)

Google's user-centric metrics, measured on real users at the **75th percentile** (p75) per page, split by mobile and desktop.

| Metric | Measures | Good | Needs improvement | Poor |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Loading: when the main content (largest image or text block) appears | **≤ 2.5 s** | ≤ 4 s | > 4 s |
| **INP** (Interaction to Next Paint) | Responsiveness: time from a user input (click, tap, key) to the next visual update, taken near the worst interaction on the page | **≤ 200 ms** | ≤ 500 ms | > 500 ms |
| **CLS** (Cumulative Layout Shift) | Visual stability: how much content unexpectedly moves | **≤ 0.1** | ≤ 0.25 | > 0.25 |

**INP replaced FID** (First Input Delay) as a Core Web Vital in March 2024. FID only measured the *delay before* the first input was handled; INP measures the full input → paint time for interactions **throughout the visit**, which is much closer to what users feel.

INP breaks into three parts, each with different fixes:

```
 user clicks            handler starts         handler ends            next frame painted
     │──── input delay ────│──── processing ────│──── presentation ────│
     main thread busy       your event           style, layout, paint
     with other work        handler code         of the update
```

| Phase | Typical cause | Fix |
|---|---|---|
| Input delay | A long task (script parsing, hydration, analytics, timers) was running when the user clicked | Break up long tasks; defer non-essential scripts; less JS on load |
| Processing | Heavy event handler (big state update, sync filtering of 10k items) | Do the minimal visual update first, then yield; move heavy work to a worker or the server |
| Presentation | Huge DOM update, layout thrashing, expensive CSS | Smaller DOM, virtualization, `content-visibility`, avoid forced sync layout |

Other useful web metrics:

| Metric | Meaning | Rough target |
|---|---|---|
| **TTFB** (Time to First Byte) | Server + network until the first byte of HTML | ≤ 0.8 s |
| **FCP** (First Contentful Paint) | First text or image on screen | ≤ 1.8 s |
| **TBT** (Total Blocking Time) | Lab metric: total time of long tasks beyond 50 ms during load. A lab proxy for INP. | ≤ 200 ms (Lighthouse) |
| **Long tasks** | Any main-thread task > 50 ms | As few as possible during interaction |
| **LoAF** (Long Animation Frames API) | Frames delayed > 50 ms, with script attribution | Use to debug INP in the field |

### 3.2 The RAIL model

Google's older but still useful way to set budgets by type of interaction:

| | Goal | Budget |
|---|---|---|
| **R**esponse | React to input | Visible response within **100 ms** (so handle events in **< 50 ms**, leaving room for other work) |
| **A**nimation | Smooth motion and scrolling | Each frame in **~10 ms** of your work (16.7 ms total at 60 Hz) |
| **I**dle | Background work | Chunks of **≤ 50 ms**, so input is never blocked for longer |
| **L**oad | Show and become interactive | Usable content in a few seconds on a **mid-range phone on a slow network**, not on your laptop |

### 3.3 Native mobile metrics

| Metric | Platform guidance |
|---|---|
| **App startup** | Android vitals flags cold starts ≥ 5 s, warm starts ≥ 2 s, hot starts ≥ 1.5 s as excessive. Apple recommends a first frame within about 400 ms of launch. |
| **ANR** (Application Not Responding) | Android shows the "App isn't responding" dialog if the main thread doesn't respond to input for **5 s**. Google Play uses ANR rate as a quality signal for store visibility. |
| **Hangs** (iOS) | Main-thread unresponsiveness ≥ 250 ms is reported as a hang in Xcode Organizer / MetricKit |
| **Slow and frozen frames** | Android vitals: slow frames (> 16 ms) and frozen frames (> 700 ms) |
| **Scroll hitches** | iOS reports hitch time ratio (ms of hitches per second of scrolling) |

### 3.4 Set an interaction budget

Turn responsiveness into a measurable requirement, like any non-functional requirement:

| Interaction | Feedback budget | Completion budget (p75 / p95) |
|---|---|---|
| Typing in any field | Character on screen < 50 ms | — |
| Button press | Visual pressed/loading state < 100 ms | Depends on action |
| Toggle / like / checkbox | Optimistic update < 100 ms | Server confirms < 1 s / 3 s |
| Navigate to another screen | Skeleton or cached content < 100 ms | Main content < 1 s / 2.5 s |
| Search suggestions | Input never lags | Results < 300 ms after the user pauses |
| Report export | Acknowledged < 100 ms, job started | Done < 30 s / 2 min, with progress |

---

## 4. The main thread: why UIs freeze

In browsers (and in iOS, Android, Flutter and most UI frameworks), **one thread** does almost everything visible: runs your JavaScript event handlers, computes styles and layout, paints, and responds to input. While it's busy with one task, **input waits in the queue**. That's the root cause of most "the UI is frozen" bugs.

```
Main thread:  [ handle click 20ms ][ ████ JSON.parse 2MB + render 10k rows: 600ms ████ ][ paint ]
User:               types a letter here ↑ ……… letter appears 600 ms later …… ↑
```

### 4.1 Keep tasks short and yield

Split long work into chunks and **yield** to the browser between them so it can handle input and paint:

```js
// Yield to the main thread so pending input and rendering can run.
function yieldToMain() {
  if (globalThis.scheduler?.yield) return scheduler.yield();   // Chromium; keeps task priority
  return new Promise(resolve => setTimeout(resolve, 0));
}

async function processAll(items) {
  let deadline = performance.now() + 40;            // ~40 ms of work per chunk
  for (const item of items) {
    process(item);
    if (performance.now() > deadline) {
      await yieldToMain();
      deadline = performance.now() + 40;
    }
  }
}
```

**Do the visible part first.** In an event handler, update what the user is looking at (button state, the checkbox, the new message) first, then yield, then do the rest (analytics, syncing other components, prefetching).

```js
button.addEventListener('click', async () => {
  button.classList.add('is-pressed');   // 1. instant visual feedback
  await yieldToMain();                  // 2. let the browser paint it
  recalculateEverything();              // 3. heavy work afterwards
  sendAnalytics();                      //    and non-urgent work last
});
```

### 4.2 Move heavy work off the main thread

| Work | Where to put it |
|---|---|
| CPU-heavy computation (parsing, filtering big datasets, image processing, crypto, diffing) | **Web Worker** (web), background thread / coroutine (`Dispatchers.Default`, GCD, isolates in Flutter) |
| Data aggregation, joins, search over large data | **The server** (send results, not raw data) |
| Network and disk I/O | Async APIs (never synchronous XHR, never disk/network on the Android/iOS main thread) |
| Non-urgent work (analytics, prefetch, logs) | `requestIdleCallback`, after interaction, or batched |

### 4.3 Rendering performance

- **Animate only `transform` and `opacity`** where possible. These run on the compositor, so animations stay smooth even when the main thread is busy. Animating `width`, `height`, `top` or `left` triggers layout every frame.
- **Avoid layout thrashing**: reading layout (`offsetHeight`, `getBoundingClientRect`) right after writing styles forces the browser to recalculate layout synchronously. Batch reads, then writes.
- **Virtualize long lists**: render only the rows on screen (e.g. TanStack Virtual, `RecyclerView` on Android, `List`/`LazyVStack` on iOS). Rendering 10,000 DOM rows is slow; rendering 30 is not.
- **`content-visibility: auto`** lets the browser skip rendering off-screen sections.
- **Keep the DOM small.** Lighthouse warns above about 1,400 nodes. Huge DOMs make style, layout and every update slower (and slow down screen readers).
- **Frameworks**: React's `useTransition` / `useDeferredValue` mark updates as non-urgent so typing stays responsive while a big list re-renders; avoid re-rendering the whole tree on every keystroke (memoization, state colocation, signals).

### 4.4 Debounce and throttle

| Technique | Behavior | Use for |
|---|---|---|
| **Debounce** | Run once after the user *stops* for N ms | Search-as-you-type requests (~200–300 ms), autosave, form validation |
| **Throttle** | Run at most once every N ms | Scroll and resize handlers, drag position updates, analytics |
| `requestAnimationFrame` | Run once per frame, right before paint | Visual updates driven by pointer movement or scroll |

Note: debounce the **expensive work**, never the **echo of the input**. The typed character must always appear instantly.

---

## 5. Feedback patterns: what to show while waiting

### 5.1 Choose by expected duration

| Expected duration | Pattern |
|---|---|
| **< ~300 ms** | Immediate pressed / selected state. **No spinner**: a spinner that flashes for 150 ms looks like a glitch and makes things feel *slower*. |
| **~300 ms – 2 s** | Inline indicator on the element itself (button shows a small spinner and "Saving…"); dim stale content instead of removing it |
| **~1 – 10 s, layout known** | **Skeleton screen** with the real layout, filled in progressively |
| **~1 – 10 s, layout unknown** | Indeterminate spinner with a label saying what's happening |
| **> ~10 s** | **Determinate progress** (percentage, steps, items done), time estimate, cancel |
| **> ~1 min or unpredictable** | Background job: let the user leave, keep working, and **notify on completion** (toast, notification, email) |

**Delay and minimum display time.** A common pattern: don't show a loading indicator until ~200–400 ms have passed (fast responses never show one), and once shown, keep it for at least ~400–500 ms (so it doesn't flicker).

```js
async function withLoading(promise, show, hide, { delay = 300, minVisible = 500 } = {}) {
  let shownAt = null;
  const timer = setTimeout(() => { shownAt = performance.now(); show(); }, delay);
  try {
    return await promise;
  } finally {
    clearTimeout(timer);
    if (shownAt !== null) {
      const remaining = minVisible - (performance.now() - shownAt);
      if (remaining > 0) await new Promise(r => setTimeout(r, remaining));
      hide();
    }
  }
}
```

### 5.2 Acknowledge every input

- **Pressed / active states** on every button and tappable element, within one frame. On mobile, tap highlights and ripples exist for this reason.
- **Change the element the user touched**, not something far away. The button says "Saving…" itself; the toggle moves itself.
- **Prevent double actions**: disable or ignore repeated submits while one is in flight (and make the backend idempotent, because the client guard isn't enough).

### 5.3 Skeleton screens

A **skeleton screen** shows the shape of the upcoming content (gray blocks for text lines, image boxes) immediately, then fills it in. The term was popularized by Luke Wroblewski around 2013.

Why they work: the page "arrives" immediately, the user can see the structure and orient themselves, and content fills in place without jumping.

Rules:
- The skeleton must **match the real layout** (same sizes and positions). A skeleton that doesn't match causes layout shift when the real content arrives, which is worse than a spinner.
- Use it for **content areas with predictable structure** (feeds, cards, tables, profiles). For actions (submitting a form) use inline button states instead.
- Keep the **shell real**: header, navigation and anything already known render as real UI, not skeletons.
- Respect `prefers-reduced-motion` (use a static or very subtle shimmer).
- Don't leave skeletons forever: after a timeout, show an error state with retry.
- Accessibility: mark the loading region `aria-busy="true"` and announce completion ("Results loaded") in a polite live region, rather than letting a screen reader read dozens of empty blocks.

### 5.4 Progressive and partial rendering

Don't wait for **everything** before showing **anything**:

- Render text first, images later (with reserved space and a low-quality placeholder or dominant-color background).
- Render the above-the-fold content first; load below-the-fold sections lazily.
- Show the first page of results; load more on scroll or on demand.
- If one panel on a dashboard is slow, render the others and show a local skeleton in the slow one.
- If a non-critical dependency fails (recommendations, reviews), render the page without it rather than failing everything.

### 5.5 Keep stale content while refreshing

When data is already on screen and you're fetching an update (refresh, filter change, pagination):

- **Keep the old content visible** (optionally dimmed with a small "Updating…" indicator) instead of replacing it with a blank spinner.
- Show cached content instantly and **revalidate in the background** ("stale-while-revalidate"). Libraries like TanStack Query, SWR, Apollo and RTK Query do this by default.
- Indicate freshness if it matters ("Updated 2 minutes ago").

### 5.6 Honest progress indicators

- **Determinate** (percentage, "3 of 12 files") whenever you can measure progress. Indeterminate spinners for long waits create anxiety ("Is it stuck?").
- **Steps** are progress too: "Uploading → Processing → Done".
- **Don't fake it.** A bar that jumps to 90% and sits there teaches users to distrust all your progress bars. If progress is truly unknown, say so and give an estimate range.
- For multi-minute operations, show a **time estimate** that updates, and the ability to **cancel**.
- Keep the UI usable during the wait whenever possible: a modal progress dialog that blocks the whole app is a last resort.

---

## 6. Optimistic UI

**Optimistic UI** = update the interface **immediately**, as if the server request already succeeded, then reconcile when the response arrives. If it fails, roll back (or mark the item as failed with a retry).

Messaging apps are the classic example: your message appears instantly in the chat (often with a "sending" indicator), and gets a checkmark when the server confirms.

### 6.1 How it works

```
User clicks "Like"
  1. UI: heart filled, count 41 → 42           (instant, < 16 ms)
  2. Request: POST /posts/7/like  (Idempotency-Key: k_93f…)
  3a. 200 OK      → keep state (optionally sync count from server: 57)
  3b. Error/timeout → roll back: heart empty, count 41, show "Couldn't like. Retry?"
```

```js
async function toggleLike(post) {
  const previous = { liked: post.liked, count: post.count };
  // 1. Optimistic update
  post.liked = !post.liked;
  post.count += post.liked ? 1 : -1;
  render(post);
  try {
    // 2. Server call; the key makes retries safe
    const saved = await api.put(`/posts/${post.id}/like`, { liked: post.liked },
                                { headers: { 'Idempotency-Key': crypto.randomUUID() } });
    Object.assign(post, saved);           // 3a. reconcile with the server's truth
  } catch {
    Object.assign(post, previous);        // 3b. roll back
    showError('Couldn’t save your like.', { retry: () => toggleLike(post) });
  }
  render(post);
}
```

In React 19, `useOptimistic` implements this pattern; TanStack Query has `onMutate` / `onError` rollback hooks; Apollo has `optimisticResponse`.

### 6.2 When to use it (and when not)

| Good fit | Bad fit |
|---|---|
| Actions that **almost always succeed** (likes, toggles, reordering, renaming, adding a to-do, sending a chat message) | Actions that **often fail** or need server validation (username availability, coupon codes, stock-limited purchases) |
| **Easily reversible** actions | **Irreversible or high-stakes** actions: payments, sending money, placing orders, deleting accounts, sending email to customers |
| The result is **predictable** on the client | The result depends on server computation (prices with tax and shipping, search results, generated IDs shown to the user) |
| Single-user or low-conflict data | Highly contended data (last seat on a flight, auction bids) |

For the "bad fit" cases, use an **instant pending state** instead ("Processing payment…", button disabled), which is still responsive, just not optimistic.

### 6.3 Pitfalls

- **Rollback must be visible and understandable.** Silently reverting a change the user made seconds ago looks like a bug. Explain it and offer retry.
- **Keep failed items, don't drop them.** For created content (messages, comments, tasks), mark the item "Not sent · Retry" rather than deleting what the user wrote.
- **Idempotency is required.** Optimistic UIs retry automatically, users retry manually, networks retry by themselves. The server must deduplicate (idempotency keys, client-generated IDs, or naturally idempotent operations like `PUT liked=true` instead of `POST toggle`).
- **Ordering.** Two quick toggles (like → unlike) can arrive out of order. Send the desired final state (`liked: false`) rather than a toggle, serialize requests per item, or use version numbers.
- **Client-generated IDs** (UUIDs) let you create items optimistically and reference them (e.g. add a comment to a not-yet-confirmed post) without waiting for a server ID.
- **Reconcile with server truth.** The server may have changed other fields (the like count went from 41 to 57 because others liked it too).

---

## 7. Visual stability: no layout shift

**Layout shift** = visible content moves after the user has started looking at or reaching for it. It's one of the most frustrating responsiveness problems: you go to tap "Cancel" and a late-loading banner pushes "Buy now" under your finger.

| Cause | Fix |
|---|---|
| Images and videos without dimensions | Always set `width` and `height` attributes (or CSS `aspect-ratio`) so the browser reserves space before loading |
| Ads, embeds, iframes, cookie banners injected above content | Reserve a fixed-size slot in advance; put banners in overlays at the bottom, not pushed in at the top |
| Content loaded asynchronously above existing content | Insert below the viewport, or reserve space, or show a "3 new posts" button instead of auto-inserting |
| Web fonts swapping in with different metrics (text reflows) | `font-display: optional` or `swap` with fallback metric overrides (`size-adjust`, `ascent-override`); preload key fonts |
| Skeletons that don't match the real content size | Make skeleton dimensions match the real layout |
| Animations of layout properties | Animate `transform` instead of `top`/`height` (transforms don't count as layout shifts) |
| Late-loading "sticky" headers or toolbars | Render them in the initial HTML with their final size |

Layout shifts that happen within 500 ms after a user input (e.g. expanding an accordion the user clicked) are expected and excluded from CLS. Unexpected shifts are the problem.

---

## 8. Stale responses and race conditions

A subtle but very common responsiveness bug: **responses arriving out of order**.

```
User types:   "a"          "ap"         "app"
Requests:     ──GET a────────────────────────────────▶ 900 ms (slow: many results)
                           ──GET ap──────────▶ 500 ms
                                        ──GET app──▶ 200 ms
Responses:                                    app ✓   ap    a  ← arrives last, overwrites!
Screen shows: results for "a" while the input says "app"
```

Fixes, usually combined:

```js
let controller;
let latestRequest = 0;

const search = debounce(async (query) => {      // 1. fewer requests
  controller?.abort();                           // 2. cancel the previous one
  controller = new AbortController();
  const requestId = ++latestRequest;
  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal: controller.signal });
    const results = await res.json();
    if (requestId !== latestRequest) return;     // 3. ignore anything stale
    renderResults(query, results);
  } catch (e) {
    if (e.name !== 'AbortError') showSearchError();
  }
}, 250);
```

- Data-fetching libraries (TanStack Query, SWR, RTK Query) handle stale responses by keying caches by query.
- The same problem appears with fast tab switching, pagination, filters, and route changes. Any time a later request can finish before an earlier one.
- On the server, cancelled requests should stop work too (propagate cancellation / context deadlines) so abandoned searches don't waste capacity.

---

## 9. Getting there sooner: prefetching, caching, streaming

Responsiveness also means **starting work before the user asks**, and **showing results as they're produced**. The [Performance note](../../runtime-operational/performance/README.md) covers caching and network optimization in depth. The ones with the biggest effect on perceived speed:

### 9.1 Predict and prefetch

| Technique | How |
|---|---|
| **Prefetch likely next pages** | `<link rel="prefetch">`, framework link prefetching (Next.js, Remix, SvelteKit prefetch links in the viewport or on hover) |
| **Prerender** | The **Speculation Rules API** (Chromium) can prerender likely next pages so navigation is near-instant |
| **Hover / touchstart intent** | Start fetching when the pointer hovers a link (~100–300 ms before the click) or on `touchstart` |
| **Preload critical resources** | `<link rel="preload">` for the hero image or key font; `fetchpriority="high"` on the LCP image; `preconnect` to critical origins |
| **Precompute on the server** | Materialized views, precomputed feeds, warm caches for popular queries |
| **Upload early** | Start uploading a photo as soon as it's selected, while the user is still writing the caption (Instagram popularized this) |

Trade-off: prefetching wastes bandwidth, battery, and server capacity when the prediction is wrong. Respect `Save-Data` and slow connections, and prefetch only high-probability next steps.

### 9.2 Cache on the client

- **HTTP caching** (`Cache-Control`, `ETag`, `stale-while-revalidate`) for static and semi-static resources.
- **In-memory query cache** so going "back" to a screen is instant (TanStack Query, SWR, Apollo, Redux).
- **Persistent cache** (IndexedDB, SQLite on mobile) so the app opens with the last-known data immediately, then refreshes.
- **Back/forward cache (bfcache)**: browsers can restore the previous page instantly from memory. Avoid things that block it (e.g. `unload` handlers, `Cache-Control: no-store` on pages that don't need it).

### 9.3 Stream instead of waiting

- **Streaming HTML / server rendering**: send the page shell and fast parts immediately, stream slow parts as they're ready (React Server Components + Suspense, HTTP chunked transfer). The user sees content long before the slowest query finishes.
- **Streaming API responses**: for large results or AI responses, stream items or tokens as they're produced (Server-Sent Events, chunked JSON / NDJSON). For LLM features, **time to first token** matters more than total generation time.
- **Paginate and cursor** large datasets instead of fetching everything.

### 9.4 Send less work to the client

- Ship less JavaScript (code splitting per route, remove unused dependencies, avoid huge client-side state libraries for simple apps).
- **Hydration** of server-rendered pages can block the main thread for seconds on slow phones; consider partial / progressive hydration or islands architecture (Astro, Qwik, React Server Components).
- Defer third-party scripts (analytics, tag managers, chat widgets, A/B testing). They're among the biggest sources of long tasks and poor INP in real sites.

---

## 10. Long-running operations

Any operation that can take more than a few seconds (exports, imports, video processing, report generation, bulk updates, AI generation) shouldn't block an HTTP request or the UI.

### 10.1 The async job pattern

```
Client                                  Server
  │ POST /exports {filters}               │
  │──────────────────────────────────────▶│ validate, enqueue job
  │◀──────────── 202 Accepted ────────────│ { jobId: "j_123", status: "queued" }
  │                                       │        worker picks job from queue
  │ GET /exports/j_123   (or SSE/WebSocket)│
  │◀──── { status: "running", progress: 40, step: "Building PDF" }
  │ …                                     │
  │◀──── { status: "done", url: "https://…/report.pdf" }
  │                                       │ + notification (in-app, email, push)
```

- Return **`202 Accepted`** with a job ID within milliseconds; do the work in a **background worker** fed by a **queue**.
- Report **progress** (percent, current step, items processed) so the UI can show honest progress.
- Let users **leave and come back**: job state lives on the server, not in a browser tab. Show running jobs in a "Downloads" / "Activity" area.
- **Notify on completion** (in-app toast, notification, email with link) for long jobs.
- Support **cancel** (and make workers check for cancellation).
- **Idempotent job creation**, so a retried POST doesn't create two exports.

### 10.2 Getting progress updates to the client

| Method | How | Good for | Trade-offs |
|---|---|---|---|
| **Polling** | Client calls `GET /jobs/:id` every N seconds | Simple jobs, low frequency | Delay up to N seconds; wasted requests (use backoff, e.g. 1 s → 2 s → 5 s) |
| **Long polling** | Server holds the request until there's news | Older environments | More complex server handling |
| **Server-Sent Events (SSE)** | One-way stream from server over HTTP | Progress, notifications, AI token streams | One direction only; fine for most "progress" use cases |
| **WebSockets** | Two-way persistent connection | Chat, collaboration, live dashboards | Stateful connections; harder to scale and load-balance |
| **Push notifications / email** | Out-of-band | Very long jobs, user has left | Needs permissions / addresses |

### 10.3 Uploads

- Show **per-file progress** (bytes uploaded), not just a spinner.
- Use **resumable uploads** (chunked; e.g. the tus protocol, S3 multipart uploads) so a dropped connection doesn't restart a 2 GB upload from zero.
- Upload **directly to object storage** with a pre-signed URL rather than through your app servers.
- Process after upload (thumbnails, transcoding) as an async job with its own status.

---

## 11. Offline and unreliable networks

A UI that freezes or shows an infinite spinner when the network is slow or gone is unresponsive by definition. Mobile networks drop, lag, and switch constantly.

- **Timeouts with clear errors.** No request should spin forever. After a reasonable timeout, show an error with retry (and retry automatically with backoff for idempotent requests).
- **Offline indicators.** Show that the app is offline and what still works; queue actions and say they'll be sent when back online.
- **Offline-first**: read from a local store (IndexedDB, SQLite), write to the local store and a sync **queue**, sync in the background. The UI responds instantly regardless of network.
- **Local-first** apps (e.g. Linear, Figma-style collaboration) keep a full local copy and sync changes with conflict resolution (CRDTs or server-ordered operations). Every interaction is local-speed.
- **Service workers** (web) enable caching the app shell, offline pages, and background sync.
- Test with **network throttling** (DevTools "Slow 4G", "Offline") and on real devices in poor-signal areas.

---

## 12. Backend / system design for responsiveness

Frontend techniques can hide latency, but they can't hide everything. Backend decisions determine what the UI is *able* to do.

| Backend decision | Effect on responsiveness |
|---|---|
| **Latency budget per interaction** | Split the end-to-end budget across hops (e.g. 200 ms total = 50 ms network + 100 ms API + 50 ms render). Each team knows its share. |
| **Optimize the tail (p95/p99), not the average** | Users feel the slow requests; a page making 10 calls almost always hits someone's p99 ([Performance §11](../../runtime-operational/performance/README.md#11-tail-latency)) |
| **Backend-for-frontend (BFF) / aggregation** | One request per screen instead of 8 sequential ones. Each round trip on mobile can cost 100+ ms. |
| **Edge / CDN, regional deployments** | Physically closer = less latency for every interaction |
| **Idempotent APIs** | Make optimistic UI, retries, and offline queues safe |
| **Client-generated IDs** | Allow optimistic creation and offline work |
| **Read-your-own-writes consistency** | After a user saves, *their* next read shows the change, even with replicas and caches. Otherwise the optimistic UI "un-does" itself on refresh. |
| **Async jobs + progress endpoints** | Long work doesn't block requests; UI can show progress ([§10](#10-long-running-operations)) |
| **Streaming responses (SSE, chunked)** | First results visible early |
| **Pagination, cursors, field selection** | Smaller, faster responses for what's on screen |
| **Timeouts and fallbacks for non-critical dependencies** | The product page renders even if the recommendation service is slow; circuit breakers stop slow dependencies from dragging everything down |
| **Separate interactive and batch workloads** | Priority queues / separate pools so a nightly import doesn't slow down user clicks |
| **Push updates (WebSocket/SSE) instead of polling** for live data | Faster updates and less load |
| **Request cancellation propagation** | Abandoned requests stop consuming capacity |
| **Rate limits that return `429` + `Retry-After`** quickly | Clients can back off gracefully instead of hanging |
| **SLOs on user-facing latency** | e.g. "p75 INP < 200 ms, p95 search API < 300 ms", tracked and alerted like availability |

---

## 13. Approaches by platform and product type

| Product type | Responsiveness priorities |
|---|---|
| **Content / news / marketing sites** | LCP and CLS: fast first content, no jumping layouts (ads!), minimal JavaScript. Server rendering + CDN. |
| **E-commerce** | Fast product and search pages; instant filters with stale content kept; prefetch product pages; **no** optimistic UI for payment, but instant pending states; stock info that doesn't shift layout. |
| **SaaS / dashboards / productivity** | INP: interactions over a long session; big tables (virtualization); keyboard actions that never lag; optimistic edits; local caches; background exports. |
| **Chat / collaboration** | Optimistic send with delivery status; real-time push (WebSocket); offline queue; presence and typing indicators; local-first for editors. |
| **Mobile apps** | Startup time; no main-thread I/O (ANR/hangs); cached data on open; 60/120 fps scrolling; resilient to bad networks. |
| **Games, drawing, music, AR/VR** | Frame time and input latency are the product: 16 ms or less, predictive rendering, client-side prediction with server reconciliation (multiplayer). VR needs very low motion-to-photon latency to avoid motion sickness. |
| **AI / LLM features** | Stream tokens (time to first token); show intermediate steps ("Searching…", "Reading 3 documents"); allow stopping; keep the input usable while generating. |
| **Developer tools / CLIs** | Instant startup (avoid heavy runtimes on every invocation); progress bars for long operations; stream output instead of buffering; watch mode with incremental rebuilds. |
| **Kiosks / embedded / TV** | Weak hardware: very small UI work per frame; immediate touch or remote-control feedback; no complex animations. |

---

## 14. Measuring and monitoring

### 14.1 Field (real users) vs lab

| | Field / RUM (Real User Monitoring) | Lab / synthetic |
|---|---|---|
| Data from | Real users, real devices and networks | Controlled test runs |
| Tells you | What users actually experience (distribution, p75, worst cases by device/country) | Why it's slow; reproducible before/after comparisons |
| Tools | `web-vitals` library, Chrome UX Report (CrUX), PageSpeed Insights field data, RUM products (Datadog, New Relic, Sentry, SpeedCurve, Firebase Performance) | Lighthouse, Chrome DevTools Performance panel, WebPageTest, Android Studio profiler, Xcode Instruments |
| Responsiveness metric | **INP** | **TBT** (proxy), DevTools interaction traces |

Use both: field data to find **what** is slow for **whom**, lab tools to find **why**.

### 14.2 Collect Core Web Vitals from real users

```js
import { onINP, onLCP, onCLS } from 'web-vitals/attribution';

function send(metric) {
  navigator.sendBeacon('/analytics/vitals', JSON.stringify({
    name: metric.name,               // 'INP' | 'LCP' | 'CLS'
    value: metric.value,
    rating: metric.rating,           // 'good' | 'needs-improvement' | 'poor'
    page: location.pathname,
    // attribution tells you WHICH element/interaction and which phase was slow
    target: metric.attribution?.interactionTarget ?? metric.attribution?.target,
  }));
}
onINP(send); onLCP(send); onCLS(send);
```

### 14.3 Practices

- **Look at p75 and above**, segmented by **device class** (low-end Android vs iPhone vs desktop), **network**, **country** and **page type**. Averages hide the users having the worst time.
- **Test on a mid-range Android phone** (or DevTools CPU throttling 4–6× + "Slow 4G"). Your laptop is not representative.
- **Performance budgets in CI**: fail the build if JavaScript bundle size, Lighthouse score, or TBT regresses beyond a threshold (Lighthouse CI, bundle size checks).
- **Trace slow interactions**: DevTools Performance panel → record → find long tasks and which script caused them. Long Animation Frames (LoAF) gives script attribution in the field.
- **Track product metrics alongside**: drop-off and rage clicks (repeated rapid clicks) often point to responsiveness problems.
- **Alert on regressions** like any SLO: a deploy that doubles INP is an incident.

---

## 15. Trade-offs

| Tension | Notes |
|---|---|
| **Optimistic UI vs correctness** | Feels instant but can show things that later turn out false. Reserve for low-risk, high-success-rate actions; always roll back visibly. |
| **Prefetching vs cost** | Faster navigation vs wasted bandwidth, battery, and server load on wrong predictions. Prefetch high-probability next steps only; respect data saver. |
| **Caching / stale-while-revalidate vs freshness** | Instant display of possibly outdated data. Fine for feeds; risky for prices, balances, stock levels (show freshness or refresh before critical actions). |
| **Rich client vs light client** | Heavy SPAs enable instant interactions after load but cost load time and INP on weak devices; server-rendered pages load fast but each interaction may need a round trip. Many apps mix both (islands, server components). |
| **Animation vs speed** | Short transitions (150–300 ms) help users follow changes; long or blocking animations make everything slower. Never make users wait for an animation to finish before they can act. |
| **Skeletons vs simplicity** | Skeletons need to match layouts and be maintained as designs change. For simple or fast views, they may not be worth it. |
| **Real-time push vs complexity** | WebSockets give instant updates but add stateful infrastructure. Polling with backoff is often good enough. |
| **Offline/local-first vs complexity** | Instant UI and offline support, but conflict resolution, sync bugs, storage limits and data migrations on clients are hard. |
| **Perceived vs actual performance** | Perceived-performance tricks hide latency; they don't remove it. If actual latency keeps growing, no amount of skeletons will save the experience. Fix the real bottlenecks too. |

---

## 16. Anti-patterns

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| **No feedback on click** | Users think it didn't register, click again → duplicates | Instant pressed/pending state; idempotent backend |
| **Full-page spinner for everything** | Hides everything the user had; even fast loads feel slow | Local indicators; keep stale content; skeletons |
| **Spinner flash** on fast operations | Looks glitchy; feels slower | Delay indicator ~300 ms; minimum display time |
| **Blank screen until all data arrives** | Feels much slower than it is | Progressive rendering; stream; skeletons |
| **Infinite spinner** with no timeout | User stuck forever, no way out | Timeouts, error state, retry |
| **Fake progress bar** (jumps to 90%, sits there) | Destroys trust | Real progress or honest indeterminate state + estimate |
| **Blocking modal during long jobs** | User can't do anything else; tab must stay open | Background job, progress in a corner, notify on completion |
| **Main-thread work on input** (sync filtering, heavy re-render per keystroke) | Typing lags, INP poor | Debounce expensive work, yield, workers, virtualization |
| **Debouncing the input itself** | Characters appear late | Only debounce the expensive side effect |
| **Layout shift** (images without size, ads pushing content) | Mis-taps, lost reading position | Reserve space; `width`/`height`; no content injection above |
| **Ignoring out-of-order responses** | Wrong results displayed | Abort + request IDs / query-keyed caches |
| **Silent optimistic rollback** | User's change "disappears" mysteriously | Visible failure state + retry |
| **Optimistic UI for payments / irreversible actions** | Shows success that didn't happen | Pending state, then confirmed result |
| **Waiting for animations** before input is accepted | Every action pays an animation tax | Interruptible, short animations |
| **Third-party scripts loaded eagerly** | Long tasks during load and interaction | Defer, lazy-load, audit, remove |
| **Sequential request waterfalls** (A then B then C) | Latency adds up, especially on mobile | Parallelize, aggregate (BFF), preload |
| **Testing only on fast devices and networks** | Problems invisible to the team | Throttling, real mid-range devices, RUM by device class |
| **Artificial delays to look "busy"** | Wastes users' time; deceptive if the work isn't real | Only show real work |

---

## 17. Checklist

**Requirements**
- [ ] Interaction budgets defined (feedback < 100 ms; completion targets per key flow)
- [ ] User-facing latency SLOs (e.g. p75 INP ≤ 200 ms, LCP ≤ 2.5 s, CLS ≤ 0.1; API p95 per endpoint)
- [ ] Target device and network profile defined (e.g. mid-range Android on 4G)

**Feedback & states**
- [ ] Every interactive element has an immediate pressed/active state
- [ ] Buttons show pending state for async actions; double submits prevented
- [ ] Loading indicators delayed ~300 ms and shown for a minimum time
- [ ] Skeletons for content with predictable layout, matching real dimensions
- [ ] Stale content kept (dimmed) during refresh; cached data shown instantly
- [ ] Determinate progress, steps and cancel for long operations
- [ ] Every request has a timeout, error state and retry; nothing spins forever
- [ ] Loading and completion announced to screen readers (`aria-busy`, live regions)
- [ ] `prefers-reduced-motion` respected for spinners, shimmers, transitions

**Client engineering**
- [ ] No long tasks (> 50 ms) during interactions; heavy work yields or runs in workers
- [ ] Expensive work debounced/throttled; input echo never delayed
- [ ] Long lists virtualized; DOM size kept reasonable
- [ ] Animations use `transform`/`opacity`
- [ ] Images/embeds have reserved space; no content injected above the viewport
- [ ] Stale/out-of-order responses ignored or aborted
- [ ] Optimistic updates (where appropriate) with visible rollback and retry
- [ ] Likely next steps prefetched; critical resources preloaded
- [ ] JavaScript and third-party scripts minimized and deferred

**Backend**
- [ ] Idempotent mutations (idempotency keys or naturally idempotent operations)
- [ ] Read-your-own-writes for the acting user
- [ ] One request per screen where possible (aggregation / BFF); no waterfalls
- [ ] Long work as async jobs: `202` + job ID, progress endpoint or push, cancel, notification
- [ ] Streaming for large or generated responses
- [ ] Timeouts, fallbacks and circuit breakers for non-critical dependencies
- [ ] Interactive traffic isolated from batch workloads

**Measurement**
- [ ] RUM for Core Web Vitals (or native equivalents) at p75, segmented by device/network
- [ ] Lab testing with CPU and network throttling
- [ ] Performance budgets in CI
- [ ] Regressions alerted on like other SLOs

---

## 18. Exercises

1. **Throttle yourself.** Open an app you built with DevTools CPU throttling at 6× and network "Slow 4G". Use it for 5 minutes. List every moment where it felt stuck, jumped, or gave no feedback.
2. **Find the long task.** Record a slow interaction in the DevTools Performance panel. Identify the long task, the function responsible, and which INP phase (input delay, processing, presentation) it affects. Fix it by yielding or moving work.
3. **Design the states.** For a "Load comments" section, design: initial skeleton, partial load, refresh with stale content, empty, error with retry, offline. Make the skeleton match the real layout exactly.
4. **Optimistic to-do list.** Build a to-do list with optimistic add/complete/delete against a fake API that fails 20% of the time and has random latency. Handle rollback, retry, out-of-order responses, and duplicate-safe retries (idempotency keys).
5. **Race condition hunt.** Build search-as-you-type against an API whose latency is *higher for shorter queries*. Reproduce the stale-results bug, then fix it with debounce + `AbortController` + request IDs.
6. **Async export.** Design (API + data model + UI) a CSV export of 5 million rows: request, job queue, progress reporting, cancellation, completion notification, download link expiry, and what the UI shows if the user refreshes the page mid-export.
7. **Measure real users.** Add the `web-vitals` snippet ([§14.2](#142-collect-core-web-vitals-from-real-users)) to a project. After a week, compare p75 INP for low-end mobile vs desktop. Which interactions are worst?
8. **System-design tie-in.** A product page calls 6 backend services (product, price, stock, reviews, recommendations, shipping estimate). Design it so the page shows useful content within 1 s even when reviews and recommendations take 3 s or fail.

---

## 19. Further reading

- **web.dev**: *Web Vitals*, *Interaction to Next Paint (INP)*, *Optimize long tasks*, *Optimize INP*, *Optimize CLS*, *Optimize LCP*, *RAIL model*.
- Nielsen Norman Group: *Response Times: The 3 Important Limits*, *Progress Indicators Make a Slow System Less Insufferable*, *Skeleton Screens 101*, *The Need for Speed*.
- David Maister, *The Psychology of Waiting Lines* (1985).
- Steven Seow, *Designing and Engineering Time: The Psychology of Time Perception in Software* (2008).
- Harrison, Amento, Kuznetsov & Bell, *Rethinking the Progress Bar* (UIST 2007); Harrison, Yeo & Hudson, *Faster Progress Bars: Manipulating Perceived Duration with Visual Augmentations* (CHI 2010).
- Buell & Norton, *The Labor Illusion: How Operational Transparency Increases Perceived Value* (Management Science, 2011).
- Ilya Grigorik, *High Performance Browser Networking* (free online): network fundamentals behind latency.
- Addy Osmani, *The Cost of JavaScript* talks and articles; *Image Optimization* (book).
- Deloitte & Google, *Milliseconds Make Millions* (2020).
- Android Developers: *Android vitals* (startup, ANR, rendering); Apple: *Improving app responsiveness*, *Reducing your app's launch time*.
- Ink & Switch, *Local-first software* (2019).
- TanStack Query docs (optimistic updates, stale-while-revalidate); React docs: `useTransition`, `useDeferredValue`, `useOptimistic`.
- Related notes: [Performance](../../runtime-operational/performance/README.md) · [Usability](../usability/README.md) · [Accessibility](../accessibility/README.md).
