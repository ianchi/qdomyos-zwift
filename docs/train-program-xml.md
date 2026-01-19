# Training Program XML Format

This document describes the custom XML format used to define training programs.
It is based on the serialization and parsing logic in `src/trainprogram.cpp`.

## Overview

- The root element is `<rows>`.
- Each workout segment is a `<row>` element with attributes describing targets.
- Optional `<repeat>` blocks can repeat a sequence of rows.
- Optional `<textevent>` child elements can annotate a row with messages.
- The custom XML format does not include a program-level description or title.
  The app only exposes descriptions/tags for Zwift `.zwo` files in the program
  lists; for XML, store metadata alongside the file (for example, in the
  filename).

## Document Structure

```xml
<rows>
  <row duration="hh:mm:ss" speed="10.5" inclination="1.5" />
  <repeat times="3">
    <row duration="00:02:00" speed="12.0" inclination="2" />
    <row duration="00:01:00" speed="9.0" inclination="0" />
  </repeat>
</rows>
```

### Root Element: `<rows>`

Contains the ordered list of workout rows and repeat blocks.

### Repeat Block: `<repeat>`

Repeats the contained rows `times` times.

Attributes:

- `times` (int, required): number of repetitions.

### Row Element: `<row>`

Each row defines a workout segment. All attributes are optional unless noted.

#### Time and Distance

- `duration` (string, `hh:mm:ss`): segment length.
- `distance` (float, km): segment distance; can be used instead of duration in
  some modes.

#### Speed and Incline

- `speed` (float, km/h): target speed. This format does not define a pace unit;
  there is no min/km field.
- `minspeed` (float, km/h): minimum speed. Parsed as a floating-point value; use
  decimals when needed (for example, `7.5`).
- `maxspeed` (float, km/h): maximum speed. Parsed as a floating-point value; use
  decimals when needed (for example, `12.3`).
- `inclination` (float, % grade): target incline/grade.

If you want to work in min/km, convert to km/h (for example, `5:00` min/km is
`12.0` km/h) and use `speed`.

#### Resistance, Power, and METs

- `resistance` (int, level): target resistance.
- `lower_resistance` (int, level): lower bound resistance.
- `upper_resistance` (int, level): upper bound resistance.
- `requested_peloton_resistance` (int, level): target Peloton resistance.
- `lower_requested_peloton_resistance` (int, level): lower Peloton resistance.
- `upper_requested_peloton_resistance` (int, level): upper Peloton resistance.
- `maxresistance` (int, level): maximum resistance.
- `power` (int, W): target power.
- `powerzone` (float, fraction): power target as a fraction of FTP.
- `mets` (int, MET): target METs.

#### Cadence and Heart Rate

- `cadence` (int, rpm): target cadence.
- `lower_cadence` (int, rpm): lower cadence bound.
- `upper_cadence` (int, rpm): upper cadence bound.
- `zonehr` (int, zone index): HR zone.
- `hrmin` (int, bpm): minimum HR.
- `hrmax` (int, bpm): maximum HR.
- `looptimehr` (int, s): HR loop time.

Note: `zonehr` targets heart-rate zones, while `powerzone` targets power as a
fraction of FTP. Use `zonehr` when you want HR-based control rather than power.
On treadmills, `powerzone` uses run FTP; on other devices it uses cycling FTP.

#### Device-Specific Attributes

Some attributes are only meaningful on certain equipment:

- Peloton-related fields such as `requested_peloton_resistance`, `pace_intensity`,
  and the `lower_`/`upper_` peloton variants apply to Peloton device workflows.
- `powerzone` depends on device type to choose run FTP (treadmill) vs cycling FTP.

#### Conflicting Targets and Precedence

The XML loader stores all provided values; it does not resolve conflicts. Runtime
behavior depends on device support and the app's control logic. The key runtime
rules observed in the control logic:

- Speed changes are emitted only when `forcespeed=1` and `speed` is provided.
  If `speed` is set but `forcespeed=0`, the row does not trigger a speed change.
- When a row has `distance > 0`, row completion uses distance; otherwise duration
  determines the row end.
- If heart-rate zone control is enabled via `zonehr`, it is applied before the
  `hrmin`/`hrmax` PID branch. Avoid setting both in the same row.

To avoid ambiguous behavior:

- Prefer a single speed control method: use `speed` (with `forcespeed=1`) or a
  `distance`-based segment, but avoid mixing multiple speed targets in one row.
- For HR guidance, choose either `zonehr` or `hrmin`/`hrmax`, not both.
- For power targets, choose either `power` or `powerzone`, not both.

#### Miscellaneous Controls

- `forcespeed` (0|1): when `1`, the row's `speed` is enforced during the segment.
  When `0`, the app may compute speed from other inputs (for example, distance).
  For `forcespeed=1`, you should also provide a `speed` value.
- `fanspeed` (float, level): fan speed.
- `pace_intensity` (int, level): Peloton pace intensity.

#### GPS and Orientation (Optional)

- `latitude` (float, degrees)
- `longitude` (float, degrees)
- `altitude` (float, m)
- `azimuth` (float, degrees)

### Text Events: `<textevent>`

Child elements of a row for time-based messages.

Attributes:

- `timeoffset` (int, s): seconds offset from the start of the row.
- `message` (string): message text.

Example:

```xml
<row duration="00:05:00" speed="10.0">
  <textevent timeoffset="60" message="Settle into pace" />
  <textevent timeoffset="240" message="Last minute push" />
</row>
```

## Ramp Segments

Ramp segments are expanded into multiple rows at load time.

### Speed Ramps

Use `speedfrom`, `speedto`, and `duration`. The loader splits the duration into
multiple 1-second rows with `forcespeed=1` and interpolated speeds.

- `speedfrom` (float, km/h)
- `speedto` (float, km/h)

```xml
<row duration="00:05:00" speedfrom="9.0" speedto="12.0" />
```

### Power Zone Ramps

Use `powerzonefrom`, `powerzoneto`, and `duration`. The loader expands the ramp
into 1-second rows. Each row’s power is calculated as a fraction of FTP
(run FTP for treadmills).

- `powerzonefrom` (float, fraction)
- `powerzoneto` (float, fraction)

```xml
<row duration="00:04:00" powerzonefrom="0.70" powerzoneto="0.90" />
```

## Typical Treadmill Program Examples

### 30-Minute Easy Run

```xml
<rows>
  <row duration="00:05:00" speed="8.0" inclination="0" />
  <row duration="00:20:00" speed="9.5" inclination="1" />
  <row duration="00:05:00" speed="8.0" inclination="0" />
</rows>
```

### Hill Intervals (Repeat Block)

```xml
<rows>
  <row duration="00:05:00" speed="8.5" inclination="0" />
  <repeat times="6">
    <row duration="00:02:00" speed="10.0" inclination="4" />
    <row duration="00:02:00" speed="8.5" inclination="1" />
  </repeat>
  <row duration="00:05:00" speed="8.0" inclination="0" />
</rows>
```

### Progressive Tempo (Speed Ramp)

```xml
<rows>
  <row duration="00:08:00" speed="8.5" inclination="1" />
  <row duration="00:12:00" speedfrom="9.0" speedto="12.0" />
  <row duration="00:05:00" speed="8.0" inclination="0" />
</rows>
```

### 10K Plan with Distance-Based Intervals

```xml
<rows>
  <row distance="3.0" zonehr="2" />
  <row distance="3.0" speed="7.0" />
  <row distance="4.0" zonehr="2" />
  <row duration="00:02:00" speed="6.0" inclination="0" />
</rows>
```
