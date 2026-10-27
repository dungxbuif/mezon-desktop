# UI primitives: TextArea, widgets, theme

**Group:** #11 — UI primitives: TextArea, widgets, theme
**Significance:** Significant
**Commits:** `71ecb58` (textarea), `bcdb9b2` (widgets), `3d582b2` (theme), `725be44` (theme bridge)
**Areas:** `crates/mezon-ui/src/components/primitives/textarea.rs`, `crates/mezon-widgets/*`, `crates/mezon-theme/*`

## Change

Three foundational UI layers were extracted into dedicated crates: `mezon-theme` (design tokens, Zed theme bridge), `mezon-widgets` (Input, Button, Icon), and a `TextArea` primitive in `mezon-ui`. These are now the shared building blocks for `mezon-ui` and `mezon-canvas`.

## Before

At `5a161c5`, UI primitives were inlined in `mezon-ui`. There was no `mezon-theme` crate (theme tokens were scattered), no `mezon-widgets` crate (Input/Button were inline components), and no `TextArea` primitive (the composer used a basic text input). `mezon-canvas` didn't exist, so there was no reuse pressure.

## After

### `mezon-theme` crate (new)

- Design tokens: colors, spacing, typography, radii, shadows — as Rust constants/enums.
- **Zed theme bridge**: `mezon-theme` depends on the vendored `theme` crate (Zed's) and exposes a `to_zed_theme()` conversion, so Mezon's tokens can drive Zed's GPUI theme system.
- Public API (Observed — `crates/mezon-theme/src/lib.rs`): `Theme`, `colors`, `spacing`, `typography`, `to_zed_theme`.
- Used by: `mezon-ui`, `mezon-canvas`, `mezon-widgets`.

### `mezon-widgets` crate (new)

- `Input` — text input with placeholder, validation, icon slot.
- `Button` — variants (primary, secondary, ghost, danger), sizes (sm, md, lg), icon support.
- `Icon` — icon component wrapping the vendored `icons` crate.
- Public API (Observed — `crates/mezon-widgets/src/lib.rs`): `Input`, `Button`, `Icon`, `init(cx)`.
- `init(cx)` registers global state (Observed — called from `mezon-ui/src/lib.rs`).
- Depends on: `mezon-theme`, `mezon-store`, `gpui`.

### `TextArea` primitive (`mezon-ui/src/components/primitives/textarea.rs`)

- Multi-line text input with: cursor, selection, undo/redo, clipboard, IME, line wrapping.
- Used by the composer (`chat_sending.rs`) and canvas (`mezon-canvas` via `mezon-widgets`).
- Not in `mezon-widgets` (Observed — in `mezon-ui`); likely because it depends on `mezon-ui`-specific state (hypothesis).

## Mechanism

### Theme flow (Observed)

```
mezon-theme::Theme (Mezon tokens)
   │
   ↓ to_zed_theme()
Zed theme (vendor/theme crate)
   │
   ↓ gpui::cx.set_theme(...)
GPUI renders with Mezon colors/typography
```

### Widget composition (Observed)

```
mezon-ui component
   │
   ├─ mezon_widgets::Button::new("Save").primary()
   ├─ mezon_widgets::Input::new().placeholder("Search...")
   └─ mezon_widgets::Icon::new("search")
   │
   ↓
rendered via gpui, themed via mezon-theme
```

### TextArea (Observed)

```
TextArea::new(text)
   │
   ├─ cursor: (line, col)
   ├─ selection: Option<(start, end)>
   ├─ history: Vec<TextState> (undo/redo)
   └─ ime: ImeState
   │
   ↓ key events
update cursor / selection / history
   │
   ↓
on_change(text) callback
```

## Motivation

- **Observed:** `mezon-theme` + `mezon-widgets` were extracted to enable reuse by `mezon-canvas` without forcing `mezon-canvas` to depend on the large `mezon-ui`.
- **Strong inference:** The Zed theme bridge exists because GPUI (vendored from Zed) expects a Zed `Theme`; Mezon's tokens must convert to that format to drive rendering.
- **Strong inference:** `TextArea` is a primitive because the composer and canvas both need multi-line text editing; sharing one implementation avoids divergence.
- **Hypothesis:** `TextArea` stayed in `mezon-ui` (not in `mezon-widgets`) because it depends on `mezon-ui`-specific state (e.g., the global clipboard or focus manager).

## Impact

- **Consistency:** all UI now uses the same tokens + widgets; no more hand-rolled colors/inputs.
- **Reuse:** `mezon-canvas` builds on `mezon-widgets` + `mezon-theme` without touching `mezon-ui`.
- **Dependencies:** `mezon-theme` depends on `gpui` + vendored `theme`; `mezon-widgets` depends on `mezon-theme`, `mezon-store`, `gpui`.
- **Maintainability:** changes to design tokens or widget behavior now happen in one place.

## Trade-offs

- **Benefit:** consistency, reuse, single source of truth for design tokens.
- **Cost:** three new crates to maintain; the extraction is a refactor that touches most of `mezon-ui`.
- **Risk:** if `TextArea` diverges from the composer's needs, it'll be forked or grow flags; the placement in `mezon-ui` (not `mezon-widgets`) suggests this risk is recognized.
- **Rollback:** re-inline `mezon-theme` + `mezon-widgets` into `mezon-ui`; `mezon-canvas` would need its own copies or revert too.

## Verification

- `crates/mezon-theme/src/lib.rs` exports `Theme`, `colors`, `spacing`, `to_zed_theme` (Observed).
- `crates/mezon-widgets/src/lib.rs` exports `Input`, `Button`, `Icon`, `init` (Observed).
- `crates/mezon-ui/src/components/primitives/textarea.rs` defines `TextArea` (Observed).
- `crates/mezon-ui/src/lib.rs:108` calls `mezon_widgets::input::init` (Observed).
- `crates/mezon-theme/Cargo.toml` depends on `gpui` + vendored `theme` (Observed).
- **Not verified:** IME correctness on all platforms; undo/redo edge cases; performance with very long text.

## Source-reading guide

1. `crates/mezon-theme/src/lib.rs` — `Theme`, tokens, `to_zed_theme`.
2. `crates/mezon-theme/src/bridge.rs` (if present) — Zed theme conversion.
3. `crates/mezon-widgets/src/lib.rs` — `Input`, `Button`, `Icon`, `init`.
4. `crates/mezon-widgets/src/input.rs` — `Input` implementation.
5. `crates/mezon-widgets/src/button.rs` — `Button` variants.
6. `crates/mezon-ui/src/components/primitives/textarea.rs` — `TextArea` (cursor, selection, IME, undo/redo).

## Learning

- **Extracting design tokens into a dedicated crate** is the standard way to enable reuse across multiple UI consumers; the key is making the crate dependency-light (only `gpui` + the theme system).
- **Bridging to a vendored theme system** (Zed's) via a conversion function (`to_zed_theme`) is cleaner than reimplementing the theme system; it lets the vendored code stay upstream-compatible.
- **Placing `TextArea` in `mezon-ui` instead of `mezon-widgets`** is a deliberate choice when the primitive has host-specific dependencies; forcing it into `mezon-widgets` would create a reverse dependency or require abstraction.

## Unknowns

- **Is IME fully supported in `TextArea`?** (Critical for CJK input. Hypothesis: yes, based on `ImeState` field; not confirmed.)
- Does `TextArea` handle very long texts (10k+ lines) efficiently? (Hypothesis: no virtualization; not confirmed.)
- Are the design tokens in `mezon-theme` synced with a Figma/design source, or hand-maintained? (Not observed.)
- Why is `Icon` in `mezon-widgets` and not directly in `mezon-theme`? (Hypothesis: `Icon` is a component, not a token; makes sense.)
