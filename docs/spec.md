# dictate

## Product and Technical Specification

| Field                | Value                                                                  |
| -------------------- | ---------------------------------------------------------------------- |
| Status               | Draft for review                                                       |
| Version              | 0.2                                                                    |
| Owner                | @joshuaboys                                                            |
| Last updated         | 2026-07-15                                                             |
| Target platform      | Ubuntu (current LTS and later) with GNOME on Wayland                   |
| Implementation       | Rust                                                                   |
| Primary use case     | Local speech-to-text dictation into desktop applications and WezTerm   |
| Planning             | [plans/index.aps.md](../plans/index.aps.md) (APS)                      |
| Decisions            | [plans/decisions/](../plans/decisions/) (ADRs)                         |

### Changes in 0.2

- Corrected activation-layer platform facts: the XDG Global Shortcuts portal
  emits distinct `Activated`/`Deactivated` signals (enabling true
  push-to-talk), GNOME custom shortcuts are press-only (toggle mode only), and
  `Escape` cannot be globally grabbed on Wayland — cancel semantics redefined
  (§8.1, §8.5).
- Added an explicit session state machine with a transition table (§13.2).
- Added a glossary (§31), risk register (§27), and a traceability matrix
  mapping spec sections to APS modules (§32).
- Clarified clipboard backend constraints on GNOME Wayland (§10.8).
- Moved resolved design decisions into Architecture Decision Records and
  remaining unknowns into the APS issues tracker (§28).
- Added concurrency and single-instance rules (§10.1), a threat-model summary
  (§18), and licensing notes (§21.4).

---

# 1. Executive summary

`dictate` is a local-first speech-to-text application for Ubuntu desktops
running GNOME and Wayland.

The user activates dictation with a global keyboard shortcut, speaks, and
receives the transcribed text in the application that had focus when dictation
began.

The preferred delivery behaviour is:

1. Insert the transcript directly into the active application.
2. For WezTerm, send the transcript directly to the originally focused pane.
3. When direct insertion is unavailable, copy the transcript to the clipboard
   and notify the user.

The first supported high-quality integration is WezTerm, including panes
connected to local shells, remote hosts and multiplexed terminal sessions.

The product deliberately avoids unrestricted keyboard simulation and other
approaches that conflict with Wayland's security model.

---

# 2. Problem statement

Speech-to-text on Linux remains fragmented, particularly on Wayland desktops.

Existing solutions commonly suffer from one or more of the following
limitations:

- They only transcribe into their own application window.
- They depend on X11 keyboard injection (e.g. `xdotool`) that does not work
  reliably — or at all — on Wayland.
- They require users to manually copy and paste every transcript.
- They do not work correctly with terminal applications.
- They do not preserve the terminal pane that was active when recording began.
- They rely entirely on cloud transcription.
- They reload the speech recognition model for every utterance.
- They do not support push-to-talk interaction.
- They can accidentally execute transcribed terminal commands.

`dictate` provides a fast, private and predictable dictation experience
designed specifically for modern Linux desktop environments.

---

# 3. Product principles

## 3.1 Local-first

Audio and transcription remain on the user's machine by default. No audio,
transcript or usage information leaves the device unless the user explicitly
enables a remote transcription provider or telemetry.

## 3.2 Wayland-native

The product works with Wayland's security model rather than attempting to
bypass it. It uses supported desktop facilities:

- XDG Desktop Portals (Global Shortcuts, Notifications).
- GNOME custom keyboard shortcuts (fallback activation path).
- IBus (post-MVP direct insertion).
- Application-specific integration APIs (WezTerm CLI and Lua).
- The desktop clipboard.

## 3.3 Application-aware delivery

Text delivery uses the most reliable available adapter for the target
application:

- WezTerm pane delivery for WezTerm.
- IBus text commitment for supported desktop applications (post-MVP).
- Clipboard fallback when no direct delivery method is available.

## 3.4 Safe terminal behaviour

Transcribed terminal input must remain editable before execution. `dictate`
never automatically submits a terminal command by sending Enter. A separate,
explicit execution action may be considered in a future version, but it is out
of scope for the initial product.

## 3.5 Fast interaction

The speech model remains loaded while the daemon is running. A normal
dictation session should feel like a keyboard interaction, not like launching
a transcription application.

## 3.6 Graceful degradation

Failure to insert text directly must never lose the transcript. The transcript
is copied to the clipboard whenever direct insertion fails, and the last
transcript remains retrievable from the daemon while it runs.

---

# 4. Goals

## 4.1 Primary goals

The product must:

- Run on current supported Ubuntu releases using GNOME and Wayland.
- Provide a global shortcut for starting and stopping speech capture.
- Support push-to-talk and toggle interaction modes.
- Capture microphone audio through PipeWire.
- Perform speech recognition locally.
- Insert text into the focused WezTerm pane.
- Work when the WezTerm pane is connected through a mux or remote session.
- Copy text to the clipboard when direct insertion is unavailable.
- Preserve the application or pane that was active when recording began.
- Avoid automatically executing terminal commands.
- Run as a persistent user-level background service.
- Provide clear feedback for recording, transcription and delivery state.

## 4.2 Secondary goals

The product should:

- Support desktop applications through IBus.
- Provide configurable speech models.
- Support custom vocabulary and term replacement.
- Support punctuation and formatting commands.
- Allow users to select microphones.
- Provide a GTK settings interface.
- Expose a CLI for automation and debugging.
- Support pluggable transcription and output adapters.

---

# 5. Non-goals

The initial product will not:

- Provide a full voice assistant.
- Execute arbitrary desktop commands based on natural language.
- Automatically run shell commands.
- Replace screen readers or accessibility software.
- Provide collaborative or cloud-hosted transcript storage.
- Support Windows or macOS.
- Provide speaker diarisation.
- Provide meeting transcription.
- Continuously record in the background.
- Attempt unrestricted synthetic keyboard injection (no `/dev/uinput`, no
  virtual keyboard protocols).
- Guarantee direct insertion into every Wayland application.
- Synchronise settings between devices.
- Store recordings by default.

---

# 6. Target users

## 6.1 Primary user

A developer or technical user who:

- Uses Ubuntu with GNOME and Wayland.
- Works heavily in WezTerm.
- Uses local or remote terminal multiplexing.
- Wants to dictate prose, prompts, commands or notes.
- Prefers local processing.
- Needs fast activation without changing applications.

## 6.2 Secondary users

- Writers and knowledge workers using Linux.
- Users with repetitive strain injuries.
- Users who prefer speech input for accessibility or productivity.
- Engineers working in remote shells.
- Users working with coding agents inside terminal panes.

---

# 7. Core user journeys

## 7.1 Push-to-talk into WezTerm

1. The user places the cursor in a WezTerm pane.
2. The user holds the configured global shortcut.
3. `dictate` records microphone input.
4. The user releases the shortcut.
5. `dictate` transcribes the recording.
6. The transcript is sent to the WezTerm pane that was focused when recording
   began.
7. The transcript appears as editable terminal input.
8. No Enter key is sent.

## 7.2 Toggle dictation into WezTerm

1. The user places the cursor in a WezTerm pane.
2. The user presses the toggle shortcut.
3. Recording begins.
4. The user presses the shortcut again.
5. Recording stops.
6. The text is transcribed and delivered to the original pane.

## 7.3 Dictation into a desktop application

1. The user focuses a supported desktop application.
2. The user starts dictation.
3. `dictate` records the active target.
4. The user finishes speaking.
5. The transcript is committed through the IBus adapter.
6. The text appears at the application cursor.

## 7.4 Clipboard fallback

1. The user starts dictation in an unsupported application.
2. The transcript cannot be inserted directly.
3. `dictate` copies the transcript to the clipboard.
4. A notification informs the user that the transcript is ready to paste.

## 7.5 Delivery failure

1. The user begins dictation in WezTerm.
2. The target pane closes before transcription finishes.
3. Direct delivery fails.
4. The transcript is copied to the clipboard.
5. The user receives a non-disruptive notification explaining the fallback.

## 7.6 Cancelled dictation

1. Recording is active.
2. The user invokes the cancel shortcut or CLI action.
3. Audio capture stops.
4. Buffered audio is discarded.
5. No transcript is delivered.

---

# 8. Interaction model

## 8.1 Default shortcuts

Suggested defaults:

| Action                 | Default shortcut  | Mechanism                                  |
| ---------------------- | ----------------- | ------------------------------------------ |
| Push-to-talk           | `Super+Alt+Space` | XDG Global Shortcuts portal (press + release) |
| Toggle dictation       | Unassigned        | Portal or GNOME custom shortcut (press only) |
| Cancel dictation       | Unassigned        | Portal shortcut; also `dictate cancel`     |
| Repeat last transcript | Unassigned        | Portal shortcut or CLI                     |
| Copy last transcript   | Unassigned        | Portal shortcut or CLI                     |

Platform constraints that shape these defaults:

- The XDG Global Shortcuts portal emits distinct `Activated` and
  `Deactivated` signals per registered shortcut, which is what makes real
  push-to-talk possible on Wayland. It is available in
  `xdg-desktop-portal-gnome` from GNOME 45; current Ubuntu LTS ships a newer
  GNOME, so this is the primary mechanism.
- GNOME custom shortcuts (Settings → Keyboard) can only run a command on key
  press. They support toggle mode (`dictate toggle`) but not push-to-talk.
  They are the fallback activation path where the portal is unavailable.
- A bare `Escape` key cannot be globally grabbed on Wayland. `Escape` cancels
  only while a `dictate` surface (e.g. the recording overlay) has keyboard
  focus. Global cancel requires a registered portal shortcut or
  `dictate cancel`.

Shortcut defaults may change based on GNOME conflicts discovered in testing.

## 8.2 Push-to-talk behaviour

- Shortcut activation begins recording.
- Shortcut deactivation ends recording.
- The target application and target pane are captured on activation.
- The target does not change if the user changes windows during transcription.
- The interaction must tolerate accidental very short presses.
- Recordings shorter than the configured minimum duration may be discarded.
- If the portal misses a release event (e.g. session lock mid-press), the
  maximum recording duration acts as a backstop (§19.7).

## 8.3 Toggle behaviour

- First invocation begins recording.
- Second invocation stops recording.
- Invoking cancel while recording discards the session.
- Starting another session while one is active must not create parallel
  recordings; the daemon rejects or coalesces the request (§10.1).

## 8.4 Feedback

The application should provide:

- A brief sound when recording starts.
- A brief sound when recording ends.
- A visual recording indicator.
- An optional microphone level indicator.
- A transcription-in-progress state.
- A delivery-success indication.
- A notification when clipboard fallback occurs.
- An error notification when transcription fails.

Sounds and notifications must be individually configurable.

## 8.5 Cancel semantics

Cancel is available through three routes, all equivalent:

1. `dictate cancel` (CLI / scripting).
2. A registered global cancel shortcut (portal).
3. `Escape` while a `dictate` surface has keyboard focus.

Cancel during `Listening` discards audio. Cancel during `Transcribing`
abandons the session and discards the transcript; it does not need to
interrupt the speech engine mid-inference, only to suppress delivery.

---

# 9. Dictation modes

## 9.1 Prose mode

Prose mode optimises the transcript for normal written communication.

It may apply:

- Sentence capitalisation.
- Punctuation.
- Paragraph breaks.
- Whitespace normalisation.
- Spoken punctuation commands.
- User-defined term replacements.

Example:

```text
Spoken:
Can you review this pull request question mark new paragraph I think the authentication change needs another test full stop

Output:
Can you review this pull request?

I think the authentication change needs another test.
```

## 9.2 Raw mode

Raw mode performs minimal transcript transformation.

It should:

- Preserve the speech engine's output.
- Avoid automatic reformatting.
- Preserve case where supported.
- Apply only essential invalid-character removal (the safety filter always
  runs; §10.9).

## 9.3 Terminal mode

Terminal mode is designed for shells, REPLs and terminal-based coding tools.

It should support explicit spoken tokens such as:

| Spoken            | Emits | Spoken          | Emits  |
| ----------------- | ----- | --------------- | ------ |
| dash              | `-`   | greater than    | `>`    |
| double dash       | `--`  | less than       | `<`    |
| forward slash     | `/`   | open quote      | `"`    |
| backslash         | `\`   | close quote     | `"`    |
| pipe              | `\|`  | open bracket    | `(`    |
| ampersand         | `&`   | close bracket   | `)`    |
| new line          | LF    | tab             | TAB    |

The token vocabulary is user-extensible (§17 Vocabulary).

Terminal mode must:

- Never automatically send Enter.
- Avoid adding sentence punctuation unless spoken.
- Preserve command casing where possible.
- Remove unsafe control characters.
- Use bracketed paste when supported.
- Warn or fall back to clipboard for multiline input when multiline delivery
  is disabled (the default; ADR-004).

Note: `new line` emits a literal line feed *within* pasted content; under
bracketed paste the shell treats it as an editable character rather than a
command submission. When bracketed paste cannot be verified and the text
contains newlines, the adapter follows the configured multiline policy rather
than risking execution.

## 9.4 Per-application mode selection

The user should be able to configure default modes by target.

Example:

```toml
[application_modes]
"org.wezfurlong.wezterm" = "terminal"
"org.gnome.TextEditor" = "prose"
"default" = "prose"
```

---

# 10. Functional requirements

## 10.1 Daemon

The system must provide a persistent user daemon named `dictated`.

The daemon must:

- Start automatically with the user session.
- Run without root privileges.
- Maintain the loaded speech model.
- Own the active recording state.
- Coordinate target detection.
- Coordinate audio capture.
- Coordinate transcription.
- Route completed transcripts to output adapters.
- Expose status and control through a local IPC interface.
- Recover cleanly from transcription worker failure.
- Prevent more than one active recording session.
- Enforce single-instance operation per user session (a second `dictated`
  must detect the existing instance and exit with a clear message).

Concurrency rules:

- Exactly one `DictationSession` may be in a non-terminal state at a time.
- `start` while a session is active is an error (CLI) or a no-op with a
  warning notification (shortcut), never a second session.
- A session that has left `Listening` (i.e. is transcribing or delivering)
  does not block a *new* recording from starting; the daemon may pipeline one
  in-flight transcription behind one active recording, but never two
  recordings.

## 10.2 Command-line interface

The system must provide a CLI named `dictate`.

Required commands:

```bash
dictate start
dictate stop
dictate toggle
dictate cancel
dictate status [--json]
dictate doctor [--json]
dictate devices
dictate models [list|install|remove|path]
dictate config [show|validate|edit]
dictate copy-last
dictate repeat-last
dictate transcribe-file <path>
```

All commands must exit non-zero on failure with an actionable message, and
support `--json` where output is structured, so the CLI is scriptable.

### Command behaviour

#### `dictate start`

- Starts a recording session.
- Captures the current target.
- Returns an error if recording is already active.

#### `dictate stop`

- Stops the active recording.
- Begins transcription.
- Returns an error if no recording is active.

#### `dictate toggle`

- Starts recording when idle.
- Stops recording when active.

#### `dictate cancel`

- Stops and discards the active recording (or suppresses delivery of an
  in-flight transcription).

#### `dictate status`

Returns structured information including:

- Daemon status.
- Current state.
- Active model.
- Selected microphone.
- Current recording duration.
- Captured target.
- Last delivery result.

A JSON output option must be supported.

#### `dictate doctor`

Checks and reports, with a pass/warn/fail result and a remediation hint per
check:

- PipeWire connectivity.
- Microphone availability.
- Global Shortcuts portal availability.
- GNOME session type and Wayland session presence.
- Model availability.
- WezTerm CLI availability.
- WezTerm mux connectivity.
- Clipboard integration.
- IBus integration (when enabled).
- User service installation and status.

#### `dictate transcribe-file <path>`

- Transcribes an existing audio file through the same engine and pipeline.
- Prints the transcript to stdout; delivers nowhere.
- Exists primarily for debugging, benchmarking and regression testing.

## 10.3 Audio capture

The audio subsystem must:

- Use PipeWire.
- Enumerate available input devices.
- Allow explicit microphone selection.
- Detect when the selected device disappears.
- Capture mono PCM audio.
- Resample to the speech model's required sample rate (16 kHz for Whisper).
- Support configurable noise gating.
- Support configurable voice activity detection.
- Maintain audio only in memory by default.
- Stop cleanly when the user cancels or releases the shortcut.
- Bound memory use via the maximum recording duration (§16).

## 10.4 Speech recognition

The first speech engine will use `whisper.cpp`.

The speech subsystem must:

- Keep the model loaded between sessions.
- Support configurable model sizes.
- Support CPU inference.
- Allow hardware-accelerated builds where available.
- Return a final stable transcript.
- Return useful failure information.
- Support language selection.
- Support automatic language detection as an option.
- Support user-defined initial prompts or vocabulary hints where available.
- Run the speech engine in a supervised subprocess so that an engine crash
  cannot take down the daemon (ADR-001).

Initial model options should include:

- Small or equivalent default model.
- Medium model for greater accuracy.
- English-specific model where appropriate.
- Quantised models for lower memory use.

The engine sits behind a `SpeechEngine` trait (§13.6) so that alternative
engines (or a future remote provider) can be added without changing the
daemon.

## 10.5 Target capture

The system must capture the output target when recording begins.

The target snapshot must include sufficient information to attempt later
delivery.

Potential target types:

```rust
pub enum TargetSnapshot {
    WezTerm {
        pane_id: u64,
        workspace: Option<String>,
        client_pid: Option<u32>,
    },
    IBus {
        context_id: String,
        application_id: Option<String>,
    },
    DesktopApplication {
        application_id: Option<String>,
        window_title: Option<String>,
    },
    Clipboard,
}
```

The target snapshot must be immutable for the duration of a dictation session.

Wayland constraint: GNOME does not expose "which window has focus" to
arbitrary clients. Target detection therefore relies on integration-provided
signals (the WezTerm plugin reporting focus, the IBus engine owning the input
context) rather than global window introspection. When no integration can
identify the target, the snapshot is `Clipboard`.

## 10.6 WezTerm integration

The WezTerm adapter is a core MVP feature.

It must:

- Detect when WezTerm is the active target.
- Identify the focused WezTerm pane.
- Record the pane ID when recording begins.
- Send the transcript to the captured pane.
- Work with local panes.
- Work with WezTerm mux panes.
- Work with remote domains where `wezterm cli send-text` supports the pane.
- Use bracketed paste behaviour where available.
- Never send Enter or append a trailing newline.
- Verify the pane still exists before delivery; fall back to the clipboard if
  it does not.
- Report delivery success or failure.

### WezTerm plugin

A small WezTerm Lua plugin should be provided.

Responsibilities may include:

- Reporting focused window and pane changes to `dictated`.
- Making the current pane identity available through a local socket or IPC
  call.
- Exposing optional WezTerm-local shortcuts.
- Reporting whether the pane is in an alternate screen.
- Reporting pane domain information.
- Supporting diagnostics.

The plugin must not contain transcription logic, and `dictated` must treat
its input as untrusted (validated) data.

### Mux behaviour

The integration must distinguish between:

- The local WezTerm GUI client.
- A local pane.
- A pane attached through a WezTerm mux server.
- A pane connected through an SSH domain.
- A shell running `tmux` or `zellij` within a WezTerm pane.

The adapter only needs to target the WezTerm pane. Content running inside
that pane is responsible for consuming pasted input.

## 10.7 IBus integration

IBus direct insertion is a post-MVP capability.

The IBus engine must:

- Appear as an optional input method.
- Pass normal physical keyboard input through.
- Receive completed transcripts from `dictated`.
- Commit transcripts into the focused input context.
- Support pre-edit text in a later version.
- Fall back safely if the input context is unavailable.
- Avoid taking permanent ownership of unrelated keyboard input.

The IBus integration is not required for WezTerm support.

## 10.8 Clipboard integration

The clipboard adapter must:

- Copy completed transcripts as plain text.
- Preserve Unicode.
- Expose the last transcript through the CLI.
- Notify the user when clipboard fallback is used.
- Avoid overwriting the clipboard when transcription fails.
- Optionally preserve and restore the previous clipboard in a future version.

Clipboard delivery must be treated as a valid delivery outcome, not an error.

Platform note: a background daemon on GNOME Wayland cannot use the core
`wl_data_device` clipboard path (it requires a focused surface). Mutter
supports the `wlr-data-control` protocol since GNOME 44, which allows
focus-less clipboard writes; the adapter should use a data-control-capable
backend (e.g. `wl-clipboard-rs`) and `dictate doctor` must verify clipboard
round-trips (see Q-003 in the issues tracker).

## 10.9 Text transformation

The transformation pipeline must be modular.

Processing stages:

```text
Raw speech engine output
→ Unicode normalisation
→ Whitespace normalisation
→ Spoken-token replacement
→ Mode-specific formatting
→ User dictionary replacement
→ Safety filtering
→ Output adapter delivery
```

Rules:

- The safety filter (control-character and escape-sequence removal) is
  unconditional — it runs in every mode, including raw mode.
- The pipeline must not silently remove substantial transcript content; any
  stage that drops more than whitespace must be observable in debug logging
  (without logging the content itself at normal levels).
- The original speech engine output is retained in memory alongside the
  formatted text (§13.3) to support debugging and `dictate status`, but is not
  persisted by default.

## 10.10 History

The MVP maintains only:

- The latest successful transcript.
- The latest raw transcript.
- The latest delivery result.

These values remain in memory and are cleared when the daemon stops.

Optional persistent transcript history may be introduced later and must be
disabled by default.

---

# 11. System architecture

## 11.1 High-level architecture

```text
┌────────────────────────────────────────────────────────┐
│                    Activation layer                    │
│                                                        │
│  XDG Global Shortcut   GNOME Shortcut   WezTerm Lua    │
│           │                  │               │         │
└───────────┼──────────────────┼───────────────┼─────────┘
            │                  │               │
            └──────────────────┴───────────────┘
                               │
                     ┌─────────▼─────────┐
                     │     dictated      │
                     │                   │
                     │ Session manager   │
                     │ Target capture    │
                     │ Delivery router   │
                     │ Configuration     │
                     └───────┬─────┬─────┘
                             │     │
               ┌─────────────┘     └──────────────┐
               │                                  │
      ┌────────▼────────┐                ┌────────▼────────┐
      │ Audio subsystem │                │ Output adapters │
      │                 │                │                 │
      │ PipeWire        │                │ WezTerm         │
      │ Resampling      │                │ IBus            │
      │ VAD             │                │ Clipboard       │
      └────────┬────────┘                └─────────────────┘
               │
      ┌────────▼────────┐
      │ Speech worker   │
      │                 │
      │ whisper.cpp     │
      │ Model lifecycle │
      └─────────────────┘
```

## 11.2 Process model

### `dictated`

A persistent Rust process responsible for orchestration.

### Speech worker

A supervised local process wrapping `whisper.cpp`.

The daemon communicates with the worker through standard input/output or a
Unix domain socket, using JSON Lines during early development with the option
to move to a framed binary protocol if profiling justifies it (Q-001).

The worker must be restartable without restarting the daemon.

### `dictate`

A lightweight CLI that communicates with the daemon over IPC. It contains no
audio or speech logic.

### `dictate-settings`

A GTK4/libadwaita configuration application (post-MVP).

### IBus engine

A separately installed integration component that communicates with the
daemon (post-MVP).

### WezTerm plugin

A Lua module loaded by the user's WezTerm configuration.

---

# 12. Suggested Rust workspace

```text
dictate/
├── Cargo.toml
├── crates/
│   ├── dictate-core/            # Domain model, state machine, pipeline traits
│   ├── dictate-daemon/          # dictated binary
│   ├── dictate-cli/             # dictate binary
│   ├── dictate-ipc/             # IPC protocol + client/server
│   ├── dictate-config/          # Config schema, load, validate
│   ├── dictate-audio/           # Audio traits, buffers, resampling, VAD
│   ├── dictate-audio-pipewire/  # PipeWire backend
│   ├── dictate-stt/             # SpeechEngine trait, worker supervision
│   ├── dictate-stt-whispercpp/  # whisper.cpp worker + protocol
│   ├── dictate-text/            # Transformation pipeline + modes
│   ├── dictate-target/          # TargetSnapshot + providers
│   ├── dictate-output/          # OutputAdapter trait + router
│   ├── dictate-output-wezterm/
│   ├── dictate-output-clipboard/
│   ├── dictate-output-ibus/
│   ├── dictate-hotkey/          # Activation trait
│   ├── dictate-hotkey-portal/   # XDG Global Shortcuts backend
│   ├── dictate-notifications/
│   └── dictate-settings/
├── integrations/
│   ├── wezterm/
│   │   └── dictate.lua
│   ├── ibus/
│   └── systemd/
├── assets/
├── packaging/
│   ├── deb/
│   ├── flatpak/
│   └── appimage/
└── docs/
```

Crates are introduced as their phase begins; the workspace need not start
with all of them. Speech models are downloaded to the XDG data directory
(`~/.local/share/dictate/models/`), never stored in the repository.

---

# 13. Core domain model

## 13.1 Session

```rust
pub struct DictationSession {
    pub id: SessionId,
    pub started_at: Instant,
    pub target: TargetSnapshot,
    pub mode: DictationMode,
    pub language: LanguageSelection,
    pub state: SessionState,
}
```

## 13.2 Session state machine

```rust
pub enum SessionState {
    Idle,
    Starting,
    Listening,
    FinalisingAudio,
    Transcribing,
    Transforming,
    Delivering,
    Completed,
    Cancelled,
    Failed,
}
```

Transitions:

| From             | Event                            | To               |
| ---------------- | -------------------------------- | ---------------- |
| Idle             | start / toggle / PTT press       | Starting         |
| Starting         | target captured, audio open      | Listening        |
| Starting         | audio unavailable                | Failed           |
| Listening        | stop / toggle / PTT release      | FinalisingAudio  |
| Listening        | max duration reached             | FinalisingAudio  |
| Listening        | cancel                           | Cancelled        |
| Listening        | audio device lost                | Failed           |
| FinalisingAudio  | below minimum duration           | Cancelled        |
| FinalisingAudio  | audio flushed                    | Transcribing     |
| Transcribing     | transcript produced              | Transforming     |
| Transcribing     | cancel                           | Cancelled        |
| Transcribing     | worker failure (after retry)     | Failed           |
| Transforming     | text formatted                   | Delivering       |
| Delivering       | any adapter succeeded            | Completed        |
| Delivering       | all adapters + clipboard failed  | Failed           |

`Completed`, `Cancelled` and `Failed` are terminal. Any undefined
state/event pair is rejected and logged; it must never panic the daemon.
Clipboard fallback that succeeds ends in `Completed` with
`fallback_used = true` (ADR-003).

## 13.3 Transcript

```rust
pub struct Transcript {
    pub raw_text: String,
    pub formatted_text: String,
    pub detected_language: Option<String>,
    pub confidence: Option<f32>,
    pub duration: Duration,
}
```

## 13.4 Delivery receipt

```rust
pub struct DeliveryReceipt {
    pub adapter: OutputAdapterKind,
    pub outcome: DeliveryOutcome,
    pub target: TargetSnapshot,
    pub fallback_used: bool,
}
```

## 13.5 Output adapter interface

```rust
#[async_trait]
pub trait OutputAdapter: Send + Sync {
    fn kind(&self) -> OutputAdapterKind;

    async fn supports(
        &self,
        target: &TargetSnapshot,
    ) -> Result<bool>;

    async fn deliver(
        &self,
        target: &TargetSnapshot,
        text: &str,
        options: &DeliveryOptions,
    ) -> Result<DeliveryReceipt>;
}
```

## 13.6 Speech engine interface

```rust
#[async_trait]
pub trait SpeechEngine: Send + Sync {
    async fn initialise(
        &mut self,
        config: SpeechEngineConfig,
    ) -> Result<()>;

    async fn transcribe(
        &self,
        audio: AudioBuffer,
        options: TranscriptionOptions,
    ) -> Result<Transcript>;

    async fn health(&self) -> Result<SpeechEngineHealth>;
}
```

---

# 14. Delivery routing

The routing order should be configurable.

Default routing:

```text
WezTerm target
→ WezTerm adapter
→ Clipboard

IBus-capable desktop target (post-MVP)
→ IBus adapter
→ Clipboard

Unknown target
→ Clipboard
```

The delivery router must:

- Attempt only adapters applicable to the captured target.
- Record all failed attempts in the delivery receipt.
- Fall back to clipboard.
- Avoid duplicate delivery (an adapter that may have partially delivered is
  treated as delivered; the router never retries into the same target).
- Never deliver to the currently active application if it differs from the
  captured target, unless explicitly configured.

---

# 15. IPC

The daemon must expose a local user-only IPC interface.

Preferred design: a D-Bus user-session service (name
`io.github.dictate.Daemon` or similar) for control and signals, with the
option of a Unix domain socket for any high-volume streaming added later.
D-Bus is already required for the portals and for IBus, and gives
signal-based state broadcasting to the CLI, overlay and settings app for free.

Suggested D-Bus surface:

```text
Methods
  Start(mode, options) -> session_id
  Stop(session_id)
  Toggle(mode, options)
  Cancel(session_id)
  GetStatus() -> status
  GetLastTranscript() -> transcript
  CopyLastTranscript()
  ReportWezTermFocus(target)      # called by the WezTerm plugin bridge

Signals
  StateChanged(session_id, state)
  DeliveryCompleted(receipt)
  Error(category, message)
```

Requirements:

- The interface must reject requests from other users (user-session bus and
  socket permissions).
- Messages must be validated; large or malformed messages are rejected.
- No TCP listener is opened by default.

---

# 16. Configuration

Location (XDG):

```text
~/.config/dictate/config.toml
```

Example:

```toml
[general]
default_mode = "prose"
language = "en"
sounds = true
notifications = true
retain_history = false

[hotkeys]
push_to_talk = "Super+Alt+Space"
toggle = ""
cancel = ""

[audio]
device = "default"
sample_rate = 16000
vad_enabled = true
minimum_recording_ms = 250
maximum_recording_seconds = 180

[speech]
engine = "whispercpp"
model = "small.en"
keep_model_loaded = true
threads = 8

[delivery]
fallback_to_clipboard = true
snapshot_target_on_start = true
notify_on_direct_success = false
notify_on_clipboard_fallback = true

[wezterm]
enabled = true
use_bracketed_paste = true
allow_multiline = false
never_send_enter = true

[ibus]
enabled = false

[privacy]
save_audio = false
save_transcripts = false
telemetry = false

[application_modes]
"org.wezfurlong.wezterm" = "terminal"
"org.gnome.TextEditor" = "prose"
"default" = "prose"
```

Rules:

- Invalid configuration must not prevent the daemon from starting. The daemon
  uses safe defaults for invalid values and reports each validation error
  (log, `dictate config validate`, and `dictate doctor`).
- Unknown keys produce warnings, not failures, to keep configs
  forward-compatible.
- The daemon should pick up config changes on restart at minimum; live reload
  is a later enhancement.

---

# 17. Settings application

The GTK settings application (post-MVP) should provide:

## General

- Default dictation mode.
- Language.
- Start and stop sounds.
- Notifications.
- Launch at login.

## Shortcuts

- Push-to-talk shortcut.
- Toggle shortcut.
- Cancel shortcut.
- Shortcut conflict detection where possible.

## Audio

- Microphone selector.
- Live input-level test.
- Noise gate.
- VAD sensitivity.
- Maximum recording length.

## Speech model

- Installed model.
- Available models.
- Download and remove model.
- Estimated memory requirements.
- Hardware acceleration status.
- Model warm-up test.

## Delivery

- WezTerm integration status.
- IBus integration status.
- Clipboard fallback.
- Multiline terminal behaviour.
- Notification preferences.

## Vocabulary

- User-defined words.
- Phrase replacements.
- Technical term dictionary.
- Per-mode replacements.

## Privacy

- Audio persistence.
- Transcript history.
- Crash reporting.
- Telemetry.
- Data directory cleanup.

## Diagnostics

- Daemon status.
- PipeWire status.
- Portal status.
- WezTerm adapter status.
- IBus status.
- Recent non-sensitive errors.
- Copy diagnostic report.

---

# 18. Privacy and security

## 18.1 Default privacy posture

By default:

- Audio is processed locally.
- Audio is not written to disk.
- Transcripts are not persisted.
- No telemetry is sent.
- No cloud API is contacted.
- No inbound network port is opened.

## 18.2 Threat model summary

Assets: microphone audio, transcript content (which may contain secrets the
user dictates), and the ability to inject text into a terminal.

| Threat                                             | Mitigation                                        |
| -------------------------------------------------- | ------------------------------------------------- |
| Another local user reads transcripts via IPC       | User-scoped bus/socket; permission checks (§18.4) |
| Transcript injected into the wrong pane/app        | Immutable target snapshot; existence check; never retarget (§10.5, §19.4) |
| Transcript executes as a command                   | No Enter ever; safety filter; bracketed paste; multiline off by default (§18.5) |
| Escape sequences smuggled through transcript text  | Unconditional control/escape filtering (§10.9)    |
| Malicious focus reports from integration plugins   | Validate plugin input; deliver only to verifiable panes |
| Sensitive content in logs or crash dumps           | No transcript/audio in logs at normal levels (§18.6) |
| Audio persisted unexpectedly                       | In-memory buffers only by default; tests assert no files (§23.5) |

## 18.3 Permissions

The application must request only the permissions it requires:

- Microphone access.
- Global shortcut registration.
- Desktop notifications.
- Clipboard access.
- IBus registration where enabled.

The product must not require:

- Root access.
- Access to `/dev/uinput`.
- Unrestricted screen capture.
- Full remote desktop control.
- Persistent keyboard monitoring.

## 18.4 IPC security

- IPC endpoints must be scoped to the current user.
- Socket permissions must prevent other users from connecting.
- Messages must be validated.
- Large or malformed messages must be rejected.
- Transcript content must not be written to logs at normal log levels.

## 18.5 Terminal safety

The terminal adapter must:

- Never automatically send Enter (a carriage return / newline that submits).
- Remove null bytes.
- Reject invalid control sequences.
- Never emit terminal escape sequences derived from transcript content.
- Treat all transcript content as text rather than commands.
- Use bracketed paste where supported.
- Support disabling multiline input (default: disabled; ADR-004).
- Fall back to clipboard when safe delivery cannot be established.

## 18.6 Logging

Logs may include:

- State transitions.
- Session IDs.
- Timings.
- Adapter outcomes.
- Error categories.
- Model status.

Logs must not include transcript text or raw audio unless debug content
logging is explicitly enabled by the user, and that switch must be clearly
labelled as exposing dictated content.

---

# 19. Error handling

## 19.1 Microphone unavailable

- Do not start recording.
- Notify the user.
- Return a useful CLI error.
- Suggest running `dictate doctor`.

## 19.2 Speech model unavailable

- Report that the configured model is missing.
- Offer an action to install it (`dictate models install`, settings app).
- Do not silently select a materially different model.

## 19.3 Speech worker crash

- Mark the active transcription as failed.
- Restart the worker (bounded restart policy; no tight crash loops).
- Preserve buffered audio only long enough to retry once where safe.
- Notify the user if the transcript cannot be recovered.

## 19.4 WezTerm pane no longer exists

- Copy the transcript to the clipboard.
- Notify the user that the original pane was unavailable.
- Do not send the transcript to a different pane.

## 19.5 IBus context unavailable

- Fall back to clipboard.
- Do not attempt synthetic keyboard injection.

## 19.6 Clipboard failure

- Keep the last transcript in daemon memory.
- Notify the user.
- Allow retrieval with:

```bash
dictate copy-last
dictate status --json
```

## 19.7 Recording too long

- Stop at the configured maximum duration.
- Transcribe the captured audio.
- Notify the user that the limit was reached.

This limit also serves as the backstop for a missed push-to-talk release
event (§8.2).

## 19.8 Portal or shortcut registration failure

- The daemon still starts; activation degrades to CLI and GNOME custom
  shortcut toggle.
- `dictate doctor` reports the portal as unavailable with remediation hints.

---

# 20. Performance requirements

Targets on representative modern desktop hardware (defined per release; e.g.
a 4-core x86-64 laptop from the last five years):

| Metric                              | Target                                     |
| ----------------------------------- | ------------------------------------------ |
| Daemon shortcut response            | Under 100 ms                               |
| Audio capture start                 | Under 150 ms                               |
| Recording feedback                  | Under 200 ms                               |
| Target snapshot                     | Under 100 ms                               |
| Model already warm                  | Required after daemon initialisation       |
| Short utterance (≤10 s) transcription | Under 2 s with the default model         |
| WezTerm delivery after transcript   | Under 250 ms                               |
| Clipboard fallback after transcript | Under 250 ms                               |
| Idle CPU usage                      | Near zero (no polling loops)               |
| Idle memory                         | Dominated by the loaded model; daemon overhead under 50 MB |

Performance measurements must be collected without logging transcript text.
`dictate transcribe-file` plus fixed audio fixtures provide the benchmarking
harness.

---

# 21. Packaging and installation

## 21.1 Initial packaging

The first supported package format is a Debian package for Ubuntu (ADR-005).

The package should install:

- `dictated`.
- `dictate`.
- `dictate-settings` (once it exists).
- A systemd user service.
- Desktop application metadata.
- Icons.
- Portal integration assets.
- The optional WezTerm Lua plugin.
- Optional IBus components (possibly a separate package; Q-007).

Speech models are not bundled; they are downloaded on demand to the XDG data
directory with checksum verification.

## 21.2 Systemd user service

```text
systemctl --user enable --now dictated.service
```

The service must:

- Restart on failure.
- Avoid aggressive restart loops (rate-limited restarts).
- Start after the graphical user session is available
  (`graphical-session.target`).
- Have access to the user PipeWire and D-Bus sessions.

## 21.3 Flatpak

Flatpak may be supported later. Portal integration fits the Flatpak model,
but the following require validation before committing to it:

- WezTerm CLI access from the sandbox.
- Host socket access.
- IBus installation.
- Model storage.
- Direct interaction with host terminal processes.

Flatpak must not block the initial Debian-package delivery (Q-008).

## 21.4 Licensing

- `whisper.cpp` is MIT-licensed and can be linked or shipped as a worker
  binary.
- Whisper model weights are MIT-licensed by OpenAI; download-on-demand also
  keeps package size acceptable.
- The project licence should be chosen before the first public release
  (suggested: MIT or Apache-2.0, matching the Rust ecosystem norm).

---

# 22. Observability and diagnostics

The daemon should expose structured metrics locally.

Useful measurements include:

- Recording duration.
- Transcription duration.
- Model loading duration.
- Target detection duration.
- Delivery duration.
- Adapter selected.
- Fallback rate.
- Speech worker crash count.
- Session cancellation count.
- Average audio queue depth.

Metrics are viewable through:

```bash
dictate status
dictate status --json
dictate doctor
```

Remote telemetry must remain opt-in and is out of scope for the MVPs.

---

# 23. Testing strategy

## 23.1 Unit tests

Unit tests must cover:

- State transitions (including every rejected state/event pair in §13.2).
- Configuration validation.
- Transcript transformations.
- Spoken punctuation replacement.
- Terminal safety filtering.
- Delivery routing.
- Clipboard fallback.
- Target immutability.
- Timeout handling.
- Model process restart logic.

## 23.2 Integration tests

Integration tests should cover:

- PipeWire microphone enumeration.
- Recording lifecycle.
- Speech worker communication.
- Clipboard writes.
- WezTerm pane detection.
- WezTerm `send-text`.
- Pane closure during transcription.
- Daemon and CLI IPC.
- Systemd user service lifecycle.

Tests requiring a live desktop session (portal, clipboard, WezTerm) run in a
separately marked suite so `cargo test` stays green in headless CI; a
scripted checklist covers what CI cannot.

## 23.3 WezTerm scenarios

The test matrix must include:

- Local shell.
- Local `tmux`.
- Local `zellij`.
- WezTerm mux server.
- WezTerm SSH domain.
- Remote shell through SSH inside a local pane.
- Coding agent running in a pane.
- Alternate-screen terminal application.
- Pane closed before delivery.
- Window switched during transcription.
- Multiple WezTerm windows.
- Multiple WezTerm GUI clients connected to one mux.

## 23.4 GNOME and Ubuntu matrix

Initial compatibility should be tested against at least:

- The current Ubuntu LTS release.
- The latest supported Ubuntu interim release where practical.
- GNOME Wayland session.
- A system with multiple microphones.
- A system with no microphone.
- PipeWire restart during operation.
- Portal unavailable or misconfigured.

## 23.5 Security tests

Tests must verify:

- No Enter is sent by the terminal adapter.
- Null bytes are removed.
- Escape sequences cannot be injected.
- IPC is inaccessible to another local user.
- Transcripts are absent from default logs.
- Audio files are not created by default.
- Clipboard fallback does not deliver to the wrong pane.

---

# 24. MVP scope

## 24.1 MVP 1: local dictation foundation

MVP 1 includes:

- Rust daemon.
- Rust CLI.
- Systemd user service.
- PipeWire audio capture.
- Toggle activation through a GNOME custom shortcut.
- Local `whisper.cpp` transcription.
- Clipboard delivery.
- Basic sound and notification feedback.
- Configuration file.
- `dictate doctor`.
- Last-transcript recovery in memory.

MVP 1 does not require: push-to-talk release detection, WezTerm integration,
IBus, GTK settings, vocabulary UI, or persistent history.

## 24.2 MVP 2: WezTerm daily-driver experience

MVP 2 includes:

- XDG Global Shortcuts integration.
- Push-to-talk.
- WezTerm Lua plugin.
- Target pane snapshot.
- `wezterm cli send-text` delivery.
- Mux-aware pane targeting.
- Terminal mode.
- Bracketed paste.
- Clipboard fallback when pane delivery fails.
- Model warm-up and persistent worker.
- Multiple WezTerm window support.

## 24.3 MVP 3: general desktop insertion

MVP 3 includes:

- IBus engine.
- Direct insertion into supported desktop applications.
- GTK settings application.
- Model management UI.
- Microphone management UI.
- Per-application modes.
- Vocabulary and phrase replacement.
- Optional pre-edit transcript display.

---

# 25. MVP acceptance criteria

## 25.1 General

The MVP is accepted when:

- The daemon starts automatically in a GNOME Wayland session.
- The user can begin and end dictation from a global shortcut.
- Audio is captured through PipeWire.
- Speech is transcribed locally.
- No audio or transcript is persisted by default.
- A failed direct delivery always falls back to clipboard where clipboard
  access is available.

## 25.2 WezTerm

The WezTerm MVP is accepted when:

- The pane focused at recording start is captured.
- A completed transcript is delivered to that pane.
- Changing focus during transcription does not change the target.
- The solution works with a local shell.
- The solution works with a pane attached through a WezTerm mux.
- The solution works when `tmux` is running inside the pane.
- The solution does not automatically execute the transcribed input.
- Closing the pane before delivery causes clipboard fallback.
- The transcript is not sent to another pane when the original pane
  disappears.

## 25.3 Performance

The MVP is accepted when:

- Shortcut activation gives feedback within 200 ms on representative hardware.
- The speech model remains warm between sessions.
- Delivery begins within 250 ms after transcription completes.
- Idle CPU use is negligible.
- The daemon recovers from a speech worker crash without requiring logout.

---

# 26. Future capabilities

Potential later capabilities include:

- Streaming partial transcript display.
- Automatic correction based on application context.
- User-defined command grammars.
- Voice editing commands.
- Cloud speech provider adapters.
- Domain-specific dictionaries.
- Multiple language profiles.
- Per-project vocabulary.
- Selection replacement.
- Dictation history.
- Transcript search.
- Audio replay.
- GNOME Shell extension (recording indicator in the top bar).
- KDE support.
- Other terminals such as Alacritty, Kitty and GNOME Console.
- Editor integrations for VS Code, Zed and JetBrains products.
- Coding-agent-specific modes.
- Remote transcription workers.
- Optional LLM-based transcript clean-up.
- Voice macros.
- Accessibility profiles.

Any LLM-based clean-up must remain optional and clearly distinguish
deterministic text replacement from model-generated rewriting.

---

# 27. Risk register

| # | Risk | Impact | Likelihood | Mitigation |
| - | ---- | ------ | ---------- | ---------- |
| R1 | Global Shortcuts portal behaves inconsistently across GNOME versions (missed release events, re-registration prompts) | Push-to-talk unreliable | Medium | GNOME custom shortcut toggle fallback; max-duration backstop; test on LTS + interim |
| R2 | Focus-less clipboard writes fail on some GNOME setups | Fallback path broken | Low–Medium | Data-control backend, `doctor` clipboard round-trip check, notification-based recovery via `copy-last` |
| R3 | Whisper accuracy/latency unacceptable on low-end CPUs | Product feels slow | Medium | Quantised models, model size selection, warm worker, hardware-accel builds |
| R4 | `wezterm cli` behaviour differs across WezTerm versions / mux setups | Core integration breaks | Medium | Version detection in `doctor`; scenario test matrix (§23.3); clipboard fallback |
| R5 | Target detection without a compositor API proves too limited beyond WezTerm | MVP 3 scope grows | Medium | IBus owns desktop-app targeting; clipboard remains the universal fallback |
| R6 | Transcript accidentally executed in a terminal | Safety failure, user harm | Low | Never-Enter invariant, bracketed paste, multiline off by default, security tests (§23.5) |
| R7 | Speech worker instability (native code crashes) | Lost dictations | Medium | Supervised subprocess, bounded restarts, single retry with preserved audio |
| R8 | GNOME shortcut defaults conflict with existing bindings | Poor first-run experience | Medium | Conflict detection where possible; easy rebinding; documented alternatives |

---

# 28. Design decisions

Resolved decisions are recorded as ADRs in
[`plans/decisions/`](../plans/decisions/):

- **ADR-001** — Rust daemon with a supervised `whisper.cpp` worker
  subprocess.
- **ADR-002** — Activation via XDG Global Shortcuts portal for push-to-talk,
  with GNOME custom shortcut toggle as fallback.
- **ADR-003** — Target snapshot at recording start; clipboard fallback is a
  normal successful outcome; WezTerm adapter ships before IBus.
- **ADR-004** — Terminal safety defaults: never send Enter, multiline
  delivery disabled by default, bracketed paste on.
- **ADR-005** — Debian package plus systemd user service as the first
  distribution format.
- **ADR-006** — Privacy defaults: no audio/transcript persistence, no
  telemetry, no network.

Open questions are tracked in [`plans/issues.md`](../plans/issues.md)
(Q-001…): speech worker wire protocol, default model choice, clipboard
backend, whisper.cpp build-vs-download, WezTerm plugin push-vs-pull focus,
audio process isolation, IBus packaging, and Flatpak viability.

---

# 29. Delivery plan

Delivery is planned and tracked through APS in
[`plans/index.aps.md`](../plans/index.aps.md). Phases map to modules:

| Phase | Content | APS modules |
| ----- | ------- | ----------- |
| 1 — Daemon and audio | Workspace, daemon lifecycle, IPC, PipeWire capture, state machine, CLI, notifications | `01-core`, `02-audio`, `05-delivery` (feedback) |
| 2 — Transcription | whisper.cpp worker, model management, warm lifecycle, normalisation, failure recovery | `03-stt`, `04-text` |
| 3 — Clipboard product (MVP 1) | GNOME toggle shortcut, clipboard adapter, last-transcript recovery, Debian packaging, diagnostics | `05-delivery`, `06-hotkeys`, `08-packaging` |
| 4 — WezTerm integration | Plugin, pane snapshot, pane delivery, mux testing, terminal mode, safety filtering | `07-wezterm`, `04-text` |
| 5 — Push-to-talk (MVP 2) | XDG Global Shortcuts portal, press/release events, PTT polish, recording overlay | `06-hotkeys` |
| 6 — Desktop insertion (MVP 3) | IBus engine, per-application configuration, GTK settings, vocabulary management | `09-ibus`, `10-settings` |

---

# 30. Definition of done

A feature is complete when:

- The implementation matches the documented behaviour.
- Unit tests cover the core logic.
- Integration tests cover the relevant platform interaction.
- Errors are actionable.
- Transcript content does not appear in standard logs.
- Clipboard fallback is tested.
- Documentation is updated.
- `dictate doctor` can diagnose the feature.
- Packaging includes all required assets.
- The feature has been exercised in a real GNOME Wayland session.
- Terminal-related changes have been verified not to send Enter or unsafe
  control sequences.
- The corresponding APS work item is validated and marked complete.

---

# 31. Glossary

| Term | Meaning |
| ---- | ------- |
| Bracketed paste | Terminal mode where pasted text is wrapped in markers so the shell treats it as literal editable input, not keystrokes |
| IBus | Intelligent Input Bus — Linux input-method framework used for committing text into applications |
| Mux (WezTerm) | WezTerm's multiplexing server; panes can live in a separate process or on a remote host |
| Pane | A single terminal within a WezTerm window/tab, addressed by pane ID |
| PipeWire | The Linux audio/video server used by modern Ubuntu for microphone capture |
| Portal (XDG) | D-Bus services exposing desktop capabilities (shortcuts, notifications) to apps in a sandbox-friendly way |
| PTT | Push-to-talk — record while a shortcut is held |
| Target snapshot | Immutable record of where the transcript must be delivered, captured when recording starts |
| VAD | Voice activity detection |
| Wayland | The display protocol used by modern GNOME; deliberately prevents global input injection/snooping |
| whisper.cpp | C/C++ port of OpenAI's Whisper speech-recognition model, suitable for local CPU inference |

---

# 32. Traceability

| Spec area | Sections | APS module |
| --------- | -------- | ---------- |
| Daemon, state machine, IPC, CLI, config | §10.1, §10.2, §13, §15, §16 | [01-core](../plans/modules/01-core.aps.md) |
| Audio capture | §10.3 | [02-audio](../plans/modules/02-audio.aps.md) |
| Speech recognition | §10.4 | [03-stt](../plans/modules/03-stt.aps.md) |
| Modes and text transformation | §9, §10.9 | [04-text](../plans/modules/04-text.aps.md) |
| Delivery routing, clipboard, feedback, history | §8.4, §10.8, §10.10, §14 | [05-delivery](../plans/modules/05-delivery.aps.md) |
| Activation and shortcuts | §8 | [06-hotkeys](../plans/modules/06-hotkeys.aps.md) |
| Target capture and WezTerm | §10.5, §10.6 | [07-wezterm](../plans/modules/07-wezterm.aps.md) |
| Packaging, service, diagnostics | §10.2 (doctor), §21, §22 | [08-packaging](../plans/modules/08-packaging.aps.md) |
| IBus | §10.7 | [09-ibus](../plans/modules/09-ibus.aps.md) |
| Settings app | §17 | [10-settings](../plans/modules/10-settings.aps.md) |
| Privacy & security | §18, §23.5 | cross-cutting (all modules) |
