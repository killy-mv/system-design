# Sustainability

> **Sustainability** (green software) = the system does its job with the **least energy, hardware and carbon emissions** practical, and the organisation measures that like any other quality.
> "How many grams of CO₂ does one request, one user or one training run cost, and what would halve it?"

Related: [Efficiency](../../runtime-operational/efficiency/README.md) (work per unit of resource; the main technical lever), [Cost efficiency](../cost-efficiency/README.md) (waste costs money and carbon alike), [Elasticity](../../runtime-operational/elasticity/README.md) (switch off what's idle), [Performance](../../runtime-operational/performance/README.md), [Projected lifetime](../projected-lifetime/README.md) (longer-lived hardware and software).

Most of what reduces emissions is ordinary efficiency work, covered on those pages. This page adds what's specific to sustainability: **how to measure carbon, embodied emissions, and when and where the work runs**.

---

## 1. Why it matters

- **Data centres and networks use a noticeable and growing share of the world's electricity**, and AI workloads are pushing that up.
- **Regulation and reporting**: large companies increasingly have to report emissions, including those of their suppliers (for example under the EU's Corporate Sustainability Reporting Directive). Their software vendors get asked for numbers.
- **Customers and procurement** ask for it in tenders.
- **It usually saves money**: an idle server wastes both electricity and budget.
- It's the right thing to do with shared resources.

---

## 2. Where software's emissions come from

| Source | What it is | Software's influence |
|---|---|---|
| **Operational emissions** | Electricity used while running: servers, storage, network, cooling, user devices | How much work is done, how efficiently, where and when |
| **Embodied emissions** | Carbon emitted to manufacture, transport and dispose of hardware | How much hardware you need and how long it stays useful |

Key terms:

| Term | Meaning |
|---|---|
| **Carbon intensity** | Grams of CO₂-equivalent per kWh of electricity (gCO₂e/kWh). Varies widely **by region** (energy mix) and **by hour** (wind, sun, demand) |
| **PUE** (power usage effectiveness) | Total data-centre energy ÷ energy used by IT equipment. 1.0 would be perfect; large cloud providers report values close to 1.1, many older facilities are well above that |
| **Energy proportionality** | An idle server still draws a substantial share of its peak power, so a few busy machines beat many idle ones |
| **Scopes 1, 2, 3** (GHG Protocol) | Direct emissions; purchased electricity; everything else in the value chain. Your cloud usage is in your scope 3 |
| **Location-based vs market-based** | Emissions by the actual local grid mix, vs after counting purchased renewable energy certificates. Know which one a number uses |

For phones and laptops, manufacturing is typically the larger part of lifetime emissions. Software that forces users to replace working devices has a real carbon cost that never shows up in your own bill.

---

## 3. Measuring: Software Carbon Intensity (SCI)

The **SCI** specification from the Green Software Foundation (also published as ISO/IEC 21031) gives a rate, not a total:

```
SCI = ((E × I) + M) per R

E = energy used by the software (kWh)
I = carbon intensity of that electricity (gCO₂e/kWh)
M = embodied emissions of the hardware, for the share and time you use it (gCO₂e)
R = functional unit: per request, per user, per order, per training run
```

Because it's a rate per unit of work, it can only be improved by actually emitting less per unit, not by buying offsets.

### Worked example: SCI of an API

Illustrative numbers. A service runs on **10 servers** averaging **200 W** each, in a data centre with **PUE 1.2**, on a grid at **400 gCO₂e/kWh**. Each server has **1,200 kgCO₂e** of embodied emissions and a 4-year life. The service handles **20 million requests a day**.

```
E = 10 × 0.2 kW × 24 h × 1.2             = 57.6 kWh / day
E × I = 57.6 × 400                       = 23,040 g / day

M = 1,200,000 g / (4 × 365 days)         ≈ 822 g per server per day
    × 10 servers                         = 8,220 g / day

SCI = (23,040 + 8,220) / 20,000 thousand requests
    ≈ 1.56 gCO₂e per 1,000 requests
```

**Change 1: right-size.** Consolidate onto 6 servers, now busier at 250 W each:

```
E × I = 6 × 0.25 × 24 × 1.2 × 400        = 17,280 g
M     = 6 × 822                          =  4,932 g
SCI   = 22,212 / 20,000                  ≈ 1.11 g per 1,000 requests   (−29%)
```

**Change 2: move to a region at 100 gCO₂e/kWh.**

```
E × I = 43.2 kWh × 100                   =  4,320 g
M                                        =  4,932 g
SCI   = 9,252 / 20,000                   ≈ 0.46 g per 1,000 requests   (−70% overall)
```

Two lessons: **region choice can matter more than any code optimisation**, and on a clean grid **embodied emissions become the larger part** (here 53%), so the remaining lever is using less hardware for longer.

**Measuring in practice**
- **Cloud provider reports**: AWS Customer Carbon Footprint Tool, Google Cloud Carbon Footprint, Azure Emissions Impact Dashboard. Good for trends; coarse and delayed.
- **Estimate from usage**: Cloud Carbon Footprint (open source) converts billing and usage data into energy and carbon estimates.
- **Measure energy directly**: Kepler (per-pod energy on Kubernetes), Scaphandre, RAPL counters, CodeCarbon (Python and ML jobs).
- **Carbon intensity data**: Electricity Maps, WattTime, national grid operators.
- **Front end**: page weight and transfer size as a proxy; tools such as Website Carbon or CO2.js.

All of these are estimates. Use them to compare options and track trends, not as precise accounting.

---

## 4. Reducing emissions

The Green Software Foundation groups the actions into three: **use less energy, use less hardware, use cleaner energy**.

| Lever | Techniques |
|---|---|
| **Energy efficiency** | Do less work: caching, fewer round trips, efficient queries and algorithms, smaller payloads, compression. Fix hot paths found by profiling ([Efficiency §3](../../runtime-operational/efficiency/README.md#3-techniques)) |
| **Hardware efficiency** | Higher utilisation: right-size, bin-pack, autoscale, scale to zero, shared multi-tenant infrastructure. Efficient processors. Keep hardware in service longer |
| **Carbon awareness: where** | Choose low-carbon regions for workloads that aren't tied to a location |
| **Carbon awareness: when** | Shift flexible work (batch jobs, ML training, backups, CI, video transcoding) to hours when the grid is cleaner |
| **Demand shaping** | Do less when energy is dirty: lower default video quality, defer non-urgent processing, "eco mode" options |
| **Data** | Retention limits, delete unused data, tier to colder storage, fewer copies. Stored data consumes energy and hardware for as long as it exists |
| **Network** | CDN and edge caching, smaller assets, modern image and video codecs, avoid autoplay and unnecessary polling |
| **Client side** | Light pages and apps, less JavaScript, dark themes on OLED screens, keep supporting older devices and OS versions |
| **AI / ML** | The smallest model that's good enough, fine-tune instead of training from scratch, quantisation and distillation, cache results, batch inference, train in clean regions and hours |
| **Environments** | Shut down non-production out of hours; expire preview environments |

### Worked example: time shifting

A nightly batch job uses **100 kWh**. On this grid, carbon intensity is about **450 gCO₂e/kWh** at 02:00 and **150 gCO₂e/kWh** at 13:00 when solar is plentiful (illustrative):

```
Run at 02:00:  100 × 450 = 45 kg CO₂e
Run at 13:00:  100 × 150 = 15 kg CO₂e      → two thirds less, same work, same code
```

If the job only has to finish "by tomorrow morning", a scheduler that reads a carbon-intensity forecast (for example through the Green Software Foundation's Carbon Aware SDK, or a carbon-aware scaler for KEDA) can pick the window. Check the effect on cost and on the systems the job loads at midday.

---

## 5. Trade-offs

| Sustainability vs | Tension |
|---|---|
| **Availability** | Redundancy and headroom mean hardware that's mostly idle. Keep what availability needs; remove what's merely forgotten |
| **Latency** | The cleanest region may be far from users; batching and deferring add delay |
| **Data residency** | Law may fix the region regardless of its grid |
| **Cost** | Usually aligned, but not always: the cheapest region or hour isn't necessarily the cleanest |
| **Features** | Higher resolution, more AI, more tracking and more retention all use more energy |
| **Simplicity** | Carbon-aware scheduling and multi-region placement add moving parts |
| **Efficiency gains** | Cheaper computing tends to be used more (the rebound effect, or Jevons paradox), so track totals as well as per-unit rates |

---

## 6. Anti-patterns

- **Greenwashing**: claiming "carbon neutral" from offsets or certificates while actual energy use grows.
- **No measurement**: sustainability as a slogan, with no number per unit of work.
- **Optimising the small part**: weeks spent on code micro-optimisations while idle environments run all year in a high-carbon region.
- **Ignoring embodied carbon**: replacing working hardware early for a small efficiency gain.
- **Ignoring the client**: heavy pages and apps that drain batteries and make old devices unusable.
- **Keeping all data forever.**
- **The largest model for every task.**
- **Cutting resilience** and calling it sustainability.

---

## 7. Checklist

- [ ] A carbon or energy metric per functional unit (SCI or a proxy) tracked over time, alongside total emissions
- [ ] Carbon intensity considered when choosing regions
- [ ] Utilisation targets and autoscaling in place; non-production shut down when unused
- [ ] Flexible workloads identified and scheduled for cleaner hours where practical
- [ ] Data retention and storage tiering policies
- [ ] Page weight and client energy use have budgets; older devices supported
- [ ] Model size and inference cost considered for AI features
- [ ] Sustainability included in design reviews next to cost and performance
- [ ] Claims are based on measured reductions, and say which accounting method they use

---

## 8. Exercises

1. Compute the SCI for a service you know, using estimates for power, PUE, carbon intensity and embodied emissions as in §3. Which term dominates? Which single change reduces it most?
2. Look up the current carbon intensity of three cloud regions you could deploy to. What would moving a batch workload between them change, and what else (latency, residency, cost) would it affect?
3. List the workloads in a typical web company that can be shifted in time by a few hours, and those that can't. What would a carbon-aware scheduler need to know about each?
4. A page weighs 4 MB and is viewed 10 million times a month. Propose changes to halve its transfer size, and explain who benefits besides the planet.

---

## 9. Further reading

Green Software Foundation: "Principles of Green Software Engineering" (learn.greensoftware.foundation), the Software Carbon Intensity specification (ISO/IEC 21031), the Carbon Aware SDK and the Green Software Patterns catalogue; Anne Currie, Sarah Hsu & Sara Bergman, *Building Green Software* (O'Reilly); AWS Well-Architected **Sustainability pillar**, and the equivalent Azure and Google Cloud guidance; the GHG Protocol (scopes 1–3); Tom Greenwood, *Sustainable Web Design*, and the W3C Web Sustainability Guidelines; International Energy Agency reports on data centres and energy; Electricity Maps and WattTime (carbon intensity data); the Kepler, Cloud Carbon Footprint and CodeCarbon projects.
