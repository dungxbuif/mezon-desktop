# Channel mini-app webview

**Group:** #9 — Channel app webview
**Significance:** Significant
**Commits:** `8e76568` (webview crate), `e31702e` (channel app UI), `0a897cf` (launch flow), `c631238` (lifecycle)
**Areas:** `crates/mezon-webview/*`, `crates/mezon-ui/src/channel_app/mod.rs`, `crates/mezon-client/src/channel_app_launch.rs`

## Change

Channels can now host a **mini-app**: a webview embedded in the channel that loads a configured URL (per-channel), providing an app-within-a-channel experience. The new `mezon-webview` crate wraps `wry` (WebKit on all platforms) and integrates with GPUI via a GTK event pump on Linux.

## Before

At `5a161c5`, there was no webview. Channels were chat-only (messages + attachments + canvas). `mezon-ui` had no `channel_app/` module. `mezon-client` had no `channel_app_launch.rs`.

## After

### `mezon-webview` crate (new)

- Wraps `wry` (v0.47) — a cross-platform webview library (WebKit2GTK on Linux, WKWebView on macOS, WebView2 on Windows).
- Public API (Observed — `crates/mezon-webview/src/lib.rs`):
  - `WebViewWindow` — a GPUI-compatible window hosting a webview.
  - `init_gtk()` (Linux only) — initializes GTK main loop for WebKit2GTK.
  - `pump_gtk_events()` — pumps GTK events from within the GPUI event loop (Linux only).
  - Channel for bidirectional JS↔Rust communication (`WebViewMessage`).

### Channel app UI (`mezon-ui/src/channel_app/mod.rs`)

- `ChannelAppWebView` — manages a webview per channel app.
- `GlobalChannelAppWindows(HashMap<i64, WindowHandle>)` — registry of active channel app windows.
- Operations: `create`, `resize`, `destroy`, `reload`, `go_back`, `go_forward`.
- Launched from a channel that has `channel.app_url` set.

### Launch flow (`mezon-client/src/channel_app_launch.rs`)

- `launch_channel_app(channel_id)` — fetches the app URL from the channel config and signals the UI to open the webview.
- Triggered when the user clicks "Open App" on a channel with an app URL.

## Mechanism

### Linux GTK integration (Observed — `mezon-app/src/main.rs:352,364`)

```
mezon-app startup (Linux)
   │
   ├─ mezon_webview::init_gtk() ──> initializes GTK main loop
   └─ register pump_gtk_events() in GPUI event loop
       │
       ↓ each GPUI frame
       pump_gtk_events() ──> processes pending GTK events (WebKit2GTK)
```

WebKit2GTK requires a GTK main loop; since GPUI has its own loop, the GTK loop must be pumped manually each frame. macOS/Windows don't need this (WKWebView/WebView2 integrate with the native loop).

### Channel app launch flow (Observed)

```
User clicks "Open App" on channel
   │
   ↓
mezon-client::channel_app_launch::launch_channel_app(channel_id)
   │
   ↓ fetch app_url from channel config
mezon-ui::channel_app::create(channel_id, app_url)
   │
   ↓
GlobalChannelAppWindows.insert(channel_id, WebViewWindow::new(app_url))
   │
   ↓
WebViewWindow loads URL, renders in a GPUI window
```

### JS↔Rust communication (Observed)

- `WebViewMessage` enum: `JsToRust { event, payload }`, `RustToJs { event, payload }`.
- Rust exposes a `window.mezon` JS API; the web app calls `window.mezon.sendMessage(...)`, which arrives as `WebViewMessage::JsToRust` in Rust.
- Rust can push events to JS via `RustToJs`.

## Motivation

- **Observed:** Channel mini-apps let channels host external web apps (games, tools, dashboards) — a Discord Activities / Slack Apps-style feature.
- **Strong inference:** `wry` is chosen over a custom webview binding because it's cross-platform and maintained (Tauri's webview layer).
- **Strong inference:** The GTK pump on Linux is a necessary workaround for WebKit2GTK's main-loop requirement; it's a known pattern in Rust+GTK apps.
- **Hypothesis:** The JS↔Rust bridge (`window.mezon`) lets mini-apps interact with the Mezon client (send messages, get user info) without a full OAuth flow.

## Impact

- **Capability:** channels can now host web apps; a new extension surface.
- **Platform:** Linux requires GTK + WebKit2GTK; macOS/Windows use native webview.
- **Dependencies:** `wry = "0.47"`, `webkit2gtk = "=2.0.1"` (Linux, pinned), `raw-window-handle`.
- **Security:** webviews load arbitrary URLs; the JS bridge exposes Mezon APIs to the web app. (See Unknowns — sandboxing not confirmed.)
- **Concurrency:** GTK event pumping adds per-frame work to the GPUI loop on Linux.

## Trade-offs

- **Benefit:** powerful extension surface; cross-platform via wry; mini-apps in channels.
- **Cost:** webview is a heavy dependency; GTK pump on Linux is fragile; security surface (arbitrary JS in the webview).
- **Risk:** if the JS bridge has no permission model, a malicious mini-app could send messages or access user data. (Not confirmed — see Unknowns.)
- **Platform fragility:** `webkit2gtk = "=2.0.1"` pin makes Linux upgrades brittle.
- **Rollback:** remove `mezon-webview` + `channel_app/` + `channel_app_launch.rs`; channels revert to chat-only.

## Verification

- `crates/mezon-webview/src/lib.rs` exports `WebViewWindow`, `init_gtk`, `pump_gtk_events`, `WebViewMessage` (Observed).
- `crates/mezon-ui/src/channel_app/mod.rs` defines `ChannelAppWebView`, `GlobalChannelAppWindows` (Observed).
- `crates/mezon-client/src/channel_app_launch.rs` defines `launch_channel_app` (Observed).
- `crates/mezon-app/src/main.rs:352,364` calls `init_gtk()` + `pump_gtk_events()` on Linux (Observed).
- `Cargo.toml` for `mezon-webview` lists `wry = "0.47"`, `webkit2gtk = "=2.0.1"` (Observed).
- **Not verified:** JS bridge permission model; webview sandboxing; behavior on webview crash.

## Source-reading guide

1. `crates/mezon-webview/src/lib.rs` — `WebViewWindow`, `init_gtk`, `pump_gtk_events`.
2. `crates/mezon-webview/src/channel.rs` (if present) — `WebViewMessage` JS bridge.
3. `crates/mezon-ui/src/channel_app/mod.rs:84` — `GlobalChannelAppWindows` registry.
4. `crates/mezon-client/src/channel_app_launch.rs` — `launch_channel_app`.
5. `crates/mezon-app/src/main.rs:352,364` — Linux GTK init + pump.
6. `crates/mezon-webview/Cargo.toml` — `wry`, `webkit2gtk` pins.

## Learning

- **Pumping a foreign event loop** (GTK) inside the host loop (GPUI) is a known workaround but adds per-frame cost and can cause subtle ordering bugs; prefer native integration where possible (macOS/Windows do).
- **Pinning native bindings** (`webkit2gtk = "=2.0.1"`) is defensive but makes security updates require a coordinated bump across the workspace.
- **JS↔Rust bridges in webviews** are a powerful extension surface but a security liability without a permission model; always check what the bridge exposes.

## Unknowns

- **Does the JS bridge have a permission model?** (Critical. Can a mini-app call `window.mezon.sendMessage` without consent? Not confirmed.)
- Is the webview sandboxed (separate process, no file access)? (Hypothesis: wry uses the native webview process, which is sandboxed by the OS; not confirmed.)
- What happens if the webview crashes? (Hypothesis: window destroyed, user notified; not confirmed.)
- Can multiple channel apps be open simultaneously? (Observed: `GlobalChannelAppWindows` is a HashMap keyed by channel_id — yes, but resource limits not tested.)
