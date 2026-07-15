# STT — whisper.cpp worker and model lifecycle

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| STT | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §10.4 · ADR-001

## Purpose

Turn captured audio into transcripts locally, fast. A supervised
`whisper.cpp` worker keeps the model warm between sessions so dictation feels
like typing, and a crash in native code never takes the daemon down.

## In Scope

- `SpeechEngine` trait implementation backed by whisper.cpp.
- Supervised worker subprocess: spawn, health checks, bounded restarts.
- Warm model lifecycle (loaded at daemon start, kept resident).
- Model management: list, download with checksum, remove (`dictate models`).
- Language selection, auto-detection option, vocabulary/prompt hints.
- `dictate transcribe-file` for debugging and benchmarking.

## Out of Scope

- Audio capture (02-audio); transcript formatting (04-text); model management
  UI (10-settings); alternative/cloud engines (future).

## Interfaces

**Depends on:**

- core — supervision hooks, config, session events.
- audio — `AudioBuffer` input.

**Exposes:**

- `SpeechEngine::transcribe` returning `Transcript` (raw text, language,
  confidence, timing).
- Engine health for `dictate status` / `dictate doctor`.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### STT-001: Supervised speech worker

- **Intent:** The speech engine runs isolated from the daemon and recovers
  from crashes on its own.
- **Expected Outcome:** Worker starts under daemon supervision, answers
  health checks, restarts after a kill without daemon restart, and restart
  attempts are rate-limited; an in-flight transcription that dies is retried
  once, then reported as failed.
- **Validation:** `cargo test -p dictate-stt supervision`
- **Confidence:** medium

### STT-002: whisper.cpp transcription

- **Intent:** Audio buffers become accurate transcripts using a warm local
  model.
- **Expected Outcome:** A known audio fixture transcribes to expected text;
  the model loads once at startup and is reused across sessions (verified by
  timing: second transcription has no load cost); failures carry useful
  detail.
- **Validation:** `cargo test -p dictate-stt-whispercpp transcribe -- --ignored`
  (fixture-based; needs model download)
- **Dependencies:** STT-001
- **Confidence:** medium

### STT-003: Model management

- **Intent:** Users can install, inspect and remove speech models safely.
- **Expected Outcome:** `dictate models` lists installed and available
  models; install downloads to the XDG data dir with checksum verification;
  a missing configured model yields an actionable error and never a silent
  substitute.
- **Validation:** `cargo test -p dictate-stt models`
- **Dependencies:** STT-002
- **Confidence:** high

### STT-004: Transcription options

- **Intent:** Language and vocabulary behaviour is user-controllable.
- **Expected Outcome:** Configured language is honoured; auto-detect mode
  populates `detected_language`; initial prompt/vocabulary hints are passed
  through where the engine supports them.
- **Validation:** `cargo test -p dictate-stt options`
- **Dependencies:** STT-002
- **Confidence:** medium

### STT-005: File transcription command

- **Intent:** A deterministic offline path exists for debugging, benchmarks
  and regression fixtures.
- **Expected Outcome:** `dictate transcribe-file <path>` prints the
  transcript to stdout using the full engine path and delivers nowhere;
  non-audio input fails with a clear error.
- **Validation:** `cargo test -p dictate-cli transcribe_file`
- **Dependencies:** STT-002
- **Confidence:** high

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
