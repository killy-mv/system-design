# Hazard warning

> **Hazard warning** = the system **tells the right person about a dangerous condition in time for them to act**, in a way they notice, understand and believe.
> "Something is going wrong. Does the person who can fix it know, do they know what to do, and do they have enough time?"

Related: [Safety](../safety/README.md) (warnings are the weakest risk control: [§6](../safety/README.md#6-risk-reduction-the-hierarchy-of-controls)), [Fail-safe](../fail-safe/README.md) (what the system does by itself), [Monitorability](../../operations-devops/monitorability/README.md) (the same problem for software on-call: [§3 alerting principles](../../operations-devops/monitorability/README.md#3-alerting-principles)), [User error protection §6](../../UI-UX/user-error-protection/README.md#6-layer-3-confirm-only-when-it-helps) (confirmation dialogs and habituation), [Accessibility](../../UI-UX/accessibility/README.md).

These are learning notes, not a substitute for the applicable standards in regulated work.

---

## 1. Where warnings fit

A warning **doesn't remove a hazard**. It hands the problem to a person. Use it for what's left after you've eliminated hazards and added automatic protection.

| Kind | When | Example |
|---|---|---|
| **Static warning** (label, sign, manual) | The hazard is always present | "Hot surface" label; high-voltage sign |
| **Dynamic warning / alarm** | An abnormal condition has started and needs a response | High-pressure alarm; "low battery, infusion will stop in 30 min" |
| **Advisory / status** | The user should know, no urgent action | "Filter due for replacement" |
| **Interlock message** | The system has already acted | "Beam disabled: door open" |

A warning only works if every step of this chain succeeds:

```
Condition detected → warning presented → noticed → understood → believed → action known → action possible in time
```

Design and test **each step**. Most failures are at "noticed" (too many alarms) and "believed" (too many false ones).

---

## 2. What makes a good alarm

ISA-18.2 defines an alarm as an audible and/or visible means of indicating an equipment malfunction, process deviation or abnormal condition **requiring a timely response**. If no response is required, it isn't an alarm.

| Property | Meaning |
|---|---|
| **Relevant** | Indicates a real abnormal condition, not normal operation or a planned state |
| **Actionable** | There is a defined operator action. No action → make it a status or a log entry |
| **Timely** | Early enough to act, not so early it's usually a false alarm |
| **Unique** | One alarm per problem, not five symptoms of the same cause |
| **Prioritised** | By severity of consequence **and** time available to respond |
| **Understandable** | Says what's wrong, where, and what to do, in the operator's words |
| **Diagnostic** | Helps find the cause, or points to the procedure |

A good warning message has four parts:

| Part | Example |
|---|---|
| Signal word / priority | **WARNING** |
| What the hazard is | Occlusion: no flow to patient |
| Consequence if ignored | Therapy interrupted |
| What to do | Check the line for kinks and clamps, then press Resume |

Compare with "ERROR 54".

---

## 3. Standards and conventions

**ANSI Z535 signal words** (product safety labels and signs; ISO 3864 is the international counterpart):

| Signal word | Meaning | Usual colours |
|---|---|---|
| **DANGER** | Hazard that, if not avoided, **will** result in death or serious injury | White on red |
| **WARNING** | Hazard that, if not avoided, **could** result in death or serious injury | Black on orange |
| **CAUTION** | Hazard that, if not avoided, could result in minor or moderate injury | Black on yellow |
| **NOTICE** | Information not related to personal injury (e.g. property damage) | White on blue |

**IEC 60601-1-8 (medical device alarms)** defines three priorities from the severity of harm and how soon it would occur:

| Priority | Operator response | Visual indicator |
|---|---|---|
| **High** | Immediate | Red, flashing |
| **Medium** | Prompt | Yellow, flashing |
| **Low** | Awareness | Cyan or yellow, steady |

It also specifies distinct auditory signals per priority, and behaviour for pausing and resetting alarms.

**ISA-18.2 / IEC 62682 and EEMUA 191 (process industry alarm management)** treat alarms as a system with a lifecycle: philosophy → identification → **rationalisation** (justify, prioritise and document every alarm) → design → operation → monitoring → audit. Commonly cited performance targets:

| Measure | Target |
|---|---|
| Average alarm rate per operator | About 1 per 10 minutes is comfortably manageable; around 2 per 10 minutes is the upper limit |
| Alarm flood | More than 10 alarms in 10 minutes; should be rare |
| Priority distribution | Roughly 80% low, 15% medium, 5% high |
| Standing (stale) alarms | Very few; an alarm that's always on carries no information |
| Chattering / fleeting alarms | None |

Use the standard that applies to your domain; these numbers are orientation, not requirements for every system.

---

## 4. Alarm fatigue and how to fight it

**Alarm fatigue**: people exposed to many alarms, most of them false or unimportant, become slow to respond, stop hearing them, silence them or turn them off. It's well documented in hospitals and control rooms, and it's the same mechanism as pager fatigue for on-call engineers and habituation to confirmation dialogs ([User error protection §6.1](../../UI-UX/user-error-protection/README.md#61-the-habituation-problem)).

Every nuisance alarm **spends trust** that the real alarm will need.

| Technique | What it does |
|---|---|
| **Rationalisation** | Each alarm must have a cause, consequence, operator action and time to respond. Delete the ones that don't. |
| **Deadband (hysteresis)** | Alarm sets at 80, clears at 77: no chattering around the limit |
| **On/off delay** | Condition must persist N seconds before alarming or clearing |
| **Suppression by state** | No "low flow" alarm while the pump is intentionally stopped |
| **First-out / root-cause grouping** | Show the first alarm of a cascade; fold consequences under it |
| **Dynamic limits** | Limits that follow the operating mode (start-up vs steady state) or the patient |
| **Escalation** | Unacknowledged high-priority alarms get louder or go to someone else |
| **Shelving with expiry** | An operator can temporarily silence a known nuisance; it comes back automatically and the shelving is logged |
| **Bad-actor review** | Regularly list the ten most frequent alarms and fix or remove them |

### Worked example: rationalising one alarm

A tank has a "level high" alarm at 80% that fires 40 times a shift during normal filling. Operators acknowledge it without looking.

| Question | Answer |
|---|---|
| What's the consequence of no response? | Overflow at 100% → spill of flammable liquid |
| What action does the operator take? | Close the inlet valve |
| How long does that take? | ~2 minutes to notice, decide and act |
| How fast can the level rise? | 3% per minute at maximum fill rate |
| So where must the alarm be? | At least 2 min × 3%/min = 6% below the trip point, plus margin |

Result:
- Normal filling now stops automatically at 85% (control, not an alarm).
- "Level high" moves to **88%**, with a 2% deadband and a 10-second delay, priority **high**. It should now fire only if the automatic stop failed.
- An **independent** high-high switch at 95% closes the inlet valve by itself ([fail-safe](../fail-safe/README.md)) and raises a separate alarm.
- The alarm text says: "T-101 level high. Automatic fill stop has failed. Close inlet valve XV-101."

The alarm went from 40 per shift to roughly never, and when it fires it means something.

---

## 5. Presenting warnings

- **More than one channel** for anything important: sound **and** light **and** text. People are deaf, colour-blind, looking elsewhere or wearing ear protection ([Accessibility](../../UI-UX/accessibility/README.md)). Never rely on colour alone.
- **Place the warning where the attention is**, and where the action is taken.
- **Urgency should match the encoding**: faster, louder and more intrusive only for higher priority. If everything is red, nothing is.
- **Don't startle**: a sudden very loud alarm degrades performance at the moment you need it.
- **Show actual state, not commanded state.** "Valve closed" must come from a position sensor, not from the fact that a close command was sent ([§7](#7-real-world-incidents)).
- **Show the age and validity of data.** A frozen display that looks normal is a hidden hazard.
- **Alarm the failure of the warning system itself**: sensor fault, lost communication, dead speaker, flat backup battery ([Monitorability §5](../../operations-devops/monitorability/README.md#5-monitoring-the-monitoring)).
- **Silencing is not resolving.** Acknowledging stops the noise; the visual indication stays until the condition clears.
- **Localise** safety text properly and use standard symbols ([Internationalization](../../UI-UX/internationalization/README.md)).
- **Test with real users** under realistic workload and noise.

The same rules cover software products: destructive-action warnings, security warnings (browser certificate pages), "data may be out of date" banners and on-call pages.

---

## 6. Anti-patterns

- Using a warning instead of fixing the hazard.
- Alarms with no defined action ("for information").
- Every alarm the same priority, colour and sound.
- One fault producing dozens of alarms (flood) with the cause buried.
- Chattering and standing alarms left for months.
- Cryptic codes that need a manual.
- Alarms that can be permanently disabled with no record.
- Alarm volume or priority that users can turn down to nothing on a life-critical device.
- Warnings so frequent that users click through them unread.
- An alarm system with no alarm for its own failure; "no alarms" treated as "all fine".
- Displaying the commanded state as if it were measured.

---

## 7. Real-world incidents

| Incident | Warning problem |
|---|---|
| **Three Mile Island** (1979) | A light showed that the relief valve had been *commanded* to close, not that it had closed; it was stuck open. Operators also faced a very large number of simultaneous alarms in the first minutes. |
| **Texaco refinery, Milford Haven** (1994) | The UK HSE investigation described operators facing an alarm flood (hundreds of alarms in the minutes before the explosion) and a display that didn't give an overview of the process. A key reference for EEMUA 191. |
| **BP Texas City** (2005) | The US Chemical Safety Board found, among many causes, that a level transmitter read incorrectly and a high-level alarm didn't operate, so operators didn't know the tower was overfilling. |
| **Therac-25** (1985–1987) | Frequent, cryptic "MALFUNCTION" messages that operators had learned to dismiss with one key ([Safety §17](../safety/README.md#17-real-world-incidents)). |

---

## 8. Checklist

- [ ] Each warning exists because a hazard remains after elimination and automatic protection
- [ ] Every alarm has a documented cause, consequence, operator action and response time
- [ ] Setpoints leave enough time for a human to respond
- [ ] Priorities assigned by consequence and urgency; few high-priority alarms
- [ ] Deadbands, delays and state-based suppression remove chattering and nuisance alarms
- [ ] Messages say what, where and what to do, in plain language
- [ ] Important alarms use at least two senses and don't rely on colour
- [ ] Displays show measured state and data age
- [ ] Failure of sensors, communication and the alarm system itself is alarmed
- [ ] Alarm rates, floods and most frequent alarms reviewed regularly
- [ ] Silencing, shelving and disabling are time-limited and logged

---

## 9. Exercises

1. List every alert, beep and warning dialog you got yesterday (phone, car, appliances, work tools). How many required action? How many did you ignore?
2. Rewrite five poor warning messages (from software or devices you use) with the four parts from §2.
3. A freezer storing vaccines has one alarm: "temperature above −15 °C". Design the full set: setpoints, delays, priorities, who gets notified and how, and what happens if the sensor or the network fails.
4. Your on-call team gets 60 pages a week and 5 need action. Apply the rationalisation questions and the techniques in §4. Which alarms go, and what replaces them? Compare with [Monitorability §3](../../operations-devops/monitorability/README.md#3-alerting-principles).

---

## 10. Further reading

ANSI/ISA-18.2 *Management of Alarm Systems for the Process Industries* and IEC 62682; EEMUA Publication 191 *Alarm Systems: A Guide to Design, Management and Procurement*; IEC 60601-1-8 (alarm systems in medical electrical equipment); ANSI Z535 series and ISO 3864 (safety signs, colours and signal words); Hollifield and Habibi, *Alarm Management: A Comprehensive Guide*; UK HSE, *The explosion and fires at the Texaco Refinery, Milford Haven, 24 July 1994*; the Kemeny Commission report on Three Mile Island; US Chemical Safety Board report on BP Texas City; The Joint Commission's Sentinel Event Alert on medical device alarm safety in hospitals; Wogalter, *Handbook of Warnings*; Google SRE book, ch. 6 "Monitoring Distributed Systems" (the same principles for paging).
