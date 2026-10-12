# Sustainability: hands-on practice notes

Planned labs for learning sustainability by **measuring energy and carbon per unit of work, then changing how much, where and when the work runs**. Theory is in [README.md](README.md).

They need **Docker**, **k6**, a **spreadsheet**, **Python** with **CodeCarbon**, and optionally **kind** + **Kepler** for per-pod energy. Direct power readings (RAPL) need Linux on bare metal; on a laptop VM or WSL, the tools fall back to estimates, which is fine for comparing before and after. A free API key from **Electricity Maps** or **WattTime** gives real carbon-intensity data; a saved CSV of one week works too.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** requests per core and right-sizing (efficiency labs 1 and 2), queue-based scaling to zero (elasticity lab 3), unit economics model (cost-efficiency lab 1), Core Web Vitals (performance lab 13).

---

## Labs

### 1. SCI on paper ⭐
- **Do:** for the demo system or `messy-shop` as you'd deploy it, build a spreadsheet with the SCI formula: servers, average power, PUE, regional carbon intensity, embodied emissions per server and lifetime, requests per day.
- **Measure:** gCO₂e per 1,000 requests, and the split between operational and embodied.
- **Then:** model three changes separately: half the servers, a region with a quarter of the carbon intensity, hardware kept 6 years instead of 4. Which wins?
- **See:** README §3 worked example.

### 2. Measure energy per request ⭐
- **Build:** run one API endpoint under a fixed k6 load for 5 minutes while measuring energy (Kepler on kind, Scaphandre, or CodeCarbon wrapped around the server process).
- **Measure:** joules or Wh per 1,000 requests for the baseline. Then repeat after one change at a time: add a cache, remove an N+1 query, enable response compression, shrink the JSON payload.
- **Check:** which change saved the most energy? Did any save latency but not energy, or the reverse?
- **See:** README §4.

### 3. Idle is not free
- **Build:** start 6 replicas of a service with no traffic and measure power for 5 minutes; then send the same total load to 6 replicas and to 2 replicas.
- **Measure:** energy per request in each case, and the idle draw as a share of loaded draw.
- **Then:** add scale-to-zero for a non-production environment (KEDA or a cron-based schedule) and estimate hours saved per week.
- **See:** README §2 (energy proportionality), §4.

### 4. Carbon-aware batch scheduling ⭐
- **Build:** a small scheduler for a batch job with a deadline ("must finish within 24 h, takes 2 h"). It reads an hourly carbon-intensity forecast (API or saved CSV) and starts the job in the lowest-carbon window that still meets the deadline.
- **Measure:** estimated emissions over a week of runs vs always starting at 02:00.
- **Break:** give it a forecast that's wrong, or a deadline that's too tight. Does it still finish on time?
- **See:** README §4 worked example.

### 5. Lighter front end
- **Do:** take a page you built (or a UI-UX demo `index.html` from this repo). Record transfer size, request count and CPU time in browser dev tools and Lighthouse; estimate emissions per view with CO2.js or Website Carbon.
- **Fix:** modern image formats and correct sizes, remove unused JavaScript and fonts, cache headers, lazy loading.
- **Measure:** before and after, including on a throttled low-end device profile.
- **See:** README §4, [Responsiveness](../../UI-UX/responsiveness/README.md).

---

## Progress

- [ ] 1. SCI on paper
- [ ] 2. Measure energy per request
- [ ] 3. Idle is not free
- [ ] 4. Carbon-aware batch scheduling
- [ ] 5. Lighter front end
