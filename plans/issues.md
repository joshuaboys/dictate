# Issues & Questions Tracker

> Development-time discoveries that emerge while building. Not a bug tracker
> replacement — a lightweight log for planning-level concerns that need
> visibility.

---

## Issues

<!--
ID format: ISS-NNN · Status: Open | Resolved | Deferred | Won't Fix
Severity: Critical | High | Medium | Low
-->

_(No issues yet)_

---

## Questions

<!--
ID format: Q-NNN · Status: Open | Answered | Deferred
Priority: High | Medium | Low
-->

### Q-001: Speech worker wire protocol — stay on JSON Lines or move to binary?

| Field      | Value            |
| ---------- | ---------------- |
| Status     | Open             |
| Priority   | Medium           |
| Discovered | Spec §27 / ADR-001 |
| Module     | STT              |

**Context:** ADR-001 starts the daemon↔worker protocol on JSON Lines for
debuggability. Audio buffers are the only high-volume payload.

**Options considered:**

1. JSON Lines + base64 audio — trivial to debug, wasteful for PCM.
2. JSON Lines control + raw PCM side-channel (fd/socket) — likely sweet spot.
3. Framed binary (MessagePack/custom) — fastest, hardest to inspect.

**Answer when:** STT-001/STT-002 profiling shows whether serialisation cost
matters.

### Q-002: Default speech model — `small.en` or multilingual `small`?

| Field      | Value    |
| ---------- | -------- |
| Status     | Open     |
| Priority   | Medium   |
| Discovered | Spec §27 |
| Module     | STT      |

**Context:** `small.en` is more accurate and faster for English; the
multilingual model serves non-English users out of the box. Spec's config
example assumes `small.en`.

**Answer when:** STT-003 defines the model catalogue; decide with real
latency/accuracy numbers on representative hardware.

### Q-003: Clipboard backend on GNOME Wayland

| Field      | Value       |
| ---------- | ----------- |
| Status     | Open        |
| Priority   | High        |
| Discovered | Spec §10.8  |
| Module     | DLV         |

**Context:** A focus-less daemon can't use the core `wl_data_device`
clipboard. Mutter supports `wlr-data-control` since GNOME 44, and newer
stacks add `ext-data-control`.

**Options considered:**

1. `wl-clipboard-rs` (data-control protocols) — in-process, no focus needed.
2. Shelling out to `wl-copy` — extra dependency, same protocols.
3. GTK/portal clipboard — needs a focused surface; wrong shape for a daemon.

**Answer when:** DLV-002 is implemented and doctor's clipboard round-trip
check passes on current Ubuntu LTS.

### Q-004: whisper.cpp — vendored build, system dependency, or downloaded worker?

| Field      | Value    |
| ---------- | -------- |
| Status     | Open     |
| Priority   | Medium   |
| Discovered | Spec §27 |
| Module     | STT, PKG |

**Context:** Affects deb size, hardware-accelerated variants, and security
update responsibility.

**Answer when:** PKG-003 packaging work forces the choice.

### Q-005: WezTerm plugin — push focus events or query on demand?

| Field      | Value    |
| ---------- | -------- |
| Status     | Open     |
| Priority   | Medium   |
| Discovered | Spec §27 |
| Module     | WEZ      |

**Context:** Push gives an always-fresh snapshot but a chattier plugin;
pull (query at recording start) is simpler but adds latency inside the
100 ms target-snapshot budget. `wezterm cli` querying may suffice without
the plugin for common cases.

**Answer when:** WEZ-001 latency measurements exist.

### Q-006: Separate audio process for crash isolation?

| Field      | Value    |
| ---------- | -------- |
| Status     | Deferred |
| Priority   | Low      |
| Discovered | Spec §27 |
| Module     | AUD      |

**Context:** The speech worker is already isolated (ADR-001). PipeWire
client code is comparatively safe Rust; a separate audio process adds IPC
for unclear benefit.

**Revisit if:** audio-related daemon crashes appear in practice.

### Q-007: Should IBus components ship as a separate Debian package?

| Field      | Value    |
| ---------- | -------- |
| Status     | Deferred |
| Priority   | Low      |
| Discovered | Spec §27 |
| Module     | IBUS, PKG |

**Context:** IBus is MVP 3 and optional; a separate package keeps the base
install lean and avoids pulling input-method dependencies onto
terminal-only users.

**Answer when:** 09-ibus moves to Ready.

### Q-008: Is Flatpak worth the host-integration complexity?

| Field      | Value      |
| ---------- | ---------- |
| Status     | Deferred   |
| Priority   | Low        |
| Discovered | Spec §21.3 |
| Module     | PKG        |

**Context:** Portals fit Flatpak, but WezTerm CLI access, host sockets, IBus
installation and model storage all need validation from inside the sandbox.

**Answer when:** after M2; the deb (ADR-005) is unblocked regardless.

### Q-009: How does dictate know WezTerm is the *focused application* at recording start?

| Field      | Value      |
| ---------- | ---------- |
| Status     | Open       |
| Priority   | High       |
| Discovered | Spec review (§10.5, risk R5) |
| Module     | WEZ, HOT   |

**Context:** GNOME Wayland exposes no "which application has focus" API to
ordinary clients. The WezTerm plugin can report WezTerm's focused *pane*,
but not whether WezTerm itself is the active window. If the user focuses
Firefox and starts dictation, the daemon still sees a valid last-known pane
and would deliver there — technically per the snapshot rules, but likely not
what the user meant.

**Options considered:**

1. Accept "last-known WezTerm pane" semantics, config-gated (e.g. a
   staleness window on focus reports) — no new components, occasionally
   surprising.
2. Small GNOME Shell extension exposing the focused app ID to the daemon —
   accurate, but a new install-and-maintain surface (currently listed as a
   future capability; may need pulling forward to MVP 2).
3. Have the WezTerm plugin report focus-gained *and* focus-lost, so a blurred
   WezTerm clears the target and dictation falls back to clipboard —
   middle ground, still blind to what gained focus.

**Answer when:** WEZ-001 design; option 3 vs 2 should be decided before
MVP 2 acceptance criteria are tested.

### Q-010: Should MVP 3 (general desktop insertion) proceed as scoped, given Vocalinux?

| Field      | Value       |
| ---------- | ----------- |
| Status     | Deferred    |
| Priority   | Medium      |
| Discovered | Prior-art review (spec §2.1) |
| Module     | IBUS, SET   |

**Context:** Vocalinux (GPLv3, Python) already serves general desktop
dictation on Linux via IBus, with an active community. dictate's
differentiators — target snapshot, terminal safety, portal-native plumbing,
daemon/CLI — are all delivered by MVP 1–2. MVP 3 is the point where dictate
starts competing head-on with a project that has a substantial head start,
and its IBus layer is Vocalinux's documented maintenance tarpit.

**Options considered:**

1. Proceed as scoped — full IBus engine + settings app.
2. Slim MVP 3 — settings app and per-application modes, keep clipboard as
   the general-desktop path, skip IBus.
3. Drop MVP 3 — position dictate purely as the terminal dictation tool and
   recommend Vocalinux for general desktop use.

**Answer when:** MVP 2 is accepted; re-assess the landscape then.

---

## Resolved

_(Nothing resolved yet)_

---

## Quick Reference

| ID Type  | Format  | Example |
| -------- | ------- | ------- |
| Issue    | ISS-NNN | ISS-001 |
| Question | Q-NNN   | Q-001   |

**Severities:** Critical > High > Medium > Low

**Reference from other docs:** `See ISS-001` or `Related: ISS-001, Q-002`
