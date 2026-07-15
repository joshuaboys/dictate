# ADR-006: Privacy defaults — nothing persisted, no telemetry, no network

| Field  | Value      |
| ------ | ---------- |
| Status | Accepted   |
| Date   | 2026-07-15 |

## Context

Dictation captures everything a user says at their desk, potentially
including secrets spoken into a terminal. Local-first privacy is the
product's core differentiator (spec §3.1) and must be the default posture,
not an option buried in settings.

## Decision

By default:

- Audio is processed in memory and never written to disk.
- Transcripts are not persisted; only the last transcript is held in daemon
  memory and cleared on stop.
- No telemetry or crash reports are sent; no cloud API is contacted; no
  inbound port is opened.
- Logs never contain transcript text or audio at normal levels; debug
  content logging is an explicit, clearly-labelled opt-in.
- Any future persistence (history), cloud engine, or telemetry ships
  disabled and opt-in.

Tests enforce the posture: no files created during capture, no transcript
strings in default logs (spec §23.5).

## Consequences

- Users lose dictation history by default — an accepted trade-off;
  `dictate copy-last` covers the common recovery case.
- Debugging user issues relies on structured metadata and `dictate doctor`
  rather than content logs.
- Every feature adding persistence or network use needs an explicit
  opt-in path and privacy review against this ADR.
