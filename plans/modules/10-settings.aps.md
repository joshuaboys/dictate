# Settings — GTK4/libadwaita configuration application (MVP 3)

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| SET | @joshuaboys | low      | Draft  |

Spec: [docs/spec.md](../../docs/spec.md) §17

## Purpose

Give non-CLI users a native GNOME settings application covering everything in
`config.toml`: modes, shortcuts, audio devices, model management, delivery
preferences, vocabulary, privacy toggles and diagnostics.

## In Scope

- GTK4/libadwaita app (`dictate-settings`) editing daemon configuration over
  IPC.
- Sections per spec §17: General, Shortcuts, Audio (with live level test),
  Speech model (download/remove/warm-up), Delivery, Vocabulary, Privacy,
  Diagnostics.

## Out of Scope

- Daemon behaviour itself (settings only reads/writes config and invokes
  existing IPC); a GNOME Shell extension indicator (future).

## Interfaces

**Depends on:**

- core — config schema and IPC.
- stt — model listing/installation.
- audio — device enumeration and level monitoring.
- delivery — notification preferences and adapter status.

**Exposes:**

- Desktop settings UI; no APIs consumed by other modules.

## Ready Checklist

Change status to **Ready** when:

- [ ] MVP 2 is accepted and MVP 3 planning starts
- [ ] Config read/write over IPC (vs direct file edit + reload) is decided
- [ ] At least one work item defined

## Work Items

_No work items yet — module is Draft. Defined when MVP 3 planning starts._

## Execution

Action plan: [../execution/SET.actions.md](../execution/SET.actions.md)
(when Ready)
