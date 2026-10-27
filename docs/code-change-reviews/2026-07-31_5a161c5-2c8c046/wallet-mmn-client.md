# MMN wallet & blockchain client

**Group:** #3 — MMN wallet & blockchain client
**Significance:** Critical
**Commits:** `5f09f0b` (mmn-client crate), `9af3123` (wallet store), `b149a1b` (key management), `dab40c1` (send flow), `65666e7` (ZK proofs), `1660ee3` (UI modal)
**Areas:** `crates/mmn-client/*`, `crates/mezon-store/src/wallet.rs`, `wallet_persist.rs`, `messages.rs`, `crates/mezon-ui/src/wallet/*`, `MMN_PARITY.md`

## Change

A full **non-custodial MMN wallet** was added: Ed25519 key generation, JSON-RPC to MMN nodes, ZK proof generation for private transfers, an indexer client for balance history, and a UI modal for send/receive/history. The implementation is documented as parity-matching the existing JS wallet (`MMN_PARITY.md`).

## Before

At `5a161c5`, there was no wallet. `mezon-store` had no `wallet.rs`, no `wallet_persist.rs`. `mezon-ui` had no `wallet/` module. The only blockchain-adjacent code was the `mezon-proto` definitions for MMN messages, but no client implementation existed.

## After

### `mmn-client` crate (new)

Public API (Observed from `crates/mmn-client/src/lib.rs`):

- `MmnClient` — JSON-RPC over HTTP to MMN nodes (`get_balance`, `send_transaction`, `get_transaction_history`).
- `DongClient` — key derivation / signature orchestration.
- `ZkClient` — ZK proof generation for private transfers (shields sender/amount).
- `IndexerClient` — HTTP to MMN indexer for historical balance/tx data.
- `WalletClients` (composite) — holds all four; used by `mezon-store::wallet`.

Dependencies (Observed): `http_client`, `ed25519-dalek`, `sha2`, `bs58`, `serde`, `serde_json`, `tokio`, `tracing`.

### `mezon-store::wallet` (new — `crates/mezon-store/src/wallet.rs`)

- `WalletStore` struct holds `WalletClients` + in-memory state (balances, pending txs).
- `WalletEvent` enum: `BalanceUpdated`, `TransactionConfirmed`, `TransactionFailed`, `WalletLocked`, `WalletUnlocked`.
- Methods: `create_wallet`, `import_wallet`, `lock`, `unlock`, `get_balance`, `send`, `shield` (ZK), `get_history`.

### Key persistence (`wallet_persist.rs`)

- **Dual storage** via `#[cfg]`:
  - Debug builds: plaintext JSON at `~/.mezon/wallet.json` (for development).
  - Release builds: OS keyring (`keyring` crate) — never touches disk.
- Mnemonic stored encrypted; spending key derived on `unlock` (password-gated).

### UI (`crates/mezon-ui/src/wallet/`)

- `wallet_modal.rs` — main modal: tabs for Send / Receive / History / Settings.
- `send_form.rs` — recipient address, amount, fee selection, "Shield (ZK)" toggle.
- `receive_view.rs` — shows address + QR code (rendered via GPUI).
- `history_list.rs` — paginated tx history from `IndexerClient`.
- `wallet_setup.rs` — first-run: create new mnemonic (12-word) or import existing.

## Mechanism

### Send flow (Observed — `wallet.rs::send`)

```
UI send_form ──> WalletStore::send(recipient, amount, shield)
                        │
                        ├─ if shield: ZkClient::generate_proof(tx) ──> MmnClient::send_shielded
                        └─ else:      MmnClient::send_transaction(signed_tx)
                        │
                        ↓
                 WalletEvent::TransactionConfirmed / Failed
                        │
                        ↓
                 UI updates history_list
```

### Key derivation (Observed — `mmn-client/src/keys.rs`)

- BIP39 mnemonic → BIP32 master key → Ed25519 spending key.
- Spending key never leaves `mmn-client`; only signatures are sent over the wire.

### Parity verification (`MMN_PARITY.md`)

- Documents that Rust implementation produces byte-identical signatures and ZK proofs to the JS wallet for the same mnemonic.
- Test vectors in `crates/mmn-client/tests/parity.rs` (Observed file exists; not all vectors read).

## Motivation

- **Observed:** `MMN_PARITY.md` explicitly states the goal: "Rust wallet must be a drop-in replacement for the JS wallet, producing identical on-chain transactions."
- **Strong inference:** Wallet is integrated into the chat client (not a separate app) because MMN is the native payment rail for the Mezon ecosystem — users send tokens within chats.
- **Strong inference:** ZK shielding is included because MMN supports private transfers and the wallet must expose that capability, not just transparent ones.
- **Hypothesis:** Dual-storage persistence (debug plaintext / release keyring) is a developer-experience compromise: tests need inspectable keys, production needs secure storage.

## Impact

- **Security surface:** Ed25519 private keys now live in the desktop process. Compromise of the app = compromise of the wallet. Mitigated by keyring storage in release + password-gated unlock.
- **Dependencies:** `ed25519-dalek`, `sha2`, `bs58`, `keyring` added to the workspace. `ed25519-dalek` is audited; `keyring` uses OS secure storage.
- **UX:** wallet is now a first-class feature with a modal accessible from the sidebar.
- **Network:** wallet adds JSON-RPC + indexer HTTP traffic, independent of the MTProto transport.

## Trade-offs

- **Benefit:** native non-custodial wallet; no separate app; ZK privacy; parity with JS ensures cross-compatibility.
- **Cost:** significant new crypto code (Ed25519, ZK) — high-severity bug surface. Key management is now a critical path.
- **Risk:** a bug in `mmn-client` could produce invalid signatures (lost funds) or leak the spending key. Parity tests mitigate but do not eliminate.
- **Compatibility:** `MMN_PARITY.md` locks the implementation to JS-compatible behavior; future MMN protocol changes require updating both.
- **Rollback:** removing `mmn-client` + `mezon-store::wallet` + `mezon-ui::wallet` reverts the feature; the rest of the app is unaffected.

## Verification

- `crates/mmn-client/src/lib.rs` exports `MmnClient`, `DongClient`, `ZkClient`, `IndexerClient`, `WalletClients` (Observed).
- `crates/mezon-store/src/wallet.rs` defines `WalletStore`, `WalletEvent` with the listed variants (Observed).
- `crates/mezon-store/src/wallet_persist.rs` contains `#[cfg(debug_assertions)]` plaintext path and `#[cfg(not(debug_assertions))]` keyring path (Observed).
- `MMN_PARITY.md` exists and documents parity (Observed).
- `crates/mmn-client/tests/parity.rs` exists (Observed; full vector set not read).
- **Not verified:** cryptographic correctness of ZK proof generation; keyring storage on all platforms; behavior on wrong password.

## Source-reading guide

1. `MMN_PARITY.md` — parity contract and test vector reference.
2. `crates/mmn-client/src/lib.rs` — public API surface.
3. `crates/mmn-client/src/keys.rs` — BIP39 → Ed25519 derivation.
4. `crates/mmn-client/src/zk.rs` — ZK proof generation (highest-risk file).
5. `crates/mezon-store/src/wallet.rs` — `WalletStore`, `send`, `shield`, `WalletEvent`.
6. `crates/mezon-store/src/wallet_persist.rs` — dual-storage `#[cfg]` paths.
7. `crates/mezon-ui/src/wallet/wallet_modal.rs` — modal entry point.
8. `crates/mezon-ui/src/wallet/send_form.rs` — send flow UI.
9. `crates/mmn-client/tests/parity.rs` — parity test vectors.

## Learning

- **Dual-storage persistence via `#[cfg]`** is a clean pattern for dev/prod secret handling, but tests can only exercise one path per build — both must be tested in CI matrix.
- **Parity testing against a reference implementation** (JS wallet) is the strongest available guarantee for cross-implementation compatibility; unit tests alone cannot prove byte-identical on-chain behavior.

## Unknowns

- Are the ZK proof circuits audited? (No audit doc found in the repo.)
- What happens if the keyring is unavailable at runtime (e.g., D-Bus secret service missing on Linux)? (Hypothesis: falls back to error; not confirmed.)
- Is the mnemonic ever logged? (Hypothesis: no, but `tracing` calls in `wallet.rs` should be audited.)
- Does the wallet re-lock on app focus loss / screensaver? (Not observed in the diff.)
