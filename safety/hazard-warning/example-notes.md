# Hazard warning: hands-on practice notes

Planned labs for learning hazard warning by **designing alarms for a simulated plant, flooding yourself with them, then cutting them down until each one means something**. Theory is in [README.md](README.md).

They need only a laptop: any language (**Python** is fine), a terminal or a simple web page for the operator display, and a spreadsheet. Optional: **Prometheus + Alertmanager** if you want to repeat lab 3 with real alerting tools. Everything is simulated.

**Suggested order:** 1 → 2 → 3 → 4 → 5.

**Related labs elsewhere:** alert routing and noise, silent failures, monitor the monitoring (monitorability labs 2, 4, 5), the `sim-pump` system ([safety notes](../safety/example-notes.md)), going to the safe state ([fail-safe notes](../fail-safe/example-notes.md)).

---

## Labs

### 1. Simulated plant with naive alarms
- **Build:** a small simulation: a tank with inlet valve, outlet pump, level, temperature and pressure sensors (or reuse `sim-pump`). Add an operator console that lists alarms and accepts commands (open/close valve, start/stop pump, acknowledge).
- **Then:** add alarms the lazy way: a high and a low alarm on every measurement, all the same priority and sound, no deadband, messages like `ALM_0042 LVL_HI`.
- **Measure:** run 30 minutes of normal simulated operation (fill, drain, start-up). Count alarms per 10 minutes, standing alarms, and how many needed any action.
- **See:** README §2, §3.

### 2. Alarm flood experiment ⭐
- **Break:** inject one root fault (outlet pump trips) and let the consequences cascade: low flow, high level, high pressure, high temperature, each chattering around its limit.
- **Measure:** play operator (or ask a friend): time from the fault to the correct action, with the naive alarm list. Record how many alarms appeared before the right one was found.
- **Keep** the recording; you'll rerun the same scenario after lab 3.
- **See:** README §4.

### 3. Rationalise and fix ⭐
- **Do:** a worksheet with one row per alarm: cause, consequence of no response, operator action, time available, priority. Delete every alarm with no action.
- **Fix:** add deadbands and on/off delays; suppress alarms that are expected in the current state (low flow while the pump is intentionally off); group consequences under a first-out alarm; set limits from "time needed to respond" as in the README §4 worked example; aim for the 80/15/5 priority split.
- **Measure:** rerun lab 1 (normal operation) and lab 2 (the fault). Compare alarms per 10 minutes and time to correct action.
- **See:** README §4.

### 4. Message and presentation design
- **Do:** rewrite every remaining alarm with the four parts (priority word, what, consequence, action). Give each priority its own colour, symbol and sound, and make sure nothing depends on colour alone.
- **Test:** show five alarms to someone who hasn't seen the system and ask "what's wrong, and what would you do?" Fix what they misread.
- **Then:** add acknowledge vs clear (silencing leaves the visual indication), shelving with an automatic expiry, and a log of who shelved what.
- **See:** README §2, §3, §5.

### 5. The alarm that never came ⭐
- **Break:** three silent failures: the level sensor freezes at a normal value; the link between plant and console drops; the alarm sound is muted.
- **Fix:** show data age on the display and alarm on stale data; a heartbeat between plant and console with a "communication lost" alarm raised *by the console*; a self-test of the annunciator at start-up; display the valve's measured position, not the last command.
- **Check:** for each alarm path, answer "how would I know if this path were broken?"
- **See:** README §5, [Monitorability §5](../../operations-devops/monitorability/README.md#5-monitoring-the-monitoring).

---

## Progress

- [ ] 1. Simulated plant with naive alarms
- [ ] 2. Alarm flood experiment
- [ ] 3. Rationalise and fix
- [ ] 4. Message and presentation design
- [ ] 5. The alarm that never came
