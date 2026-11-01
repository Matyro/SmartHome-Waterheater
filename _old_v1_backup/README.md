## Hot water heater automations

Each YAML file in this folder is intended to represent **one automation**. Since these are currently managed from **Settings → Automations & scenes**, keep importing/editing them individually in the UI. Do not concatenate these files into `automations.yaml`; Home Assistant UI-managed automations also need their own stored IDs, which the UI maintains.

When replacing the former combined controller and hysteresis automations, disable or remove those old entries in Home Assistant before enabling the replacement automations, to avoid duplicate controllers writing to the same switches.

### Function map

| File | Responsibility |
| --- | --- |
| `Water Heater - Request Heat Below 45C.yaml` | Request heat below 45°C, including after Home Assistant starts. |
| `Water Heater - Stop Normal Heating Above 55C.yaml` | End heating above 55°C only when the last disinfection completion is less than seven days ago. |
| `Water Heater - Check Weekly Disinfection Cycle.yaml` | Request heating when the last disinfection completion is at least seven days old. |
| `Water Heater - Complete Disinfection Cycle.yaml` | At above 62°C during disinfection, clear demand flags and record completion. |
| `Water Heater - Control Phase 1.yaml` | Control phase 1 from heat demand and >1200 W surplus held for two minutes. |
| `Water Heater - Control Phase 2.yaml` | Control phase 2 from heat demand and >2300 W surplus held for two minutes. |
| `Water Heater - Party Mode Control.yaml` | Runs phase 3 for Party Mode; the phase 1 and 2 controllers handle their own Party Mode overrides. |
| `Water Heater - Reset Party Mode Above 55C.yaml` | Automatically turns Party Mode off at 55°C or higher, including if enabled while already hot. |
| `Water Heater - Energy Tracking.yaml` | Estimate energy from phases 1–3 while they are on. |
| `Water Heater - Runaway Protection.yaml` | Shut down phases 1–3 and latch a fault above 10 kWh estimated energy. |
| `Heater Turn Off (65°C Reached).yaml` | Shut down phases 1–3 and latch a fault above 65°C, including after startup. |
| `Water Heater - Fault off.yaml` | Shut down phases 1–3 for a latched fault or either sensor fault; fail safe on startup. |
| `Water Heater - Reset Cycle Energy.yaml` | Reset the estimated energy total daily at 00:01 when no fault is latched. |

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
- Party Mode bypasses solar-power checks and energizes phases 1–3 until the tank reaches 55°C, then clears its helper. Phase 3 is switched by its own automation, while the phase 1 and 2 controllers manage those phases. It remains subject to sensor-fault, fault-lockout, and over-temperature safety protections; verify that all three phases can safely run together before enabling it. Normal/weekly heat demand can still continue independently after Party Mode resets, subject to its usual solar thresholds.
- Energy tracking is an estimate assuming 1 kW per active phase (including phase 3) and minute-level updates, not a meter reading. The 10 kWh protection is therefore approximate and resets daily when not faulted.
- Disinfection is marked complete immediately once the sensed temperature is above 62°C; no minimum hold time is enforced.
- Disinfection due status is derived directly from `input_datetime.water_heater_last_legionella_cycle`; `input_boolean.water_heater_desinfection_needed` is no longer used and can be removed from Home Assistant. If the datetime is unset or invalid, a disinfection cycle is treated as due.
- Disinfection heating is subject to the same surplus-power thresholds as normal heating; without enough solar surplus, it waits for suitable power rather than forcing the heater on.
- Check the actual sensor and switch entity IDs in each automation against your Home Assistant installation.