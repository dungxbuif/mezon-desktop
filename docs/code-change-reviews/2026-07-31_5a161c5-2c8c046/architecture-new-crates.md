# Architecture: 8 new crates & workspace split

**Group:** #1 — New crate architecture & workspace split
**Significance:** Critical
**Commits:** `2ccbb5f` (cli+mcp), `2801bc4` (canvas), `5f09f0b` (wallet), `8e76568` (webview), `33ef428` (emoji/voice), + 8 `Cargo.toml` additions
**Areas:** `crates/mezon-{canvas,mcp,webview,stream,theme,cli,widgets}`, `crates/mmn-client`, `Cargo.toml`

## Change

The workspace grew from ~12 to **20 crates**. Eight new crates were extracted, organized into four functional layers. The workspace `Cargo.toml` `members` list was extended and `mezon-active-windows` was deliberately excluded (Observed comment: "orphan (unused) + does not compile").

## Before

At `5a161c5`, the workspace contained the original chat-oriented crates: `mezon-app`, `mezon-ui`, `mezon-client`, `mezon-store`, `mezon-voice`, `mezon-video`, `mezon-audio`, `mezon-native`, `mezon-updater`, `mezon-proto`, `mezon-i18n`, plus vendored GPUI/Zed crates. UI primitives (input, button, theme) were inlined inside `mezon-ui`; there was no canvas editor, no MCP server, no webview, no streaming session crate, no blockchain client. Logic for these concerns either did not exist or was embedded in `mezon-ui`/`mezon-store`.

## After

Eight new crates, grouped by layer:

| Layer              | Crate                           | Role                                                                                   |
| ------------------ | ------------------------------- | -------------------------------------------------------------------------------------- |
| UI primitives      | `mezon-theme`, `mezon-widgets`  | Foundation: theme tokens + Input/Button/Icon — shared by `mezon-ui` and `mezon-canvas` |
| UI feature         | `mezon-canvas`, `mezon-webview` | Canvas editor (pure GPUI, TipTap/Quill) + channel app webview (wry)                    |
| Runtime/media      | `mezon-stream`                  | WebRTC viewer session (livekit + signaling WS)                                         |
| External interface | `mezon-mcp`, `mezon-cli`        | MCP server (HTTP/stdio for AI) + CLI entry                                             |
| Domain client      | `mmn-client`                    | MMN blockchain client (Ed25519 + JSON-RPC + ZK)                                        |

### Dependency graph (Observed from `Cargo.toml` + `lib.rs` imports)

```
mezon-app (binary)
 ├── mezon-cli ──┬── mezon-mcp ──┬── mezon-client, mezon-proto, rmcp, axum
 │               └── mezon-native (control IPC)
 ├── mezon-ui
 │    ├── mezon-canvas ──┬── mezon-widgets, mezon-theme, mezon-store, mezon-i18n, ui, menu
 │    ├── mezon-webview ── wry, raw-window-handle (linux: gtk, webkit2gtk)
 │    ├── mezon-theme ──── gpui, theme(vendor)
 │    └── mezon-widgets ── mezon-theme, mezon-store, gpui
 ├── mezon-store
 │    ├── mezon-stream ── livekit, tokio-tungstenite, mezon-voice
 │    └── mmn-client ──── http_client, ed25519-dalek, sha2, bs58
 └── (mezon-mcp runtime wired via mezon-app/src/mcp.rs + ControlServer)
```

**Layering is one-directional with no cycles** (Observed): `mezon-theme`/`mezon-widgets` sit at the UI base; `mezon-canvas`/`mezon-webview` in the middle; `mezon-ui` aggregates; `mezon-app` is the binary host. `mezon-mcp`/`mezon-cli`/`mmn-client`/`mezon-stream` are peers that do not depend on UI.

## Mechanism

### Integration into `mezon-app` startup (Observed — `crates/mezon-app/src/main.rs`)

1. Line 90: `mezon_cli::try_run(args)` — CLI mode early-exit (returns `Some(exit_code)` → exit; `None` → start GUI). `is_cli_invocation` distinguishes from `mezonapp://` deep-links.
2. Line 304: create `mcp_cmd_tx/rx` channel.
3. Line 352/364 (Linux): `mezon_webview::init_gtk()` + `pump_gtk_events()` — WebKit2GTK needs a GTK main loop pumped inside the GPUI event loop.
4. `mezon-ui::init` → `mezon_canvas::init(cx)` (`mezon-ui/src/lib.rs:108`) + `mezon_widgets::input::init`.

### MCP runtime (`crates/mezon-app/src/mcp.rs`)

`McpRuntime::start` builds `McpController` + `ControlServer` (local IPC socket for CLI), `set_backend` attaches `AppApi` + UI command channel. `register_mcp_server_hooks` (`main.rs:894`) connects the MCP controller to UI actions.

### Canvas navigation injection

`mezon-ui/src/canvas_navigation.rs:7` `init(cx)` registers `CanvasNavigationHooks` (`navigate_to_canvas`, `active_canvas_id`, `confirm_delete_canvas`). The hook maps `route.canvas_id` → `Route::Canvas { clan_id, channel_id, canvas_id }` and calls `crate::router::navigate`. This lets `mezon-canvas` stay decoupled from the UI router via a **global-hooks pattern** (`GlobalCanvasNavigation`).

### Channel app webview lifecycle

`mezon-ui/src/channel_app/mod.rs` manages `ChannelAppWebView` (create/resize/destroy/reload/go_back) via a global registry `GlobalChannelAppWindows(HashMap<i64, WindowHandle>)`.

### Stream session

`mezon-store/src/stream.rs:10` holds `session: Option<StreamSession>` and exposes `StreamEvent` to UI.

### Wallet aggregation

`mezon-store/src/wallet.rs` composes `MmnClient` + `DongClient` + `ZkClient` + `IndexerClient` into `WalletClients`.

## Motivation

- **Observed:** `MMN_PARITY.md` documents the requirement to match the JS wallet implementation, justifying a dedicated `mmn-client` crate with parity test vectors.
- **Strong inference:** Extracting `mezon-theme`/`mezon-widgets` enables reuse between `mezon-ui` and the new `mezon-canvas` without forcing `mezon-canvas` to depend on the large `mezon-ui`.
- **Strong inference:** `mezon-mcp`/`mezon-cli` are isolated because they are external-facing surfaces (AI agents, CLI) with different dependency profiles (`rmcp`, `axum`, `clap`) that should not pollute the GUI binary's hot path.
- **Hypothesis:** `mezon-stream` is separate because WebRTC/livekit pulls in heavy native deps and a dedicated OS thread + tokio runtime that must not block the UI executor.

## Impact

- **Consumers:** `mezon-ui` now re-exports from `mezon-theme`/`mezon-widgets`/`mezon-canvas`/`mezon-webview`; `mezon-store` consumes `mezon-stream`/`mmn-client`; `mezon-app` consumes `mezon-cli`/`mezon-mcp`.
- **Developer workflow:** new UI features should use `mezon-theme` + `mezon-widgets` instead of hand-rolling input/button rendering. New external integrations go through `mezon-mcp`.
- **Build:** workspace `resolver = "2"`, `edition = "2024"`. New platform-specific deps: `webkit2gtk = "=2.0.1"` (pinned, Linux), `wry = "0.47"`.

## Trade-offs

- **Benefit:** clear separation of concerns; `mezon-canvas` can evolve its TipTap parser without touching `mezon-ui`; `mmn-client` crypto is isolated and testable.
- **Cost:** more crates to maintain; concurrency model is now non-uniform (see below).
- **Compatibility:** `mezon-active-windows` exclusion is a deliberate regression (rich-presence deferred) — documented, not a bug.
- **Failure mode:** `webkit2gtk` pinning makes Linux webview upgrades fragile.
- **Rollback:** each crate is independently removable by reverting its `Cargo.toml` addition and the integration points in `mezon-app`/`mezon-ui`/`mezon-store`.

## Verification

- `Cargo.toml` workspace `members` list (Observed) includes all 8 new crates and the exclusion comment for `mezon-active-windows`.
- `crates/mezon-ui/src/lib.rs:108` calls `mezon_canvas::init(cx)`.
- `crates/mezon-app/src/main.rs:90` calls `mezon_cli::try_run`.
- No cyclic dependencies detected by tracing `Cargo.toml` `[dependencies]` + `lib.rs` `pub use`/`use` statements.

## Source-reading guide

1. `Cargo.toml` — workspace `members` (lines ~3-50) and the `mezon-active-windows` exclusion comment.
2. `crates/mezon-app/src/main.rs:90` — `mezon_cli::try_run` (CLI early-exit).
3. `crates/mezon-app/src/main.rs:304,352,364` — MCP channel + Linux GTK init.
4. `crates/mezon-app/src/mcp.rs` — `McpRuntime::start`, `ControlServer`, `set_backend`.
5. `crates/mezon-ui/src/lib.rs:108` — `mezon_canvas::init`.
6. `crates/mezon-ui/src/canvas_navigation.rs:7` — `CanvasNavigationHooks` global pattern.
7. `crates/mezon-ui/src/channel_app/mod.rs:84` — `GlobalChannelAppWindows` registry.
8. `crates/mezon-store/src/stream.rs:10` — `StreamSession` holder.
9. `crates/mezon-store/src/wallet.rs:106-109` — `WalletClients` composition.
10. Each new crate's `src/lib.rs` for the public API surface.

## Learning

- The **global-hooks pattern** (`GlobalCanvasNavigation`) is a reusable way to let a leaf crate (`mezon-canvas`) trigger navigation without depending on the router crate — the host (`mezon-ui`) injects callbacks at init.
- **Dual-storage persistence** (debug JSON / release keyring) via `#[cfg]` is a clean way to keep secrets out of plaintext in dev while using OS secure storage in production — but it creates two code paths that tests can only cover one at a time.

## Unknowns

- Why does `mezon-stream` spawn an OS thread + a new tokio runtime instead of reusing `mezon-client`'s `transport_runtime`? (Hypothesis: WebRTC blocking calls + frame pump cadence; not confirmed.)
- Is the `webkit2gtk = "=2.0.1"` pin required for a specific API, or just cautious version locking?
