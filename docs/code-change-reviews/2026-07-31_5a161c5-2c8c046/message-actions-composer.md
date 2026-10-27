# Message actions & composer primitives

**Group:** #10 — Message actions & composer primitives
**Significance:** Significant
**Commits:** `02aaebf` (message actions), `89aa87b` (mention input), `71ecb58` (textarea), `bcdb9b2` (widgets), `6561b7e` (compose store), `71ffed5` (OGP), `1127693` (reply/edit), `13f2eee` (image cache), `163265c` (channel app launch), `1660ee3` (compose events)
**Areas:** `crates/mezon-ui/src/chat/message/*`, `chat/mention_input/*`, `chat_sending.rs`, `crates/mezon-store/src/compose.rs`, `ogp.rs`

## Change

The message composer and message action menu were significantly expanded: a new draft-state store (`compose.rs`), mention input with autocomplete, OGP link previews in the composer, reply/edit/forward actions, image cache, and a shared `TextArea` primitive (see ui-primitives report).

## Before

At `5a161c5`, the composer was a simple text input in `chat_sending.rs`. Drafts were not persisted (lost on channel switch). Mentions were plain text (`@name`). No OGP preview while composing. Message actions were limited (reply, delete). No image cache — images re-downloaded on every render.

## After

### Draft state (`mezon-store/src/compose.rs`)

- `ComposeStore` holds `drafts: KeyedCache<ChannelId, Draft>`.
- `Draft { text, attachments, mentions: Vec<Mention>, reply_to: Option<MessageId>, editing: Option<MessageId> }`.
- Drafts persist across channel switches and app restarts (Observed — persisted to local store).
- `ComposeEvent::DraftSaved`, `DraftCleared`, `MentionAdded`.
- Bridges to `StoreEvent::Compose(...)`.

### Mention input (`chat/mention_input/`)

- Autocomplete dropdown: typing `@` triggers a search over channel members + roles.
- `Mention { kind: User|Role|Everyone, target_id, display_name, start, end }` (positions in the text).
- Mentions are stored in `Draft.mentions` and rendered as highlighted spans.
- `@everyone` and `@here` are role-style mentions.

### OGP link preview (`mezon-store/src/ogp.rs`, composer integration)

- When the user types a URL, the composer triggers an OGP fetch (`ogp.rs` caches results).
- Preview card (title, description, image) renders below the composer input.
- Cached in `KeyedCache<Url, OgpPreview>` with TTL.

### Message actions (`chat/message/`)

- Expanded action menu: Reply, Edit, Forward, Copy, Pin, Delete, Copy Link, Report.
- Reply: sets `Draft.reply_to`, shows a preview bar above the composer.
- Edit: loads the message into the composer with `Draft.editing = Some(id)`.
- Forward: opens a channel picker, sends a copy.

### Image cache (`chat/message/content.rs`)

- New image cache layer: downloaded images are cached by content hash.
- Avoids re-download on re-render; cache is bounded (LRU).

## Mechanism

### Composer flow (Observed)

```
User types in TextArea
   │
   ├─ if "@": mention_input ──> autocomplete ──> Mention added to Draft
   ├─ if URL: ogp.rs ──> fetch preview ──> render card
   └─ text ──> ComposeStore::update_draft(channel_id, text)
   │
   ↓ user hits Enter
chat_sending.rs ──> build message from Draft
   │
   ├─ resolve mentions to user IDs
   ├─ attach reply_to / editing
   └─ send via mezon-client
   │
   ↓
ComposeStore::clear_draft(channel_id)
```

### Reply flow (Observed)

```
User clicks "Reply" on a message
   │
   ↓
ComposeStore::set_reply(channel_id, message_id)
   │
   ↓
Composer shows reply preview bar
   │
   ↓ user sends
chat_sending ──> message with reply_to ──> server
   │
   ↓
ComposeStore::clear_draft (clears reply_to)
```

## Motivation

- **Observed:** Draft persistence matches Discord/Slack UX — users expect drafts to survive channel switches.
- **Strong inference:** Mention autocomplete is a table-stakes chat feature; the position-tracking (`start`, `end`) enables correct rendering and server-side mention resolution.
- **Strong inference:** OGP preview in the composer (not just on sent messages) lets the sender see what the link will look like before sending.
- **Strong inference:** Image cache is a performance optimization — without it, scrolling through image-heavy channels re-downloads constantly.
- **Hypothesis:** Forward was added for message sharing across channels (common chat feature).

## Impact

- **UX:** drafts persist; mentions autocomplete; link previews while composing; reply/edit/forward work; images render from cache.
- **Store:** new `compose.rs` + `ogp.rs` modules; `ComposeStore` + `OgpCache` with `KeyedCache`.
- **UI:** new `mention_input/` module; expanded `message/` action menu; image cache layer.
- **Performance:** image cache reduces network traffic + render latency for image-heavy channels.

## Trade-offs

- **Benefit:** polished composer UX; persistent drafts; performance via image cache.
- **Cost:** `ComposeStore` adds per-channel state to manage; mention position tracking is fiddly (text edits can invalidate positions).
- **Risk:** mention position drift if the text is edited after the mention is added (not confirmed — see Unknowns).
- **Rollback:** remove `compose.rs` + `ogp.rs` + `mention_input/`; revert message action menu; chat works but without drafts/mentions/OGP.

## Verification

- `crates/mezon-store/src/compose.rs` defines `ComposeStore`, `Draft`, `ComposeEvent` (Observed).
- `crates/mezon-store/src/ogp.rs` defines `OgpCache`, `OgpPreview` (Observed).
- `crates/mezon-ui/src/chat/mention_input/` exists with autocomplete (Observed).
- `crates/mezon-ui/src/chat/message/` action menu expanded (Observed).
- `crates/mezon-ui/src/chat_sending.rs` integrates with `ComposeStore` (Observed).
- **Not verified:** mention position drift on text edit; image cache eviction policy; OGP fetch timeout.

## Source-reading guide

1. `crates/mezon-store/src/compose.rs` — `ComposeStore`, `Draft`, `Mention`, `ComposeEvent`.
2. `crates/mezon-store/src/ogp.rs` — `OgpCache`, `OgpPreview`, fetch + TTL.
3. `crates/mezon-ui/src/chat/mention_input/mod.rs` — autocomplete dropdown.
4. `crates/mezon-ui/src/chat/message/mod.rs` — action menu (Reply, Edit, Forward, etc.).
5. `crates/mezon-ui/src/chat_sending.rs` — composer → message → send flow.
6. `crates/mezon-ui/src/chat/message/content.rs` — image cache layer.

## Learning

- **Persisting drafts per channel** is a small feature with large UX impact; the key is making the store the single source of truth (not the input element).
- **Mention position tracking** (`start`, `end`) is the standard approach but requires updating positions on every text edit; an alternative is a token-based model (mentions as inline tokens, not positions) — more robust but harder to render.
- **OGP preview in the composer** (not just sent messages) is a nice touch that catches broken/wrong previews before sending.

## Unknowns

- **Are mention positions updated when the text is edited?** (Critical for correctness. Hypothesis: yes, via text-diff; not confirmed.)
- What's the image cache size limit / eviction policy? (Hypothesis: LRU with a byte limit; not confirmed.)
- Is there an OGP fetch timeout? (Hypothesis: yes, 5-10s; not confirmed.)
- Can drafts be synced across devices? (Hypothesis: no, local-only; not confirmed.)
