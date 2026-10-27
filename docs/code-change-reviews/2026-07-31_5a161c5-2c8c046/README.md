# Code change review — 2026-07-31 (5a161c5 → 2c8c046)

**Status:** PASS
**Review date:** 2026-07-31
**Reviewer:** GitHub Copilot (automated, `track-codebase-evolution` skill)

## Range

|       | Short SHA | Full SHA                                   | Timestamp (author TZ)     |
| ----- | --------- | ------------------------------------------ | ------------------------- |
| Start | `5a161c5` | `5a161c5d3f28f99efb14d2faffb03ccaa02bb363` | 2026-07-15 22:00:22 -0700 |
| End   | `2c8c046` | `2c8c0462b8f6be2705d53b79a6f10c2a0bf98b28` | 2026-07-30 09:00:11 -0700 |

- **Branch:** `develop`
- **Repository:** `mezonai/mezon-desktop` (`https://github.com/mezonai/mezon-desktop.git`)
- **Ancestry:** `5a161c5` IS ancestor of `2c8c046` (verified via `git merge-base --is-ancestor`)
- **Merge:** fast-forward, no conflicts. Local `develop` was 303 commits behind `mezon/develop`.
- **Scale:** 303 commits · 435 files changed · +118,519 / −7,121 · version 0.1.26 → 0.1.30

## Five-minute summary

This 14-day sprint is an **architectural leap** that turns `mezon-desktop` from a basic chat client into a multi-capability desktop app. The most consequential changes:

1. **Workspace split into 8 new crates** (`mezon-theme`, `mezon-widgets`, `mezon-canvas`, `mezon-webview`, `mezon-stream`, `mezon-mcp`, `mezon-cli`, `mmn-client`), organized into 4 functional layers: UI primitives, UI features, runtime/media, and external interfaces. Dependencies are one-directional with no cycles.
2. **Transport resilience**: TCP-first with HTTP fallback (only when socket closed — avoids duplicate non-idempotent sends) + token renewal backoff (30s → 15min ×2).
3. **MMN blockchain wallet**: full `WalletStore` + `mmn-client` (Ed25519 signing, JSON-RPC, ZK proofs, indexer) with dual-storage persistence (debug JSON / release OS keyring).
4. **MCP AI integration**: `mezon-mcp` exposes the app to AI agents over HTTP/stdio + local IPC control socket, with `read_only` gating and local-origin enforcement.
5. **Clan administration system**: roles with permissions, channel permission overrides (allow/deny/neutral), community settings, audit log, webhooks, archived channels, categories.
6. **Voice/stream channel maturity**: stable pagination (swap-not-reorder), device select, screen share (incl. Linux Wayland portal), PIP, fullscreen, WebRTC viewer session.
7. **Canvas editor**: TipTap/Quill rich-text rendered in pure GPUI, integrated as first-class message content type with router navigation.
8. **Channel mini-apps**: `wry`-based webview with HTTPS-only, media-deny, navigation-origin lock, Windows WebView2 init queue.
9. **Message actions breadth**: poll, pin/unpin confirm, ephemeral, buzz, share location/contact, send token, cross-message text selection, context-menu clipboard, OGP link preview.
10. **UI primitives foundation**: new `TextArea` (custom GPUI element, full keyboard semantics, undo/redo, word boundary) + `MentionInput` + `mezon-theme` token bridge to Zed's `theme` crate.

## Change groups (ordered by significance)

| #   | Group                                                      | Significance | Deep report                                                |
| --- | ---------------------------------------------------------- | ------------ | ---------------------------------------------------------- |
| 1   | New crate architecture & workspace split                   | Critical     | [architecture-new-crates.md](architecture-new-crates.md)   |
| 2   | Transport resilience: HTTP fallback + token renewal        | Critical     | [transport-resilience.md](transport-resilience.md)         |
| 3   | MMN wallet & blockchain client                             | Critical     | [wallet-mmn-client.md](wallet-mmn-client.md)               |
| 4   | MCP AI integration & CLI                                   | Critical     | [mcp-ai-integration.md](mcp-ai-integration.md)             |
| 5   | Store domain expansion & architecture standardization      | Critical     | [store-domain-expansion.md](store-domain-expansion.md)     |
| 6   | Voice/stream channel & screen share                        | Critical     | [voice-stream-channel.md](voice-stream-channel.md)         |
| 7   | Clan administration (roles, permissions, community, audit) | Critical     | [clan-administration.md](clan-administration.md)           |
| 8   | Canvas editor integration                                  | Significant  | [canvas-editor.md](canvas-editor.md)                       |
| 9   | Channel mini-app webview                                   | Significant  | [channel-app-webview.md](channel-app-webview.md)           |
| 10  | Message actions & composer primitives                      | Significant  | [message-actions-composer.md](message-actions-composer.md) |
| 11  | UI primitives: TextArea, widgets, theme                    | Significant  | [ui-primitives.md](ui-primitives.md)                       |
| 12  | Notification, inbox, profile, sidebar, media channel       | Significant  | [ui-features-misc.md](ui-features-misc.md)                 |

Supporting and mechanical changes (i18n, CI, lockfile, formatting, vendored libspa tests) are recorded in [coverage.md](coverage.md) without a deep dive.

## Suggested reading paths

### 5-minute

Read this README's "Five-minute summary" and "Change groups" table.

### 30-minute

Read [architecture-new-crates.md](architecture-new-crates.md) (the workspace split), [transport-resilience.md](transport-resilience.md) (HTTP fallback trade-offs), and [wallet-mmn-client.md](wallet-mmn-client.md) (the riskiest new surface).

### Deep review

Read all Critical reports in order (1–7), then the Significant reports (8–12). Use the source-reading guides at the end of each report to verify claims in code. Cross-check [evidence.md](evidence.md) for the commit/path inventory and [coverage.md](coverage.md) for group-to-commit mapping.

## Links

- **Evidence:** [evidence.md](evidence.md)
- **Coverage:** [coverage.md](coverage.md)
- **Quality review:** [quality-review.md](quality-review.md)
- **Previous review:** none (first review in this journal)
- **Relevant project docs:** [`docs/01-architecture-overview.md`](../../01-architecture-overview.md), [`docs/05-migration-plan.md`](../../05-migration-plan.md), [`docs/06-auth-flow.md`](../../06-auth-flow.md), [`docs/16-transport-deep-dive.md`](../../16-transport-deep-dive.md)

## Exclusions, limitations, and unresolved questions

- **No prior review baseline** — this is the first review; the start SHA was chosen as the local `develop` HEAD before the merge.
- **UI analysis is after-state inferred** — the UI subagent lacked direct `git show` access for before/after diffs on `mezon-ui`; before-states are inferred from module/symbol structure. See [quality-review.md](quality-review.md) for the impact.
- **Tray/SNI not located in `mezon-ui`** — likely in `mezon-native`/`mezon-app`; out of scope for this review.
- **i18n full audit incomplete** — only `tt.json` was spot-checked; 10 other locale files need verification for new keys.
- **Concurrency model non-uniformity** across `mezon-stream` (OS thread + tokio), `mezon-mcp` (shared `transport_runtime`), `mezon-cli` (new runtime) — documented as a risk but not load-tested.
- **Unresolved questions:**
  - Is `dev-wallet.json` in `.gitignore`? (wallet persistence leak risk)
  - Is `baked_env.rs` (generated, contains secrets) in `.gitignore`?
  - Does the server support combined user+action filtering for audit log, or is client-side filtering loading all logs?
  - Does `HttpFallbackSession.base_url` ever resolve to `http://` (plaintext token risk)?
