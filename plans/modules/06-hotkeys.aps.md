# Hotkeys — activation via GNOME shortcuts and the Global Shortcuts portal

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| HOT | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §8 · ADR-002

## Purpose

Let the user start, stop and cancel dictation from anywhere on a Wayland
desktop. Toggle mode ships first on GNOME custom shortcuts (press-only);
real push-to-talk arrives with the XDG Global Shortcuts portal, whose
Activated/Deactivated signals are the only Wayland-legitimate press/release
source.

## In Scope

- Activation abstraction so toggle and PTT share session semantics.
- GNOME custom-shortcut path: documented setup invoking `dictate toggle`
  (MVP 1).
- XDG Global Shortcuts portal registration, press/release handling, and
  re-registration across daemon restarts (MVP 2).
- Cancel routes: portal shortcut, CLI, Escape on a focused dictate surface.
- Short-press tolerance and missed-release backstop behaviour.
- Recording overlay/indicator surface (MVP 2 polish).

## Out of Scope

- Session state machine itself (01-core); shortcut configuration UI
  (10-settings).

## Interfaces

**Depends on:**

- core — IPC control methods, session events, config for bindings.

**Exposes:**

- Activation events (start/stop/cancel) into the daemon session manager.
- Portal availability status for `dictate doctor`.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### HOT-001: Toggle activation via GNOME custom shortcut

- **Intent:** MVP 1 users can dictate hands-on-keyboard with stock GNOME.
- **Expected Outcome:** Documented setup binds a GNOME custom shortcut to
  `dictate toggle`; repeated presses alternate start/stop against a live
  daemon; `dictate doctor` reports whether a binding exists where detectable.
- **Validation:** `cargo test -p dictate-cli toggle` (plus scripted desktop
  checklist)
- **Confidence:** high

### HOT-002: Global Shortcuts portal registration

- **Intent:** dictate owns desktop-blessed global shortcuts on Wayland.
- **Expected Outcome:** Configured shortcuts register through the portal with
  user consent; registration survives daemon restart without re-prompting
  where the portal allows; portal absence degrades gracefully and is
  reported by doctor.
- **Validation:** `cargo test -p dictate-hotkey-portal registration -- --ignored`
  (requires desktop session)
- **Dependencies:** HOT-001
- **Confidence:** low

### HOT-003: Push-to-talk

- **Intent:** Hold-to-record makes dictation feel instant.
- **Expected Outcome:** Portal Activated starts recording and Deactivated
  stops it; presses shorter than the minimum are discarded quietly; a missed
  release is capped by max duration; focus changes mid-hold don't move the
  captured target.
- **Validation:** `cargo test -p dictate-hotkey ptt` (state-level), desktop
  checklist for end-to-end
- **Dependencies:** HOT-002
- **Confidence:** medium

### HOT-004: Cancel routes

- **Intent:** Backing out of a dictation is always one gesture away.
- **Expected Outcome:** Cancel works via CLI, registered portal shortcut, and
  Escape when a dictate surface is focused; cancel during Listening discards
  audio, during Transcribing suppresses delivery; no transcript escapes a
  cancelled session.
- **Validation:** `cargo test -p dictate-core cancel_paths`
- **Dependencies:** HOT-001
- **Confidence:** high

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
