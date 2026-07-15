# Audio — PipeWire capture, devices, VAD

| ID  | Owner       | Priority | Status |
| --- | ----------- | -------- | ------ |
| AUD | @joshuaboys | high     | Ready  |

Spec: [docs/spec.md](../../docs/spec.md) §10.3

## Purpose

Capture microphone audio through PipeWire and hand the speech subsystem a
clean, correctly-sampled, in-memory buffer — with device selection, loss
detection, and duration limits enforced.

## In Scope

- PipeWire input-device enumeration and explicit selection.
- Mono PCM capture, resampled to the engine's required rate (16 kHz).
- Configurable noise gate and voice activity detection (neural VAD with an
  amplitude-threshold fallback — see spec §10.3; Silero/ONNX is the
  field-proven candidate).
- Minimum/maximum recording duration enforcement.
- Device-loss detection during capture.
- In-memory-only buffers by default.

## Out of Scope

- Transcription (03-stt); session orchestration (01-core); persistence of
  audio (explicitly excluded by ADR-006).

## Interfaces

**Depends on:**

- core — session events (start/stop/cancel) and configuration.

**Exposes:**

- Device listing for `dictate devices` and doctor checks.
- `AudioBuffer` consumed by the speech engine.

## Ready Checklist

- [x] Purpose and scope are clear
- [x] Dependencies identified
- [x] At least one work item defined

## Work Items

### AUD-001: Device enumeration and selection

- **Intent:** Users can see and choose their microphone.
- **Expected Outcome:** `dictate devices` lists PipeWire input devices with a
  default marked; configured device is used for capture; a missing configured
  device produces an actionable error, not a silent switch.
- **Validation:** `cargo test -p dictate-audio-pipewire devices` (plus manual
  `dictate devices` on a desktop session)
- **Confidence:** medium

### AUD-002: Capture pipeline

- **Intent:** Recording produces engine-ready audio without touching disk.
- **Expected Outcome:** Start/stop yields a mono 16 kHz in-memory buffer;
  stop and cancel tear the stream down cleanly; a test asserts no files are
  created during capture.
- **Validation:** `cargo test -p dictate-audio capture`
- **Dependencies:** AUD-001
- **Confidence:** medium

### AUD-003: Duration limits, noise gate and VAD

- **Intent:** Accidental taps and runaway recordings are handled
  automatically.
- **Expected Outcome:** Recordings under `minimum_recording_ms` are
  discarded; recording stops at `maximum_recording_seconds` with the user
  notified; noise gate and VAD are configurable and bypassable.
- **Validation:** `cargo test -p dictate-audio limits_vad`
- **Dependencies:** AUD-002
- **Confidence:** medium

### AUD-004: Device-loss resilience

- **Intent:** Unplugging the mic mid-recording fails safely.
- **Expected Outcome:** Device disappearance during capture ends the session
  in Failed with a clear notification; captured-so-far audio is offered for
  transcription when above minimum duration; daemon remains healthy.
- **Validation:** `cargo test -p dictate-audio device_loss`
- **Dependencies:** AUD-002
- **Confidence:** low

## Execution

Action plans in [../execution/](../execution/) as work items are picked up.
