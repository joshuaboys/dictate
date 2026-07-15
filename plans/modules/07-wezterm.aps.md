# WezTerm — pane targeting, safe pane delivery, Lua plugin

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| WEZ | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §10.5, §10.6, §23.3 · ADR-003, ADR-004

## Purpose

Make WezTerm the flagship integration: capture the focused pane when
recording starts and deliver the transcript to that exact pane — local, mux
or SSH-domain — as editable input that is never executed. This is the module
that makes `dictate` a terminal daily driver.

## In Scope

- WezTerm target detection and pane snapshot (`pane_id`, workspace, client).
- Delivery via the WezTerm CLI with bracketed paste, no Enter, no trailing
  newline.
- Pane-existence verification before delivery; clipboard fallback when the
  pane is gone — never a different pane.
- Lua plugin: focus/pane reporting to the daemon, diagnostics, optional
  WezTerm-local bindings (input treated as untrusted; see Q-005).
- Multiline policy enforcement (off by default) at the point of delivery.
- The spec §23.3 scenario matrix (tmux/zellij inside panes, mux server, SSH
  domains, alternate screen, multiple windows/clients).

## Out of Scope

- Terminal-mode text formatting (04-text); generic routing (05-delivery);
  other terminals (future).

## Interfaces

**Depends on:**

- delivery — adapter registration and fallback chain.
- text — terminal-mode formatted, safety-filtered text.
- core — `TargetSnapshot::WezTerm`, IPC for plugin reports.

**Exposes:**

- WezTerm `OutputAdapter` and target provider.
- Adapter/CLI/mux status for `dictate doctor`.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### WEZ-001: Pane target capture

- **Intent:** Recording start pins the exact pane the transcript belongs to.
- **Expected Outcome:** With WezTerm focused, the snapshot holds the focused
  pane's ID (plus workspace/client where available) within the 100 ms budget;
  the snapshot is immutable for the session; focus changes afterwards are
  ignored.
- **Validation:** `cargo test -p dictate-output-wezterm target_capture`
- **Confidence:** medium

### WEZ-002: Safe pane delivery

- **Intent:** Transcripts arrive in the pane as editable input, never as an
  executed command.
- **Expected Outcome:** Text reaches the captured pane via the WezTerm CLI
  with bracketed paste where available; no Enter or trailing newline is ever
  sent (asserted by tests on the exact bytes handed to WezTerm); multiline
  input follows the configured policy, defaulting to clipboard fallback.
- **Validation:** `cargo test -p dictate-output-wezterm delivery_safety`
- **Dependencies:** WEZ-001
- **Confidence:** medium

### WEZ-003: Pane-gone fallback

- **Intent:** A vanished target can't misdirect or lose a transcript.
- **Expected Outcome:** Pane existence is verified immediately before
  delivery; a missing pane triggers clipboard fallback with a notification;
  the transcript is never sent to any other pane.
- **Validation:** `cargo test -p dictate-output-wezterm pane_gone`
- **Dependencies:** WEZ-002
- **Confidence:** high

### WEZ-004: Lua plugin

- **Intent:** WezTerm itself tells the daemon which pane matters.
- **Expected Outcome:** `integrations/wezterm/dictate.lua` loads from a user
  config, reports focused pane/window changes (or answers queries — Q-005) to
  the daemon, and exposes a diagnostics hook; the daemon validates all plugin
  input; plugin absence degrades to CLI-based detection.
- **Validation:** plugin smoke script under `integrations/wezterm/` plus
  `cargo test -p dictate-output-wezterm plugin_bridge`
- **Dependencies:** WEZ-001
- **Confidence:** low

### WEZ-005: Mux and remote domains

- **Intent:** Panes behind a mux server or SSH domain are first-class
  targets.
- **Expected Outcome:** Delivery succeeds to mux-attached and SSH-domain
  panes, and into panes running tmux/zellij (the pane consumes the paste);
  unreachable domains fall back to clipboard with a clear notification.
- **Validation:** scripted scenario matrix (spec §23.3) recorded in
  `docs/testing/wezterm-matrix.md`
- **Dependencies:** WEZ-002, WEZ-003
- **Confidence:** low

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
