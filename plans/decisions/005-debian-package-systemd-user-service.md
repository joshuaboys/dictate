# ADR-005: Debian package + systemd user service as the first distribution format

| Field  | Value      |
| ------ | ---------- |
| Status | Accepted   |
| Date   | 2026-07-15 |

## Context

The target platform is Ubuntu GNOME. The daemon must start with the user
session, restart on failure, and reach the user's PipeWire and D-Bus
sessions. Flatpak's sandbox complicates exactly the things dictate needs
most (WezTerm CLI on the host, host sockets, IBus components), while a
native package makes them trivial.

## Decision

- Ship a Debian package first, installing `dictated`, `dictate`, the
  systemd **user** unit, desktop metadata, icons and the WezTerm Lua plugin.
- Run the daemon via `systemctl --user`, ordered after
  `graphical-session.target`, with rate-limited restart-on-failure.
- Never bundle speech models; download on demand to the XDG data directory
  with checksum verification.
- Defer Flatpak/AppImage until host-integration questions are validated
  (Q-008); Flatpak must not block the deb.

## Consequences

- Fastest path to the target audience (Ubuntu developers), full host access
  for the WezTerm and IBus integrations.
- We own security updates for the packaged binaries; no store distribution
  yet.
- Package layout must anticipate an eventual IBus split (Q-007) and keep the
  base package model-free and small.
