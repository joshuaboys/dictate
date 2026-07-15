# Spikes — platform validation before foundation work

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| SPK | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §27 (risks R1, R2, R4), §29 Phase 0

## Purpose

Convert the plan's three biggest platform unknowns into recorded facts
before foundation work bakes in assumptions. Each spike is a half-day
throwaway experiment on a real Ubuntu GNOME Wayland session; the deliverable
is a findings note and an answered question or confirmed decision — never
production code.

## In Scope

- Throwaway scripts/prototypes (kept under `docs/spikes/` with their
  findings; not part of the workspace build).
- Answering or re-scoping Q-003, and validating the assumptions behind
  ADR-002 (portal press/release) and ADR-004/R4 (WezTerm send-text bytes).
- Feeding results back into the spec, ADRs and issues tracker.

## Out of Scope

- Production crates or reusable abstractions — those belong to the real
  modules once the facts are in.

## Interfaces

**Depends on:**

- Nothing — runs before and alongside CORE-001.

**Exposes:**

- Findings notes consumed by 05-delivery (Q-003), 06-hotkeys (R1) and
  07-wezterm (R4).

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### SPK-001: Focus-less clipboard write on stock Ubuntu GNOME

- **Intent:** Prove a background daemon can set the clipboard on current
  Ubuntu LTS GNOME Wayland — MVP 1's delivery path depends on it.
- **Expected Outcome:** A minimal headless process writes Unicode text to
  the clipboard via a data-control-capable backend and it pastes into a GUI
  app; findings (backend choice, GNOME versions checked, failure modes)
  recorded and Q-003 answered or re-scoped.
- **Validation:** `test -f docs/spikes/SPK-001-clipboard.md` and Q-003
  updated in `plans/issues.md`
- **Confidence:** medium

### SPK-002: Global Shortcuts portal press/release reliability

- **Intent:** Confirm the portal delivers usable Activated/Deactivated pairs
  for push-to-talk before MVP 2 is designed around it.
- **Expected Outcome:** A prototype registers a shortcut and logs
  press/release events; findings cover consent-dialog behaviour,
  re-registration across restarts, missed-release frequency under fast
  tapping and session lock; ADR-002 confirmed or amended.
- **Validation:** `test -f docs/spikes/SPK-002-portal.md`
- **Confidence:** medium

### SPK-003: WezTerm send-text byte-level behaviour

- **Intent:** Establish exactly what bytes reach a pane via the WezTerm CLI
  so the never-Enter and bracketed-paste invariants are grounded in fact.
- **Expected Outcome:** Findings document observed bytes for text with and
  without embedded newlines, paste vs no-paste modes, and behaviour into
  local, mux and tmux-hosting panes on the current WezTerm release; risk R4
  and the 07-wezterm work items updated if reality differs.
- **Validation:** `test -f docs/spikes/SPK-003-wezterm.md`
- **Confidence:** medium

## Execution

Spikes are self-contained; no action plans needed.
