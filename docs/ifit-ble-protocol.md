# iFit / NordicTrack BLE Protocol Notes

This document summarizes the iFit-style BLE protocol as implemented in the codebase.
It focuses on the NordicTrack/ProForm bike flow and the iFit virtual device emulator.

## Scope

- **Real devices**: NordicTrack/ProForm bikes using the iFit UUIDs.
- **Virtual device**: `virtualbike` emulates iFit devices for client apps.

These notes are derived from the code paths that read/write the iFit BLE frames.

## BLE UUIDs and Roles

The iFit protocol uses a vendor UUID pair:

- **Write characteristic**: `00001534-1412-efde-1523-785feabcd123`
- **Notify characteristic**: `00001535-1412-efde-1523-785feabcd123`

The bike implementation writes frames to `00001534` and subscribes to notifications
on `00001535`. The virtual device receives frames on `00001534` and responds by
writing to `00001535`.【F:src/devices/proformbike/proformbike.cpp†L3235-L3323】
【F:src/virtualdevices/virtualbike.cpp†L615-L676】

## Initialization and Polling (NordicTrack/ProForm Bikes)

The NordicTrack/ProForm bike code sends a repeating set of "no-op" frames and
variant-specific frames to keep the device streaming metrics and to send control
requests. The exact payloads differ by model (for example, NordicTrack GX 2.7
uses its own `noOpData2/3/5` sequences).【F:src/devices/proformbike/proformbike.cpp†L890-L1119】

Key points:

- A sequence of init frames (`initData10..12`) is sent after discovery to prime
  the protocol state machine.【F:src/devices/proformbike/proformbike.cpp†L3235-L3273】
- A periodic polling loop writes different frames per counter tick; some cycles
  also inject resistance writes via `innerWriteResistance()`.
  【F:src/devices/proformbike/proformbike.cpp†L890-L1180】
- NordicTrack GX 2.7 and related models use dedicated frame variants in the
  polling loop.【F:src/devices/proformbike/proformbike.cpp†L919-L1119】

## Virtual iFit Device Handshake

The virtual iFit device listens for incoming frames on `00001534` and responds
with a scripted sequence of replies ("ifit ans 1" through "ifit ans 16") to
emulate the device handshake and status updates. The reply payloads are fixed
hex frames with dynamic metric fields injected in response `ans 11`.
【F:src/virtualdevices/virtualbike.cpp†L615-L907】

### Status Update ("ans 11")

For the main status reply sequence, the emulator writes multiple frames and
patches in current metrics:

- **Resistance** (byte 11 of `reply2`) uses the last requested iFit resistance
  when available and is clamped to `0x26`.【F:src/virtualdevices/virtualbike.cpp†L762-L785】
- **Watts** (bytes 12-13 of `reply2`) are encoded as a 16-bit integer.
  【F:src/virtualdevices/virtualbike.cpp†L781-L787】
- **Distance** (bytes 14-16 of `reply2`) uses odometer meters encoded as 24 bits.
  【F:src/virtualdevices/virtualbike.cpp†L781-L788】
- **Cadence** (byte 18 of `reply2`) is a single byte.
  【F:src/virtualdevices/virtualbike.cpp†L781-L788】
- **Elapsed time** (bytes 6-7, 11-12, 15-16 of `reply3`) uses seconds since the
  first iFit frame in the session.
  【F:src/virtualdevices/virtualbike.cpp†L785-L799】
- **Speed** (bytes 13-14 of `reply3`) is encoded as `speed * 100`.
  【F:src/virtualdevices/virtualbike.cpp†L791-L799】
- **Calories** (bytes 3-4 and 10-11 of `reply4`) use a scaled value; the code
  applies a constant multiplier before encoding.
  【F:src/virtualdevices/virtualbike.cpp†L772-L808】

A checksum-like byte is updated based on several field values and written to both
`reply3[19]` and `reply4[19]`.【F:src/virtualdevices/virtualbike.cpp†L801-L809】

### Resistance Requests ("ans 14")

When the emulator receives a frame starting with `FF 0D 02`, it treats byte 12
as the requested iFit resistance. The code stores the requested value and, when
allowed, translates it to a device resistance or inclination change.
【F:src/virtualdevices/virtualbike.cpp†L859-L887】

### Stop Requests ("ans 15")

A specific frame pattern triggers a stop request; the emulator sets a stop flag
and emits a response sequence indicating stop state. The next status cycle will
use the stop-specific reply in "ans 12".
【F:src/virtualdevices/virtualbike.cpp†L892-L904】

## Device-Specific Behavior Notes

- NordicTrack/ProForm bikes share the same iFit UUIDs but use model-specific
  polling frames and resistance write behaviors.
  【F:src/devices/proformbike/proformbike.cpp†L890-L1180】
- The virtual iFit device adapts resistance requests depending on device
  capabilities (direct iFit-compatible resistance, incline-based mapping, or
  Peloton-style conversion).
  【F:src/virtualdevices/virtualbike.cpp†L874-L887】

## Practical Implications

- If you are integrating a new NordicTrack/ProForm model, start by cloning an
  existing "no-op" frame set and adjusting model-specific frames in the polling
  loop.
- For iFit client compatibility, the handshake frame sequence and the dynamic
  metric encoding in "ans 11" are essential.
