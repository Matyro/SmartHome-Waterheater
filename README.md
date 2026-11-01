## Resistiv Water Heater automations

Each YAML file in this folder is intended to represent **one automation**. Since these are currently managed from **Settings → Automations & scenes**, keep importing/editing them individually in the UI. Do not concatenate these files into `automations.yaml`; Home Assistant UI-managed automations also need their own stored IDs, which the UI maintains.

The previous 13 automations were consolidated into 4 (originals are kept in `_old_v1_backup/`, which is not meant for import). Disable or remove **all 13 old automations** in Home Assistant before enabling the replacements, to avoid duplicate controllers writing to the same switches.

### Function map

| File | Responsibility |
| --- | --- |
| `Resistiv Water Heater - Heat Demand.yaml` | Set heat demand below 45°C, clear it above 55°C (when no disinfection is due), run the weekly disinfection (due after 7 days, completed above 62°C). |
| `Resistiv Water Heater - Heater Control.yaml` | Switch phases 1–3: Party Mode (below 55°C), or heat demand plus surplus >1200 W (phase 1) / >2300 W (phase 2) held for two minutes. Phase 3 is Party Mode only. Clears Party Mode at 55°C or higher. |
| `Resistiv Water Heater - Safety.yaml` | Shut down phases 1–3 and latch a fault above 65°C, above 10 kWh estimated energy, or on any NTC sensor fault; enforces shutdown while latched and on startup. |
| `Resistiv Water Heater - Energy Tracking.yaml` | Estimate energy from active phases every minute; reset daily at 00:01 when no fault is latched. |
### Helpers

Create these helpers in Home Assistant and retain the entity IDs exactly:

- `input_number.water_heater_cycle_energy_kwh` — estimated energy in kWh; reset daily only when no fault is latched.
- `input_boolean.water_heater_fault` — safety lockout; manually investigate before clearing.
- `input_datetime.water_heater_last_legionella_cycle` — date/time of the last completed disinfection cycle; an unset or invalid value is treated as due.
- `input_boolean.water_heater_needs_heat` — normal/weekly heating demand.
- `input_boolean.water_heater_party_mode` — manual Party/Force Heat toggle; independently requests all three phases until 55°C.

Create the Party Mode helper as a **Toggle** under **Settings → Devices & services → Helpers**, set its entity ID to `input_boolean.water_heater_party_mode`, and add it to a dashboard if you want a quick manual switch.

### Important behavior and assumptions

- The 65°C cutoff is configured as a **latched safety fault**, not just a normal stop. Clear `input_boolean.water_heater_fault` manually only after checking the heater and sensors.
- A fault sensor that is `on`, `unknown`, or `unavailable` also triggers the fail-safe shutdown and fault latch; both sensors must explicitly report `off` before heating is allowed.
- If the 10 kWh limit caused the fault, reset the energy helper below 10 kWh after investigation before clearing the fault; the protection re-latches if the helper remains above its limit.
- Normal surplus control operates phases 1 and 2 and requires a valid tank temperature below 65°C. Safety shutdowns include phase 3. Verify that this phase assignment matches the wiring.
- Party Mode bypasses solar-power checks and energizes phases 1–3 until the tank reaches 55°C, then clears its helper. It remains subject to sensor-fault, fault-lockout, and over-temperature safety protections; verify that all three phases can safely run together before enabling it. Normal/weekly heat demand can still continue independently after Party Mode resets, subject to its usual solar thresholds.
- Energy tracking is an estimate assuming 1 kW per active phase (including phase 3) and minute-level updates, not a meter reading. The 10 kWh protection is therefore approximate and resets daily when not faulted.
- Disinfection heat demand is now requested as soon as the last cycle is 7 days old (previously only at 05:00); the 05:00 and startup runs remain as a safety net.
- Disinfection is marked complete immediately once the sensed temperature is above 62°C; no minimum hold time is enforced.
- Disinfection due status is derived directly from `input_datetime.water_heater_last_legionella_cycle`; `input_boolean.water_heater_desinfection_needed` is no longer used and can be removed from Home Assistant. If the datetime is unset or invalid, a disinfection cycle is treated as due.
- Disinfection heating is subject to the same surplus-power thresholds as normal heating; without enough solar surplus, it waits for suitable power rather than forcing the heater on.
- Check the actual sensor and switch entity IDs in each automation against your Home Assistant installation.

### Known issue: trace viewer errors

In the automation trace timeline, conditions (for example in the `if` blocks of `Resistiv Water Heater - Heater Control.yaml`) may show `Error in describing condition: can't access property "includes" of undefined` (or `Cannot read properties of undefined (reading 'includes')` in Chrome). This is a Home Assistant **frontend display bug** and does not affect the automations: the conditions are evaluated correctly and the result is shown beneath the error. See [home-assistant/frontend#54871](https://github.com/home-assistant/frontend/issues/54871) (reported against core 2026.10.1); this project is not tracking or fixing it.

### AI assistance

This project was partially developed and optimized with the assistance of AI. Review and validate the automations against your Home Assistant setup before use.