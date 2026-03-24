# PR 1 Checklist: Stellar Chain Support in `open-wallet-standard/core`

## Purpose

This checklist defines the first upstream PR against `open-wallet-standard/core`.

PR 1 is intentionally narrow:

- add Stellar as a supported chain family in the existing OWS universal-wallet model
- derive and store a Stellar signer account
- add metadata conventions for linked OZ smart accounts so a `C...` address can be surfaced as the primary Stellar wallet identity without changing the core `accounts[]` model
- update only the touched tests and docs required for Stellar to land

PR 1 is not the auth-entry signing PR. That belongs in PR 2.

## Scope

Included:

- `ChainType::Stellar`
- canonical Stellar chain ids for testnet and pubnet
- CAIP parsing and alias handling for Stellar
- SLIP-44 coin type `148`
- Stellar derivation path support
- derived `G...` account generation in wallet creation and import
- linked `C...` smart-account metadata and primary-account surfacing
- wallet lifecycle coverage with Stellar present
- CLI and SDK account visibility for Stellar

Explicitly not included:

- `signAuthEntry`
- fee payer or relayer flow
- `AuthPayload` construction
- OZ smart-account deployment helpers
- delegated signer support
- Channels-backed `signAndSend`
- WebAuthn or `secp256r1`
- broad cleanup unrelated to touched Stellar paths

## Repo Files Likely Touched

Rust core and signer registration:

- `ows/crates/ows-core/src/chain.rs`
- `ows/crates/ows-core/src/lib.rs`
- `ows/crates/ows-signer/src/chains/mod.rs`
- `ows/crates/ows-signer/src/chains/stellar.rs`
- `ows/crates/ows-signer/src/lib.rs`
- `ows/crates/ows-lib/src/ops.rs`

CLI and bindings:

- `ows/crates/ows-cli/src/commands/info.rs`
- `ows/crates/ows-cli/src/commands/derive.rs`
- `bindings/node/src/lib.rs`
- `bindings/node/__test__/index.spec.mjs`
- `bindings/python/src/lib.rs`
- `bindings/python/python/ows/__init__.py`
- `bindings/python/tests/test_bindings.py`

Docs:

- `README.md`
- `docs/01-storage-format.md` if linked smart-account metadata needs to be documented there
- `docs/07-supported-chains.md`
- `docs/06-wallet-lifecycle.md` if lifecycle wording needs Stellar-specific clarification
- `docs/sdk-cli.md`
- `docs/sdk-node.md`
- `docs/sdk-python.md`
- `bindings/node/README.md`
- `bindings/python/README.md`

## Implementation Checklist

### Chain registration

- add `Stellar` to `ChainType`
- add namespace mapping for `stellar`
- add coin type mapping to `148`
- add `stellar:testnet` and `stellar:pubnet` to known chains
- add `stellar` shorthand parsing in the same style as existing aliases

### Signer registration

- create `ows/crates/ows-signer/src/chains/stellar.rs`
- implement address derivation for Stellar Ed25519 keys
- implement default derivation path for Stellar
- register the signer in `signer_for_chain(...)`

### Wallet derivation

- ensure `create_wallet(...)` derives a Stellar account
- ensure `import_wallet_mnemonic(...)` derives a Stellar account
- ensure `import_wallet_private_key(...)` supports Stellar through the existing Ed25519 path
- keep the stored account shape aligned with existing `account_id`, `address`, `chain_id`, `derivation_path`

### Smart-account profile metadata

- add a metadata convention for linked OZ smart accounts
- support a `primarySmartAccount` field for Stellar wallets
- ensure linked `C...` addresses round-trip without breaking existing wallet schema expectations
- ensure the derived `G...` signer remains the custody anchor in `accounts[]`

### CLI and SDK exposure

- ensure wallet create/list/info surfaces show Stellar accounts
- ensure mnemonic derive supports Stellar
- ensure Node and Python wallet/account outputs include Stellar
- ensure linked `C...` smart-account metadata is visible when present
- ensure Stellar wallet output can mark the linked `C...` account as primary

## Test Checklist

Rust:

- `ChainType` serde round-trip includes `Stellar`
- namespace mapping test covers `stellar`
- coin type test covers `148`
- known-chain parsing covers `stellar:testnet` and `stellar:pubnet`
- deterministic mnemonic -> Stellar `G...` address fixture
- wallet create/import/export/list lifecycle test with Stellar present
- wallet metadata round-trip test for linked `C...` smart-account entries

Node:

- wallet account count and chain presence updated for Stellar
- `deriveAddress(..., "stellar")` test added if surfaced through bindings
- wallet output includes linked smart-account metadata when present

Python:

- wallet account count and chain presence updated for Stellar
- `derive_address(..., "stellar")` test added if surfaced through bindings
- wallet output includes linked smart-account metadata when present

Docs/tests touched by family-count assumptions:

- update only the assertions and examples that fail because Stellar is now part of the supported family set

## Acceptance Criteria

- `createWallet()` produces a Stellar account alongside the existing chain set
- mnemonic import reproduces the same Stellar address deterministically
- Stellar addresses are visible through CLI, Node, and Python surfaces
- linked `C...` smart-account metadata round-trips cleanly if present
- a linked `C...` account can be surfaced as the primary Stellar wallet identity without changing how `accounts[]` works
- touched docs describe Stellar as a supported chain family
- no new signing operation is introduced in this PR
- no unrelated cleanup is bundled into the diff

## Suggested Local Validation

From the OWS repo:

```bash
cd ows
cargo build --workspace
cargo test --workspace
```

Bindings:

```bash
cd bindings/node && npm test
cd bindings/python && pytest
```

If the binding test commands differ upstream, use the repo's existing CI entrypoints rather than inventing new ones.

## Reviewer Framing

Open the PR as:

- one new chain family in the current universal-wallet architecture
- optional metadata-backed smart-wallet profile for linked OZ `C...` accounts
- no provider abstraction
- no relayer logic
- no auth-entry signing yet
- minimal touched test/doc updates required by the feature

One-sentence framing:

`This PR adds Stellar as a supported chain family in OWS’s existing wallet model by deriving and exposing a Stellar signer account and optionally surfacing linked OZ smart-account metadata, without changing the signing interface yet.`
