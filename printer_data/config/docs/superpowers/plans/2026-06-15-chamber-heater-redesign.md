# Chamber Heater Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor chamber heater control into a shared `_HEATSOAK_CORE` macro, add a standalone `HEATSOAK` command for console-triggered heatsoak, and replace the broken control loop with a two-state `heating`/`maintaining` state machine whose target can be updated at runtime.

**Architecture:** `START_CHAMBER_TEMP_CONTROL` owns the desired chamber ambient temperature as a gcode variable. A rewritten `chamber_heater_control` delayed gcode reads from it every 10 seconds and drives `heater_chamber` via a two-state state machine. `_HEATSOAK_CORE` handles the wait-to-temperature sequence and hands off to the control loop. Both `PRINT_START_HEATSOAK` and the new `HEATSOAK` macro use `_HEATSOAK_CORE` as their shared implementation.

**Tech Stack:** Klipper gcode macros (Jinja2 templating), Mainsail UI. No build step — changes take effect after `FIRMWARE_RESTART` in the Mainsail console.

---

## File Map

| File | Change |
|---|---|
| `macros/chamber_heater.cfg` | Full rewrite — state machine, `_HEATSOAK_CORE`, `HEATSOAK`, `SET_CHAMBER_TARGET`, `STOP_HEATSOAK` |
| `macros/print_start/heatsoak.cfg` | Refactor `PRINT_START_HEATSOAK` to call `_HEATSOAK_CORE` |
| `macros/print_end.cfg` | Add two lines to stop the control loop on print end |

All paths are relative to `/home/pi/printer_data/config/`.

---

## Task 1: Rewrite `chamber_heater.cfg`

**Files:**
- Modify: `macros/chamber_heater.cfg`

This replaces the entire file. The old delayed gcode had a bug (`chamber_target` used where `chamber_temp` was defined) and read target from `PRINT_START_SETTINGS`. The new version introduces a proper two-state state machine and stores its own target.

- [ ] **Step 1: Replace the entire contents of `macros/chamber_heater.cfg`**

```cfg
# Two-state chamber heater control loop.
# State 'heating':    heater at 70°C — max safe for the ASA heater enclosure.
# State 'maintaining': heater at target+5°C — gentle hold once chamber is at temp.
#
# Transitions:
#   chamber_temp < target - 2          → force back to 'heating'
#   chamber_temp >= target + 1         → transition to 'maintaining'
#   target - 2 <= chamber_temp < target+1 → hold current state
#
[delayed_gcode chamber_heater_control]
initial_duration: 0.0
gcode:
  {% set S = printer['gcode_macro START_CHAMBER_TEMP_CONTROL'] %}
  {% set target = S.target %}
  {% set state = S.state %}
  {% set chamber_temp = printer['temperature_sensor chamber'].temperature %}

  {% if target == 0 %}
    # Target zeroed — loop stops, do not reschedule.
  {% elif chamber_temp < target - 2 %}
    SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=state VALUE='"heating"'
    SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET=70
    UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=10
  {% elif chamber_temp >= target + 1 %}
    SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=state VALUE='"maintaining"'
    SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET={target + 5}
    UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=10
  {% else %}
    {% if state == 'heating' %}
      SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET=70
    {% else %}
      SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET={target + 5}
    {% endif %}
    UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=10
  {% endif %}


[gcode_macro START_CHAMBER_TEMP_CONTROL]
variable_target: 0
variable_state: 'heating'
gcode:
  {% set target = params.TARGET | int %}
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=target VALUE={target}
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=state VALUE='"heating"'
  UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=1
  SET_DISPLAY_TEXT MSG="Chamber heater control activated"


# Private macro — not shown in Mainsail. Owns the chamber heatsoak wait sequence.
# Never writes gcode variables — only consumes the CHAMBER param.
[gcode_macro _HEATSOAK_CORE]
gcode:
  {% set chamber = params.CHAMBER | int %}
  SET_DISPLAY_TEXT MSG="Heating chamber to {chamber}c"
  SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET=70
  TEMPERATURE_WAIT SENSOR="temperature_sensor chamber" MINIMUM={chamber}
  SET_HEATER_TEMPERATURE HEATER=heater_chamber TARGET=0
  START_CHAMBER_TEMP_CONTROL TARGET={chamber}


# Standalone heatsoak trigger for console/Mainsail use.
# Bed is fixed at 110°C. CHAMBER defaults to 55°C.
[gcode_macro HEATSOAK]
gcode:
  {% set chamber = params.CHAMBER | default(55) | int %}
  SET_DISPLAY_TEXT MSG="Heatsoak: Chamber {chamber}c"
  SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=110
  M106 S77
  _HEATSOAK_CORE CHAMBER={chamber}


# Update the desired chamber ambient temp at runtime.
# Pin this macro to the Mainsail dashboard for one-click access during a print.
# Resets state to 'heating' so the loop re-evaluates from scratch.
[gcode_macro SET_CHAMBER_TARGET]
gcode:
  {% set target = params.TARGET | default(55) | int %}
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=target VALUE={target}
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=state VALUE='"heating"'
  SET_DISPLAY_TEXT MSG="Chamber target: {target}c"


# Stop the control loop immediately, turn off all heaters and fans.
[gcode_macro STOP_HEATSOAK]
gcode:
  UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=0
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=target VALUE=0
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=state VALUE='"heating"'
  TURN_OFF_HEATERS
  M107
  SET_DISPLAY_TEXT MSG="Chamber heater stopped"
```

- [ ] **Step 2: Restart Klipper and verify the file loads cleanly**

In the Mainsail console run:
```
FIRMWARE_RESTART
```
Expected: no red error messages in the console. The macros `HEATSOAK`, `SET_CHAMBER_TARGET`, and `STOP_HEATSOAK` appear in the Mainsail macro list (note: `_HEATSOAK_CORE` will NOT appear — that's correct).

- [ ] **Step 3: Verify `STOP_HEATSOAK` executes without errors**

In the Mainsail console run:
```
STOP_HEATSOAK
```
Expected: display text shows "Chamber heater stopped", `heater_chamber` target goes to 0, no console errors.

- [ ] **Step 4: Verify `SET_CHAMBER_TARGET` executes without errors**

```
SET_CHAMBER_TARGET TARGET=55
```
Expected: display text shows "Chamber target: 55c", no errors.

- [ ] **Step 5: Verify the control loop self-terminates when target is 0**

```
START_CHAMBER_TEMP_CONTROL TARGET=0
```
Expected: display text shows "Chamber heater control activated", but the loop fires once after 1 second and immediately stops (target is 0). `heater_chamber` stays at 0. No repeated firing.

- [ ] **Step 6: Verify the control loop runs and applies the heating state**

```
START_CHAMBER_TEMP_CONTROL TARGET=80
```
Expected: after ~1 second the loop fires. Since current chamber temp is almost certainly below `80 - 2 = 78°C`, `heater_chamber` target is set to 70°C. The loop reschedules itself every 10 seconds. Confirm by checking `heater_chamber` target in the Mainsail temperature panel.

Then stop it:
```
STOP_HEATSOAK
```

---

## Task 2: Refactor `print_start/heatsoak.cfg`

**Files:**
- Modify: `macros/print_start/heatsoak.cfg`

Replace the body of `PRINT_START_HEATSOAK`. The chamber heatsoak sequence moves into `_HEATSOAK_CORE`. The bed-only path (target_chamber == 0) is unchanged.

- [ ] **Step 1: Replace the entire contents of `macros/print_start/heatsoak.cfg`**

```cfg
[gcode_macro PRINT_START_HEATSOAK]
gcode:
  {% set S = printer['gcode_macro PRINT_START_SETTINGS'] %}
  {% set target_bed = S.target_bed %}
  {% set target_chamber = S.target_chamber %}
  {% set x_wait = S.x_wait %}
  {% set y_wait = S.y_wait %}

  # No chamber target — short bed-only heatsoak
  {% if target_chamber == 0 %}
    SET_DISPLAY_TEXT MSG="Bed: {target_bed}c"
    G1 X{x_wait} Y{y_wait} Z50 F9000
    M106 S77
    M190 S{target_bed}
    SET_DISPLAY_TEXT MSG="Soak for 3 min"
    G4 P180000

  # Chamber target set — heat chamber first, then bring bed to print temp
  {% else %}
    SET_DISPLAY_TEXT MSG="Target Chamber: {target_chamber}c"
    G1 X{x_wait} Y{y_wait} Z100 F9000
    M106 S77
    SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=110
    _HEATSOAK_CORE CHAMBER={target_chamber}
    SET_DISPLAY_TEXT MSG="Bed: {target_bed}c"
    M190 S{target_bed}
  {% endif %}

  # Preheat hotend to safe temp for probing
  SET_DISPLAY_TEXT MSG="Hotend: 150c"
  M104 S150
```

- [ ] **Step 2: Restart Klipper and verify the file loads cleanly**

```
FIRMWARE_RESTART
```
Expected: no errors. `PRINT_START_HEATSOAK` is not a pinned macro button so it won't appear in the list — that is correct, it is called internally by the print start sequence.

---

## Task 3: Patch `print_end.cfg`

**Files:**
- Modify: `macros/print_end.cfg`

Add two lines before `TURN_OFF_HEATERS` to stop the control loop and zero its target. Without this the delayed gcode keeps self-scheduling after a print ends.

- [ ] **Step 1: Edit `macros/print_end.cfg` — insert loop stop before `TURN_OFF_HEATERS`**

Current file:
```cfg
[gcode_macro PRINT_END]
gcode:
  M400                           ; wait for buffer to clear
  G92 E0                         ; zero the extruder
  G1 E-10.0 F3600                ; retract filament
  G91                            ; relative positioning
  G0 Z1.00 X20.0 Y20.0 F20000    ; move nozzle to remove stringing
  TURN_OFF_HEATERS
  ...
```

Replace with:
```cfg
[gcode_macro PRINT_END]
gcode:
  M400                           ; wait for buffer to clear
  G92 E0                         ; zero the extruder
  G1 E-10.0 F3600                ; retract filament
  G91                            ; relative positioning
  G0 Z1.00 X20.0 Y20.0 F20000    ; move nozzle to remove stringing
  UPDATE_DELAYED_GCODE ID=chamber_heater_control DURATION=0
  SET_GCODE_VARIABLE MACRO=START_CHAMBER_TEMP_CONTROL VARIABLE=target VALUE=0
  TURN_OFF_HEATERS

  M107                           ; turn off fan
  G1 Z2 F3000                    ; move nozzle up 2mm
  G90                            ; absolute positioning
  G0 X{printer.toolhead.axis_maximum.x - 10} Y{printer.toolhead.axis_maximum.y - 10} F3600
  BED_MESH_CLEAR
  M84
```

- [ ] **Step 2: Restart Klipper and verify the file loads cleanly**

```
FIRMWARE_RESTART
```
Expected: no errors.

- [ ] **Step 3: Verify `PRINT_END` stops the loop**

First start the control loop:
```
START_CHAMBER_TEMP_CONTROL TARGET=55
```
Confirm the loop is running (check `heater_chamber` target in temperature panel — should be 70°C within 10 seconds).

Then run:
```
PRINT_END
```
Expected: `heater_chamber` target returns to 0, all heaters off, toolhead parks at rear. No further changes to `heater_chamber` after 10 seconds (loop has stopped).
