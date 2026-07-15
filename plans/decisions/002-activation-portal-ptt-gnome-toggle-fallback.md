# ADR-002: XDG Global Shortcuts portal for push-to-talk, GNOME custom shortcut toggle as fallback

| Field  | Value      |
| ------ | ---------- |
| Status | Accepted   |
| Date   | 2026-07-15 |

## Context

Wayland forbids global key grabs by ordinary clients. The only
desktop-blessed mechanisms on GNOME are:

- The **XDG Global Shortcuts portal** (`org.freedesktop.portal.GlobalShortcuts`),
  which emits distinct `Activated` and `Deactivated` signals per shortcut —
  the only legitimate source of press *and release* events, which
  push-to-talk requires. Available in `xdg-desktop-portal-gnome` since
  GNOME 45; current Ubuntu LTS ships newer.
- **GNOME custom shortcuts**, which run a command on key *press only* —
  sufficient for toggle mode, unusable for push-to-talk.

Evdev/uinput-level listening is rejected on principle (spec §3.2, §18.3).

## Decision

- MVP 1 activation: GNOME custom shortcut invoking `dictate toggle`
  (documented setup, no portal dependency).
- MVP 2: register shortcuts through the Global Shortcuts portal;
  push-to-talk maps Activated→start, Deactivated→stop.
- Portal unavailability degrades to toggle + CLI, reported by
  `dictate doctor`.
- Global `Escape` cancel is impossible on Wayland; cancel is a registered
  portal shortcut, the CLI, or Escape on a focused dictate surface.

## Consequences

- MVP 1 ships with zero portal risk; portal quirks (R1) only gate MVP 2.
- Missed release events are backstopped by the maximum recording duration.
- Users on pre-45 GNOME still get toggle mode.
- The activation layer needs an abstraction so both paths drive identical
  session semantics (module 06-hotkeys).
