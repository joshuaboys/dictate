# ADR-004: Terminal safety defaults — never Enter, multiline off, bracketed paste on

| Field  | Value      |
| ------ | ---------- |
| Status | Accepted   |
| Date   | 2026-07-15 |

## Context

Delivering dictated text into a shell is the product's riskiest action: a
newline submits, and mis-transcribed speech could execute a destructive
command. Escape sequences embedded in text can also corrupt or manipulate a
terminal.

## Decision

- The terminal adapter **never** sends Enter or appends a trailing newline;
  there is no configuration that changes this in the initial product.
- Bracketed paste is used whenever supported, so embedded line feeds remain
  editable content instead of submissions.
- Multiline delivery is **disabled by default** (`allow_multiline = false`);
  multiline transcripts fall back to clipboard unless the user opts in.
- The safety filter (null bytes, control characters, escape sequences) runs
  unconditionally in every mode, including raw mode.
- Security tests (spec §23.5) assert these invariants on the exact bytes
  handed to the terminal.

## Consequences

- Users must press Enter themselves — a deliberate friction that keeps
  transcribed commands editable and reviewable.
- Some multiline dictations take the clipboard path by default; opting in is
  a conscious, per-user choice.
- Any future "dictate and execute" feature requires a separate, explicit
  action and its own ADR.
