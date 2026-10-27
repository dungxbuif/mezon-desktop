# MCP AI integration & CLI

**Group:** #4 — MCP AI integration & CLI
**Significance:** Critical
**Commits:** `2ccbb5f` (cli+mcp crates), `ccd6181` (transport bridge), `12ddbc2` (hooks), `8ce4aa9` (shared transport)
**Areas:** `crates/mezon-mcp/*`, `crates/mezon-cli/*`, `crates/mezon-app/src/mcp.rs`, `crates/mezon-app/src/main.rs`

## Change

Mezon now exposes an **MCP (Model Context Protocol) server** so external AI agents can drive the app (read channels, send messages, manage guilds), plus a **CLI** that can run the app headless or issue commands to a running instance via a local IPC control server.

## Before

At `5a161c5`, there was no MCP server, no CLI. The app was GUI-only, launched via `mezon-app`. There was no way for an external program (AI agent, script) to interact with a running Mezon instance except by simulating UI input.

## After

### `mezon-mcp` crate (new)

- Implements the MCP server protocol (uses `rmcp` crate + `axum` for HTTP transport).
- Two transports: **stdio** (for local AI agent integration) and **HTTP** (for remote agents).
- Exposes "tools" (MCP terminology for callable functions): `send_message`, `read_channel_history`, `list_channels`, `get_user_profile`, `create_canvas`, etc. (Observed tool list in `crates/mezon-mcp/src/tools.rs`).
- Each tool wraps a `RequestFrame` sent to the shared `transport_runtime` (same channel as GUI).

### `mezon-cli` crate (new)

- `try_run(args) -> Option<i32>` — entry point called from `mezon-app/src/main.rs:90`.
  - If args match a CLI command → execute and return `Some(exit_code)` (headless mode).
  - If args are empty or match GUI flags → return `None` (GUI starts normally).
- CLI commands (Observed): `mezon mcp serve` (start MCP server), `mezon wallet balance`, `mezon wallet send`, `mezon channel list`, `mezon message send`.
- When a GUI instance is already running, CLI commands are sent to it via the **ControlServer** local IPC socket (not executed in the new process).

### `McpRuntime` (`crates/mezon-app/src/mcp.rs`)

- `McpRuntime::start` builds:
  - `McpController` — holds `Sender<RequestFrame>` to `transport_runtime` + `Receiver<ApiResponse>`.
  - `ControlServer` — local IPC socket (Unix socket on macOS/Linux, named pipe on Windows) for CLI↔GUI communication.
- `set_backend` attaches `AppApi` (UI actions: navigate, open modal, focus channel) + the UI command channel.
- `register_mcp_server_hooks` (`main.rs:894`) connects MCP controller callbacks to UI actions (e.g., `send_message` tool → `mezon-ui` chat send).

## Mechanism

### MCP request flow (Observed)

```
AI Agent ──(stdio/HTTP)──> mezon-mcp server
                              │
                              ↓ tool call (e.g., send_message)
                          McpController
                              │
                              ↓ RequestFrame
                          transport_runtime ──> server
                              │
                              ↓ ApiResponse
                          McpController ──> AI Agent (tool result)
```

### CLI↔GUI flow (Observed)

```
mezon-cli (new process)
   │
   ├─ if no running GUI: execute command headlessly (own transport_runtime)
   └─ if GUI running: connect to ControlServer IPC socket
                       │
                       ↓ command
                    ControlServer ──> McpController ──> UI action / transport
                       │
                       ↓ result
                    mezon-cli prints result, exits
```

### Tool registration (Observed — `crates/mezon-mcp/src/tools.rs`)

Tools are registered with `rmcp`'s tool schema. Each tool has:

- `name`, `description` (for the AI agent's prompt).
- `input_schema` (JSON Schema for arguments).
- `handler: async fn(args) -> ToolResult`.

The handler builds a `RequestFrame`, sends it via `McpController`, awaits the response, and serializes the result as MCP tool output.

## Motivation

- **Observed:** `mezon-cli` README and `mezon-mcp` docstrings describe the goal: "let AI agents and scripts operate Mezon programmatically."
- **Strong inference:** MCP is chosen over a custom API because it's an emerging standard (Anthropic-backed) that AI agents (Claude, etc.) can use without Mezon-specific integration.
- **Strong inference:** The CLI shares the MCP transport because the two surfaces have identical needs (send requests to a running app, get results) — reusing `McpController` avoids duplicating the request/response plumbing.
- **Hypothesis:** stdio transport is for local agent integration (Claude Code, etc.); HTTP is for remote/orchestrated agents.

## Impact

- **Programmability:** Mezon is now scriptable and AI-controllable. This is a major capability expansion.
- **Security:** the ControlServer IPC socket is local-only (Observed: Unix socket path under `~/.mezon/`). Any local process can connect. (See Unknowns — auth not confirmed.)
- **Architecture:** the MCP server runs in-process with the GUI (not a separate daemon), sharing the transport and auth. This simplifies auth but couples MCP lifetime to GUI lifetime.
- **Dependencies:** `rmcp`, `axum`, `clap` added.

## Trade-offs

- **Benefit:** standard AI-agent integration via MCP; CLI for automation; no separate daemon to manage.
- **Cost:** in-process MCP server means an MCP crash can affect the GUI; the local IPC socket is a new attack surface.
- **Risk:** if the ControlServer socket has no auth, any local user can issue commands (send messages, spend wallet) to a running Mezon instance. (Not confirmed — see Unknowns.)
- **Compatibility:** MCP protocol version pinned via `rmcp` version; future MCP spec changes require updating.
- **Rollback:** revert `mezon-mcp` + `mezon-cli` crates and the `main.rs:90` / `mcp.rs` integration; GUI-only mode is restored.

## Verification

- `crates/mezon-mcp/src/lib.rs` exports the MCP server builder (Observed).
- `crates/mezon-mcp/src/tools.rs` defines the tool list (Observed).
- `crates/mezon-cli/src/lib.rs` defines `try_run` (Observed).
- `crates/mezon-app/src/main.rs:90` calls `mezon_cli::try_run` (Observed).
- `crates/mezon-app/src/mcp.rs` defines `McpRuntime`, `McpController`, `ControlServer` (Observed).
- `crates/mezon-app/src/main.rs:894` calls `register_mcp_server_hooks` (Observed).
- **Not verified:** ControlServer socket authentication; MCP server behavior under concurrent tool calls; CLI command routing when GUI is running.

## Source-reading guide

1. `crates/mezon-cli/src/lib.rs` — `try_run`, CLI command dispatch.
2. `crates/mezon-mcp/src/lib.rs` — MCP server builder, transport selection.
3. `crates/mezon-mcp/src/tools.rs` — tool definitions and handlers.
4. `crates/mezon-app/src/main.rs:90` — CLI early-exit.
5. `crates/mezon-app/src/main.rs:894` — `register_mcp_server_hooks`.
6. `crates/mezon-app/src/mcp.rs` — `McpRuntime::start`, `ControlServer`, `set_backend`.

## Learning

- **Sharing the transport between GUI and MCP** is elegant (single auth, single connection) but creates a backpressure question: if an AI agent floods the transport, GUI requests stall. A priority queue or separate sub-channels would mitigate (not observed in the diff).
- **CLI↔GUI via local IPC** is the standard pattern for "single-instance" apps (Telegram, Discord do similar); the key is detecting an existing instance and routing to it rather than starting a second one.

## Unknowns

- **Does the ControlServer IPC socket require authentication?** (Critical security question. Hypothesis: no auth observed, but `mezon-native` control server security commit `4b54805` may add it — not confirmed.)
- What is the MCP tool authorization model? Can an AI agent call `wallet send` without user confirmation? (Hypothesis: there is a confirmation gate in `register_mcp_server_hooks`; not confirmed.)
- Is the MCP server enabled by default, or opt-in via config? (Hypothesis: opt-in, given the CLI command `mezon mcp serve`.)
- What happens if the AI agent sends a tool call while the GUI is showing a modal? (Hypothesis: queued; not confirmed.)
