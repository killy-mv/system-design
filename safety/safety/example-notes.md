# Safety: hands-on practice notes

Planned labs for learning safety by **analysing a small simulated device, building its protection, then trying to defeat it**. Theory is in [README.md](README.md).

They need only a laptop: a language you're comfortable with (**Python** is fine), a test framework with property-based testing (**pytest + Hypothesis**), a spreadsheet for the worksheets, and any diagram tool. No real hardware, and nothing here is validated for real use.

**Suggested order:** 1 → 2 → 3 → 5 → 4 → 6 → 7 → 8. Do the fail-safe and hazard-warning labs alongside labs 5–6.

**Related labs elsewhere:** safe states and watchdogs ([fail-safe notes](../fail-safe/example-notes.md)), alarm design ([hazard-warning notes](../hazard-warning/example-notes.md)), in-memory race (concurrency lab 1), common-mode fault (fault tolerance lab 3), property-based tests (testability lab 6).

---

## The shared practice system: `sim-pump`

A simulated infusion pump, small enough to analyse completely. All labs in this folder (and several in fail-safe and hazard-warning) use it.

```
   nurse (CLI / test script)
          │  set drug, rate, start, stop, bolus
          ▼
   ┌──────────────┐  motor steps   ┌──────────────┐  flow   ┌─────────┐
   │  controller  │ ─────────────→ │  motor model │ ──────→ │ patient │
   │  (the doer)  │ ←───────────── │  + line model│         │  model  │
   └──────────────┘  flow sensor   └──────────────┘         └─────────┘
```

- **Simulated time**: a `tick()` function advances the world by 100 ms, so tests run fast and deterministically.
- **Patient model**: just accumulates delivered volume; "harm" = total dose over a limit within a time window.
- **Fault hooks**: sensor stuck / noisy / disconnected, motor stuck on, motor stalled, controller hang, corrupted rate value, clock jump.
- Start it deliberately **naive**: no limits, no monitor, no checks. The labs add them.

---

## Labs

### 1. Hazard log and risk matrix ⭐
- **Do:** write the intended use (who, where, which drugs) in five lines. Then list at least 8 hazards for `sim-pump` (over-delivery, under-delivery, air in line, delivery to the wrong patient, delayed therapy, …) with causes, severity and existing controls (none yet).
- **Check:** every hazard is a *system state*, not a component failure ("pump delivers too much", not "sensor broken").
- **Keep** this hazard log open for all later labs and update it.
- **See:** README §3, §4.

### 2. FMEA worksheet
- **Do:** one row per component failure mode (sensor, motor, keypad, display, clock, power, drug library entry): effect, severity, occurrence, detection, action. Aim for 15+ rows.
- **Check:** which hazards from lab 1 did the FMEA *not* reach? (Usually the ones where nothing is broken, such as a correct pump given a wrong order.)
- **See:** README §5.1.

### 3. Fault tree for "overdose delivered"
- **Do:** draw the tree down to basic events. Find the minimal cut sets. Mark every cut set of size 1.
- **Then:** assign rough probabilities to basic events and compute the top event, once assuming independence and once with a shared power supply or shared sensor. See how much one common cause changes the answer.
- **See:** README §5.2.

### 4. STPA on the same pump ⭐
- **Do:** draw the control structure (nurse, controller, motor, sensor, patient, drug library). Fill the four-column unsafe-control-action table for *start*, *stop*, *set rate* and *bolus*. Write two loss scenarios per UCA.
- **Compare:** list what STPA found that labs 2 and 3 didn't. Turn each UCA into a safety constraint and add it to the hazard log.
- **See:** README §5.3.

### 5. Build the doer / checker ⭐
- **Build:** an independent safety monitor: separate module with no imports from the controller, its own (second) flow sensor, hard dose limits per drug, a stale-data timeout, and an `enable` output the motor requires on every tick.
- **Break:** use the fault hooks one at a time (motor stuck on, sensor stuck, controller hang, corrupted rate). Record for each: was harm prevented, and how long did it take?
- **Then:** make the monitor read the *controller's* sensor value instead of its own, and rerun. Note which faults now get through.
- **Check:** the monitor is under ~300 lines and you can explain every line.
- **See:** README §8 worked example.

### 6. A Therac-style race ⭐
- **Build:** a mode setting with a slow physical change: switching between "low rate" and "high rate" tubing takes 8 simulated seconds, and the UI lets the operator edit the order while it's still moving.
- **Break:** find the edit sequence (change mode, then quickly change back and start) that leaves the software's belief and the physical state different. Add a cryptic "MALFUNCTION 12" that the operator can clear with one key, and see how easy it is to continue.
- **Fix:** an interlock that checks the *measured* configuration immediately before start; a state machine with no editable window during transitions; a meaningful message.
- **See:** README §17 (Therac-25), §10.

### 7. Reuse outside the validated range
- **Build:** a rate conversion that stores mL/h in a 16-bit integer, "validated" for the original pump (max 999 mL/h). Now reuse it in a "high-flow" model (max 60,000 mL/h) without changes.
- **Break:** show the wraparound or exception. If you handled the exception by shutting the controller down, check what the motor does next.
- **Fix:** explicit ranges and units in types; a property-based test over the *new* operating range; a documented list of assumptions that reuse must re-check.
- **See:** README §9, §17 (Ariane 5).

### 8. Write the safety case
- **Do:** for the two worst hazards, draw a GSN-style argument: claim → strategy → sub-claims → evidence (your worksheets, test results, review notes) → assumptions and context.
- **Check:** mark every claim with weak or missing evidence. Ask someone else to attack it for 20 minutes and record what they find.
- **See:** README §11.

---

## Bonus: read and compare

- Leveson and Turner's Therac-25 paper, then re-read your lab 6 notes.
- The Ariane 501 inquiry report (short and very readable), then lab 7.
- One chapter of the STPA Handbook, then redo part of lab 4.

---

## Progress

- [ ] 1. Hazard log and risk matrix
- [ ] 2. FMEA worksheet
- [ ] 3. Fault tree for "overdose delivered"
- [ ] 4. STPA on the same pump
- [ ] 5. Build the doer / checker
- [ ] 6. A Therac-style race
- [ ] 7. Reuse outside the validated range
- [ ] 8. Write the safety case
