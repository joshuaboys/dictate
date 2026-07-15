# Delivery — adapter routing, clipboard, feedback, history

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| DLV | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §8.4, §10.8, §10.10, §14 · ADR-003

## Purpose

Get the finished transcript to where the user expects it — the captured
target first, the clipboard as a first-class fallback — and tell the user
what happened. A transcript must never be silently lost.

## In Scope

- `OutputAdapter` routing per target type with fallback chain and delivery
  receipts.
- Clipboard adapter that works from a focus-less background daemon on GNOME
  Wayland (see Q-003).
- Feedback: start/stop sounds, notifications for fallback and errors, all
  individually configurable.
- In-memory last-transcript state; `dictate copy-last` / `repeat-last`.

## Out of Scope

- WezTerm adapter internals (07-wezterm) and IBus adapter (09-ibus) — they
  implement the trait this module routes through; text formatting (04-text).

## Interfaces

**Depends on:**

- core — session flow, `TargetSnapshot`, `DeliveryReceipt`, IPC.
- text — formatted transcript input.

**Exposes:**

- Router invoked at `Delivering`; adapter registration for WezTerm/IBus.
- Last-transcript retrieval over IPC.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### DLV-001: Delivery router

- **Intent:** Each transcript reaches exactly one correct destination via
  the best available adapter.
- **Expected Outcome:** Routing follows spec §14 defaults; only adapters
  matching the captured target are attempted; failed attempts are recorded in
  the receipt; no duplicate delivery; delivery never retargets to the
  currently focused app when it differs from the snapshot.
- **Validation:** `cargo test -p dictate-output router`
- **Confidence:** high

### DLV-002: Clipboard adapter

- **Intent:** The clipboard works as a universal, focus-less delivery path.
- **Expected Outcome:** Plain-text Unicode transcripts land on the clipboard
  from the background daemon in a GNOME Wayland session; failed transcription
  never overwrites the clipboard; success ends the session Completed with
  `fallback_used` set when it was a fallback.
- **Validation:** `cargo test -p dictate-output-clipboard` (plus manual
  round-trip check on a desktop session; see Q-003)
- **Dependencies:** DLV-001
- **Confidence:** medium

### DLV-003: Feedback (sounds and notifications)

- **Intent:** The user always knows the session state without watching a
  window.
- **Expected Outcome:** Start/stop sounds play when enabled; clipboard
  fallback and errors raise notifications; direct-success notification is off
  by default; every channel is individually configurable per spec §8.4.
- **Validation:** `cargo test -p dictate-notifications`
- **Dependencies:** DLV-001
- **Confidence:** medium

### DLV-004: Last-transcript recovery

- **Intent:** A transcript survives delivery failure for the daemon's
  lifetime.
- **Expected Outcome:** Latest raw/formatted transcript and delivery result
  are held in memory only; `dictate copy-last` re-copies, `dictate
  repeat-last` re-delivers to a fresh target; state clears on daemon stop;
  nothing is written to disk.
- **Validation:** `cargo test -p dictate-daemon last_transcript`
- **Dependencies:** DLV-001, DLV-002
- **Confidence:** high

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
