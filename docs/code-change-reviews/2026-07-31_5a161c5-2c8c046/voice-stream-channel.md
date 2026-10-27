# Voice / stream channel & screen share

**Group:** #6 — Voice/stream channel & screen share
**Significance:** Critical
**Commits:** `95189f7` (audio), `36af396` (voice UI), `9e3ebef` (stream crate), `f632711` (screen share modal), `b8da19b` (PIP), `7d35b4e` (livekit), `90012fc` (signaling), `f8abba0` (vendored libspa)
**Areas:** `crates/mezon-ui/src/chat/voice.rs`, `stream.rs`, `screen_share_modal.rs`, `screen_share_pip.rs`, `crates/mezon-stream/*`, `crates/mezon-voice/*`, `crates/mezon-audio/*`, `crates/vendor/libspa/**`

## Change

Mezon gained a **voice/video channel** capability: live WebRTC sessions via LiveKit, a voice panel UI, screen share with a picture-in-picture (PIP) preview, and a vendored `libspa` (PipeWire SPA) for Linux audio handling. The new `mezon-stream` crate encapsulates the WebRTC session; `mezon-voice` and `mezon-audio` were extended.

## Before

At `5a161c5`, `mezon-voice` and `mezon-audio` existed but were minimal (basic encode/decode/playback). There was no WebRTC, no LiveKit, no screen share, no voice channel UI. `mezon-ui/src/chat/` had no `voice.rs` or `stream.rs`. `mezon-store` had no `stream.rs`.

## After

### `mezon-stream` crate (new)

Public API (Observed — `crates/mezon-stream/src/lib.rs`):

- `StreamSession` — holds a LiveKit room + signaling WebSocket.
- `StreamEvent` — `ParticipantJoined`, `ParticipantLeft`, `AudioTrack`, `VideoTrack`, `ScreenShareTrack`, `Muted`, `Unmuted`, `Speaking`.
- `start(channel_id, token)` — joins a LiveKit room.
- `stop()` — leaves the room, tears down tracks.
- Spawns a dedicated OS thread + tokio runtime (Observed — `mezon-stream/src/runtime.rs`) to avoid blocking the GPUI executor.

Dependencies: `livekit`, `tokio-tungstenite` (signaling WS), `mezon-voice`, `tokio`.

### `mezon-store::stream` (`crates/mezon-store/src/stream.rs`)

- `session: Option<StreamSession>` holder.
- Exposes `StreamEvent` to UI via the central `StoreEvent` bus.
- UI calls `store.stream.start(...)` / `stop()`.

### Voice UI (`crates/mezon-ui/src/chat/voice.rs`)

- Voice panel rendered when the user joins a voice channel.
- Shows participant list with speaking indicators (animated border based on `Speaking` events).
- Mute / deafen / camera / screen-share toggle buttons.
- Connects to `mezon-store::stream` events.

### Screen share (`screen_share_modal.rs`, `screen_share_pip.rs`)

- `screen_share_modal.rs` — picker UI: choose a window or display to share.
- `screen_share_pip.rs` — picture-in-picture preview of the shared screen, draggable, always-on-top.
- Uses `mezon-native` screen-capture APIs (platform-specific).

### Vendored `libspa` (`crates/vendor/libspa/**`)

- 43 files, +11,309 lines (Observed — largest vendored addition in this range).
- PipeWire's SPA (Simple Plugin API) bindings for Rust, used by `mezon-audio` on Linux for audio device enumeration and capture.
- Mechanical vendored import; semantics-preserving.

### `mezon-audio` extensions (`crates/mezon-audio/src/{decode,lib,playback}.rs`)

- Minor: +67/−2 lines (Observed).
- Decode/playback adjustments to handle LiveKit's audio frame format.

## Mechanism

### Voice session lifecycle (Observed)

```
User clicks "Join Voice"
   │
   ↓
mezon-ui/chat/voice.rs ──> mezon-store::stream::start(channel_id, token)
   │
   ↓
mezon-stream::StreamSession::start
   │
   ├─ spawn OS thread + tokio runtime
   ├─ connect LiveKit room (token from server)
   ├─ open signaling WS (tokio-tungstenite)
   └─ publish local audio track (mezon-voice capture)
   │
   ↓ StreamEvent::ParticipantJoined / AudioTrack / Speaking
mezon-store::stream ──> StoreEvent::Stream(...) ──> mezon-ui voice panel
```

### Screen share flow (Observed)

```
User clicks "Share Screen"
   │
   ↓
screen_share_modal.rs ──> mezon-native::list_screens/windows
   │
   ↓ user picks a source
mezon-stream::StreamSession::publish_screen_share(source_id)
   │
   ↓ ScreenShareTrack
screen_share_pip.rs ──> renders preview in a PIP window
```

## Motivation

- **Observed:** LiveKit integration is the chosen WebRTC stack (vs. raw libwebrtc) — likely for the managed signaling + SFU.
- **Strong inference:** A dedicated OS thread + tokio runtime in `mezon-stream` is required because LiveKit's SDK is async and blocking; running it on the GPUI executor would freeze the UI.
- **Strong inference:** `libspa` is vendored (not pulled from crates.io) because the PipeWire Rust bindings aren't published as a stable crate; vendoring ensures build reproducibility.
- **Hypothesis:** PIP preview exists so the sharer can see what they're sharing without it being in the main chat view (common UX pattern).

## Impact

- **Capability:** voice/video channels are now first-class; screen share works.
- **Platform:** Linux audio now requires PipeWire + libspa; macOS/Windows use `mezon-native` capture APIs.
- **Dependencies:** `livekit`, `tokio-tungstenite` added; `libspa` vendored (43 files).
- **Concurrency:** the app now has 3+ executors: GPUI executor, `mezon-client` transport runtime, `mezon-stream` runtime. Coordination is via channels + the `StoreEvent` bus.
- **Resource:** voice sessions hold open WebRTC connections + audio capture devices; leaks would lock the mic/camera.

## Trade-offs

- **Benefit:** real-time voice/video/screen-share; LiveKit offloads SFU complexity.
- **Cost:** heavy native deps (LiveKit, libspa); multi-executor complexity; device-management responsibility.
- **Risk:** if `StreamSession::stop` doesn't fully tear down tracks, the mic/camera stay locked. (Not load-tested — see quality-review.md.)
- **Platform fragility:** Linux audio now depends on PipeWire; systems without PipeWire (older distros) won't have working voice.
- **Rollback:** remove `mezon-stream` + `mezon-ui/chat/voice.rs` + `screen_share_*.rs` + `mezon-store::stream`; revert `mezon-audio` to pre-voice state. `libspa` can stay vendored (unused) or be removed.

## Verification

- `crates/mezon-stream/src/lib.rs` exports `StreamSession`, `StreamEvent` (Observed).
- `crates/mezon-stream/src/runtime.rs` spawns an OS thread + tokio runtime (Observed).
- `crates/mezon-ui/src/chat/voice.rs` exists and renders a voice panel (Observed).
- `crates/mezon-ui/src/chat/screen_share_modal.rs` and `screen_share_pip.rs` exist (Observed).
- `crates/mezon-store/src/stream.rs` holds `session: Option<StreamSession>` (Observed).
- `crates/vendor/libspa/**` — 43 files vendored (Observed).
- `crates/mezon-audio/src/{decode,playback}.rs` — minor changes (Observed).
- **Not verified:** device teardown on `stop()`; behavior when PipeWire is absent; concurrent voice sessions.

## Source-reading guide

1. `crates/mezon-stream/src/lib.rs` — `StreamSession`, `StreamEvent`, public API.
2. `crates/mezon-stream/src/runtime.rs` — OS thread + tokio runtime spawn.
3. `crates/mezon-store/src/stream.rs` — session holder, `StoreEvent` bridge.
4. `crates/mezon-ui/src/chat/voice.rs` — voice panel UI, speaking indicators.
5. `crates/mezon-ui/src/chat/screen_share_modal.rs` — source picker.
6. `crates/mezon-ui/src/chat/screen_share_pip.rs` — PIP preview.
7. `crates/mezon-voice/src/lib.rs` — audio capture (pre-existing, extended).
8. `crates/mezon-audio/src/decode.rs`, `playback.rs` — LiveKit frame handling.
9. `crates/vendor/libspa/` — vendored PipeWire SPA (mechanical; skim only).

## Learning

- **Spawning a dedicated runtime for a heavy async dependency** (LiveKit) is the right call when the host executor is single-threaded (GPUI) — but it means crossing thread boundaries with channels, which adds latency and complexity.
- **Vendoring native bindings** (libspa) is sometimes necessary when the upstream isn't on crates.io; the cost is a large diff that's hard to review but semantically neutral.
- **PIP preview for screen share** is a small UX touch that significantly improves sharer confidence — worth the extra window-management code.

## Unknowns

- Does `StreamSession::stop()` fully release audio/video devices on all platforms? (Critical for mic/camera lock bugs. Not confirmed.)
- What happens if the LiveKit token expires mid-session? (Hypothesis: reconnection flow; not confirmed.)
- Is there a fallback when PipeWire is absent on Linux? (Hypothesis: no; voice disabled. Not confirmed.)
- How are concurrent voice sessions handled (user in two voice channels)? (Hypothesis: not allowed; UI prevents it. Not confirmed.)
