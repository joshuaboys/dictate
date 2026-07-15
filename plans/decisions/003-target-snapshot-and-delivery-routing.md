# ADR-003: Target snapshot at recording start; clipboard fallback is success; WezTerm before IBus

| Field  | Value      |
| ------ | ---------- |
| Status | Accepted   |
| Date   | 2026-07-15 |

## Context

Transcription takes seconds, during which the user may switch windows. GNOME
Wayland offers no general "which window is focused" API to normal clients, so
universal direct insertion is impossible; delivery capability varies by
target. A transcript that lands in the wrong place is worse than one that
lands on the clipboard.

## Decision

- Capture an immutable `TargetSnapshot` when recording **starts**; delivery
  goes only to that target, never to whatever is focused later.
- Route by target type (spec §14); the clipboard is the universal terminus
  of every chain, and a successful clipboard fallback is a **Completed**
  session (`fallback_used = true`), not a failure.
- Build the WezTerm adapter (pane-addressed, works today via the WezTerm
  CLI) before the IBus adapter; IBus is MVP 3 and never required for
  terminal use.
- If a captured target no longer exists, fall back to clipboard — never
  retarget (e.g. never a different pane).

## Consequences

- Predictability: users learn "text goes where I started dictating".
- UX and copy must present clipboard fallback as normal (notification, not
  error).
- Target detection quality depends on integrations (WezTerm plugin, IBus
  context); unknown targets simply mean clipboard.
- A failed transcription never touches the clipboard, so fallback never
  destroys clipboard contents for nothing.
