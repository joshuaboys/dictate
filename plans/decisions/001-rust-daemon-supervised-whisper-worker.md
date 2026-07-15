# ADR-001: Rust daemon with a supervised whisper.cpp worker subprocess

| Field  | Value      |
| ------ | ---------- |
| Status | Accepted   |
| Date   | 2026-07-15 |

## Context

Dictation must feel like typing: the speech model has to stay warm between
utterances, so transcription lives in a long-running process. whisper.cpp is
native C/C++ code; a crash in it must not take down the daemon that owns
session state, shortcuts and IPC. The daemon itself needs to be a reliable,
low-overhead user service.

## Decision

- Implement `dictated` and `dictate` in Rust, as a multi-crate workspace.
- Run whisper.cpp in a separate worker process supervised by the daemon
  (bounded restarts, health checks), communicating over stdio/Unix socket.
- Keep the model loaded in the worker for the daemon's lifetime
  (`keep_model_loaded = true` default).
- Put the engine behind a `SpeechEngine` trait so other engines can be added
  without daemon changes.

## Consequences

- A native crash costs at most one utterance (single retry policy), never
  the daemon.
- Worker restart re-pays model load once; acceptable and observable via
  metrics.
- Slightly more plumbing (worker protocol — see Q-001) than in-process
  bindings, but the isolation and independent restartability justify it.
- Q-006 (separate *audio* process) remains open; this ADR covers only the
  speech worker.
