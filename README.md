# dictate

Local-first speech-to-text dictation for Ubuntu desktops running GNOME on
Wayland.

Hold a global shortcut, speak, release — the transcript appears in the
application that was focused when you started, with first-class delivery
into WezTerm panes (local, mux and SSH domains) and clipboard fallback
everywhere else. Transcription runs locally via a warm `whisper.cpp` model;
nothing is persisted and nothing leaves the machine by default, and the
terminal adapter never sends Enter.

**Status:** planning — no code yet.

## Documents

- [Product and technical specification](docs/spec.md)
- [Delivery plan (APS root index)](plans/index.aps.md)
- [Architecture decisions](plans/decisions/)
- [Open questions](plans/issues.md)

Planning uses [APS (Anvil Plan Spec)](plans/aps-rules.md): the spec defines
*what and why*; `plans/modules/` holds the executable work items.
