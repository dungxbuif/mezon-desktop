# Store domain expansion & architecture standardization

**Group:** #5 — Store domain expansion
**Significance:** Critical
**Commits:** `41a07a5`, `f196e7e`, `6acec35`, `bcb6288`, `9a70598`, `1660ee3`, `8b886cf` + 15 new module additions
**Areas:** `crates/mezon-store/src/{wallet,channel_media,channel_role_permissions,channel_settings,channel_users,compose,ogp,webhook,canvas,audit_log,auto_update,clan_load,events,notification_*,quick_menu,stream,ui_state}.rs`, `channel.rs`, `roles.rs`, `messages.rs`, `build.rs`

## Change

`mezon-store` expanded from a chat-core store (~10 modules) to a **full domain store** with 15+ new modules covering wallet, media, permissions, compose, OGP, webhooks, canvas, audit, notifications, stream, and UI state. A **breaking change** to `ClanRoles` was introduced. A `KeyedCache` standardization was applied across modules.

## Before

At `5a161c5`, `mezon-store` had: `channel.rs`, `messages.rs`, `roles.rs`, `clan.rs`, `user.rs`, `connection.rs`, `pinned.rs`, `threads.rs`, `search.rs`, `attachments.rs`, and a few others. Caching was ad-hoc (each module rolled its own `HashMap`-based cache). `ClanRoles` was a flat struct. There was no wallet, no OGP, no webhook, no audit log, no compose, no canvas, no stream store.

## After

### New modules (Observed — `crates/mezon-store/src/`)

| Module                            | Role                                                           |
| --------------------------------- | -------------------------------------------------------------- |
| `wallet.rs` + `wallet_persist.rs` | MMN wallet state + key storage (see wallet report)             |
| `channel_media.rs`                | Media channel metadata (files, images, videos)                 |
| `channel_role_permissions.rs`     | Per-role permission matrix per channel                         |
| `channel_settings.rs`             | Channel settings (name, topic, slowmode, etc.)                 |
| `channel_users.rs`                | Member list per channel (distinct from clan members)           |
| `compose.rs`                      | Draft message state (text, attachments, mentions) per channel  |
| `ogp.rs`                          | Open Graph Protocol link previews (URL → title/desc/image)     |
| `webhook.rs`                      | Channel webhook configuration + delivery state                 |
| `canvas.rs`                       | Canvas document references + sync state                        |
| `audit_log.rs`                    | Clan audit log entries (role changes, kicks, deletes)          |
| `auto_update.rs`                  | Update channel state (version, download progress)              |
| `clan_load.rs`                    | Clan data loading/refresh orchestration                        |
| `events.rs`                       | Central event bus (`StoreEvent`)                               |
| `notification_*.rs`               | Notification settings + state (multiple files)                 |
| `quick_menu.rs`                   | Quick-action menu state                                        |
| `stream.rs`                       | WebRTC stream session state (see voice report)                 |
| `ui_state.rs`                     | Ephemeral UI state (selected channel, scroll position, modals) |

### `ClanRoles` breaking change (`roles.rs`)

- **Before:** `ClanRoles { owner, admins: Vec, members: Vec }` (flat lists).
- **After:** `ClanRoles { owner_id, roles: HashMap<RoleId, Role> }` where `Role` has `permissions: PermissionSet`.
- This is a **schema-breaking change**: persisted clan data from before this commit cannot be deserialized as `ClanRoles` without migration. (Observed: no migration file found — see Unknowns.)

### `KeyedCache` standardization

- New `KeyedCache<K, V>` struct in `crates/mezon-store/src/lib.rs` (Observed).
- Replaces ad-hoc `HashMap` caches across modules.
- Provides: `get_or_insert`, `invalidate`, `invalidate_all`, TTL support.
- Adopted by: `channel.rs`, `messages.rs`, `channel_users.rs`, `channel_settings.rs`, `ogp.rs`, `webhook.rs`.

### Central event bus (`events.rs`)

- `StoreEvent` enum aggregates events from all modules: `ChannelLoaded`, `MessageReceived`, `WalletUpdated`, `StreamStarted`, `CanvasSynced`, `AuditLogAppended`, etc.
- `mezon-ui` subscribes to `StoreEvent` via `mezon-store::subscribe()` instead of per-module subscriptions.

### `build.rs` additions

- New `build.rs` in `mezon-store` (Observed) — generates code from `mezon-proto` definitions (likely for `StoreEvent` variants and request/response types). Reduces hand-written boilerplate.

## Mechanism

### `KeyedCache` pattern (Observed)

```rust
pub struct KeyedCache<K, V> {
    map: HashMap<K, V>,
    ttl: Option<Duration>,
    inserted_at: HashMap<K, Instant>,
}
impl<K, V> KeyedCache<K, V> {
    pub fn get_or_insert(&mut self, key: K, loader: impl FnOnce() -> V) -> &mut V;
    pub fn invalidate(&mut self, key: &K);
    pub fn invalidate_all(&mut self);
}
```

Modules use it as: `channels: KeyedCache<ChannelId, Channel>` instead of raw `HashMap`.

### Event bus flow (Observed)

```
Module (e.g., wallet.rs)
   │
   ↓ WalletEvent::BalanceUpdated
events.rs::StoreEvent::Wallet(BalanceUpdated)
   │
   ↓ broadcast
mezon-ui subscribers (sidebar, modal, etc.)
```

## Motivation

- **Observed:** The 15 new modules map 1:1 to new features (wallet, canvas, stream, OGP, webhooks, audit log) — the store grew because the app grew.
- **Strong inference:** `KeyedCache` standardization reduces duplicated cache logic and makes invalidation consistent — a maintainability refactor that piggybacked on the feature work.
- **Strong inference:** The central `StoreEvent` bus replaces N per-module subscriptions with 1, simplifying the UI's update path.
- **Strong inference:** `ClanRoles` was restructured to support per-role permissions (needed by `channel_role_permissions.rs`) — the old flat-list model couldn't express "role X can do Y in channel Z."
- **Hypothesis:** `build.rs` codegen from proto ensures store types stay in sync with the wire protocol.

## Impact

- **Consumers:** `mezon-ui` now imports from ~25 store modules instead of ~10; the subscription model changed from per-module to central bus.
- **Persistence:** `ClanRoles` schema change means old persisted data is incompatible. (Migration status unknown — see Unknowns.)
- **Performance:** `KeyedCache` with TTL reduces memory growth from unbounded caches.
- **Maintainability:** standardization makes the store more uniform; new modules follow the `KeyedCache` + `StoreEvent` pattern.

## Trade-offs

- **Benefit:** feature-complete store; consistent caching; single event bus; proto-driven types.
- **Cost:** `ClanRoles` is a breaking change with no observed migration — existing users may lose clan data on upgrade.
- **Risk:** the central `StoreEvent` bus could become a bottleneck if every UI widget subscribes to every event (fan-out). Filtering at the subscriber is needed.
- **Rollback:** individual modules are removable; `ClanRoles` rollback requires a reverse migration.

## Verification

- 15+ new module files exist in `crates/mezon-store/src/` (Observed via `ls`).
- `KeyedCache` defined in `crates/mezon-store/src/lib.rs` (Observed).
- `StoreEvent` enum in `events.rs` aggregates module events (Observed).
- `ClanRoles` restructured to `HashMap<RoleId, Role>` (Observed in `roles.rs`).
- `build.rs` exists in `mezon-store` (Observed).
- **Not verified:** migration logic for `ClanRoles`; whether all modules actually adopted `KeyedCache` or only some.

## Source-reading guide

1. `crates/mezon-store/src/lib.rs` — `KeyedCache` definition, module declarations.
2. `crates/mezon-store/src/events.rs` — `StoreEvent` enum, `subscribe()`.
3. `crates/mezon-store/src/roles.rs` — `ClanRoles` new structure (breaking change).
4. `crates/mezon-store/src/channel_role_permissions.rs` — per-role permission matrix.
5. `crates/mezon-store/src/compose.rs` — draft message state.
6. `crates/mezon-store/src/ogp.rs` — OGP link preview cache.
7. `crates/mezon-store/src/audit_log.rs` — clan audit log.
8. `crates/mezon-store/src/build.rs` — codegen from proto.
9. `crates/mezon-store/src/wallet.rs` — wallet store (see wallet report).

## Learning

- **Standardizing cache infrastructure** (`KeyedCache`) alongside feature work is efficient but means the refactor and the features land together — harder to review and bisect.
- **A central event bus** is cleaner than N per-module subscriptions but shifts complexity from "many subscriptions" to "one bus with many event types"; the trade-off favors the bus when modules are numerous.
- **Schema-breaking changes without observed migrations** are a red flag — always check for migration files when a persisted struct's shape changes.

## Unknowns

- **Is there a migration for `ClanRoles`?** (No migration file found in the diff. Hypothesis: the app re-fetches clan data from the server on first run after upgrade, discarding the old local cache. Not confirmed.)
- Does `KeyedCache` TTL get applied uniformly, or do some caches opt out? (Hypothesis: opt-out per module.)
- Is `StoreEvent` broadcast synchronous or async? (Hypothesis: async, via a channel; not confirmed.)
- Does `build.rs` regenerate types on every build, or only when proto changes? (Hypothesis: every build, via `build.rs` timestamp check.)
