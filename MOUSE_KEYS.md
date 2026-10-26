# Mouse Keys (experimental)

This branch adds QMK-style **mouse keys** to libhmk: cursor movement and
vertical scrolling from keycodes, so a mouse layer can be mapped onto any
layout. Previously libhmk only supported mouse *buttons* (`MS_BTN1`–`MS_BTN5`).

> **Status: untested.** This was developed without hardware. It compiles by
> inspection against the libhmk patterns but has never run on a board. Treat
> the first flash as experimental and keep a known-good firmware around.

## New keycodes

| Keycode | Aliases | Action |
|---------|---------|--------|
| `SP_MOUSE_UP` | `MS_UP` | Move cursor up |
| `SP_MOUSE_DOWN` | `MS_DOWN` | Move cursor down |
| `SP_MOUSE_LEFT` | `MS_LEFT` | Move cursor left |
| `SP_MOUSE_RIGHT` | `MS_RGHT` | Move cursor right |
| `SP_MOUSE_WHEEL_UP` | `MS_WHU` | Scroll up |
| `SP_MOUSE_WHEEL_DOWN` | `MS_WHD` | Scroll down |

The keycodes live in the free special-keycode space (`0xA0`–`0xA7`, between
`SP_MOUSE_BUTTON_5` and `SP_MO_MIN`), so no EEPROM layout migration is needed.

Horizontal scroll uses the **AC Pan** axis. The stock TinyUSB mouse report
descriptor has a vertical wheel only, so `usb_descriptors.c` writes the mouse
descriptor explicitly with an extra AC Pan axis, matching `hid_mouse_report_t`
(buttons, x, y, wheel, pan).

They behave like QMK's mouse keys: hold the key and the cursor accelerates
linearly from 1x to `MOUSEKEY_MAX_SPEED` over `MOUSEKEY_TIME_TO_MAX` steps.

## Configuration (compile-time)

All values have defaults and can be overridden in a keyboard's config:

| Define | Default | Meaning |
|--------|---------|---------|
| `MOUSEKEY_DELAY` | 10 ms | Delay before movement starts after a key press |
| `MOUSEKEY_INTERVAL` | 20 ms | Time between movement steps |
| `MOUSEKEY_MOVE_DELTA` | 2 | Base step size in counts |
| `MOUSEKEY_MAX_SPEED` | 10 | Maximum speed multiplier |
| `MOUSEKEY_TIME_TO_MAX` | 40 | Steps until maximum speed is reached |
| `MOUSEKEY_WHEEL_DELAY` | 10 ms | Delay before scrolling starts |
| `MOUSEKEY_WHEEL_INTERVAL` | 50 ms | Time between scroll steps |

## Known limitations

- **hmkconf support lives in a companion PR** on the same branch name
  (`feature/mouse-keys`) so the keycodes are assignable from the UI.
- No acceleration-profile keycodes (`MS_ACL0/1/2`-style). The curve is fixed
  at compile time.

## Roadmap: analog (Hall-effect) speed control

The current implementation is **binary**: a movement key is either held or
not, and speed comes from time-based acceleration only. Key travel depth is
ignored - which throws away the main advantage of a Hall-effect board.

A follow-up could make speed proportional to key depth:

- A mouse-movement binding contributes `direction x normalized_depth` to the
  axis velocity each tick, instead of a fixed step.
- Depth is taken from the per-key analog value with a small dead zone at the
  top of travel (noise/graze protection) and a soft saturation curve near the
  bottom (so bottoming out does not feel like a cliff).
- Speed becomes positional rather than temporal: press lightly for precise
  nudging, lean deeper to cross the screen, and ease off to slow instantly.
  Rapid Trigger composes naturally with this (a quick light tap is a small,
  slow nudge - no ramp to wait for).

This requires deeper integration than keycodes: the mouse task must read live
per-key analog values, and "mouse axis" is arguably a new advanced-key type
rather than a plain keycode. The binary implementation ships first because it
is verifiable without hardware; the analog mode is the natural v2.

## Implementation notes

- Movement keycodes have no entry in `keycode_to_hid`; they are tracked as a
  bitmask in `hid.c` and polled by `hid_mouse_task()`, which is called once
  per main loop iteration and generates time-based deltas.
- The mouse report is only sent when it changes, using the existing
  round-robin report sender.
- Axis deltas are clamped to the HID ±127 range.

## iOS / iPadOS caution

libhmk also supports an **XInput gamepad mode** (`xinput.c`). If a board is
used with iOS or iPadOS, leave gamepad mode disabled: Apple's USB stack is
strict about gamepad-class HID devices, and a keyboard that also claims
gamepad interfaces is exactly the pattern that made 2026 Cipulot snapshot
builds fail on iOS (a joystick HID collection caused full rejection of the
device). The plain keyboard + boot mouse composite used here is the safe
combination.
