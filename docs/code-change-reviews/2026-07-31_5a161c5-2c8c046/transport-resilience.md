# Transport resilience: HTTP fallback + token renewal

**Group:** #2 — Transport resilience
**Significance:** Critical
**Commits:** `8ce4aa9` (HTTP fallback), `ccd6181` (token renewal), `12ddbc2` (MCP transport), `528c9f3` (proto bump)
**Areas:** `crates/mezon-client/src/transport.rs`, `transport_runtime.rs`, `mezon-store/src/connection.rs`, `mezon-proto/proto/mtproto/transport.proto`

## Change

The transport layer gained an **HTTP fallback path** alongside the existing MTProto TCP transport, plus a **token renewal flow** that re-authenticates without tearing down the session. The MCP server now has its own transport bridge to the same client.

## Before

At `5a161c5`, `mezon-client` used a single transport: an MTProto-style TCP framing layer (`transport_adapter.rs` + `abridged_tcp_adapter.rs`) wrapped by `transport_runtime.rs` (a tokio runtime + `Connection` actor). All client→server traffic went through one TCP socket. If the socket died, the whole client had to reconnect and re-handshake. There was no HTTP path. Token refresh was implicit (server pushed new tokens; client stored them).

## After

### HTTP fallback (`transport.rs`)

- New `HttpFallback` struct wraps a `reqwest`-style client (`http_client::Client`) and a base URL.
- `Transport` enum extended: `Tcp(AbridgedTcp) | Http(HttpFallback)`.
- `transport_runtime.rs` now tracks `transport_health: TransportHealth` (`Healthy | Degraded | Failed`) and a `failover_state` machine.
- On N consecutive TCP failures (Observed threshold = 3 in `transport_runtime.rs`), the runtime switches to `Http` and continues sending the same `RequestFrame`s serialized as JSON POSTs to `/api/v2/...`.
- Recovery: a background probe (`network_probe.rs`, pre-existing) pings the TCP endpoint; on success the runtime switches back to `Tcp` and resumes binary framing.

### Token renewal (`connection.rs`)

- `Connection::renew_token` (new method) sends a `RenewTokenRequest` with the current refresh token, receives a new `{access_token, refresh_token, expires_in}`, and **hot-swaps** the in-memory credentials without dropping the transport.
- The store emits a `ConnectionEvent::TokenRenewed` that `mezon-ui` observes to refresh the auth UI state.
- Renewal is triggered by either (a) a 401 response from the server, or (b) a timer set to `expires_in - 60s`.

### MCP transport bridge (`mezon-app/src/mcp.rs`)

- `McpController` holds a `Sender<RequestFrame>` to the same `transport_runtime` input channel that the GUI uses.
- This means **MCP requests and GUI requests share the same transport and the same auth token** — a single connection serves both surfaces.

## Mechanism

### Failover state machine (Observed — `transport_runtime.rs`)

```
Healthy ──(3 consecutive TCP failures)──> Degraded ──(probe succeeds)──> Healthy
   │                                          │
   │                                  (probe fails / timeout)
   │                                          ↓
   └──────────────────────────────────────> Failed (HTTP only)
```

- In `Degraded`, the runtime sends via HTTP but keeps the TCP socket open for probe recovery.
- In `Failed`, TCP is closed; only HTTP is used until a successful probe reopens TCP.

### Token renewal sequence (Observed — `connection.rs`)

```
GUI/MCP ──> RequestFrame ──> transport_runtime ──> server
                                              │
                                              ↓ 401
                                       renew_token()
                                              │
                                              ↓ new tokens
                                       ConnectionEvent::TokenRenewed
                                              │
                                              ↓
                                       UI refreshes auth state
                                       (retry original request)
```

## Motivation

- **Observed:** `transport.proto` adds `HttpFallbackConfig` and `RenewTokenRequest`/`Response` messages — the protocol explicitly supports these flows.
- **Strong inference:** HTTP fallback exists for environments where raw TCP is blocked (corporate proxies, restrictive NAT, cloud egress filtering) — a common deployment concern for desktop chat clients.
- **Strong inference:** Hot-swap token renewal avoids the cost of a full reconnect+handshake on every token expiry (which at 1-hour tokens would be visible to users as periodic freezes).
- **Hypothesis:** Sharing the transport between MCP and GUI avoids double-auth and double-connection overhead when an AI agent is driving the app concurrently with the user.

## Impact

- **Reliability:** the client now has two independent paths to the server; a single socket failure no longer takes the app offline.
- **Auth:** token renewal is now proactive (timer-based) rather than reactive-only, reducing 401 storms.
- **MCP:** AI agents can issue requests without opening a second authenticated session.
- **Performance:** HTTP path is slower (JSON + TLS handshake per request, no multiplexing) — acceptable for fallback, not for steady state.

## Trade-offs

- **Benefit:** resilience in hostile networks; no user-visible reconnect on token expiry.
- **Cost:** two code paths to maintain (binary TCP framing vs JSON HTTP); the failover state machine adds complexity and a new class of bugs (state drift between transports).
- **Risk:** if the HTTP endpoint and TCP endpoint disagree on session state (e.g., one thinks the session is expired), requests can fail in confusing ways. (Not load-tested — see quality-review.md.)
- **Rollback:** revert `transport.rs` `Http` variant + `transport_runtime.rs` failover; the TCP-only path remains functional.

## Verification

- `transport.proto` defines `HttpFallbackConfig`, `RenewTokenRequest`, `RenewTokenResponse` (Observed).
- `transport_runtime.rs` contains `TransportHealth` enum and `failover_state` field (Observed).
- `connection.rs` contains `renew_token` method and `ConnectionEvent::TokenRenewed` variant (Observed).
- `mezon-app/src/mcp.rs` `McpController` holds a `Sender<RequestFrame>` to `transport_runtime` (Observed).
- **Not verified:** end-to-end failover behavior under packet loss; token renewal under concurrent 401s.

## Source-reading guide

1. `crates/mezon-client/src/transport.rs` — `Transport` enum, `HttpFallback` struct.
2. `crates/mezon-client/src/transport_runtime.rs` — `TransportHealth`, `failover_state`, probe loop.
3. `crates/mezon-client/src/connection.rs` — `renew_token`, `ConnectionEvent::TokenRenewed`.
4. `crates/mezon-client/src/network_probe.rs` — probe logic (pre-existing, reused for recovery).
5. `crates/mezon-proto/proto/mtproto/transport.proto` — `HttpFallbackConfig`, `RenewTokenRequest`.
6. `crates/mezon-app/src/mcp.rs` — `McpController` transport bridge.

## Learning

- A **failover state machine with a probe-driven recovery** is the standard pattern for dual-transport clients; the key is keeping the probe cheap and the state transitions idempotent.
- **Hot-swapping credentials** is safer than reconnecting because it preserves in-flight request queues — but it requires the server to support token refresh on the same session.

## Unknowns

- What is the exact TCP failure threshold (Observed as 3, but is it configurable)?
- Does the HTTP fallback path support the full request set, or only a subset? (Hypothesis: full set, since it serializes the same `RequestFrame`s — not confirmed.)
- Is the renewal timer per-session or global? (Hypothesis: per-session.)
