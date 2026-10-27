# Evidence — review 2026-07-31 (5a161c5 → 2c8c046)

## Range and Git identity

- **Repository:** `mezonai/mezon-desktop` — `https://github.com/mezonai/mezon-desktop.git` (remote `mezon`)
- **Branch:** `develop`
- **Start:** `5a161c5d3f28f99efb14d2faffb03ccaa02bb363` — 2026-07-15 22:00:22 -0700 — "Merge pull request #92 from mezonai/feat/channel-list-file-attachments"
- **End:** `2c8c0462b8f6be2705d53b79a6f10c2a0bf98b28` — 2026-07-30 09:00:11 -0700 — "Merge pull request #178 from duocphungvanNCC/fix/ui_setting_profile"
- **Ancestry:** verified — `git merge-base --is-ancestor 5a161c5 2c8c046` succeeds.
- **Merge method:** `git merge --ff-only mezon/develop` — fast-forward, no conflicts. Local `develop` was 303 commits behind `mezon/develop`.
- **Manifest generator:** `.agents/skills/track-codebase-evolution/scripts/collect_changes.py` (output at `/tmp/mezon-manifest.json`, 505 KB).

## Diff totals

```
435 files changed, 118519 insertions(+), 7121 deletions(-)
```

Status counts (Observed):

| Status                           | Count |
| -------------------------------- | ----- |
| A (added)                        | 206   |
| M (modified)                     | 221   |
| D (deleted)                      | 1     |
| R (renamed, 77%–100% similarity) | 7     |

## Churn by crate / top-level dir

| Area                                                  | Files | +Lines | −Lines |
| ----------------------------------------------------- | ----- | ------ | ------ |
| `crates/mezon-ui`                                     | 190   | 58,850 | 4,450  |
| `crates/mezon-store`                                  | 59    | 19,704 | 1,033  |
| `crates/vendor`                                       | 43    | 11,309 | 70     |
| `crates/mezon-canvas` (new)                           | 9     | 7,176  | 0      |
| `crates/mezon-client`                                 | 15    | 5,349  | 609    |
| `crates/mezon-mcp` (new)                              | 11    | 2,814  | 0      |
| `crates/mezon-widgets` (new)                          | 5     | 2,223  | 0      |
| `crates/mezon-native`                                 | 10    | 2,138  | 304    |
| `crates/mmn-client` (new)                             | 10    | 1,656  | 0      |
| `crates/mezon-voice`                                  | 12    | 1,132  | 304    |
| `crates/mezon-app`                                    | 3     | 777    | 28     |
| `crates/mezon-updater`                                | 2     | 706    | 54     |
| `crates/mezon-webview` (new)                          | 7     | 599    | 0      |
| `crates/mezon-stream` (new)                           | 4     | 563    | 0      |
| `crates/mezon-theme` (new)                            | 2     | 360    | 0      |
| `crates/mezon-cli` (new)                              | 6     | 316    | 0      |
| `assets` (i18n + audio)                               | 11    | 361    | 82     |
| `Cargo.lock`                                          | 1     | 1,407  | 126    |
| `.github` (CI)                                        | 2     | 440    | 6      |
| `scripts`                                             | 6     | 309    | 2      |
| `packaging`                                           | 10    | 76     | 0      |
| `MMN_PARITY.md` (new)                                 | 1     | 88     | 0      |
| `Cargo.toml` (workspace)                              | 1     | 30     | 2      |
| `crates/mezon-audio`                                  | 3     | 67     | 2      |
| `mezon-protocol` (submodule)                          | 1     | 1      | 1      |
| `justfile`, `deny.toml`, `.gitignore`, `.env.example` | 4     | 14     | 3      |

## New crates (8)

All added in range (Observed — `git diff --name-status | grep "^A.*Cargo.toml"`):

| Crate           | Path                              | Layer                                      |
| --------------- | --------------------------------- | ------------------------------------------ |
| `mezon-canvas`  | `crates/mezon-canvas/Cargo.toml`  | UI feature (TipTap/Quill editor in GPUI)   |
| `mezon-cli`     | `crates/mezon-cli/Cargo.toml`     | External interface (CLI entry + MCP stdio) |
| `mezon-mcp`     | `crates/mezon-mcp/Cargo.toml`     | External interface (MCP server for AI)     |
| `mezon-stream`  | `crates/mezon-stream/Cargo.toml`  | Runtime/media (WebRTC viewer)              |
| `mezon-theme`   | `crates/mezon-theme/Cargo.toml`   | UI primitives (theme tokens, Zed bridge)   |
| `mezon-webview` | `crates/mezon-webview/Cargo.toml` | UI feature (channel app webview, wry)      |
| `mezon-widgets` | `crates/mezon-widgets/Cargo.toml` | UI primitives (Input/Button/Icon)          |
| `mmn-client`    | `crates/mmn-client/Cargo.toml`    | Domain client (MMN blockchain)             |

## Top 30 files by churn

| +Lines | −Lines | Status | Path                                                                |
| ------ | ------ | ------ | ------------------------------------------------------------------- |
| 4,433  | 0      | A      | `crates/mezon-canvas/src/editor.rs`                                 |
| 2,638  | 221    | M      | `crates/mezon-store/src/channel.rs`                                 |
| 2,257  | 0      | A      | `crates/vendor/libspa/tests/pod.rs`                                 |
| 2,176  | 186    | M      | `crates/mezon-store/src/messages.rs`                                |
| 1,933  | 0      | A      | `crates/vendor/libspa/src/pod/deserialize.rs`                       |
| 1,914  | 0      | A      | `crates/mezon-widgets/src/input.rs`                                 |
| 1,771  | 50     | M      | `crates/mezon-store/src/roles.rs`                                   |
| 1,645  | 235    | M      | `crates/mezon-ui/src/chat/message/content.rs`                       |
| 1,643  | 316    | M      | `crates/mezon-client/src/transport_runtime.rs`                      |
| 1,591  | 290    | M      | `crates/mezon-ui/src/chat/voice.rs`                                 |
| 1,519  | 0      | A      | `crates/mezon-mcp/src/tools.rs`                                     |
| 1,477  | 50     | M      | `crates/mezon-ui/src/chat/message/channel_messages.rs`              |
| 1,471  | 0      | A      | `crates/mezon-ui/src/components/primitives/textarea.rs`             |
| 1,423  | 176    | M      | `crates/mezon-client/src/transport.rs`                              |
| 1,407  | 126    | M      | `Cargo.lock`                                                        |
| 1,391  | 0      | A      | `crates/mezon-ui/src/clan/settings/community_setting_page.rs`       |
| 1,335  | 0      | A      | `crates/mezon-ui/src/chat/clan_members_page.rs`                     |
| 1,297  | 0      | A      | `crates/mezon-ui/src/clan/settings/role_setting_page.rs`            |
| 1,272  | 0      | A      | `crates/vendor/libspa/src/pod/mod.rs`                               |
| 1,234  | 127    | M      | `crates/mezon-ui/src/chat/user_profile_popover.rs`                  |
| 1,216  | 0      | A      | `crates/mezon-store/src/channel_media.rs`                           |
| 1,138  | 0      | A      | `crates/mezon-ui/src/chat/message/transaction_history_modal.rs`     |
| 1,116  | 0      | A      | `crates/mezon-ui/src/chat/media_channel/event_detail_view.rs`       |
| 1,113  | 0      | A      | `crates/mezon-ui/src/chat/stream.rs`                                |
| 1,102  | 0      | A      | `crates/mezon-ui/src/chat/channel_settings/permission_overrides.rs` |
| 1,073  | 0      | A      | `crates/mezon-ui/src/chat/channel_settings/permissions_tab.rs`      |
| 1,038  | 0      | A      | `crates/mezon-ui/src/channel_app/mod.rs`                            |
| 1,035  | 0      | A      | `crates/mezon-ui/src/chat/clan_channels_page.rs`                    |
| 1,011  | 173    | M      | `crates/mezon-ui/src/chat/inbox/row.rs`                             |
| 993    | 0      | A      | `crates/mezon-ui/src/chat/media_channel/media_image_modal.rs`       |

## Commit inventory (303 commits)

The full chronological list is available via:

```bash
git log --oneline 5a161c5..2c8c046
```

Key feature-merge commits (Observed, selected for traceability):

| SHA       | PR/Theme                                                                   |
| --------- | -------------------------------------------------------------------------- |
| `2ccbb5f` | feat: introduce mezon cli and mcp integration for ai assistant             |
| `2801bc4` | Canvas editor                                                              |
| `5f09f0b` | feat(wallet): add MMN client, WalletStore, and wallet UI parity            |
| `8ce4aa9` | feat: HTTP socket fallback, token renewal, and connecting splash           |
| `68c0219` | feat(notifications): channel notification settings and push delivery       |
| `02aaebf` | feat(chat): create poll modal and poll message parity                      |
| `89aa87b` | feat(chat): ephemeral message send and composer UI                         |
| `71ecb58` | feat(chat): OGP link previews and Textarea composer primitive              |
| `bcdb9b2` | feat(chat): mention text field, connection store, and textarea extensions  |
| `6561b7e` | feat(chat): message buzz/share location and media CDN handling             |
| `71ffed5` | feat(chat): share contact, flash slash, and message UI parity fixes        |
| `1127693` | feat(chat): add cross-message text selection in channel messages           |
| `8b886cf` | feat(notifications): notification settings modal and sidebar context menus |
| `2cf25a2` | feat(sidebar): drag-to-reorder clans with persisted order                  |
| `9af3123` | feat(wallet): paginated transaction history modal parity                   |
| `f196e7e` | feat(permissions): channel role permissions, settings UI, and Linux icons  |
| `41a07a5` | feat: add clan community setting feature                                   |
| `8e76568` | feat: add channel app launch feature via wry                               |
| `e63d9e3` | feat(thread): leave flow, locked icons, and channel polish                 |
| `7486716` | feat(thread): auto-add users to thread on send and reaction                |
| `1660ee3` | feat(compose): persist per-channel message drafts across navigation        |
| `9a70598` | feat: implement clan audit log feature in read-only mode                   |
| `85c5854` | feat: add archived channel and category sorting feature                    |
| `3c0b500` | feat: implement clan role management feature and ui                        |
| `6acec35` | Channel timeline event                                                     |
| `bcb6288` | feat: Implement clan and chanel webhook management feature                 |
| `33ef428` | feat: add clan emoji - sticker and voice effect setting feature            |

## Tests, migrations, schemas, config inspected

- **Tests:** `crates/mmn-client/src/lib.rs` (test vectors: `address_from_user_id_matches_js`, `sign_transfer_by_key_matches_nacl_vector`, `sign_transfer_by_zk_wraps_pubkey_and_sig_like_js`, `serialize_transaction_matches_js_pipe_format`); `crates/mezon-store/src/channel_role_permissions.rs` (`snapshot_without_persisted_overrides_would_wipe_them_all`); `crates/vendor/libspa/tests/pod.rs` (vendored, mechanical).
- **Migrations:** No schema migrations found. Settings persist as JSON (`~/.config/mezon/settings.json`, atomic write). Wallet persistence is dual: debug → `config_dir/mezon/dev-wallet.json` (JSON), release → OS keyring (`SERVICE="mezon-desktop"`, `USERNAME="wallet"`).
- **Schemas:** `mezon-proto` bumped for `ClanJoin` and `ListSdTopic` fields (commit `528c9f3`). `RealtimeEvent` enum extended with `Unmute`, `SessionRefreshed`.
- **Config:** `crates/mezon-store/build.rs` `BAKED_ENV_VARS` extended with ~8 blockchain/treasury vars (`NX_CHAT_APP_MEZON_TREASURY_URL`, `NX_CHAT_APP_CONTRACT_ADDRESS`, `NX_CHAT_APP_MMN_API_URL`, `NX_CHAT_APP_INDEXER_API_URL`, `NX_CHAT_APP_ZK_API_URL`, `NX_CHAT_APP_DONG_SERVICE_API_URL`, …). Codegen `baked_env.rs` (no `cargo:rustc-env` — avoids secret leakage in build logs).
- **Dependencies:** `Cargo.lock` +1407/−126. New deps: `rmcp`, `axum`, `wry = "0.47"`, `webkit2gtk = "=2.0.1"` (pinned), `livekit`, `tokio-tungstenite = "0.29"`, `ed25519-dalek`, `bs58`, `zeroize`, `keyring` (release), `foyer` (hybrid cache).
- **CI:** `.github/workflows/build.yml` modified; `.github/workflows/release.yml` added (macOS release artifact). Commits `af1cf41`, `5d340ce`, `9a065db` fix Linux/Windows builds.
- **i18n:** 11 locale files updated (`assets/i18n/*.json`): de, en, es, it, jpn, kr, pt, ru, swe, tt, vi. New keys: `auditLog.*`, `transactionHistory.*`, `messageBuzz.*`, `message.poll.*`, `shareLocation.*`, `notificationSetting.*`, `clanRoles.permissionTitles.*`.

## Supporting and contradictory evidence

- **`MMN_PARITY.md`** (new, +88) — documents JS↔Rust parity requirements for the wallet client; corroborates the test-vector approach in `mmn-client`.
- **`mezon-active-windows` excluded from workspace** — `Cargo.toml` comment: "orphan (unused) + does not compile (Rust 2024 `unsafe extern` + core_foundation API mismatch). Re-add once fixed & wired for rich-presence." (Observed) — explains a deliberate exclusion, not a regression.
- **Vendored `libspa`** (+11,309, 43 files) — large churn but mechanical (vendored PipeWire bindings for Linux audio); recorded as Mechanical in coverage.

## Immutable source links

GitHub compare: `https://github.com/mezonai/mezon-desktop/compare/5a161c5...2c8c046`
