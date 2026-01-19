# HR Zone Control and Limits

This document describes how the app determines heart-rate zone targets and
range limits, and which settings control the behavior.

## Overview

The app supports two mutually exclusive HR control modes:

1. **PID on Heart Zone**: target a heart-rate zone index (1-5).
2. **PID on HR min/max**: target an explicit HR range in bpm.

If both are configured, the zone-based PID branch runs first and the min/max
branch is skipped in that cycle.
【F:src/homeform.cpp†L6952-L7107】

## HR Zone Boundaries (BPM)

QZ defines HR zones as percentage bands of the user's maximum heart rate:

- Max HR defaults to `220 - age` unless a max HR override is enabled.
- The zone thresholds are configured as percentages:
  - `heart_rate_zone1`, `heart_rate_zone2`, `heart_rate_zone3`, `heart_rate_zone4`.
- Zone ranges are derived by multiplying each percentage by Max HR and dividing
  by 100. For example, Zone 2 spans from `zone1% * MaxHR` up to
  `zone2% * MaxHR - 1`.

These thresholds are used to compute the current zone and to display the BPM
range for the active zone in the UI.
【F:src/homeform.cpp†L6611-L6726】【F:src/homeform.cpp†L9344-L9357】
【F:src/qzsettings.cpp†L212-L215】【F:src/qzsettings.h†L607-L617】

## PID on Heart Zone (Zone-Based Control)

### Inputs

- Global setting: `treadmill_pid_heart_zone` ("Disabled" or 1-5).
- Training program override: `zoneHR` in the current row.
- Optional bounds from the current row:
  - `minSpeed`, `maxSpeed`, `maxResistance`.
- Control interval:
  - `loopTimeHR` from the current row, or a default of 10 seconds.
- Optional behavior flags:
  - `trainprogram_pid_pushy` to allow extra nudges toward the target zone.
  - `trainprogram_pid_ignore_inclination` to disable treadmill incline
    compensation in the zone-based branch.
【F:src/homeform.cpp†L6952-L7098】【F:src/qzsettings.cpp†L221-L222】
【F:src/qzsettings.cpp†L861-L862】【F:src/qzsettings.cpp†L965-L966】

### Behavior

- If a training program row has `zoneHR`, it overrides the global setting and
  updates the setting to the selected zone (or disables if zone <= 0).
- Every `loopTimeHR` seconds, the app compares the current HR zone with the
  target zone and adjusts speed or resistance:
  - **Treadmill**: adjust speed by 0.2 km/h within `minSpeed`/`maxSpeed`.
  - **Bike**: adjust resistance by 1, or adjust power if ERG mode is active
    using `pid_heart_zone_erg_mode_watt_step`.
  - **Rower**: adjust resistance by 1.
- For treadmills, if incline changes and incline compensation is enabled, the
  app recalculates a speed that preserves current wattage before the zone logic
  step.
【F:src/homeform.cpp†L6952-L7098】【F:src/qzsettings.cpp†L1027-L1027】

## PID on HR Min/Max (Range-Based Control)

### Inputs

- Global settings: `treadmill_pid_heart_min`, `treadmill_pid_heart_max`.
- Training program override: `HRmin`, `HRmax` in the current row.
- Optional bounds from the current row:
  - `minSpeed`, `maxSpeed`, `maxResistance`.
- Control interval:
  - `loopTimeHR` from the current row, or a default of 10 seconds.
【F:src/homeform.cpp†L7101-L7221】【F:src/qzsettings.cpp†L680-L681】

### Behavior

- If a training program row has `HRmin` and `HRmax`, it overrides the global
  min/max settings.
- Every `loopTimeHR` seconds, the app compares the current HR against the range
  and adjusts speed or resistance to move back into the band:
  - **Treadmill**: adjust speed by 0.2 km/h within `minSpeed`/`maxSpeed`.
  - **Bike**: adjust resistance by 1 within `maxResistance`.
  - **Rower**: adjust resistance by 1.
- If `hrmax` is 0 or -1, it is treated as 220.
【F:src/homeform.cpp†L7101-L7240】

## Precedence and Conflicts

- Zone-based control takes precedence over min/max control when enabled.
- Training program values override global settings during that row.
- Bounds (`minSpeed`, `maxSpeed`, `maxResistance`) are applied only when
  provided by the active training program row.
【F:src/homeform.cpp†L6952-L7240】

## Configuration References

- `treadmill_pid_heart_zone` (settings UI and QZ settings store).
- `treadmill_pid_heart_min`, `treadmill_pid_heart_max` (settings UI).
- `trainprogram_pid_pushy`, `trainprogram_pid_ignore_inclination`.
- `pid_heart_zone_erg_mode_watt_step` for ERG step size on bikes.
【F:src/settings.qml†L7388-L7475】【F:src/qzsettings.cpp†L221-L222】
【F:src/qzsettings.cpp†L680-L681】【F:src/qzsettings.cpp†L861-L862】
【F:src/qzsettings.cpp†L965-L966】【F:src/qzsettings.cpp†L1027-L1027】
