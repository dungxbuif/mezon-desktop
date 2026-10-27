# Canvas editor integration

**Group:** #8 — Canvas editor
**Significance:** Significant
**Commits:** `2801bc4` (canvas crate), `75b7a67` (navigation), `551cb72` (message content), `4f7df21` (persistence)
**Areas:** `crates/mezon-canvas/*`, `crates/mezon-ui/src/canvas_navigation.rs`, `chat/message/content.rs`, `crates/mezon-store/src/canvas.rs`

## Change

A **canvas editor** was added as a new first-class surface in Mezon: a rich-text/document editor (TipTap/Quill-style) rendered in pure GPUI, accessible as a channel-level document. Canvases are linked from messages and navigated to via a route.

## Before

At `5a161c5`, there was no canvas. `mezon-ui` had no `canvas_navigation.rs`. `mezon-store` had no `canvas.rs`. Messages had no canvas-link content type. The only rich content was text + attachments + embeds (OGP).

## After

### `mezon-canvas` crate (new)

- Pure-GPUI rich-text editor (no webview — unlike `mezon-webview`).
- TipTap/Quill-style document model: blocks (paragraph, heading, list, code, image) with inline marks (bold, italic, code, link).
- Public API (Observed — `crates/mezon-canvas/src/lib.rs`):
  - `CanvasView` — GPUI view for rendering/editing a canvas.
  - `init(cx)` — registers hooks (called from `mezon-ui/src/lib.rs:108`).
  - Document model: `CanvasDoc { blocks: Vec<Block> }`, `Block { kind, content, marks }`.

### Navigation (`canvas_navigation.rs`)

- `CanvasNavigationHooks` global pattern (see architecture report).
- `navigate_to_canvas(clan_id, channel_id, canvas_id)` → `Route::Canvas { ... }`.
- `active_canvas_id()` → current canvas.
- `confirm_delete_canvas()` — confirmation dialog before deletion.

### Message content type (`chat/message/content.rs`)

- New `MessageContent::CanvasLink { canvas_id, title, preview }` variant.
- Renders as a card with the canvas title + a thumbnail preview of the first block.
- Click → `navigate_to_canvas`.

### Persistence (`mezon-store/src/canvas.rs`)

- `CanvasStore` holds `canvases: KeyedCache<CanvasId, CanvasDoc>`.
- `CanvasEvent::Synced`, `Updated`, `Deleted`.
- Sync: local edits are sent to the server; server pushes updates to other clients.
- `StoreEvent::Canvas(CanvasEvent)` bridges to the central bus.

## Mechanism

### Canvas open flow (Observed)

```
User clicks canvas link in message
   │
   ↓
MessageContent::CanvasLink ──> CanvasNavigationHooks::navigate_to_canvas
   │
   ↓
mezon-ui router ──> Route::Canvas { clan_id, channel_id, canvas_id }
   │
   ↓
mezon-canvas::CanvasView::new(canvas_id)
   │
   ├─ mezon-store::canvas::load(canvas_id) ──> CanvasDoc
   └─ render blocks in GPUI
```

### Edit sync flow (Observed)

```
User edits canvas
   │
   ↓
CanvasView ──> mezon-store::canvas::update(canvas_id, blocks)
   │
   ↓
CanvasEvent::Updated ──> StoreEvent::Canvas(Updated)
   │
   ├─ mezon-ui re-renders
   └─ mezon-client sends update to server
   │
   ↓
server broadcasts to other clients ──> CanvasEvent::Synced
```

## Motivation

- **Observed:** Canvas is a channel-level document — a persistent rich-text surface tied to a channel, complementing the ephemeral chat.
- **Strong inference:** Pure-GPUI rendering (not webview) was chosen for performance and consistency with the rest of the UI; webview would introduce a separate rendering pipeline.
- **Strong inference:** The TipTap/Quill block model is chosen because it's the de-facto standard for collaborative editors, easing future real-time collaboration.
- **Hypothesis:** Canvas links in messages let users reference documents in conversation, similar to Slack posts.

## Impact

- **Capability:** channels now have a persistent document surface alongside the chat.
- **UI:** new route (`Route::Canvas`), new message content type, new navigation hooks.
- **Store:** new `canvas.rs` module with `KeyedCache` + sync.
- **Dependencies:** `mezon-canvas` depends on `mezon-widgets`, `mezon-theme`, `mezon-store`, `mezon-i18n`, `ui` (vendor) — no new external deps.

## Trade-offs

- **Benefit:** first-class document editing in the chat client; pure-GPUI for consistency.
- **Cost:** a full editor implementation (block model, marks, rendering, sync) is large; maintaining it is ongoing.
- **Risk:** collaborative sync (multiple editors) is complex; conflict resolution is not visible in this diff (see Unknowns).
- **Rollback:** remove `mezon-canvas` + `canvas_navigation.rs` + `MessageContent::CanvasLink` + `mezon-store::canvas`; canvases become inaccessible but chat works.

## Verification

- `crates/mezon-canvas/src/lib.rs` exports `CanvasView`, `init`, `CanvasDoc`, `Block` (Observed).
- `crates/mezon-ui/src/canvas_navigation.rs:7` defines `CanvasNavigationHooks` (Observed).
- `crates/mezon-ui/src/chat/message/content.rs` has `MessageContent::CanvasLink` variant (Observed).
- `crates/mezon-store/src/canvas.rs` defines `CanvasStore`, `CanvasEvent` (Observed).
- `crates/mezon-ui/src/lib.rs:108` calls `mezon_canvas::init(cx)` (Observed).
- **Not verified:** conflict resolution for concurrent edits; performance with large documents.

## Source-reading guide

1. `crates/mezon-canvas/src/lib.rs` — public API, `CanvasView`, `init`.
2. `crates/mezon-canvas/src/doc.rs` — `CanvasDoc`, `Block`, `Mark` model.
3. `crates/mezon-canvas/src/render.rs` — GPUI rendering of blocks.
4. `crates/mezon-ui/src/canvas_navigation.rs:7` — `CanvasNavigationHooks`.
5. `crates/mezon-ui/src/chat/message/content.rs` — `MessageContent::CanvasLink`.
6. `crates/mezon-store/src/canvas.rs` — `CanvasStore`, sync, `CanvasEvent`.

## Learning

- **Pure-GPUI rich-text editors** are feasible but require implementing the block/mark model from scratch (no DOM, no contenteditable); the trade-off is performance + consistency vs. implementation effort.
- **The global-hooks pattern** (`CanvasNavigationHooks`) cleanly decouples the leaf crate (`mezon-canvas`) from the router — a reusable pattern for any feature that needs navigation without a direct router dependency.

## Unknowns

- **How are concurrent edits resolved?** (OT? CRDT? Last-write-wins? Not visible in the diff. Critical for collaboration.)
- Is there an offline mode (edit locally, sync when online)? (Hypothesis: yes, via `CanvasStore` cache; not confirmed.)
- What's the max canvas size? (Performance concern for rendering + sync.)
- Can canvases be exported (markdown, HTML)? (Not observed.)
