# OWS Stellar Implementation Surface Audit

## Purpose

This document narrows the upstream work for adding a new OWS chain family, using Stellar as the forcing case. It focuses on the OWS reference implementation surface: Rust crates, CLI, Node/Python bindings, tests, fixtures, and likely refactor pressure.

This is intentionally not a Stellar product-spec. It answers: what in `open-wallet-standard/core` would actually need to change, what should be split across PRs, and where hidden implementation constraints already exist.

Primary upstream sources:

- `https://github.com/open-wallet-standard/core`
- `https://docs.openwallet.sh/`

## High-Level Finding

The OWS spec text says a new chain can be added without changing the standard, but the current reference implementation is not plugin-like. Adding a new family today requires coordinated changes across:

- `ows-core` chain registry and alias parsing
- `ows-signer` signer registry and per-chain module set
- `ows-lib` universal-wallet derivation, key selection, RPC/broadcast flow, and signing API
- CLI help text and all-chains display paths
- Node and Python binding tests and docs

If the new family reuses an existing curve, the change is moderate. If it needs a new curve or a new signing primitive beyond `sign_message` / `sign_transaction` / `sign_and_send`, the change becomes architectural.

There is also already some drift inside the current repo:

- `ChainType` includes `Spark`, but `ALL_CHAIN_TYPES` does not, so universal wallet derivation is not actually “all registered families”.
- some signer integration tests omit `Sui` and some wallet/binding tests still assume an 8-family matrix.
- `docs/07-supported-chains.md` presents Spark as a supported family, while `ows-core` default RPC config does not include a Spark endpoint.

That existing drift means a Stellar PR should include a small consistency cleanup instead of just appending another family and inheriting the same mismatch pattern.

For Stellar specifically:

- `Ed25519` reuse is favorable
- chain-family addition is straightforward
- transaction/broadcast support is the larger constraint
- a new operation such as `sign_auth_entry` is likely a separate PR from base chain-family support

## Exact Files And Modules Likely Needing Changes

### 1. Chain Model And Registry

Core files:

- `ows/crates/ows-core/src/chain.rs`
- `ows/crates/ows-core/src/lib.rs`
- `ows/crates/ows-core/src/config.rs`

Why:

- `ChainType` is a closed enum.
- `ALL_CHAIN_TYPES` is a fixed array used for universal wallet derivation.
- `KNOWN_CHAINS` is a hard-coded registry for alias resolution and default chain IDs.
- `namespace()`, `default_coin_type()`, `from_namespace()`, `Display`, and `FromStr` all require manual extension.
- built-in RPC defaults are keyed by exact chain IDs in `Config::default_rpc()`.

Likely Stellar touchpoints:

- add `ChainType::Stellar`
- add Stellar entries to `KNOWN_CHAINS`
- add namespace and coin-type mapping
- add alias parsing support in `parse_chain`
- add default RPC config keys if `sign_and_send` / broadcasting is supported

Important implementation note:

- `parse_chain()` leaks unknown CAIP strings via `Box::leak` for dynamic namespace fallback. Any new namespace automatically participates once `from_namespace()` supports it.
- `ALL_CHAIN_TYPES` is not equivalent to “all enum variants” today, so reviewers should treat it as an explicit derivation allowlist, not a derived registry.

### 2. Signer Trait Surface And Registry

Signer files:

- `ows/crates/ows-signer/src/traits.rs`
- `ows/crates/ows-signer/src/chains/mod.rs`
- `ows/crates/ows-signer/src/lib.rs`
- `ows/crates/ows-signer/src/curve.rs`

New module expected:

- `ows/crates/ows-signer/src/chains/stellar.rs`

Why:

- `signer_for_chain()` is an explicit `match` on `ChainType`.
- each chain family is a concrete module under `src/chains/`.
- `ChainSigner` assumes the signing surface is address derivation plus:
  - `sign`
  - `sign_message`
  - `sign_transaction`
  - optional `extract_signable_bytes`
  - optional `encode_signed_transaction`
  - `default_derivation_path`

Constraint:

- this trait is good for classic “derive address, sign bytes, maybe encode tx” chains.
- if Stellar support requires a separate authorization-signing primitive, that does not fit cleanly into the existing trait without either:
  - a trait extension
  - a new library-level API that bypasses `ChainSigner`
  - a chain-specific enum/operation abstraction

Curve note:

- `curve.rs` currently only supports `Secp256k1` and `Ed25519`.
- Stellar can likely reuse `Ed25519`, so no curve expansion is needed for base support.
- if upstream later wants passkey-oriented `secp256r1`, `KeyPair` and curve plumbing become a larger refactor.

### 3. Universal Wallet Derivation And Key Selection

Library file:

- `ows/crates/ows-lib/src/ops.rs`

Why this file matters more than it first appears:

- `derive_all_accounts()` loops over `ALL_CHAIN_TYPES`
- `derive_all_accounts_from_keys()` also loops over `ALL_CHAIN_TYPES`
- `decrypt_signing_key()` picks key material by curve via `signer_for_chain(chain_type)`
- `create_wallet()` and mnemonic import derive one account per chain family automatically
- `import_wallet_private_key()` is built around a two-curve world

Important hidden constraint:

- `KeyPair` is hard-coded to exactly two stored keys:
  - `secp256k1`
  - `ed25519`
- exported private-key wallets serialize to JSON with exactly those two fields
- `key_for_curve()` is a `match` over only those two curves

Implication:

- adding Stellar with `Ed25519` fits the current model
- adding any future third curve does not
- the private-key wallet format and import/export behavior are not general “N curve” abstractions

### 4. Send/Broadcast Pipeline

Primary file:

- `ows/crates/ows-lib/src/ops.rs`

Related config file:

- `ows/crates/ows-core/src/config.rs`

Why:

- `sign_and_send()` delegates into `sign_encode_and_broadcast()`
- `sign_encode_and_broadcast()` assumes:
  1. signer can extract signable bytes
  2. signer can sign
  3. signer can encode a full signed transaction
  4. OWS can resolve an RPC URL
  5. OWS can broadcast by chain type

Hidden constraint:

- even if basic signing support lands first, `sign_and_send` may have to remain unsupported for Stellar until there is:
  - a stable transport path in OWS
  - transaction encoding support in the signer path
  - broadcast implementation in `ops.rs`

That makes “derive + sign” a cleaner first PR than “full send support”.

### 5. CLI Surface

Files:

- `ows/crates/ows-cli/src/main.rs`
- `ows/crates/ows-cli/src/commands/info.rs`
- `ows/crates/ows-cli/src/commands/derive.rs`
- `ows/crates/ows-cli/src/commands/sign_transaction.rs`
- `ows/crates/ows-cli/src/commands/send_transaction.rs`
- `ows/crates/ows-cli/src/commands/wallet.rs`

Why:

- command help text and examples enumerate supported chains
- `wallet info` prints supported families from `ALL_CHAIN_TYPES`
- mnemonic derive for “all chains” loops over `ALL_CHAIN_TYPES`
- wallet import help text currently describes curve families and source chains in a hard-coded way

Constraint:

- CLI docs and help currently mix normative chain support with implementation details such as “all 8 accounts”
- adding a 9th family requires text changes across CLI help, SDK docs, tests, and examples

### 6. Node Bindings

Files:

- `bindings/node/src/lib.rs`
- `bindings/node/__test__/index.spec.mjs`
- `docs/sdk-node.md`
- `bindings/node/README.md`

Why:

- the NAPI layer mostly mirrors `ows_lib`, so the code changes are probably small if no new API is added
- tests and docs hard-code current chain count and chain list

Specific current assumptions:

- tests expect `wallet.accounts.length === 8`
- tests enumerate exact chain arrays
- docs repeatedly say “all 8 chains”

If a new API lands:

- the NAPI layer needs new exported methods and return types
- test coverage has to be added here, not just in Rust

### 7. Python Bindings

Files:

- `bindings/python/src/lib.rs`
- `bindings/python/python/ows/__init__.py`
- `bindings/python/tests/test_bindings.py`
- `docs/sdk-python.md`
- `bindings/python/README.md`

Why:

- same pattern as Node: thin wrapper, but tests/docs assume the current family count and current exported operations

Specific current assumptions:

- tests expect `len(wallet["accounts"]) == 8`
- docs say “all 8 chain accounts”
- import docs describe source chains in current two-curve buckets

## Hidden Implementation Constraints And Refactor Pressure

### Constraint 1: The Spec Says “No Core Changes”; The Repo Says Otherwise

`docs/07-supported-chains.md` says adding a new chain does not require core changes. The current codebase contradicts that operationally:

- enum extension is required
- registry extension is required
- signer dispatch extension is required
- universal-wallet derivation extension is required
- tests/docs assume a fixed chain count

The likely conclusion for upstream is:

- no normative spec change may be required
- reference implementation changes absolutely are required

### Constraint 2: Two-Curve Wallet Format

The private-key wallet path is designed around exactly two curves. This is acceptable for Stellar if it stays on `Ed25519`, but it is a real design boundary.

Affected logic is concentrated in:

- `ows/crates/ows-lib/src/ops.rs`

Refactor pressure only if needed:

- replace `KeyPair { secp256k1, ed25519 }` with a map keyed by curve/family
- version or migrate private-key export format
- generalize import/export docs and tests

### Constraint 3: `sign_transaction` Is The Default Escape Hatch

The current OWS shape heavily assumes that new chains can fit into:

- message signing
- raw tx signing
- optional tx re-encoding and broadcasting

If Stellar support requires a distinct signing primitive, forcing it into `sign_transaction` will make the API and tests misleading. That suggests a phased approach:

- first PRs: family support, derivation, addressing, maybe raw message support
- later PR: new signing primitive and bindings

### Constraint 4: “Universal Wallet” Means Test Churn Everywhere

Because wallets auto-derive one account per supported family, adding one family changes:

- wallet account counts
- CLI output
- docs screenshots/examples
- Node tests
- Python tests
- any characterization tests asserting exact counts

This is not difficult, but it means chain-family additions are inherently cross-cutting in the current repo.

Related existing inconsistency:

- the repo already uses different effective family sets in different places:
  - `ChainType` and signer registry include more than the universal-wallet derivation set
  - SDK docs, binding tests, and signer integration tests do not all agree on the same set

For Stellar, that argues for either:

- making `ALL_CHAIN_TYPES` the single canonical “auto-derived families” list and auditing all docs/tests against it, or
- deriving that list mechanically from a stronger registry model in a follow-up refactor

### Constraint 5: Broadcast Support Is Not Modular

Even with a signer module implemented, `sign_and_send` is only production-ready when:

- config defaults exist or explicit RPC is supplied
- transaction encoding is supported
- chain broadcast logic exists in `ows_lib`

That argues for splitting chain-family support from full send-path support.

## Likely Upstream PR Slicing

### PR 1: Chain Family Registration And Derivation

Goal:

- make OWS aware of Stellar as a chain family
- derive Stellar addresses from mnemonic/private-key wallets
- update universal-wallet account generation

Expected files:

- `ows/crates/ows-core/src/chain.rs`
- `ows/crates/ows-core/src/lib.rs`
- `ows/crates/ows-signer/src/chains/mod.rs`
- `ows/crates/ows-signer/src/chains/stellar.rs`
- `ows/crates/ows-signer/src/lib.rs`
- `ows/crates/ows-lib/src/ops.rs`
- `ows/crates/ows-cli/src/commands/info.rs`
- `ows/crates/ows-cli/src/commands/derive.rs`
- docs and tests that hard-code 8 chains

Acceptance target:

- create/import wallet now yields one more account
- `derive_address(..., "stellar")` works
- message signing may still be stubbed or limited depending on implementation choice

Why this slice is good:

- smallest coherent implementation PR
- avoids overloading reviewers with broadcast/API redesign at the same time

### PR 2: Signing Surface For Stellar

Goal:

- support the minimum signing operation needed for the new family in the OWS Rust core and bindings

Two possible shapes:

- if existing `sign_message` / `sign_transaction` is enough:
  - mostly signer and tests
- if a new primitive is needed:
  - trait/API extension in `ows-signer`
  - new library export in `ows-lib`
  - NAPI/PyO3 binding additions
  - SDK docs updates

Expected files:

- `ows/crates/ows-signer/src/traits.rs`
- `ows/crates/ows-signer/src/chains/stellar.rs`
- `ows/crates/ows-lib/src/ops.rs`
- `bindings/node/src/lib.rs`
- `bindings/python/src/lib.rs`
- `bindings/python/python/ows/__init__.py`
- `docs/sdk-node.md`
- `docs/sdk-python.md`
- `docs/02-signing-interface.md` if the reference implementation exposes a new operation that should be documented

Why separate it:

- this is where API design debates start
- easier to review once family support already exists

### PR 3: CLI And Docs Completion

Goal:

- finish CLI examples, supported-chain docs, SDK reference docs, and installation-facing materials

Expected files:

- `docs/07-supported-chains.md`
- `docs/sdk-cli.md`
- `docs/sdk-node.md`
- `docs/sdk-python.md`
- `bindings/node/README.md`
- `bindings/python/README.md`
- CLI help text in `ows/crates/ows-cli/src/main.rs`

Why separate it:

- lets maintainers review behavior separately from prose
- reduces churn in core code review

### PR 4: Full Send/Broadcast Support

Goal:

- support `sign_and_send` for the new family if OWS should own that path

Expected files:

- `ows/crates/ows-core/src/config.rs`
- `ows/crates/ows-lib/src/ops.rs`
- possibly new transport/helper modules if introduced
- CLI send-path docs and tests

Why this should be last:

- it is the most implementation-specific part
- it is the easiest scope to defer

## Test Locations And Fixture Implications

### Rust Core And Signer Tests

Current locations:

- `ows/crates/ows-core/src/chain.rs`
- `ows/crates/ows-core/src/config.rs`
- `ows/crates/ows-signer/src/curve.rs`
- `ows/crates/ows-signer/src/lib.rs`
- `ows/crates/ows-lib/src/ops.rs`

Impacted assertions:

- enum serde and display coverage
- namespace and coin-type mappings
- `ALL_CHAIN_TYPES.len()`
- signer registry coverage
- full pipeline tests for “all supported families”
- wallet lifecycle tests asserting exact account counts

Current gap to account for:

- the existing Rust signer integration matrix is already incomplete for the full enum set, so a Stellar PR should not assume current tests provide full family coverage.

New fixture needs:

- deterministic address fixture from a fixed mnemonic
- deterministic private-key address fixture if private-key import is supported
- signing vector fixtures if the signer implementation exposes deterministic signatures

### Node Tests

Current location:

- `bindings/node/__test__/index.spec.mjs`

Impacted assumptions:

- account count `8`
- exact supported-chain arrays
- docs/examples embedded in test names/comments

Minimum additions:

- address derivation test for new family
- wallet account presence assertion
- signing test if public API exposes it

### Python Tests

Current location:

- `bindings/python/tests/test_bindings.py`

Impacted assumptions:

- account count `8`
- chain-presence assertions
- import docs mirrored in tests

Minimum additions:

- same as Node

### Fixture Strategy Recommendation

Use one fixed mnemonic-based characterization vector across:

- Rust
- Node
- Python

That keeps the family-addition review tight and avoids per-language drift.

For any new signing primitive, add at least:

- one success vector
- one malformed-input vector
- one unsupported-operation vector if send/broadcast is deferred

## Recommended Upstream Strategy

The lowest-risk path is:

1. upstream Stellar as a chain family plus address derivation first
2. keep PR 1 free of new signing API surface if possible
3. add the Stellar-specific signing operation in a second PR only after maintainers agree on shape
4. defer `sign_and_send` until there is clear agreement that OWS should own the transport path

That sequence matches the current codebase boundaries:

- chain family support is cross-cutting but conceptually simple
- signing API extensions are the real design decision
- send/broadcast support is the most implementation-heavy and least portable piece

## Reviewer Checklist

- Does the new family fit the existing `ChainSigner` trait without distorting semantics?
- Does it reuse an existing curve, or are we accidentally introducing third-curve assumptions?
- Are we updating every place that assumes “8 chains”?
- Are we keeping docs clear about normative spec vs reference implementation behavior?
- Are we deferring full broadcast support until the core signer surface is stable?
- If a new operation is added, are Node and Python bindings updated in the same PR?
