# Core — daemon, session state machine, IPC, CLI, configuration

| ID   | Owner       | Priority | Status |
| ---- | ----------- | -------- | ------ |
| CORE | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §10.1, §10.2, §13, §15, §16

## Purpose

Provide the orchestration backbone of `dictate`: the persistent user daemon
(`dictated`) that owns the single dictation session state machine, a
user-scoped IPC surface, the `dictate` CLI, and configuration loading with
safe-default validation. Every other module plugs into this one.

## In Scope

- Rust workspace scaffolding and shared domain types (session, transcript,
  receipts, adapter traits).
- Daemon lifecycle: startup, single-instance enforcement, clean shutdown,
  subsystem supervision.
- Session state machine (spec §13.2) with single-active-session rule.
- IPC (D-Bus user session service) for control, status and signals.
- CLI commands: start, stop, toggle, cancel, status (`--json`), config.
- Configuration schema, loading, validation with safe defaults.
- Structured logging that excludes transcript content.

## Out of Scope

- Audio capture (02-audio), transcription (03-stt), text transformation
  (04-text), delivery adapters (05-delivery), shortcut registration
  (06-hotkeys), packaging (08-packaging).

## Interfaces

**Depends on:**

- Nothing (foundation module).

**Exposes:**

- Domain types and traits (`SessionState`, `TargetSnapshot`, `Transcript`,
  `OutputAdapter`, `SpeechEngine`) consumed by all other modules.
- IPC methods and signals consumed by the CLI, hotkeys, WezTerm bridge and
  (later) settings app.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### CORE-001: Rust workspace scaffold

- **Intent:** A buildable multi-crate workspace exists matching the spec's
  architecture so all later work has a home.
- **Expected Outcome:** `cargo build --workspace` and `cargo test --workspace`
  succeed on a clean checkout; core, daemon, cli, ipc and config crates exist
  with CI-friendly lint (fmt + clippy) passing.
- **Validation:** `cargo fmt --check && cargo clippy --workspace -- -D warnings && cargo test --workspace`
- **Confidence:** high

### CORE-002: Session state machine

- **Intent:** Dictation session lifecycle is modelled explicitly so invalid
  transitions are impossible and observable.
- **Expected Outcome:** All transitions in spec §13.2 are implemented; every
  undefined state/event pair is rejected without panicking; only one session
  can be in a non-terminal state; tests cover every row of the transition
  table plus rejection cases.
- **Validation:** `cargo test -p dictate-core state_machine`
- **Confidence:** high

### CORE-003: Daemon lifecycle

- **Intent:** `dictated` runs as a reliable long-lived user process.
- **Expected Outcome:** Daemon starts, refuses to start a second instance
  with a clear message, shuts down cleanly on SIGTERM, and logs state
  transitions without transcript content.
- **Validation:** `cargo test -p dictate-daemon lifecycle`
- **Dependencies:** CORE-001
- **Confidence:** high

### CORE-004: IPC control surface

- **Intent:** Local, user-only IPC lets clients control and observe the
  daemon.
- **Expected Outcome:** Start/Stop/Toggle/Cancel/GetStatus/GetLastTranscript
  work over the user session bus; state-change signals are emitted; requests
  from other users are rejected; malformed and oversized messages are
  rejected without crashing.
- **Validation:** `cargo test -p dictate-ipc`
- **Dependencies:** CORE-002, CORE-003
- **Confidence:** medium

### CORE-005: CLI

- **Intent:** `dictate` gives users and scripts full control of the daemon.
- **Expected Outcome:** `dictate start|stop|toggle|cancel|status` behave per
  spec §10.2 (correct errors when no/duplicate session; `status --json`
  emits stable structured output); all failures exit non-zero with an
  actionable message.
- **Validation:** `cargo test -p dictate-cli`
- **Dependencies:** CORE-004
- **Confidence:** high

### CORE-006: Configuration loading and validation

- **Intent:** User configuration is honoured without ever preventing daemon
  startup.
- **Expected Outcome:** `~/.config/dictate/config.toml` per spec §16 is
  loaded; invalid values fall back to safe defaults with reported validation
  errors; unknown keys warn rather than fail; `dictate config validate`
  surfaces problems.
- **Validation:** `cargo test -p dictate-config`
- **Dependencies:** CORE-001
- **Confidence:** high

### CORE-007: Content-safe structured logging

- **Intent:** Diagnostics are useful while dictated content stays private.
- **Expected Outcome:** Logs carry state transitions, session IDs, timings
  and error categories; a test proves transcript text and audio never appear
  at normal log levels; debug content logging requires an explicit opt-in.
- **Validation:** `cargo test -p dictate-core logging_privacy`
- **Dependencies:** CORE-003
- **Confidence:** high

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
