# Text — transformation pipeline, dictation modes, safety filter

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| TXT | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §9, §10.9 · ADR-004

## Purpose

Turn raw engine output into text that is correct for its destination: prose
for documents, raw for fidelity, terminal for shells — with an unconditional
safety filter so transcript content can never smuggle control characters or
escape sequences into a terminal.

## In Scope

- Staged pipeline: unicode/whitespace normalisation → spoken-token
  replacement → mode formatting → user dictionary → safety filter.
- Prose mode: capitalisation, punctuation, paragraph commands.
- Raw mode: minimal transformation.
- Terminal mode: spoken symbol tokens, no added punctuation, casing
  preserved.
- Safety filter: strip null bytes, control characters and escape sequences —
  in every mode.
- User-defined term replacements; per-target default mode selection.

## Out of Scope

- Delivery mechanics like bracketed paste and multiline policy enforcement
  (05-delivery / 07-wezterm consume this module's output); vocabulary UI
  (10-settings); LLM-based clean-up (future).

## Interfaces

**Depends on:**

- core — `Transcript` type, mode configuration.

**Exposes:**

- `format(raw, mode, target) -> formatted_text` used by the session flow
  before delivery.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### TXT-001: Pipeline framework

- **Intent:** Transformations are composable stages with observable
  behaviour.
- **Expected Outcome:** Stages run in the spec's order per mode; a stage
  removing more than whitespace is visible in debug logs (without content at
  normal levels); raw engine output is preserved alongside formatted text.
- **Validation:** `cargo test -p dictate-text pipeline`
- **Confidence:** high

### TXT-002: Safety filter

- **Intent:** No transcript can carry executable or terminal-corrupting
  bytes.
- **Expected Outcome:** Null bytes, C0/C1 control characters (except
  permitted LF/TAB in terminal mode) and ANSI escape sequences are removed in
  every mode including raw; adversarial fixtures (spec §23.5) pass.
- **Validation:** `cargo test -p dictate-text safety`
- **Dependencies:** TXT-001
- **Confidence:** high

### TXT-003: Prose mode

- **Intent:** Dictated prose reads like written text.
- **Expected Outcome:** Sentence capitalisation, spoken punctuation
  ("question mark", "full stop") and paragraph commands ("new paragraph")
  produce the spec §9.1 example output; whitespace is normalised.
- **Validation:** `cargo test -p dictate-text prose`
- **Dependencies:** TXT-001
- **Confidence:** high

### TXT-004: Terminal mode

- **Intent:** Dictated commands come out shell-ready but never shell-run.
- **Expected Outcome:** Spoken symbol tokens from spec §9.3 map to their
  characters; no sentence punctuation is added; casing is preserved; "new
  line" yields LF content only — output never ends with an
  execution-triggering newline added by the pipeline.
- **Validation:** `cargo test -p dictate-text terminal_mode`
- **Dependencies:** TXT-001, TXT-002
- **Confidence:** medium

### TXT-005: Raw mode

- **Intent:** Users who want engine-faithful output get it.
- **Expected Outcome:** Output matches engine text except for the safety
  filter; no reformatting or case changes are applied.
- **Validation:** `cargo test -p dictate-text raw_mode`
- **Dependencies:** TXT-001, TXT-002
- **Confidence:** high

### TXT-006: User dictionary and per-target modes

- **Intent:** Personal vocabulary and per-application defaults shape output
  without code changes.
- **Expected Outcome:** Configured term/phrase replacements apply in the
  documented stage order; `[application_modes]` selects the mode from the
  target snapshot with a default fallback.
- **Validation:** `cargo test -p dictate-text dictionary_modes`
- **Dependencies:** TXT-001
- **Confidence:** high

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
