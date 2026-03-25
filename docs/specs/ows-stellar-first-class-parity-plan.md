# OWS Stellar First-Class Parity Plan

## Status

Draft

## Last Updated

2026-03-25

## Purpose

This document maps the current `open-wallet-standard/core` chain implementations against the current Stellar draft integration and defines what it would take to bring Stellar to first-class parity in a way that matches how OWS actually works today.

This is not a generic “add another chain” note. It is specifically about:

- Stellar support in the existing OWS universal-wallet architecture
- OpenZeppelin smart-account support as the primary Stellar wallet model
- OpenZeppelin Relayer / Channels as the preferred fee-payment and submission path

## Primary Sources

OWS:

- https://docs.openwallet.sh/
- https://github.com/open-wallet-standard/core

Stellar:

- https://developers.stellar.org/docs/build/guides/transactions/signing-soroban-invocations
- https://developers.stellar.org/docs/learn/fundamentals/contract-development/authorization
- https://developers.stellar.org/docs/build/guides/auth/contract-authorization

OpenZeppelin:

- https://docs.openzeppelin.com/stellar-contracts/accounts/signers-and-verifiers
- https://docs.openzeppelin.com/stellar-contracts/accounts/policies
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/packages/accounts
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/examples/multisig-smart-account
- https://docs.openzeppelin.com/relayer/stellar

Local implementation references:

- [chain.rs](../../ows-core/ows/crates/ows-core/src/chain.rs)
- [wallet_file.rs](../../ows-core/ows/crates/ows-core/src/wallet_file.rs)
- [traits.rs](../../ows-core/ows/crates/ows-signer/src/traits.rs)
- [mod.rs](../../ows-core/ows/crates/ows-signer/src/chains/mod.rs)
- [ops.rs](../../ows-core/ows/crates/ows-lib/src/ops.rs)
- [types.rs](../../ows-core/ows/crates/ows-lib/src/types.rs)
- [stellar.rs](../../ows-core/ows/crates/ows-signer/src/chains/stellar.rs)

## Executive Summary

The right parity target is not “make Stellar look exactly like EVM.”

The right target is:

1. Make Stellar a first-class chain family in the universal wallet model.
2. Make the OZ `C...` smart account the first-class wallet identity for Stellar smart-wallet flows.
3. Keep the derived Ed25519 `G...` signer as the custody anchor in OWS storage.
4. Make `signAuthEntry` the canonical Stellar smart-wallet signing primitive.
5. Add a first-class relayer-backed send profile for Stellar smart-wallet flows instead of pretending generic raw-transaction `signAndSend` is the right abstraction.

That is how Stellar reaches first-class parity without forcing a fake EVM-style model onto Soroban smart accounts.

## What “First-Class Parity” Means In OWS

In this repo, a chain is first-class when it has most or all of the following:

- a `ChainType` and canonical identifiers
- universal-wallet derivation via `ALL_CHAIN_TYPES`
- address derivation and account entries in `accounts[]`
- CLI + Node + Python exposure
- docs in supported chains and signing interface
- deterministic signing behavior with test vectors
- clear statement of what send/broadcast paths are supported

Important repo reality:

- not every OWS chain has full `signAndSend` parity today
- only some chains implement `encode_signed_transaction`
- broadcast support is already uneven across chains

So for this repo, first-class parity does not require Stellar to mimic EVM raw-transaction flows. It requires Stellar to have a correct, documented, tested chain profile with an equally first-class smart-wallet profile layered on top.

## Current OWS Chain Pattern Map

### EVM

Current behavior:

- HD-derived account in `accounts[]`
- generic `signTransaction`
- generic `signAndSend`
- chain-specific `signTypedData`
- address is the operational wallet identity

Why it feels first-class:

- it has a complete identity, signing, encoding, and broadcast story
- it also has a chain-specific extra operation beyond the generic interface

### Solana

Current behavior:

- HD-derived account in `accounts[]`
- generic `signTransaction`
- `extract_signable_bytes`
- `encode_signed_transaction`
- generic `signAndSend`

Why it matters for Stellar:

- Solana proves OWS already tolerates a chain whose wire transaction format needs special extraction and re-encoding rules
- this is a good precedent for “generic interface plus chain-specific transaction semantics”

### Sui

Current behavior:

- HD-derived account in `accounts[]`
- generic `signTransaction`
- chain-specific intent hashing inside the signer
- chain-specific encoded wire signature format
- generic `signAndSend`

Why it matters for Stellar:

- Sui proves OWS already supports a chain where “sign transaction” means “apply chain-specific intent rules first”
- it is the best precedent for a chain profile with nontrivial signing semantics that still feels first-class

### Bitcoin / Cosmos / Tron / TON / Filecoin / Spark

Current behavior:

- all are present as chain families
- all participate in wallet derivation
- signing support exists
- send/broadcast support is mixed and not uniform

Why this matters:

- OWS today already distinguishes between “supported chain family” and “full transport parity”
- Stellar can become first-class before it has a universal raw-XDR send path

## Current Stellar Draft Map

What exists now in the draft branch:

- `ChainType::Stellar`
- Stellar pubnet and testnet identifiers
- Ed25519 `G...` address derivation
- universal-wallet inclusion
- CLI + Node + Python exposure
- classic Stellar `TransactionEnvelope` signing via `signTransaction`
- `signAuthEntry`
- explicit rejection of generic Stellar `signAndSend`
- real auth-entry XDR fixtures
- real classic-envelope signing fixtures

What is still missing:

- first-class OZ smart-account identity model
- linked `C...` account metadata conventions
- `WalletInfo` / binding surface for smart-account identities
- real OZ verifier compatibility tests
- relayer / Channels-backed submission profile
- wallet creation or linking flow for smart accounts
- explicit conformance profile for Stellar smart-wallet behavior

## The Core Architectural Insight

Stellar has two valid identities in this system:

1. the signer identity: an Ed25519 `G...` account derived from the OWS mnemonic
2. the wallet identity: an OZ smart-account `C...` contract address

OWS needs both.

The right split is:

- store the `G...` signer in `accounts[]` because it is seed-derived and custody-backed
- store the `C...` smart account in wallet metadata because it is deployed state, not derivation output
- present the `C...` smart account as the primary wallet identity for Stellar smart-wallet flows in the CLI and bindings

This is the biggest structural change still missing from the draft branch.

## Target Parity Model For Stellar

### 1. Chain Parity

Stellar should have the same baseline chain-family standing as Solana or Sui:

- canonical chain ids
- derivation path and address encoding
- universal-wallet participation
- docs and fixtures
- Node / Python / CLI coverage

### 2. Smart-Wallet Parity

Stellar should also have a first-class smart-wallet profile:

- linked OZ smart-account `C...` identities
- a designated primary smart account per network
- signer metadata for the underlying Ed25519 key
- verifier metadata for OZ `Signer::External`
- relayer metadata for Channels / Relayer send paths

### 3. Signing Parity

For Stellar smart-wallets, the parity primitive is not `signTransaction`.

The parity primitive is:

- `signAuthEntry`

That should be treated the same way OWS already treats `signTypedData` for EVM:

- a chain-specific first-class operation that exists because the generic signing interface is not enough

### 4. Transport Parity

Stellar should not pretend to have parity by forcing smart-wallet sends through generic raw-transaction signing.

Transport parity for Stellar should mean:

- first-class documentation that Stellar smart-wallet submission is relayer-backed
- a supported relayer transport profile
- explicit mapping to OZ Relayer authorization modes, especially `xdr` auth-entry support

## Recommended Wallet Model

## Storage

Keep `accounts[]` unchanged for the custody anchor:

- include the derived Stellar `G...` signer account

Add metadata conventions for smart accounts:

```json
{
  "metadata": {
    "stellar": {
      "smart_accounts": [
        {
          "network": "stellar:testnet",
          "address": "C...",
          "primary": true,
          "signer_address": "G...",
          "signer_public_key": "hex...",
          "signer_kind": "external_ed25519",
          "verifier_contract": "C...",
          "context_rule_mode": "default",
          "relayer": {
            "provider": "openzeppelin",
            "mode": "channels"
          }
        }
      ]
    }
  }
}
```

This keeps storage backward-compatible while making the smart account a first-class concept.

## Binding Surface

Extend `WalletInfo` with a binding-friendly smart-account view rather than hiding everything inside opaque metadata.

Recommended additions:

- `primary_account`
- `smart_accounts[]`

For Stellar, `primary_account` should point at the `C...` wallet identity when a linked smart account exists.

## Recommended Signing Model

### Keep

- `signMessage` for the underlying Ed25519 signer where useful
- `signAuthEntry` for smart-wallet auth

### Add

- validation-oriented fixture coverage for real `HashIdPreimageSorobanAuthorization` bytes
- a typed return shape that can be fed directly into OZ `Signer::External`

### Do Not Force

- generic `signTransaction` for smart-wallet flows

If classic G-account transaction-envelope signing is later added, it should be documented as a separate Stellar classic profile, not as the primary smart-wallet path.

## Recommended Send Model

### Current Draft

- `signAndSend` is intentionally unsupported

### Parity Target

Add a first-class Stellar smart-wallet send profile that is relayer-backed.

Preferred direction:

- use OZ Relayer / Channels
- support the relayer’s auth mode that accepts pre-signed Soroban authorization entry XDR
- keep fee payment and source-account management outside the signer primitive

Important design point:

- this is not the same as EVM `signAndSend`
- it is a relayer transport profile on top of the signing layer

That can still be first-class if it is clearly modeled and documented.

## Plan

### Phase 1: Finish Chain-Family Parity

Goal:

- make Stellar chain support as solid as any other OWS chain family

Work:

- keep `ChainType::Stellar`, aliasing, derivation, docs, and bindings
- add real address-derivation vectors
- add real auth-entry preimage fixtures
- define the Stellar chain-profile conformance claim

Exit criteria:

- Stellar is fully documented in supported chains, storage, signing, and conformance docs
- all bindings and CLI surfaces are aligned
- fixtures are stable and deterministic

### Phase 2: Add First-Class Smart-Account Identity

Goal:

- make the OZ `C...` account a first-class wallet identity rather than an external convention

Work:

- define metadata schema for linked smart accounts
- surface `primary_account` and `smart_accounts[]` in `WalletInfo`
- add CLI display for linked smart accounts
- add helper operations for linking / unlinking smart accounts

Exit criteria:

- a wallet can contain both its custody anchor and its smart-account identity
- SDK consumers do not need to parse raw metadata to discover the `C...` wallet

### Phase 3: Make Signing Correct Against OZ

Goal:

- prove that OWS-produced signatures are valid inputs to OZ smart accounts

Work:

- add real preimage XDR fixtures
- add end-to-end compatibility tests against OZ `ed25519_verifier`
- prove a signature validates in `__check_auth`
- document exactly what OWS signs and what it does not build

Exit criteria:

- at least one fixture-backed path proves OWS `signAuthEntry` output works with OZ `Signer::External`

### Phase 4: Add Relayer / Channels Transport Parity

Goal:

- make submission a first-class part of the Stellar smart-wallet story

Work:

- define a relayer-backed send profile
- support Channels / Relayer request construction
- document authorization mode mapping, especially pre-signed XDR auth entries
- surface relayer configuration in wallet metadata or config

Exit criteria:

- Stellar smart-wallet flows have a supported submit path, not just an off-box note

### Phase 5: Conformance And Examples

Goal:

- make the implementation upstreamable and reviewable

Work:

- add machine-readable vectors
- add CLI / Node / Python examples
- add one documented end-to-end example: create wallet, link smart account, sign auth entry, send through relayer

Exit criteria:

- the feature can be described as a coherent OWS Stellar profile rather than a narrow draft patch

## PR Sequencing

### PR 1: Chain Profile Hardening

Scope:

- keep current Stellar chain family support
- keep `signAuthEntry`
- keep generic Stellar `signTransaction` unsupported
- add real fixtures and docs hardening

### PR 2: Smart-Account Identity Model

Scope:

- metadata conventions
- binding and CLI surface for linked `C...` accounts
- primary-account semantics

### PR 3: OZ Compatibility Proof

Scope:

- real compatibility tests with OZ verifier flow
- explicit contract of what OWS signs

### PR 4: Relayer / Channels Send Profile

Scope:

- submission adapter and docs
- no attempt to masquerade as generic raw-transaction send

## Risks And Open Questions

### 1. Smart-Account Metadata Shape

Open question:

- should OWS standardize a cross-chain smart-account metadata shape, or should Stellar metadata remain chain-specific for now

Recommendation:

- keep it Stellar-specific for the first upstream pass

### 2. `signAuthEntry` Validation Depth

Open question:

- should OWS validate that input bytes are actually canonical XDR preimages before signing

Recommendation:

- keep the current byte-signing primitive, but add optional validation helpers and real fixtures so misuse is harder

### 3. Classic Stellar Support

Open question:

- should OWS eventually support classic transaction-envelope signing for G-accounts

Recommendation:

- yes, but treat it as a separate Stellar classic profile after smart-wallet parity lands

### 4. Relayer Surface Placement

Open question:

- should relayer-backed send be a core `signAndSend` implementation detail or a separate Stellar send method

Recommendation:

- do not force it into raw `signAndSend` semantics until the API shape is proven

## Recommended Immediate Next Step

The next document should be a file-by-file implementation plan for Phase 2 and Phase 3:

- exact Rust types to extend
- exact binding types to extend
- exact metadata schema
- exact OZ compatibility fixture strategy

That is the shortest path from the current draft branch to a credible upstream PR set.
