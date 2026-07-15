# Packaging — systemd user service, doctor, Debian package, docs

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| PKG | @joshuaboys | medium   | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §10.2 (doctor), §21, §22 · ADR-005

## Purpose

Make `dictate` installable and diagnosable by someone who didn't build it: a
Debian package that installs the binaries, a systemd user service that keeps
the daemon alive across the session, `dictate doctor` to explain what's
wrong, and enough documentation to go from install to first dictation.

## In Scope

- systemd user unit: restart-on-failure with rate limiting, ordered after
  the graphical session, access to user PipeWire and D-Bus.
- `dictate doctor`: every check in spec §10.2 with pass/warn/fail and a
  remediation hint; `--json` output.
- Debian package: binaries, unit, desktop metadata, icons, WezTerm plugin,
  assets. Models download on demand — never bundled.
- Install/quickstart documentation.

## Out of Scope

- Flatpak/AppImage (deferred; Q-008); IBus packaging split (Q-007); release
  automation (later).

## Interfaces

**Depends on:**

- core — binaries and IPC health for doctor and the service.
- All modules — doctor surfaces their status checks.

**Exposes:**

- `dictated.service`, the `.deb`, and the doctor diagnostics users run
  first.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### PKG-001: systemd user service

- **Intent:** The daemon is always there when the desktop is.
- **Expected Outcome:** `systemctl --user enable --now dictated.service`
  starts the daemon after the graphical session with PipeWire/D-Bus access;
  it restarts on failure without tight loops; stop/disable behave cleanly.
- **Validation:** scripted service lifecycle check on a desktop session
  (`integrations/systemd/test-service.sh`)
- **Confidence:** high

### PKG-002: dictate doctor

- **Intent:** One command explains why dictation isn't working.
- **Expected Outcome:** Doctor runs every spec §10.2 check, each with
  pass/warn/fail and a remediation hint; `--json` emits stable structured
  output; failures caused by missing optional integrations warn rather than
  fail.
- **Validation:** `cargo test -p dictate-cli doctor`
- **Confidence:** high

### PKG-003: Debian package

- **Intent:** Install is one `apt install`, not a build from source.
- **Expected Outcome:** The `.deb` installs binaries, user unit, desktop
  metadata, icons and the WezTerm plugin on current Ubuntu LTS; lintian is
  clean or has documented overrides; purge removes what install added; no
  models are bundled.
- **Validation:** `packaging/deb/build-and-verify.sh` (build, lintian,
  install/purge in a container)
- **Dependencies:** PKG-001
- **Confidence:** medium

### PKG-004: Install and quickstart docs

- **Intent:** A new user reaches first successful dictation unaided.
- **Expected Outcome:** Docs cover install, service enablement, GNOME
  shortcut setup, model download, WezTerm plugin setup and troubleshooting
  via doctor; README links them.
- **Validation:** docs walkthrough performed on a clean Ubuntu VM (checklist
  in the doc)
- **Dependencies:** PKG-002, PKG-003
- **Confidence:** high

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
