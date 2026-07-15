<!--
APS Index — non-executable. Do not create tasks here.
Focus on intent, scope, modules, risks. See: plans/aps-rules.md
-->

# dictate — Local speech-to-text for GNOME Wayland

| Field   | Value       |
| ------- | ----------- |
| Status  | Ready       |
| Owner   | @joshuaboys |
| Created | 2026-07-15  |
| Spec    | [docs/spec.md](../docs/spec.md) |

## Problem

Speech-to-text on Linux is fragmented and largely broken on Wayland: existing
tools rely on X11 keyboard injection, only transcribe into their own window,
lose the terminal pane that was focused, or ship audio to the cloud. There is
no fast, private, terminal-aware dictation tool for GNOME Wayland — and none
that can safely deliver text into WezTerm panes (local, mux or remote)
without risking accidental command execution.

`dictate` is a local-first dictation daemon: global shortcut → PipeWire
capture → local whisper.cpp transcription → delivery to the application that
was focused when recording began, with clipboard fallback as a first-class
outcome. Full detail: [docs/spec.md](../docs/spec.md).

## Success Criteria

- [ ] **MVP 1** — In a GNOME Wayland session: toggle shortcut starts/stops
      recording, audio is captured via PipeWire, transcribed locally by a
      warm whisper.cpp worker, and the transcript lands on the clipboard with
      a notification. Nothing is persisted by default. (Spec §24.1, §25.1)
- [ ] **MVP 2** — Push-to-talk via the XDG Global Shortcuts portal delivers
      the transcript into the WezTerm pane focused at recording start —
      including mux/tmux panes — without ever sending Enter; a closed pane
      falls back to clipboard, never another pane. (Spec §24.2, §25.2)
- [ ] **MVP 3** — IBus commits transcripts into supported desktop apps; GTK
      settings app manages models, microphones, modes and vocabulary.
      (Spec §24.3)
- [ ] Performance: warm model between sessions, feedback < 200 ms after
      shortcut, delivery < 250 ms after transcription, idle CPU near zero.
      (Spec §20, §25.3)
- [ ] Security invariants hold under test: no Enter sent, no escape-sequence
      injection, no transcript in logs, IPC user-scoped. (Spec §23.5)

## What's Next

Prioritised queue of ready work:

| #   | Work Item | Module                                | Owner       | Status |
| --- | --------- | ------------------------------------- | ----------- | ------ |
| 1   | CORE-001  | [core](./modules/01-core.aps.md)      | @joshuaboys | Ready  |
| 2   | CORE-002  | [core](./modules/01-core.aps.md)      | @joshuaboys | Ready  |
| 3   | CORE-003  | [core](./modules/01-core.aps.md)      | @joshuaboys | Ready  |
| 4   | CORE-004  | [core](./modules/01-core.aps.md)      | @joshuaboys | Ready  |
| 5   | CORE-005  | [core](./modules/01-core.aps.md)      | @joshuaboys | Ready  |
| 6   | AUD-001   | [audio](./modules/02-audio.aps.md)    | @joshuaboys | Ready  |

<!-- Update this table as work items complete or priorities shift -->

## Modules

| Module                                        | ID   | Purpose                                            | Status | Priority | Dependencies |
| --------------------------------------------- | ---- | -------------------------------------------------- | ------ | -------- | ------------ |
| [core](./modules/01-core.aps.md)              | CORE | Daemon, session state machine, IPC, CLI, config    | Ready  | high     | —            |
| [audio](./modules/02-audio.aps.md)            | AUD  | PipeWire capture, devices, VAD                     | Ready  | high     | core         |
| [stt](./modules/03-stt.aps.md)                | STT  | whisper.cpp worker, model lifecycle                | Ready  | high     | core, audio  |
| [text](./modules/04-text.aps.md)              | TXT  | Transformation pipeline, modes, safety filter      | Ready  | high     | core         |
| [delivery](./modules/05-delivery.aps.md)      | DLV  | Adapter routing, clipboard, feedback, history      | Ready  | high     | core, text   |
| [hotkeys](./modules/06-hotkeys.aps.md)        | HOT  | GNOME toggle fallback, portal push-to-talk         | Ready  | high     | core         |
| [wezterm](./modules/07-wezterm.aps.md)        | WEZ  | Pane snapshot, pane delivery, Lua plugin           | Ready  | high     | delivery, text |
| [packaging](./modules/08-packaging.aps.md)    | PKG  | systemd service, doctor, Debian package, docs      | Ready  | medium   | core         |
| [ibus](./modules/09-ibus.aps.md)              | IBUS | IBus engine for desktop-app insertion (MVP 3)      | Draft  | low      | delivery     |
| [settings](./modules/10-settings.aps.md)      | SET  | GTK4/libadwaita settings application (MVP 3)       | Draft  | low      | core, stt, delivery |

## Constraints

- Ubuntu current LTS+, GNOME on Wayland only; no X11-injection techniques,
  no `/dev/uinput`, no virtual-keyboard protocols (spec §3.2, §5).
- Local-first by default: no audio/transcript persistence, no telemetry, no
  network listeners (ADR-006).
- The terminal adapter must never send Enter; transcribed commands stay
  editable (ADR-004).
- Rust workspace; speech engine isolated in a supervised subprocess
  (ADR-001).
- Transcript content must never appear in logs at normal levels (spec §18.6).

## Milestones

### M1: Clipboard product (MVP 1)

- **Target:** end of Phase 3 (spec §29)
- **Includes:** core, audio, stt, text (prose/raw), delivery (clipboard),
  hotkeys (GNOME toggle), packaging (service + doctor + deb)

### M2: WezTerm daily driver (MVP 2)

- **Target:** end of Phase 5
- **Includes:** wezterm, text (terminal mode), hotkeys (portal PTT + overlay)

### M3: Desktop insertion (MVP 3)

- **Target:** end of Phase 6
- **Includes:** ibus, settings

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
| ---- | ------ | ---------- | ---------- |
| Portal push-to-talk unreliable across GNOME versions | high | medium | GNOME toggle fallback; max-duration backstop (spec R1) |
| Focus-less clipboard writes fail on some setups | high | low-med | data-control backend; doctor round-trip check (spec R2) |
| Whisper too slow on low-end CPUs | med | medium | quantised/smaller models; warm worker (spec R3) |
| `wezterm cli` drift across versions | high | medium | version check in doctor; scenario matrix; clipboard fallback (spec R4) |
| Transcript executed in terminal | high | low | never-Enter invariant + security tests (spec R6) |

Full register: spec §27.

## Decisions

- **D-001:** Rust daemon + supervised whisper.cpp worker subprocess —
  crash isolation without losing warm-model latency
  ([ADR-001](./decisions/001-rust-daemon-supervised-whisper-worker.md))
- **D-002:** Portal push-to-talk, GNOME custom-shortcut toggle fallback —
  only portal delivers press/release on Wayland
  ([ADR-002](./decisions/002-activation-portal-ptt-gnome-toggle-fallback.md))
- **D-003:** Snapshot target at recording start; clipboard fallback is
  success; WezTerm before IBus
  ([ADR-003](./decisions/003-target-snapshot-and-delivery-routing.md))
- **D-004:** Terminal safety defaults: never Enter, multiline off,
  bracketed paste on
  ([ADR-004](./decisions/004-terminal-safety-defaults.md))
- **D-005:** Debian package + systemd user service first; Flatpak deferred
  ([ADR-005](./decisions/005-debian-package-systemd-user-service.md))
- **D-006:** Privacy defaults: nothing persisted, no telemetry, no network
  ([ADR-006](./decisions/006-privacy-defaults.md))

## Open Questions

Tracked in [issues.md](./issues.md):

- [ ] Q-001 — Speech worker wire protocol (JSON Lines now; revisit)
- [ ] Q-002 — Default model choice (`small.en` vs multilingual `small`)
- [ ] Q-003 — Clipboard backend on GNOME Wayland
- [ ] Q-004 — Bundle whisper.cpp vs build-from-source vs download
- [ ] Q-005 — WezTerm plugin: push focus events vs query on demand
- [ ] Q-006 — Separate audio process for crash isolation?
- [ ] Q-007 — IBus components as a separate Debian package?
- [ ] Q-008 — Flatpak viability
