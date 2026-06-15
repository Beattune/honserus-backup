# Chamber Heater Redesign

**Date:** 2026-06-15

## Goal

Two improvements:
1. Allow heatsoak to be triggered manually from the Mainsail console for mechanical checks (belt tension etc.) without a slicer-initiated print start.
2. Allow the desired chamber ambient temperature to be changed at runtime during a print.

---

## Hardware (unchanged)

| Object | Pin | Role |
|---|---|---|
| `heater_generic heater_chamber` | PA3 (HE2 via SSR) | PTC heating element |
| Heater thermistor | PF4 | PID sensor, placed in heater fins — must not change |
| `temperature_sensor chamber` | PF6 | Ambient chamber temperature |
| `heater_fan chamber_heater_fan` | PD13 | Circulation fan, tied to `heater_chamber` |

---

## Existing Bugs Fixed

- `chamber_heater.cfg` uses `chamber_target` where `chamber_temp` was defined — the control loop never evaluated correctly. Fixed in the refactor.
- `PRINT_END` calls `TURN_OFF_HEATERS` but leaves the `chamber_heater_control` delayed gcode running indefinitely. Fixed by stopping the loop in `PRINT_END`.

---

## Architecture

### Source of truth for desired chamber ambient temp

`START_CHAMBER_TEMP_CONTROL` gains two gcode variables:

- `variable_target: 0` — the desired chamber ambient temperature in °C. `0` means inactive.
- `variable_state: 'heating'` — current control loop state: `'heating'` or `'maintaining'`.

The control loop reads exclusively from these variables. `PRINT_START_SETTINGS` is no longer read by the control loop.

### Changing the target at runtime

`SET_CHAMBER_TARGET TARGET=<n>` writes a new value to `START_CHAMBER_TEMP_CONTROL.target` and resets state to `'heating'`. Pin this macro to the Mainsail dashboard for one-click access during a print.

---

## Control Loop State Machine

The `chamber_heater_control` delayed gcode runs every 10 seconds.

**Inputs:**
- `target` — from `START_CHAMBER_TEMP_CONTROL.target`
- `state` — from `START_CHAMBER_TEMP_CONTROL.state`
- `chamber_temp` — from `temperature_sensor chamber`

**Logic:**

```
if target == 0:
    stop loop (do not reschedule)

elif chamber_temp < target - 2:
    state → 'heating'
    heater_chamber TARGET = 70

elif chamber_temp >= target + 1:
    state → 'maintaining'
    heater_chamber TARGET = target + 5

else:  # target - 2 <= chamber_temp < target + 1  (hold band)
    hold current state
    if state == 'heating':   heater_chamber TARGET = 70
    if state == 'maintaining': heater_chamber TARGET = target + 5

reschedule in 10s
```

**State transitions:**

| Condition | Transition |
|---|---|
| `chamber_temp < target - 2` | → `heating` (always, from either state) |
| `chamber_temp >= target + 1` | → `maintaining` |
| `target - 2 <= chamber_temp < target + 1` | hold current state |

The 70°C heater setpoint in `heating` state is the safe maximum for the ASA enclosure around the heater element.

---

## Shared Core: `_HEATSOAK_CORE`

Private macro (underscore prefix, not shown in Mainsail). Owns the chamber portion of heatsoak only. Never writes any gcode variables.

**Param:** `CHAMBER` (int, required) — desired ambient target in °C.

**Sequence:**
1. Display: `Heating chamber to {CHAMBER}c`
2. `SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET=70` — boost heat fast
3. `TEMPERATURE_WAIT SENSOR="temperature_sensor chamber" MINIMUM={CHAMBER}` — block until ambient reached
4. `SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET=0` — cut boost
5. `START_CHAMBER_TEMP_CONTROL TARGET={CHAMBER}` — hand off to control loop

---

## Macro Inventory

### Modified: `chamber_heater.cfg`

**`[delayed_gcode chamber_heater_control]`** — implements state machine above. Reads from `START_CHAMBER_TEMP_CONTROL`, not from `PRINT_START_SETTINGS`.

**`[gcode_macro START_CHAMBER_TEMP_CONTROL]`**
- Gains `variable_target: 0` and `variable_state: 'heating'`
- Accepts `TARGET` param, sets both variables, kicks off delayed gcode with `DURATION=1`

**`[gcode_macro _HEATSOAK_CORE]`** — new, described above.

**`[gcode_macro HEATSOAK]`** — new standalone trigger.
- Param: `CHAMBER` default `55`
- Sets bed to 110°C (non-blocking)
- `M106 S77` (part fans for air circulation)
- Calls `_HEATSOAK_CORE CHAMBER={CHAMBER}`

**`[gcode_macro SET_CHAMBER_TARGET]`** — new runtime control.
- Param: `TARGET` default `55`
- Writes `target` and resets `state` to `'heating'` on `START_CHAMBER_TEMP_CONTROL`
- Display: `Chamber target: {TARGET}c`

**`[gcode_macro STOP_HEATSOAK]`** — new.
- `UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=0`
- `SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=target VALUE=0`
- `SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=state VALUE='"heating"'`
- `TURN_OFF_HEATERS`
- `M107`
- Display: `Chamber heater stopped`

### Modified: `print_start/heatsoak.cfg`

**`[gcode_macro PRINT_START_HEATSOAK]`** — refactored to use `_HEATSOAK_CORE`.

- Reads `target_bed` and `target_chamber` from `PRINT_START_SETTINGS` (unchanged interface with slicer).
- If `target_chamber == 0`: existing bed-only path unchanged (M190, 3 min wait).
- If `target_chamber > 0`:
  1. Display, move to center, `M106 S77`
  2. `SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=110`
  3. `_HEATSOAK_CORE CHAMBER={target_chamber}`
  4. Display, `M190 S{target_bed}` — wait for bed to reach print target
- Preheat hotend to 150°C (unchanged).

### Modified: `print_end.cfg`

**`[gcode_macro PRINT_END]`** — add before `TURN_OFF_HEATERS`:
```
UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=0
SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=target VALUE=0
```

---

## File Changes Summary

| File | Change |
|---|---|
| `macros/chamber_heater.cfg` | Full refactor — state machine, new macros |
| `macros/print_start/heatsoak.cfg` | Refactored to call `_HEATSOAK_CORE` |
| `macros/print_end.cfg` | Stop control loop on print end |
