# Safety

> **Safety** = the system **doesn't cause unacceptable harm** to people, property or the environment, in normal use, under foreseeable misuse, and when things break.
> "If this software does the wrong thing, or stops at the wrong moment, can somebody get hurt? What stops that?"

**ISO/IEC 25010:2023** added safety as a top-level quality characteristic, with five sub-characteristics: operational constraint, risk identification, **fail safe**, **hazard warning** and safe integration.

Related: [Fail-safe](../fail-safe/README.md) (failing into a safe state), [Hazard warning](../hazard-warning/README.md) (telling people about danger in time), [Fault tolerance](../../runtime-operational/fault-tolerance/README.md) and [Reliability](../../runtime-operational/reliability/README.md) (keeping the system working), [User error protection](../../UI-UX/user-error-protection/README.md) (human error in ordinary products), [Compliance](../../security/compliance/README.md).

These are **learning notes**. Real safety-critical work is done under the applicable standards and regulations, with trained safety engineers and independent assessment. Nothing here replaces that.

---

## Table of contents

1. [Why safety matters to software engineers](#1-why-safety-matters-to-software-engineers)
2. [Safety vs reliability vs security](#2-safety-vs-reliability-vs-security)
3. [Vocabulary: harm, hazard, risk](#3-vocabulary-harm-hazard-risk)
4. [The safety lifecycle](#4-the-safety-lifecycle)
5. [Hazard analysis methods](#5-hazard-analysis-methods)
6. [Risk reduction: the hierarchy of controls](#6-risk-reduction-the-hierarchy-of-controls)
7. [Integrity levels and standards](#7-integrity-levels-and-standards)
8. [Safety architecture patterns](#8-safety-architecture-patterns)
9. [Software practices for safety](#9-software-practices-for-safety)
10. [Human factors](#10-human-factors)
11. [Safety cases](#11-safety-cases)
12. [Safety and security together](#12-safety-and-security-together)
13. [Machine learning in safety-related systems](#13-machine-learning-in-safety-related-systems)
14. [Approaches by domain](#14-approaches-by-domain)
15. [Trade-offs](#15-trade-offs)
16. [Anti-patterns](#16-anti-patterns)
17. [Real-world incidents](#17-real-world-incidents)
18. [Checklist](#18-checklist)
19. [Exercises](#19-exercises)
20. [Further reading](#20-further-reading)

---

## 1. Why safety matters to software engineers

- Software now controls things that used to be controlled by hardware and trained operators: brakes, infusion pumps, radiation machines, trains, chemical plants, power grids.
- Software **doesn't wear out**. It fails because of design errors, and those are present in every copy at once. Redundant copies of the same software share the same bug ([Fault tolerance §4](../../runtime-operational/fault-tolerance/README.md#4-correlated-failures-the-real-enemy)).
- "Ordinary" software can be safety-related too: emergency dispatch, navigation that routes people into danger, clinical decision support, medication dose calculators, building access, industrial IoT dashboards operators rely on.
- Harm is often **irreversible**. You can't roll back an overdose. That changes the engineering: "ship, observe, fix" isn't an acceptable strategy for the hazardous parts.

**The first design question is always: can the hazard be removed entirely?** Everything else in these notes is for the hazards that remain.

---

## 2. Safety vs reliability vs security

| Quality | Question | Fails when |
|---|---|---|
| **Reliability** | Does it do what the specification says, over time? | The system deviates from its spec |
| **Safety** | Does it avoid causing harm? | Someone or something is hurt, even if the spec was followed |
| **Security** | Is it protected from people who want to misuse it? | An attacker gets it to do something it shouldn't |

They overlap but are different properties:

- **Reliable but unsafe**: the software does exactly what was specified, and the specification was wrong for the situation. Many software-related accidents involve software that worked "correctly" against flawed or incomplete requirements.
- **Safe but unreliable**: a train that stops whenever a sensor is uncertain is safe and annoying. A machine that never starts is perfectly safe.
- **Safety and availability can pull in opposite directions**: the safe response to doubt is often to stop ([Fail-safe](../fail-safe/README.md)), while availability wants to keep going.

Safety is a **system property**. A component isn't "safe" on its own; it's safe or unsafe in a particular system, environment and use. The same valve controller can be harmless on a water tank and dangerous on a reactor.

---

## 3. Vocabulary: harm, hazard, risk

| Term | Meaning | Infusion pump example |
|---|---|---|
| **Harm** | Injury or damage to health, property or environment | Patient receives an opioid overdose |
| **Hazard** | A system state that, together with certain conditions, leads to harm | Pump delivers more than the prescribed rate |
| **Hazardous situation** | People are exposed to the hazard | Pump is connected to a patient |
| **Cause** | What can produce the hazard | Unit mix-up (mg vs mL), stuck motor, software fault, wrong drug library entry |
| **Risk** | Combination of the **severity** of harm and the **probability** it occurs | Catastrophic × remote |
| **Risk control / mitigation** | A measure that reduces risk | Dose limits, independent flow sensor, alarm, stopping the pump |
| **Residual risk** | What remains after controls | Accepted only if judged tolerable |

A typical risk matrix:

| Probability ↓ / Severity → | Negligible | Minor | Serious | Catastrophic |
|---|---|---|---|---|
| **Frequent** | Medium | High | High | High |
| **Occasional** | Low | Medium | High | High |
| **Remote** | Low | Low | Medium | High |
| **Improbable** | Low | Low | Low | Medium |

Things to know about risk matrices:
- The **probability of a software fault can't be estimated reliably**. Many standards tell you to assume a software failure *will* happen and judge the risk mostly by severity (IEC 62304 takes this approach).
- **ALARP** ("as low as reasonably practicable", used in UK regulation): reduce risk until the cost of further reduction is grossly disproportionate to the benefit.
- A matrix is a way to **prioritise**, not a proof of safety.

---

## 4. The safety lifecycle

Most standards follow the same loop:

```
Define the system and its intended use
        ↓
Identify hazards  ──────────────→  hazard log (kept for the system's whole life)
        ↓
Estimate and evaluate risk
        ↓
Define safety requirements (what must always / never happen)
        ↓
Design risk controls  →  implement  →  verify each control works
        ↓
Evaluate residual risk  →  safety case  →  release
        ↓
Monitor in the field  →  incidents and near misses feed back into the hazard log
```

Key ideas:
- **Safety requirements come from hazards**, not from features. "The pump shall never deliver more than the hard limit for the selected drug" exists because of the overdose hazard.
- **Traceability**: hazard → safety requirement → design element → code → test → evidence, in both directions. If a line of the chain is missing, you can't show the hazard is controlled.
- **Changes reopen the analysis.** A "small" change, a new environment or reused software needs a change impact analysis ([§17](#17-real-world-incidents): Ariane 5).
- Safety work starts **at the concept stage**. Adding it at the end is expensive and usually leaves only weak controls (warnings, procedures).

---

## 5. Hazard analysis methods

| Method | Direction | Question it asks | Good for | Weak at |
|---|---|---|---|---|
| **PHA** (preliminary hazard analysis) | Top-down, early | What could hurt someone here? | First list of hazards at concept stage | Detail |
| **FMEA / FMECA** (failure mode and effects analysis) | Bottom-up | If this component fails this way, what happens? | Hardware, single failures, completeness per component | Combinations of failures; hazards with no failure at all |
| **FTA** (fault tree analysis) | Top-down | What combinations of events cause this top event? | Combinations, redundancy, single points of failure, probability estimates | Finding hazards you didn't think of |
| **HAZOP** (hazard and operability study) | Deviation-based | What if there's *more / less / no / reverse / late* flow, pressure, data? | Process plants; adapted for data flows and software | Slow; needs a good team |
| **STPA** (system-theoretic process analysis) | Top-down, control-based | Which **control actions** are unsafe, and why would they happen? | Software-heavy systems, human–automation interaction, requirements flaws | Doesn't produce probabilities |

Use more than one. FMEA and FTA assume accidents come from **component failures**. STPA also finds accidents where **nothing failed**: every component did what it was designed to do and the interaction was unsafe.

### 5.1 FMEA in one table

| Component | Failure mode | Effect | Sev | Occ | Det | Action |
|---|---|---|---|---|---|---|
| Flow sensor | Stuck at last value | Over-delivery not detected | 9 | 3 | 7 | Plausibility check against motor steps; alarm on no change |
| Motor driver | Stuck on | Free flow of drug | 10 | 2 | 4 | Independent cut-off relay driven by the safety monitor |
| Keypad | Key bounce (double digit) | 10× dose entered | 9 | 4 | 5 | Debounce; dose limits; confirmation screen with units |

Severity, occurrence and detection are scored 1–10. Many teams multiply them into a **risk priority number (RPN)**, but a high-severity item deserves action whatever the product says.

### 5.2 Fault tree analysis

```
                 Overdose delivered (top event)
                           │
                         [OR]
          ┌────────────────┴─────────────────┐
   Wrong rate commanded              Rate correct, delivery wrong
          │                                   │
        [OR]                                [AND]
   ┌──────┴───────┐                 ┌─────────┴──────────┐
 Entry error   Software          Motor runs fast    Monitor fails
 not caught    computes wrong                        to stop it
```

- **OR gates** are where single failures get through. **AND gates** are where your protection is.
- A **minimal cut set** is a smallest combination of events that causes the top event. A cut set of size 1 is a single point of failure.
- An AND gate only helps if its inputs are **independent**. A shared power supply, shared sensor or shared code turns it back into an OR.

### 5.3 STPA

1. **Define losses and hazards.** Loss: patient harmed. Hazard: pump delivers more drug than prescribed.
2. **Draw the control structure**: controllers (nurse, pump software), control actions (start, set rate, stop), feedback (flow sensor, display), controlled process (drug delivery).
3. **Find unsafe control actions (UCAs).** For each control action, ask four questions:

| Control action | Not provided | Provided when unsafe | Too early / late / wrong order | Stopped too soon / applied too long |
|---|---|---|---|---|
| **Stop infusion** | Not stopped when the dose is complete | Stopped during a life-sustaining infusion | Stopped late after an occlusion | n/a |
| **Start bolus** | Not given when needed | Given during lockout interval | Given before the new rate is confirmed | Continues past the bolus volume |

4. **Find loss scenarios**: why would each UCA happen? Wrong feedback, wrong process model ("the software believes the pump is stopped"), delayed data, confusing display, conflicting commands.

Each UCA becomes a **safety constraint** ("the pump must not deliver a bolus during the lockout interval"), and each scenario becomes a design requirement or a test.

---

## 6. Risk reduction: the hierarchy of controls

Controls aren't equally strong. Prefer the top of the list (this ordering appears in ISO 12100 for machinery and ISO 14971 for medical devices):

| Rank | Control | Example |
|---|---|---|
| 1 | **Eliminate the hazard / inherently safe design** | A pump mechanism that physically can't free-flow; lower voltage; no hazardous mode at all |
| 2 | **Protective measures** (guards, interlocks, automatic shutdown) | Door interlock cuts the beam; over-pressure valve; independent safety monitor |
| 3 | **Information for safety** (warnings, alarms, training, procedures) | Alarm, label, manual, checklist |

- A hazard removed needs no alarm, no training and no vigilance.
- Warnings and procedures depend on people noticing, understanding and acting under stress. They're the **weakest** control and shouldn't be the only one for a severe hazard ([Hazard warning](../hazard-warning/README.md)).
- **Defence in depth**: several independent layers, because each has holes (the Swiss cheese model, [User error protection §2.3](../../UI-UX/user-error-protection/README.md#23-the-swiss-cheese-model-and-just-culture)).

---

## 7. Integrity levels and standards

Standards classify functions by how bad a failure would be, then require **more rigour** for higher levels: more analysis, more independence, stricter coding and testing.

| Domain | Standard | Levels (least → most critical) | Level is decided by |
|---|---|---|---|
| General (E/E/PE systems) | **IEC 61508** | SIL 1 → SIL 4 | Required risk reduction |
| Process industry | IEC 61511 | SIL 1 → SIL 4 | Layers-of-protection / risk graph |
| Automotive | **ISO 26262** | QM, ASIL A → ASIL D | Severity × exposure × controllability |
| Aviation software | **DO-178C** | DAL E → DAL A | Failure condition severity (no effect → catastrophic) |
| Medical device software | **IEC 62304** (with ISO 14971) | Class A → Class C | Worst possible harm (none → death or serious injury) |
| Railway | EN 50126 / 50128 / 50129 | SIL 0 → SIL 4 | Tolerable hazard rate |
| Machinery | ISO 13849 | PL a → PL e | Severity, frequency, possibility of avoidance |

IEC 61508 target failure measures for a safety function:

| SIL | Low demand: average probability of failure on demand | High demand / continuous: dangerous failures per hour |
|---|---|---|
| 1 | ≥ 10⁻² to < 10⁻¹ | ≥ 10⁻⁶ to < 10⁻⁵ |
| 2 | ≥ 10⁻³ to < 10⁻² | ≥ 10⁻⁷ to < 10⁻⁶ |
| 3 | ≥ 10⁻⁴ to < 10⁻³ | ≥ 10⁻⁸ to < 10⁻⁷ |
| 4 | ≥ 10⁻⁵ to < 10⁻⁴ | ≥ 10⁻⁹ to < 10⁻⁸ |

What to remember:
- A level applies to a **safety function**, not to a product or a programming language. "SIL 3 PLC" means it's *suitable for use in* a SIL 3 function.
- The numbers are for **random hardware failures**. For software (systematic faults), the level sets the required **process rigour**. Nobody can measure that a program fails once in 10⁹ hours.
- DO-178C structural coverage shows how rigour scales: statement coverage at Level C, decision coverage at Level B, **MC/DC** (modified condition/decision coverage) at Level A.
- **Keep the high-integrity part small.** Isolate it so most of the system can be built at a lower level ([§8](#8-safety-architecture-patterns)).

---

## 8. Safety architecture patterns

| Pattern | Idea | Example |
|---|---|---|
| **Safe state** | A defined state where the hazard is absent; reach it on any detected fault | Beam off, pump stopped, signals at red ([Fail-safe](../fail-safe/README.md)) |
| **Interlock** | An unsafe action is physically or logically impossible while a condition holds | Microwave door switch; points can't move while a train is on them |
| **Doer / checker (safety monitor)** | A complex channel does the work; a small, simple, independent channel checks the result and can force the safe state | Flight control command/monitor lanes; pump controller + independent dose monitor |
| **Safety envelope / limiter** | Hard limits enforced outside the complex logic | Maximum dose rate, speed limiter, geofence |
| **Simplex architecture** | An advanced controller runs inside the envelope of a simple verified one, which takes over when limits are approached | Research and autonomous systems |
| **Redundancy with voting** | Several channels, majority decides | 2-out-of-3 sensors, triple modular redundancy |
| **Diverse redundancy** | Channels built differently (hardware, software, team, principle) to avoid common-cause failure | Dissimilar flight computers; pressure switch + pressure transmitter |
| **Watchdog** | Something independent notices the controller has stopped or hung | Hardware watchdog timer resets the MCU and de-energises outputs |
| **Partitioning** | Safety and non-safety software can't interfere (memory, time, I/O) | Hypervisors, ARINC 653 partitions, separate MCU for the safety function |
| **Plausibility checks** | Reject sensor values that can't be true | Range, rate-of-change, cross-check against another sensor, stale-data timeout |
| **Degraded modes** | Lose function in steps, keeping the safest ones | "Limp-home" mode in cars |

### Worked example: doer / checker

```
            ┌──────────────────────────┐
 dose order │  Control channel (doer)  │ motor commands
 ─────────→ │  UI, drug library, PID,  │ ─────────────────┐
            │  logging, networking     │                  ▼
            └──────────────────────────┘             ┌─────────┐
                                                     │  Motor  │──→ patient
            ┌──────────────────────────┐   enable    └─────────┘
 own flow   │  Safety monitor (checker)│ ─────────────────▲
 sensor ──→ │  ~300 lines: hard limits,│   (relay opens unless the monitor
            │  watchdog, stale data    │    keeps asserting "OK")
            └──────────────────────────┘
```

- The doer can be large and change often. The checker is **small enough to review completely** and changes rarely.
- The checker has its **own sensor, own processor and own way to stop the motor**. If it shares the doer's sensor reading or code library, it shares the doer's faults.
- The enable signal is **de-energise-to-trip**: if the checker crashes or loses power, the motor stops.
- Only the checker and its path need the highest integrity level.

---

## 9. Software practices for safety

- **Requirements first.** Studies of safety-critical software (for example Lutz's analysis of NASA spacecraft errors) found that safety-related faults came mostly from requirements and interface misunderstandings, not from coding slips. Specify what must happen in **every** state, including "sensor missing", "value out of range", "message late", "two commands at once".
- **Define behaviour for off-nominal cases**: start-up, shutdown, mode changes, power loss mid-action, restart with stale state.
- **Keep it simple** ([Simplicity](../../development-evolution/simplicity/README.md)). Complexity you can't analyse is a hazard in itself.
- **Restricted language subsets and rules**: MISRA C/C++, SPARK Ada, the JPL "Power of Ten" rules (bounded loops, no dynamic allocation after initialisation, no recursion, check every return value).
- **Determinism**: bounded execution time, fixed scheduling, no unbounded queues or garbage-collection pauses in the safety path.
- **Defensive handling of data**: units in types, range checks at every boundary, explicit handling of overflow and conversion, timestamps and freshness on every sensor value.
- **Concurrency with great care**: races in safety logic are hard to test for ([Concurrency](../../runtime-operational/concurrency/README.md)); prefer designs where they can't occur.
- **Verification beyond testing**: static analysis, model checking, formal proof for the most critical parts (seL4, the B method used for Paris Métro line 14), structural coverage, fault injection, hardware-in-the-loop simulation.
- **Independence**: someone other than the author reviews and verifies; at high levels, a separate organisation.
- **Configuration and data are code**: drug libraries, calibration tables and thresholds get the same review, versioning and testing ([Configurability](../../operations-devops/configurability/README.md)).
- **Tools count too**: a compiler or test tool that can introduce or hide errors needs qualification or cross-checking.
- **Reuse is not automatically safe**: re-validate reused software against the *new* system's hazards and operating range.

---

## 10. Human factors

People are part of the system: operators, maintainers, patients, bystanders.

- **"Operator error" is a starting point for investigation, not a conclusion.** Ask why the action made sense to the person at the time ([User error protection §2](../../UI-UX/user-error-protection/README.md#2-understanding-human-error)).
- **Mode confusion**: the automation is in a different mode from what the operator believes. Make modes few and unmistakable ([User error protection §4.5](../../UI-UX/user-error-protection/README.md#45-avoid-modes-or-make-them-unmistakable)).
- **Automation surprises**: the system acts without the operator knowing why, or with more authority than they expected.
- **Ironies of automation** (Bainbridge): the more reliable the automation, the less practised the human is at taking over, and they're asked to take over exactly when things are hardest.
- **Alarm fatigue and habituation**: too many warnings teach people to ignore them ([Hazard warning](../hazard-warning/README.md)).
- **Workarounds**: if the safe way is slow or irritating, people bypass it. A bypassed interlock is a design finding.
- **Foreseeable misuse** is in scope. If people will predictably do it, the design must handle it.
- **Usability testing is safety evidence** in medical devices (IEC 62366) and should be elsewhere.
- **Safety culture**: near misses reported without blame, the authority to stop work, and schedule pressure that doesn't override safety findings.

---

## 11. Safety cases

A **safety case** is a structured argument, supported by evidence, that a system is acceptably safe for a given use in a given environment.

```
Claim:     The pump is acceptably safe for use on general wards
  Strategy: Argue over each identified hazard
    Claim:    H1 (over-delivery) is acceptably controlled
      Evidence: hazard log entry, monitor design review, fault-injection test report, usability study
    Claim:    H2 (air embolism) is acceptably controlled
      Evidence: ...
  Context:  Intended use, users, environment, assumptions
```

- **Goal Structuring Notation (GSN)** is a common way to draw it: goals, strategies, evidence ("solutions"), context, assumptions.
- The value is in making **assumptions explicit** and showing where evidence is thin.
- It's a **living document**: updated with every change and every field incident.
- Beware the "paper safety" trap: a large document that argues the system is safe without anyone trying to show it isn't. Good safety cases actively look for counter-evidence.

---

## 12. Safety and security together

A system that can be attacked can be made unsafe. A connected pump, car or plant controller needs both.

- An attacker is **not a random failure**: they will find the one combination a probability argument dismissed.
- [Integrity](../../security/integrity/README.md) and [authentication](../../security/authentication/README.md) of commands, firmware and configuration are safety controls.
- Controls can conflict: security wants to **lock out** on doubt; safety may need **access in an emergency** ("break-glass" access, audited afterwards; see [Authorization](../../security/authorization/README.md) and [Auditability](../../security/auditability/README.md)).
- Security patches must be fast; safety re-validation is slow. Architect so that the safety function is isolated and rarely needs patching.
- Standards pairing the two: IEC 62443 (industrial), ISO/SAE 21434 (automotive), alongside the safety standards above.

---

## 13. Machine learning in safety-related systems

- An ML model has **no complete specification** to verify against, and its failures are hard to predict from its structure.
- Treat it as an untrusted "doer": put a **deterministic safety envelope or monitor** around it ([§8](#8-safety-architecture-patterns)).
- **ISO 21448 (SOTIF)** covers hazards from *insufficient performance* with no fault present (a camera blinded by sun, an object the model wasn't trained on). **UL 4600** describes safety cases for autonomous products.
- Define the **operational design domain** (where the system is allowed to operate) and detect when you've left it.
- Monitor in the field for distribution shift, and keep a path for a human or a simple fallback to take over **with enough time to do so**.

---

## 14. Approaches by domain

| Domain | Typical hazards | Dominant approach |
|---|---|---|
| **Medical devices** | Overdose, wrong therapy, missed alarm, delayed treatment | ISO 14971 risk management, IEC 62304 classes, usability engineering, regulatory approval |
| **Automotive** | Unintended acceleration, loss of braking/steering | ISO 26262 ASILs, redundant sensors, fail-operational for driver assistance, SOTIF |
| **Aviation** | Loss of control, misleading information to crew | DO-178C, dissimilar redundancy, certification authority oversight |
| **Railway** | Collision, derailment | Fail-safe signalling and interlocking, SIL 4, formal methods |
| **Process / chemical / nuclear** | Fire, explosion, toxic release | Independent safety instrumented systems (IEC 61511), layers of protection, alarm management |
| **Industrial robots / machinery** | Crushing, collision with people | Guards, light curtains, safety-rated stop, ISO 13849 |
| **Consumer IoT** (heaters, locks, e-scooters) | Fire, burns, trapping people, loss of control | Hardware cut-outs independent of firmware, safe behaviour when offline, safe OTA updates |
| **Web / cloud software with physical consequences** (dispatch, navigation, health apps, plant dashboards) | Wrong or stale information acted on; unavailable in an emergency | Show data age and uncertainty, degrade visibly, high availability for emergency paths, clear statements of intended use |

For ordinary web software, the transferable habits are: ask "what's the worst thing that can happen if this is wrong?", show **stale or missing data as stale or missing** instead of as a normal value, and keep hard limits outside the clever code.

---

## 15. Trade-offs

| Trade-off | Tension |
|---|---|
| **Safety vs availability** | Stopping on doubt is safe; spurious stops cost money and push people to bypass the protection |
| **Safety vs usability** | Confirmations, interlocks and alarms add friction; too much friction creates workarounds and alarm fatigue |
| **Safety vs cost and time to market** | Analysis, independence and certification are expensive; see [Time to market](../../business-project/time-to-market/README.md) |
| **Safety vs features / change** | Every change reopens the analysis, so safety-critical parts evolve slowly |
| **Redundancy vs complexity** | Extra channels add failure modes (voters, synchronisation, disagreement handling) |
| **Automation vs human skill** | More automation, less practised operators |
| **Safety vs security** | Emergency access vs lock-out; fast patching vs re-validation |

Reducing one risk can raise another: a pump that stops on every doubt delays therapy, which is also harm. Evaluate the **overall** residual risk.

---

## 16. Anti-patterns

- Treating safety as a document written after the design is done.
- "The software is tested, so it's safe": testing against a flawed requirement proves nothing about hazards.
- **Removing a hardware interlock because software now does that job** ([§17](#17-real-world-incidents): Therac-25).
- Reusing software in a new environment without re-validating its assumptions.
- A single sensor feeding a function with the authority to cause a catastrophe.
- Redundant channels that share a sensor, power supply, clock or code.
- A safety monitor that reads the same (possibly wrong) values as the thing it monitors.
- Using a warning or a line in the manual as the only control for a severe hazard.
- Showing the *commanded* state as if it were the *actual* state.
- Error messages operators can't interpret and learn to dismiss.
- Probability arguments for software ("this bug is 10⁻⁹") with no basis.
- Blaming the operator and closing the investigation.
- Letting schedule pressure decide the integrity level.

---

## 17. Real-world incidents

| Incident | What happened | Lessons |
|---|---|---|
| **Therac-25** (1985–1987) | A computer-controlled radiation therapy machine gave massive overdoses in six known accidents, several of them fatal. Software inherited from earlier models contained race conditions; the earlier models' **hardware interlocks had been removed** and safety relied on software alone. Operators saw cryptic messages such as "MALFUNCTION 54" and could resume with one key press. | Don't replace independent hardware protection with software. Reused code isn't proven code. Error messages must be meaningful. Investigate incident reports instead of assuming the machine can't be at fault. |
| **Ariane 5 flight 501** (1996) | About 37 seconds after lift-off, the inertial reference software (reused from Ariane 4) converted a 64-bit floating-point value to a 16-bit signed integer; Ariane 5's trajectory produced a larger value, the conversion overflowed, and the unhandled exception shut down both the backup and the active unit, which ran identical software. The rocket veered and self-destructed. The failing function wasn't needed after lift-off. | Re-validate reused software against the new operating range. Identical redundancy doesn't protect against design faults. Shutting down isn't a safe state for a system that must keep flying. Don't run code you don't need. |
| **Boeing 737 MAX / MCAS** (2018–2019) | Two crashes (Lion Air 610, Ethiopian Airlines 302) killed 346 people. MCAS, a function that automatically trimmed the nose down, acted on **one angle-of-attack sensor**; when that sensor gave a wrong reading, MCAS activated repeatedly. Pilots hadn't been told about MCAS in their manuals and training, and investigations criticised the assumptions made about how quickly crews would diagnose and respond. | No single sensor for a function with that much authority. Limit the authority of automation. Tell operators what the automation does. Validate assumptions about human response with real people. Re-do the hazard analysis when a function's authority grows during development. |
| **Uber ATG test vehicle, Tempe** (2018) | A test vehicle in autonomous mode struck and killed a pedestrian. The NTSB found, among other things, that the system didn't correctly classify a pedestrian crossing outside a crosswalk, that automatic emergency braking was not enabled in that mode, and that the safety driver was not monitoring the road. | A human monitor isn't a reliable safety mechanism for long, dull tasks. Don't disable an independent protection layer without replacing it. Safety culture matters. |

Read the primary reports: details in summaries (including this one) are simplified.

---

## 18. Checklist

**Analysis**
- [ ] Intended use, users and environment written down, including foreseeable misuse
- [ ] Hazard log exists and is maintained for the system's whole life
- [ ] More than one analysis method used (e.g. FMEA + STPA)
- [ ] Each hazard has a severity, controls and an owner
- [ ] Applicable standards and integrity levels identified

**Design**
- [ ] Hazards eliminated by design where possible, before adding protection or warnings
- [ ] Safe state defined for each function, and reached on any detected fault
- [ ] No single sensor, component or person as the only barrier for a severe hazard
- [ ] Safety function is small, isolated and independent of the function it protects
- [ ] Redundant channels checked for common causes (power, sensor, clock, code, data)
- [ ] Stale, missing and implausible data handled explicitly

**Software**
- [ ] Safety requirements trace to design, code, tests and evidence
- [ ] Behaviour specified for every mode, including start-up, shutdown and restart
- [ ] Coding standard, static analysis and independent review in place
- [ ] Fault injection shows each control actually works
- [ ] Configuration and data tables reviewed and tested like code

**People and operations**
- [ ] Operators know what the automation does and which mode it's in
- [ ] Alarms are few, prioritised and actionable
- [ ] Change impact analysis for every change and every reuse
- [ ] Near misses and field incidents feed back into the hazard log
- [ ] Safety case current, with assumptions stated

---

## 19. Exercises

1. Pick a device you use (electric kettle, lift, e-scooter, garage door). List three hazards, their causes, and the controls you can observe. Which level of the hierarchy of controls is each?
2. Write an FMEA table with at least six rows for a software-controlled space heater.
3. Draw a fault tree for "lift door opens with no car present". Find the minimal cut sets. Is there any cut set of size 1?
4. Do the four-column STPA table for the control action "apply brakes" in an automatic emergency braking system.
5. A safety monitor and its controller both read the same temperature sensor over the same bus. List the faults this monitor can't catch, then redesign it.
6. Read the Ariane 501 inquiry board report. Which of its recommendations apply to a web service you've worked on?
7. Your team wants to replace a physical emergency-stop button with a button on a touchscreen. Write the argument for and against.
8. Sketch a safety case (claims, evidence, assumptions) for one hazard of a smart door lock: "occupants can't get out during a fire".

---

## 20. Further reading

Nancy Leveson, *Engineering a Safer World* (free from MIT Press; STAMP and STPA) and *Safeware*; Leveson and Thomas, *STPA Handbook* (free); Leveson and Turner, "An Investigation of the Therac-25 Accidents" (IEEE Computer, 1993); the Ariane 501 Inquiry Board report (Lions report, 1996); the Indonesian KNKT and Ethiopian final accident reports and the US House committee report on the 737 MAX; NTSB report on the 2018 Tempe collision; James Reason, *Human Error* and *Managing the Risks of Organizational Accidents*; Lisanne Bainbridge, "Ironies of Automation" (1983); Sidney Dekker, *The Field Guide to Understanding "Human Error"*; Robyn Lutz, "Analyzing Software Requirements Errors in Safety-Critical, Embedded Systems" (1993); Gerard Holzmann, "The Power of Ten: Rules for Developing Safety-Critical Code"; Phil Koopman, *Better Embedded System Software* and his safety lectures; the standards themselves: IEC 61508, ISO 26262, DO-178C, IEC 62304, ISO 14971, ISO 12100; the GSN Community Standard.
