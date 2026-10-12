# Fail-safe: hands-on practice notes

Planned labs for learning fail-safe design by **killing parts of a simulated controller and checking where the outputs end up**. Theory is in [README.md](README.md).

They need only a laptop: any language (**Python** is fine), a test framework, and **Docker Compose** for lab 4. Everything is simulated; no real hardware.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** the `sim-pump` system and the doer/checker monitor ([safety notes](../safety/example-notes.md), lab 5), dynamic config with last known good (configurability lab 2), common-mode fault and a chaos experiment (fault tolerance labs 3–4), announcing the trip ([hazard-warning notes](../hazard-warning/example-notes.md)).

---

## Labs

### 1. Heater controller with a safe state ⭐
- **Build:** a simulated room (temperature rises when the heater is on, falls when off), a temperature sensor, a relay and a thermostat loop. Model the relay as *normally closed* first: heater on when the control signal is absent.
- **Break:** crash the controller process; cut the "wire" (drop the control signal). Watch the simulated temperature run away.
- **Fix:** make the relay *normally open*, write an explicit state machine (`INIT → IDLE → HEATING → FAULT`) where every unknown event leads to `FAULT`, and `FAULT` means heater off.
- **Check:** a table of faults vs resulting state, like the one in README §3.
- **See:** README §2, §3.

### 2. Watchdog and heartbeat ⭐
- **Build:** a separate watchdog process (or thread with its own timer) that opens the relay unless it receives a heartbeat every 500 ms.
- **Break:** hang the control loop with an infinite loop or a deadlock. First send heartbeats from a background timer thread: the watchdog stays happy while the controller is dead. Then send them from the main loop only after the sensor read and control step succeed.
- **Measure:** time from hang to heater off. How does the heartbeat period trade detection time against spurious trips?
- **See:** README §3 (watchdog, positive health signals).

### 3. Sensor faults: stale, stuck, implausible
- **Break:** inject four sensor faults: disconnected (no data), stuck at the last value, out of range (−200 °C), and a jump of 30 °C in one sample.
- **Fix:** timestamp every reading; reject readings older than 2 s; add range and rate-of-change checks; optionally add a second sensor and trip on disagreement. Latch the fault and require a manual reset.
- **Check:** the stuck sensor is the hard one. What independent evidence could detect it? (Heater on for 10 minutes with no temperature change; a second sensor; a thermal fuse in the simulation.)
- **See:** README §3.

### 4. Fail open or fail closed in a service
- **Build:** a small API with three dependencies in Docker Compose: an auth service, a rate-limiter store (Redis) and a feature-flag service.
- **Break:** stop each dependency in turn and record what the API does today. Look for accidental choices: a `catch` that returns `true`, a missing flag read as `false`, a limiter that blocks everyone.
- **Fix:** decide and implement on purpose: auth fails closed; the rate limiter fails open with a local fallback limit; flags have coded safe defaults; invalid config keeps the last known good version. Write the decisions in a table.
- **Check:** a test per dependency that asserts the failure direction.
- **See:** README §4.

### 5. Fault-injection campaign
- **Do:** for either the heater or `sim-pump`, list every fault you can inject (power, each wire, each sensor fault, controller hang, corrupted setpoint, clock jump, relay welded shut) and run them all automatically in a test suite.
- **Check:** every row ends in the safe state within a stated time, and the trip is latched and reported. Any row that doesn't is a finding: add an independent layer (for the welded relay, a simulated thermal fuse).
- **Then:** analyse the safe state itself. For the pump, "stopped" delays therapy: what does the user need to be told?
- **See:** README §5, §7.

---

## Progress

- [ ] 1. Heater controller with a safe state
- [ ] 2. Watchdog and heartbeat
- [ ] 3. Sensor faults: stale, stuck, implausible
- [ ] 4. Fail open or fail closed in a service
- [ ] 5. Fault-injection campaign
