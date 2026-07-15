# Project Context

> User-owned file. APS will never overwrite this.

## Overview

`dictate` is a local-first speech-to-text dictation tool for Ubuntu desktops
running GNOME on Wayland. A global shortcut records the microphone, a warm
local whisper.cpp model transcribes it, and the transcript is delivered to
the application that was focused when recording began — with first-class
support for WezTerm panes (local, mux and SSH domains) and clipboard
fallback as a normal outcome. It never sends Enter to a terminal and
persists nothing by default.

- Full specification: [docs/spec.md](../docs/spec.md)
- Root plan: [plans/index.aps.md](./index.aps.md)

## Team

- @joshuaboys — owner/solo developer (APS profile: solo).

## Tech Stack

- Rust (multi-crate workspace) for daemon (`dictated`), CLI (`dictate`) and
  adapters.
- whisper.cpp in a supervised worker subprocess (ADR-001).
- PipeWire (audio), D-Bus (IPC + XDG portals), systemd user services.
- WezTerm Lua for the terminal integration plugin; GTK4/libadwaita for the
  post-MVP settings app; IBus for post-MVP desktop insertion.
- Packaging: Debian package first (ADR-005).

## Conventions

- Planning follows APS: read `plans/aps-rules.md` first; work items are the
  execution authority; validate before marking complete.
- Module IDs: CORE, AUD, STT, TXT, DLV, HOT, WEZ, PKG, IBUS, SET.
- Rust hygiene: `cargo fmt --check` and `cargo clippy -- -D warnings` clean;
  tests accompany every work item.
- Never log transcript or audio content at normal log levels — this is a
  reviewable invariant, not a preference (ADR-006).
- Terminal delivery never sends Enter (ADR-004); treat it as
  non-configurable.

## Active Decisions

See [plans/decisions/](./decisions/) — ADR-001 through ADR-006 cover process
model, activation, delivery routing, terminal safety, packaging and privacy
defaults. Open unknowns live in [plans/issues.md](./issues.md) (Q-001…Q-008).
