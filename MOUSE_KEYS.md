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

They come in two flavors:

### Analog depth control (default)

The Hall-effect advantage: **speed follows key travel depth**. When a
movement key is held, the cursor step size scales linearly from 1x at the
dead zone to `MOUSEKEY_MAX_SPEED` at the bottom of travel, using the live
per-key distance reported by the matrix. Press lightly for precise nudging,
lean deeper to cross the screen, ease off to slow down instantly - no ramp
to wait for. Rapid Trigger composes naturally: a quick light tap is a small,
slow nudge.

A small dead zone at the top of travel suppresses noise and accidental
grazes: a held key shallower than `MOUSEKEY_ANALOG_DEAD_ZONE` produces no
motion.

### Time-based acceleration (fallback)

Set `MOUSEKEY_ANALOG_ENABLE = 0` for classic QMK-style behavior: speed is
independent of depth and accelerates linearly from 1x to `MOUSEKEY_MAX_SPEED`
over `MOUSEKEY_TIME_TO_MAX` steps.

## Configuration (compile-time)

All values have defaults and can be overridden in a keyboard's config:

| Define | Default | Meaning |
|--------|---------|---------------|
| `MOUSEKEY_ANALOG_ENABLE` | 1 | Speed proportional to travel depth |
| `MOUSEKEY_ANALOG_DEAD_ZONE` | 8 | Depth (0-255) below which a held key produces no motion |
| `MOUSEKEY_INTERVAL` | 20 ms | Time between movement steps |
| `MOUSEKEY_MOVE_DELTA` | 2 | Base step size in counts |
| `MOUSEKEY_MAX_SPEED` | 10 | Speed multiplier at the bottom of travel |
| `MOUSEKEY_DELAY` | 10 ms | (analog off) Delay before movement starts |
| `MOUSEKEY_TIME_TO_MAX` | 40 | (analog off) Steps until maximum speed |
| `MOUSEKEY_WHEEL_DELAY` | 10 ms | Delay before scrolling starts |
| `MOUSEKEY_WHEEL_INTERVAL` | 50 ms | Time between scroll steps |

## Known limitations

- **hmkconf support lives in a companion PR** on the same branch name
  (`feature/mouse-keys`) so the keycodes are assignable from the UI.
- No acceleration-profile keycodes (`MS_ACL0/1/2`-style). The curve is fixed
  at compile time.

## Tuning notes

- The analog curve is currently **linear** in depth. If deep presses feel
  abrupt near the bottom, a soft saturation curve (easing the last ~20% of
  travel toward `MOUSEKEY_MAX_SPEED`) is the natural next tweak - kept out of
  this change until the feel can be judged on hardware.
- When a movement keycode is registered, the layout binds it to the physical
  key so the mouse task can read its depth. If the same keycode is bound to
  multiple keys, the most recent binding wins.
- Keycodes injected without a physical key (e.g. by external tools) are
  treated as fully pressed.

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
