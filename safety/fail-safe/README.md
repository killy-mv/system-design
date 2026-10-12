# Fail-safe

> **Fail-safe** = when the system fails, it **automatically ends up in a state that doesn't cause harm**, without needing power, software or a person to get it there.
> "The controller just crashed, lost power or lost its sensor. What are the outputs doing right now?"

Related: [Safety](../safety/README.md) (hazards, risk and where fail-safe fits; [§8](../safety/README.md#8-safety-architecture-patterns) for architecture patterns), [Hazard warning](../hazard-warning/README.md) (telling people the system has gone to its safe state), [Fault tolerance](../../runtime-operational/fault-tolerance/README.md) (keeping working *through* faults), [Recoverability](../../runtime-operational/recoverability/README.md) (getting back afterwards).

These are learning notes, not a substitute for the applicable safety standards in regulated work.

---

## 1. Fail-safe vs related terms

| Term | On failure the system… | Example |
|---|---|---|
| **Fail-safe** | Goes to a state where the hazard is absent | Railway signal falls to red; heater switches off |
| **Fail-operational** | Keeps performing its function (needs redundancy) | Fly-by-wire flight controls; steer-by-wire |
| **Fail-degraded / limp-home** | Keeps a reduced, safer function | Car limits speed and gear after a transmission fault |
| **Fail-silent** | Stops producing output instead of producing wrong output | A sensor node that goes quiet when its self-test fails |
| **Fail-fast** | Stops immediately and visibly at the first sign of a bug | Process crashes on a failed assertion |
| **Fail-secure** | Stays locked / denies access | Vault door stays locked on power loss |
| **Fault-tolerant** | Masks the fault; users don't notice | RAID, 2-out-of-3 voting |

**Fail-safe and fail-secure can be opposites.** On power loss, a fire exit must unlock (safe for people) and a vault must stay locked (secure for assets). Decide per door, and write down why.

**Fail-safe needs a safe state to exist.** A train can stop. An aircraft in flight can't, and neither can a ventilator or a car at motorway speed: those need fail-operational design, at least long enough to reach a safe condition.

---

## 2. Classic examples

| System | Fail-safe mechanism |
|---|---|
| **Railway air brakes** | Air pressure holds the brakes *off*. A broken hose or a separated train loses pressure and the brakes apply. |
| **Railway signals** | A broken wire, failed relay or lost power gives a red (or dark, treated as red) signal. |
| **Lift safety gear** | If the car overspeeds or the rope fails, spring-loaded brakes grip the guide rails. |
| **Dead man's switch / vigilance device** | The driver must keep acting; if they stop, power is cut and brakes apply. |
| **Reactor control rods** | Held up by electromagnets; loss of power drops them into the core by gravity. |
| **Spring-return valves** | A spring drives the valve to its safe position (open or closed, chosen per valve) when air or power is lost. |
| **Traffic light conflict monitor** | An independent unit detects conflicting greens and forces the junction to flashing red. |
| **Fuses and circuit breakers** | The fault current itself opens the circuit. |

The common idea: **the safe state is the one that needs no energy and no decision.** Energy and a healthy controller are needed to *leave* it.

---

## 3. Design principles

| Principle | Meaning |
|---|---|
| **Define the safe state first** | Per function and per operating mode. "Off" isn't always safe (cooling pump, ventilator, landing gear). |
| **De-energise to trip** | The safe state is the unpowered state. A cut wire, blown fuse or dead controller all lead to it. |
| **Positive "I'm healthy" signals** | Require a continuously refreshed signal to *stay* in the active state. Silence means stop. Never rely on a "fault" message arriving. |
| **Watchdog** | An independent timer that forces the safe state if the controller stops kicking it. Kick it only after real health checks, not from a timer interrupt that survives a hung main loop. |
| **Treat missing and stale data as unsafe** | A sensor value needs a timestamp and a validity flag. No fresh value → assume the worst. |
| **Plausibility checks** | Out-of-range, impossible rate of change, or disagreement between sensors → safe state. |
| **Independence** | The mechanism that forces the safe state doesn't depend on the thing that might have failed (separate hardware, power, code path). |
| **Latching** | Once tripped, stay in the safe state until a deliberate, checked reset. No automatic restart into the same fault. |
| **Safe by default** | Initial values, start-up state and reset state are the safe ones. |
| **Announce it** | Going to the safe state must be visible, with the reason ([Hazard warning](../hazard-warning/README.md)). |

### Worked example: a heater controller

A software thermostat switches a heater relay. Hazard: overheating and fire.

**Not fail-safe:**

```
relay is "normally closed" (heater ON when the relay coil is unpowered)
controller sends "OFF" when temp > setpoint
```

Controller crash, loose wire to the relay coil, or a sensor stuck at 18 °C → heater on forever.

**Fail-safe:**

```
relay is "normally open"        → no coil power = heater OFF
controller must pulse an enable line every 500 ms; a hardware watchdog
  drops the relay if pulses stop → hang or crash = heater OFF
temperature older than 2 s, < -20 °C or > 120 °C, or changing
  impossibly fast               → heater OFF, fault latched
independent thermal fuse on the heater at 150 °C
                                → OFF even if everything above is wrong
```

| Fault | Result |
|---|---|
| Controller hangs | Watchdog opens relay |
| Power to controller lost | Relay opens |
| Wire to relay cut | Relay opens |
| Sensor disconnected / stuck | Stale or implausible reading → off, latched |
| Relay contacts weld shut | Thermal fuse opens the circuit |

Note the last row: the relay itself can fail in the dangerous direction, so one more **independent, simpler** layer sits behind it.

---

## 4. Fail-safe in ordinary software

The same question applies with no hardware in sight: **when a dependency is down or the data is doubtful, which way do we fail?**

| Situation | Fail closed (deny / stop) | Fail open (allow / continue) |
|---|---|---|
| Authorisation service unreachable | Usual choice: deny ([Authorization](../../security/authorization/README.md)) | Only for explicitly low-risk actions |
| Fraud check times out on a payment | Hold or decline | Accept small amounts, review later |
| Rate limiter's store is down | Protects the backend, blocks users | Keeps users working, risks overload |
| Feature-flag service unreachable | Each flag has a coded **safe default** | |
| Config push fails validation | Keep **last known good** config ([Configurability](../../operations-devops/configurability/README.md)) | |
| Dosage / price / limit lookup fails | Refuse to proceed; never substitute 0 or a guess | |
| Deploy health check fails | Stop the rollout and roll back ([Deployability](../../operations-devops/deployability/README.md)) | |

Other software fail-safes: transactions that roll back when interrupted, databases that go read-only instead of risking corruption, `terraform plan` before `apply`, default-deny firewalls, circuit breakers ([Availability §8](../../runtime-operational/availability/README.md#8-resilience-patterns-containing-failures)).

Rules of thumb:
- **Decide per case and write it down.** The dangerous outcome is failing in whichever direction the exception handler happened to choose.
- **A fallback value is a decision.** `catch { return true }` and `limit = config.get("limit") or 0` are fail-open and fail-unknown in disguise.
- **Test the failure direction**, not only the happy path.

---

## 5. Trade-offs

- **Safety vs availability**: every spurious trip is downtime. Too many, and people bypass the protection. Voting such as 2-out-of-3 sensors reduces spurious trips without giving up safety.
- **The safe state can have its own hazards**: a stopped lift traps people; a stopped chemical process may need cooling to continue; an emergency stop on a motorway is dangerous. Analyse the safe state too.
- **Fail-safe vs fail-secure**: see §1. Resolve conflicts explicitly.
- **Restart**: latching is safe and slow; automatic restart is convenient and can cycle through the fault repeatedly.

---

## 6. Anti-patterns

- No defined safe state; "it won't fail" as the plan.
- Energise-to-trip: the safety action needs the power, wire or software that just failed.
- Relying on a "fault" message to arrive from a component that may be dead.
- Using the last good sensor value indefinitely.
- A watchdog kicked from an interrupt, so it's happy while the main loop is hung.
- The same processor and code computing the command and checking it.
- Catch-all exception handlers that default to "allow" or "continue".
- Automatic reset that puts the system straight back into the hazardous state.
- A safe state nobody has ever actually triggered in a test.
- Going to the safe state silently, so operators don't know protection has acted.

---

## 7. Checklist

- [ ] Safe state defined per function and per mode, and its own hazards analysed
- [ ] Loss of power, signal or controller leads to the safe state without any action
- [ ] Active state requires a continuously refreshed health signal
- [ ] Watchdog is independent and is kicked only after real health checks
- [ ] Stale, missing and implausible inputs lead to the safe state
- [ ] Trips latch; reset is deliberate and checked
- [ ] Fail-open / fail-closed decided and documented for every external dependency
- [ ] Every fail-safe path exercised by fault injection, regularly
- [ ] Entering the safe state is announced with its cause

---

## 8. Exercises

1. For a building's doors (main entrance, fire exits, server room, medicine cabinet), decide fail-safe or fail-secure on power loss. Justify each.
2. Find three `catch` blocks or default values in code you've written. For each: which way does it fail? Is that the direction you'd choose on purpose?
3. A drone loses its control link. List the candidate "safe states" (hover, land, return home, cut motors). Which is safe where? What would you need to know to choose?
4. Your API gateway can't reach the auth service. Design the behaviour per endpoint class (public read, user data, payments, health checks), including how long any cached decision may be used.

---

## 9. Further reading

Nancy Leveson, *Safeware* and *Engineering a Safer World* (design for safety, safe states); Phil Koopman, *Better Embedded System Software* (watchdog timers, safe shutdown) and his "Embedded System Safety" lectures; IEC 61508 and IEC 61511 (safety functions, de-energise-to-trip practice in the process industry); railway signalling texts on the fail-safe principle and vital relays; Jim Gray, "Why Do Computers Stop and What Can Be Done About It?" (fail-fast); Google SRE book, ch. 22 "Addressing Cascading Failures" (fail-open vs fail-closed choices in services); Michael Nygard, *Release It!* (stability patterns).
