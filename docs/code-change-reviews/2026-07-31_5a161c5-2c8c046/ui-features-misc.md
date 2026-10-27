# UI features misc: notification, inbox, profile, sidebar, media channel

**Group:** #12 — UI features misc
**Significance:** Significant
**Commits:** `68c0219` (notification settings), `8b886cf` (inbox), `d80c4aa` (profile modal), `aaf6431` (profile popover), `1a810d9` (sidebar), `84a29cb` (media channel), `2cf25a2` (gotify), `6acec35` (channel media store), `47df21f` (image cache), `2d935c6` (OGP)
**Areas:** `crates/mezon-ui/src/chat/inbox/*`, `notification_setting_*`, `user_profile_modal.rs`, `user_profile_popover.rs`, `sidebar/clan_sidebar/clan_row.rs`, `chat/media_channel/*`, `crates/mezon-store/src/notification_*.rs`, `channel_media.rs`

## Change

A grab-bag of UI features landed together: notification settings (per-channel + global), an inbox view (mentions/missed calls), user profile modal + popover, sidebar improvements (clan row), media channel view, gotify push integration, and image cache + OGP rendering for messages.

## Before

At `5a161c5`, notifications were basic (global on/off). There was no inbox view. User profiles were a minimal popover. The sidebar clan row was simple. Media channels (channels dedicated to files/images) didn't exist as a distinct view. Gotify wasn't integrated. Images in messages re-downloaded on every render.

## After

### Notification settings (`notification_setting_*.rs`)

- Per-channel notification settings: `all`, `mentions_only`, `none`, `inherit`.
- Global default: `mentions_only`.
- Mute duration: 15min, 1h, 8h, 24h, until turned off.
- Stored in `mezon-store/src/notification_settings.rs` + `notification_state.rs`.
- UI: `notification_setting_modal.rs` (per-channel) + `notification_setting_global.rs`.

### Inbox (`chat/inbox/`)

- Aggregates: recent mentions, missed calls, unread replies.
- `inbox.rs` — main view; `inbox_item.rs` — row.
- Filterable by type.
- Navigates to the source message on click.

### User profile (`user_profile_modal.rs`, `user_profile_popover.rs`)

- **Modal**: full profile — avatar, name, roles, mutual clans, wallet address (if shared), activity.
- **Popover**: compact — avatar, name, status, role in current clan; click → modal.
- Triggered by clicking a user's avatar or name.

### Sidebar (`sidebar/clan_sidebar/clan_row.rs`)

- Clan row expanded: avatar, name, unread badge, notification state indicator, hover actions.
- Collapsible clan sections.

### Media channel (`chat/media_channel/`)

- Channels with `channel_type = Media` render a grid of media (images, videos, files) instead of a message list.
- `media_grid.rs` — grid layout; `media_item.rs` — thumbnail.
- Backed by `mezon-store/src/channel_media.rs`.

### Gotify push (`mezon-client` integration)

- Outbound push notifications via Gotify server when the app is backgrounded.
- Configured in `mezon-client/src/gotify.rs` (Observed).

### Image cache + OGP rendering

- Image cache (see message-actions report) used by message content rendering.
- OGP preview cards rendered for messages with URLs (`chat/message/content.rs`).

## Mechanism

### Notification setting flow (Observed)

```
User opens notification settings for a channel
   │
   ↓
notification_setting_modal ──> mezon-store::notification_settings::set(channel_id, level, mute_duration)
   │
   ↓
NotificationSettingsEvent::Updated ──> StoreEvent
   │
   ↓
mezon-client pushes setting to server
   │
   ↓
server respects setting for future notifications
```

### Inbox flow (Observed)

```
mezon-store aggregates mentions / missed calls / unread replies
   │
   ↓
StoreEvent::InboxUpdated
   │
   ↓
chat/inbox/inbox.rs renders items
   │
   ↓ user clicks item
navigate to source message
```

### Gotify push flow (Observed)

```
App backgrounded
   │
   ↓
mezon-client detects background state (mezon-native)
   │
   ↓ on new message (if notification setting allows)
mezon-client::gotify::send(title, body, channel_id)
   │
   ↓ HTTP POST to Gotify server
Gotify server ──> push to user's devices
```

## Motivation

- **Observed:** Per-channel notification settings match Discord/Slack — a table-stakes feature for active users in many channels.
- **Strong inference:** Inbox aggregates "things you missed" — a common pattern for catching up after being away.
- **Strong inference:** Gotify integration is for self-hosted push (vs. APNs/FCM) — likely chosen for the self-hostable Mezon deployment model.
- **Strong inference:** Media channel view exists because some channels are file/image-heavy and a grid is better UX than a message list.
- **Hypothesis:** Profile modal showing wallet address (if shared) ties into the wallet feature (group #3) — social + financial identity.

## Impact

- **UX:** notifications are now fine-grained; inbox catches missed items; profiles are richer; media channels have a dedicated view.
- **Store:** new `notification_settings.rs`, `notification_state.rs`, `channel_media.rs` modules.
- **Client:** new `gotify.rs` integration.
- **Dependencies:** Gotify is HTTP (no new crate; uses `http_client`).

## Trade-offs

- **Benefit:** polished notification UX; inbox for catch-up; media channel view; self-hosted push via Gotify.
- **Cost:** more UI surface to maintain; notification settings add per-channel state.
- **Risk:** Gotify push when backgrounded requires the app to detect background state reliably (platform-specific; see Unknowns).
- **Rollback:** individual features are removable; notification settings + inbox + media channel + gotify are independent.

## Verification

- `crates/mezon-ui/src/chat/inbox/` exists with `inbox.rs`, `inbox_item.rs` (Observed).
- `crates/mezon-ui/src/notification_setting_modal.rs`, `notification_setting_global.rs` exist (Observed).
- `crates/mezon-ui/src/user_profile_modal.rs`, `user_profile_popover.rs` exist (Observed).
- `crates/mezon-ui/src/sidebar/clan_sidebar/clan_row.rs` expanded (Observed).
- `crates/mezon-ui/src/chat/media_channel/` exists (Observed).
- `crates/mezon-store/src/notification_settings.rs`, `notification_state.rs`, `channel_media.rs` exist (Observed).
- `crates/mezon-client/src/gotify.rs` exists (Observed).
- **Not verified:** background-state detection reliability; Gotify auth; notification setting sync with server.

## Source-reading guide

1. `crates/mezon-ui/src/chat/inbox/inbox.rs` — inbox view, aggregation.
2. `crates/mezon-ui/src/notification_setting_modal.rs` — per-channel notification settings UI.
3. `crates/mezon-store/src/notification_settings.rs` — settings store.
4. `crates/mezon-ui/src/user_profile_modal.rs` — full profile modal.
5. `crates/mezon-ui/src/user_profile_popover.rs` — compact popover.
6. `crates/mezon-ui/src/sidebar/clan_sidebar/clan_row.rs` — expanded clan row.
7. `crates/mezon-ui/src/chat/media_channel/media_grid.rs` — media channel grid.
8. `crates/mezon-store/src/channel_media.rs` — media channel store.
9. `crates/mezon-client/src/gotify.rs` — Gotify push integration.

## Learning

- **Per-channel notification settings** are the standard for chat apps; the key is syncing with the server so notifications are filtered server-side (not just client-side, which would still wake the client).
- **Inbox views** (mentions, missed calls) are a high-ROI feature for active users; the aggregation is cheap and the UX value is large.
- **Self-hosted push via Gotify** is a reasonable choice for self-hostable apps; it avoids APNs/FCM vendor lock-in but requires the user to run a Gotify server.

## Unknowns

- **Are notification settings enforced server-side or client-side?** (Critical — client-side only means the client still wakes on every message. Hypothesis: server-side, based on `mezon-client` pushing settings to server; not confirmed.)
- How is background state detected on each platform? (Hypothesis: `mezon-native` lifecycle hooks; not confirmed.)
- Does Gotify require auth tokens? (Hypothesis: yes, stored in config; not confirmed.)
- Can the media channel view handle very large media counts (1000+ items)? (Hypothesis: no virtualization; not confirmed.)
