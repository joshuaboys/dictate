# IBus — direct insertion into desktop applications (MVP 3)

| ID   | Owner       | Priority | Status |
| ---- | ----------- | -------- | ------ |
| IBUS | @joshuaboys | low      | Draft  |

Spec: [docs/spec.md](../../docs/spec.md) §10.7

## Purpose

Commit completed transcripts directly into focused desktop applications
through IBus, replacing clipboard fallback with true insertion for apps that
support input methods — without ever taking ownership of normal keyboard
input.

## In Scope

- An IBus engine that passes physical keyboard input through untouched.
- Receiving transcripts from `dictated` and committing them to the focused
  input context.
- `TargetSnapshot::IBus` capture and safe fallback to clipboard when the
  context is unavailable.
- Pre-edit (in-progress transcript) display in a later iteration.

## Out of Scope

- WezTerm delivery (07-wezterm is preferred for WezTerm targets even when
  IBus is enabled); synthetic keyboard injection of any kind.

## Interfaces

**Depends on:**

- delivery — adapter registration and routing.
- core — IPC between engine component and daemon.

**Exposes:**

- IBus `OutputAdapter`; IBus status for `dictate doctor`.

## Ready Checklist

Change status to **Ready** when:

- [ ] MVP 2 (WezTerm daily driver) is accepted
- [ ] Q-007 (separate IBus package?) is answered
- [ ] Engine/daemon IPC design is sketched in a design doc
- [ ] At least one work item defined

## Work Items

_No work items yet — module is Draft. Defined when MVP 3 planning starts._

## Execution

Action plan: [../execution/IBUS.actions.md](../execution/IBUS.actions.md)
(when Ready)
