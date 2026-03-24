# Implementation Plan: OWS for Stellar Smart Wallets

## Status

Draft

## Last Updated

2026-03-24

## Purpose

This document turns the companion spec into an execution plan. It breaks the work into phases, defines deliverables, dependencies, verification strategy, risks, and decision gates.

The intended delivery is an upstream feature PR sequence against `open-wallet-standard/core`, not a repo-wide cleanup project.

Companion document:

- `docs/specs/ows-stellar-smart-wallet-spec.md`
- `docs/specs/ows-stellar-upstreaming-plan.md`
- `docs/specs/ows-stellar-implementation-surface-audit.md`
- `docs/specs/ows-stellar-pr1-checklist.md`

Research artifacts:

- `docs/research/ows-open-wallet-standard.json`
- `docs/research/stellar-smart-wallet-auth-entry.json`

Official OWS documents that shape this plan:

- `docs/00-specification.md`
- `docs/01-storage-format.md`
- `docs/02-signing-interface.md`
- `docs/03-policy-engine.md`
- `docs/04-agent-access-layer.md`
- `docs/05-key-isolation.md`
- `docs/06-wallet-lifecycle.md`
- `docs/07-supported-chains.md`
- `docs/08-conformance-and-security.md`

Primary upstream sources:

- https://docs.openwallet.sh/
- https://github.com/open-wallet-standard/core

Primary Stellar / OZ sources for this plan:

- https://developers.stellar.org/docs/build/guides/transactions/signing-soroban-invocations
- https://developers.stellar.org/docs/learn/fundamentals/contract-development/authorization
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/README.md
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/mod.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/storage.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/ed25519.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/examples/multisig-smart-account/ed25519-verifier/src/contract.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/webauthn.rs
- https://docs.openzeppelin.com/relayer/1.4.x/guides/stellar-channels-guide

## Executive Plan

Deliver the work in four phases:

1. Foundation and wallet profile
2. Stellar signing surface
3. OpenZeppelin smart-account integration
4. Channels-backed end-to-end flow and hardening

The recommended MVP is:

- Stellar testnet only
- Ed25519 only
- OWS `signAuthEntry`
- OpenZeppelin `Signer::External` single-signer proof with a default context rule
- linked `C...` smart-account metadata with primary-account surfacing
- optional 2-of-3 multisig proof after the single-signer path is stable
- OZ Channels as the preferred send path once signing works

Passkeys, WebAuthn, and secp256r1 should remain out of scope until the Ed25519 path is stable.

## Phase 0: Project Setup

## Goals

- establish repo structure
- pin scope
- document primary design decisions
- define test environments

## Tasks

- create docs structure
- store research artifacts in repo
- publish architecture spec
- publish implementation plan
- draft the upstream design note or issue before interface-affecting code work
- decide the initial implementation language and packaging strategy
- decide whether local implementation work happens in:
  - a fork of OWS
  - or a local checkout of upstream with a feature branch

## Deliverables

- spec document
- implementation plan
- initial repo layout

## Exit Criteria

- scope is explicit
- MVP is narrowed to Ed25519 and testnet
- implementation ownership is clear

## Phase 1: OWS Stellar Foundation and Wallet Profile

## Goals

- add Stellar to the OWS data model
- support key derivation and address generation
- make the core library understand Stellar as a chain family
- define how linked OZ smart accounts are represented and surfaced without breaking current OWS wallet storage expectations

## Work Items

### 1. Chain Model

- add `Stellar` to OWS chain enums
- add namespace parsing for Stellar
- register known Stellar testnet and pubnet chains
- add coin type `148`
- update only the touched tests and docs that must change because Stellar increases the supported family set

### 2. Derivation

- implement Stellar HD derivation using `m/44'/148'/{index}`
- derive Ed25519 keys from mnemonic
- generate classic Stellar G-addresses from public keys

### 3. Wallet Serialization

- ensure Stellar accounts are persisted as normal wallet accounts
- verify account id formatting
- define provisional CAIP-style chain ids
- preserve compatibility with existing OWS wallet artifact expectations
- preserve unknown metadata fields during non-destructive updates
- reject unsupported schema versions and unknown required fields
- keep derived `G...` signer accounts in `accounts[]` for storage compatibility
- define linked `C...` smart-account metadata and a `primarySmartAccount` convention for Stellar wallets
- surface the `C...` account as the primary operational wallet identity in CLI and SDK output when metadata is present

### 4. CLI and SDK Plumbing

- allow `stellar:testnet` and `stellar:pubnet` selection in CLI and SDK
- surface Stellar wallet info in account lists
- resolve any friendly aliases to canonical Stellar chain ids before downstream handling
- display linked smart-account metadata and the primary `C...` wallet when present

### 5. API Key and Policy Compatibility

- ensure Stellar requests work under the existing API key file model
- preserve owner mode and agent mode behavior exactly
- preserve fail-closed policy semantics for token-backed requests
- preserve lifecycle semantics for create, import, list, export, and delete flows

## Deliverables

- Stellar chain type support in core
- Stellar account derivation in wallet creation and import flows
- CLI and SDK recognition of Stellar accounts
- linked smart-account metadata shape and display behavior

## Verification

- unit tests for chain parsing
- unit tests for derivation path generation
- golden tests for known mnemonic to G-address derivation
- tests for canonical id parsing and alias normalization
- wallet lifecycle test including create, import, export, and list
- wallet artifact round-trip tests under existing OWS schema expectations
- wallet metadata round-trip tests for linked `C...` smart-account entries
- tests that update existing docs/examples asserting hard-coded family counts

## Exit Criteria

- a new OWS wallet can produce a Stellar account entry
- a wallet imported from a mnemonic reproduces the same Stellar address
- a wallet can persist and re-load a linked OZ smart-account `C...` address without schema breakage
- the feature lands without unrelated changes outside touched Stellar code paths and affected tests/docs

## Phase 2: Stellar Signing Surface

## Goals

- implement the right signing primitive for Stellar smart wallets
- support auth-entry signing as a first-class operation

## Work Items

### 1. Define API Surface

- add `signAuthEntry` request/response types
- use serialized auth-entry preimage input, not a higher-level auth builder
- prefer base64 XDR as the canonical payload encoding in the MVP
- settle the exported operation name early so upstream review can decide between `signAuthEntry` and a less ambiguous preimage-specific name
- decide whether raw hash signing is exposed or kept internal
- decide how the new operation is described in profile and conformance language
- explicitly keep nonce selection, expiration selection, and invocation-tree construction outside OWS core
- define a Stellar-aware policy context rather than overloading generic transaction fields
- reserve a later path for a Channels-backed `signAndSend` transport rather than treating raw RPC broadcast as the target model

### 2. Implement Signing Logic

- build Stellar signing adapter
- sign `HashIdPreimageSorobanAuthorization` payloads with Ed25519
- return signature bytes in a binding-friendly format
- ensure the signed payload matches Soroban host expectations exactly

### 3. Optional Classic Support

- optionally support classic transaction-envelope signing for G-accounts
- keep this separate from auth-entry signing
- do not let classic support block the smart-account PR path

### 4. CLI Support

- add a command for auth-entry signing
- support input through file, stdin, or inline payload
- ensure CLI surfaces the primary linked `C...` smart account when targeting Stellar smart-wallet flows

### 5. SDK Bindings

- expose `signAuthEntry` in Node bindings
- expose `signAuthEntry` in Python bindings if required for scope
- keep Node and Python behavior aligned with the Rust core and shared fixtures

## Deliverables

- library-level `signAuthEntry`
- CLI support for auth-entry signing
- Node SDK support
- upstreamable Rust-first implementation that fits the current OWS architecture
- a clear unsupported-path error for raw `signAndSend` under the Stellar smart-wallet profile until Channels transport lands

## Verification

- unit tests that verify signature correctness against Stellar SDK validation logic
- fixture-based tests using known auth-entry preimages
- error handling tests for malformed payloads
- local policy branch tests
- binding parity tests across Rust, Node, and Python for the same fixtures
- negative tests for unsupported `signAndSend` and `signTypedData` behavior under the Stellar smart-wallet profile
- negative tests for expired API keys and denied policy paths
- machine-readable vectors suitable for future OWS conformance claims
- vectors that prove replay and expiration fields are signed as part of the preimage, not reinterpreted by OWS

## Exit Criteria

- a client can ask OWS to sign a Stellar auth entry and receive a valid Ed25519 signature
- unsupported Stellar operations fail clearly and consistently

## Phase 3: OpenZeppelin Smart Account Integration

## Goals

- prove that OWS-managed Stellar keys work with OpenZeppelin smart accounts
- establish the minimum proof needed to justify the upstream feature

## Work Items

### 1. Single-Signer Example

- deploy OZ Ed25519 verifier
- deploy OZ smart account with one external signer
- configure one `ContextRuleType::Default` rule
- configure signer key from OWS-derived public key
- use `Signer::External` only

### 2. Multisig Example

- deploy threshold policy
- deploy smart account with 3 Ed25519 signers
- require 2-of-3 approvals
- use `Signer::External` only

This can be deferred until after the single-signer path if it risks delaying the upstream PR.

### 3. Client Integration

- build helper code to:
  - simulate in recording mode
  - extract auth-entry preimages
  - build OpenZeppelin `AuthPayload.signers`
  - select and attach one `context_rule_id` per auth context
  - sign with OWS
  - attach signatures
  - re-simulate in enforcing mode

### 4. Account Metadata

- persist linked smart-account addresses locally
- define minimal metadata format with `primarySmartAccount`
- surface the linked `C...` account as the main wallet identity in example UX

### 5. Signer and Verifier Constraints

- enforce OZ external-key size assumptions in fixtures and examples
- verify canonical key handling for duplicate external signers
- document why `Delegated` signers are excluded from the MVP

## Deliverables

- testnet deployment scripts or examples
- client helper layer
- end-to-end single-signer example
- optional end-to-end multisig example
- linked smart-account metadata example and docs

## Verification

- contract tests using OZ example patterns
- integration tests against Stellar testnet or local environment
- successful `__check_auth` validation using OWS signatures
- multisig threshold validation
- explicit validation that `context_rule_ids` length matches auth-context count
- negative tests for delegated-signer paths remaining unsupported in the MVP
- validation that the linked `C...` smart account is surfaced as primary in example wallet output

## Exit Criteria

- the OZ smart account accepts OWS-generated signatures
- the single-signer example is reproducible

## Phase 4: Channels-Backed End-to-End Submission

## Goals

- complete the full smart-wallet flow
- support real submission through OZ Channels

This phase is not required for the first upstream signing PR if auth-entry signing and `__check_auth` verification are already proven, but it is the preferred send story for the full Stellar smart-wallet profile.

## Work Items

### 1. Channels Interface

- define request payload to OZ Channels
- decide trust boundaries and validation requirements
- document that the channel-account source account spends the sequence number in this flow

### 2. Build and Submit

- extract `func` and `auth` from the assembled invocation
- submit through OZ Channels
- capture returned transaction id, hash, and status
- keep raw XDR fallback and fee-bump as optional later enhancements, not part of MVP scope

### 3. Safety Checks

- ensure relayer or channel account is never accidentally added to auth tree
- reject expired or malformed auth payloads
- verify network passphrase and target RPC consistency

### 4. Optional Self-Hosted Relayer

- document a minimal self-hosted relayer service or script only as a fallback path
- keep scope small and testnet-only for MVP

## Deliverables

- Channels request/response contract
- working submission example
- end-to-end integration script

## Verification

- end-to-end transaction success on testnet
- negative tests for malformed auth entries
- negative tests for replay or expiration failures
- Channels-side or relayer-side rejection tests

## Exit Criteria

- a contract-account transaction can be:
  - built
  - simulated
  - auth-signed by OWS
  - re-simulated
  - submitted through Channels

## Phase 5: Hardening and Developer Experience

## Goals

- improve operational quality
- reduce integration friction
- document security boundaries

## Work Items

### 1. Local Policy Extensions

- add Stellar-aware local policy hooks
- optionally inspect:
  - contract address
  - function name
  - asset or token involved
  - max spend values

### 2. Observability

- log signing events without leaking secrets
- emit request ids for tracing
- add debug tooling for failed auth flows
- add Stellar audit-log entries that fit the existing append-only OWS model

### 3. DX Improvements

- scaffold scripts for smart-account deployment
- docs for local development
- examples for Node integration

### 4. Security Review

- review key handling and zeroization assumptions
- review Channels / relayer trust boundaries
- review auth-entry payload handling
- review replay and expiration handling
- confirm policy-before-decrypt for token-backed requests
- confirm unknown declarative rules and executable-policy failures deny

## Deliverables

- improved policies
- diagnostics
- deployment and usage docs
- security checklist

## Exit Criteria

- the MVP is understandable and operable by another engineer without tribal knowledge

## Deferred Phase: Passkey / WebAuthn Support

## Reason for Deferral

This is explicitly not part of the MVP because OWS currently does not support:

- secp256r1 as a signer curve
- WebAuthn credential formats
- passkey-specific signing UX

## Future Scope

- add `secp256r1` to OWS
- support WebAuthn signature packaging
- integrate with OZ WebAuthn verifier contracts
- support passkey-centric smart wallet flows
- support OZ WebAuthn key-data format of 65-byte public key plus credential-id suffix
- support structured `WebAuthnSigData`, not only raw signatures

## Dependencies

## External Dependencies

- OWS codebase access and willingness to modify or fork
- Stellar SDK support for auth-entry assembly and validation
- Stellar testnet access
- OpenZeppelin Stellar smart account contracts
- OpenZeppelin Channels / Relayer service or plugin for end-to-end testing
- optional self-hosted relayer implementation as a later fallback

## Internal Dependencies

- decision on repo structure
- decision on package boundaries
- stable canonical encoding for auth-entry inputs

## Recommended Repo Structure

If implementation is added to this repo, use:

```text
docs/
  research/
  specs/
packages/
  ows-stellar-adapter/
  stellar-auth-client/
examples/
  single-signer/
  multisig-2-of-3/
services/
  channels-transport/
```

If this repo is documentation-first for now, keep:

```text
docs/
  research/
  specs/
```

until implementation starts.

## Milestones

## Milestone 1

Stellar chain support lands in OWS data model and wallet derivation, with linked smart-account metadata shape.

Definition of done:

- wallet creation shows Stellar account
- derivation tests pass
- wallet metadata round-trips a linked `C...` account

## Milestone 2

`signAuthEntry` works locally.

Definition of done:

- signatures verify against Stellar tooling
- SDK and CLI support exist

## Milestone 3

OpenZeppelin single-signer smart account works with OWS key.

Definition of done:

- successful testnet auth flow with a default context rule and linked `C...` account

## Milestone 4

Channels-backed submission works for the single-signer smart account.

Definition of done:

- end-to-end Channels submission confirmed on testnet

## Milestone 5

2-of-3 multisig smart account works with OWS keys.

Definition of done:

- threshold policy path validated

## Risks

## Technical Risks

- CAIP identifier choice for Stellar may need adjustment.
- OWS's current architecture may require more invasive refactoring than expected.
- Stellar auth-entry signing may require tighter coupling with Stellar SDK types than ideal.
- Channels integration may introduce hidden complexity in simulation, transaction assembly, and transport error handling.
- contract-account UX may differ enough from other OWS chains to create abstraction leakage.
- OpenZeppelin published docs may lag the `main` branch source, which affects rule-selection and policy-hook assumptions.
- trying to bundle Channels transport or multisig extras into the first upstream PR may slow acceptance.

## Product Risks

- supporting both classic and smart-wallet Stellar flows at once may dilute the MVP
- early passkey pressure could expand scope prematurely
- Channels assumptions may become product decisions before the wallet layer is stable

## Mitigations

- keep MVP testnet-only
- ship Ed25519 only first
- make auth-entry signing explicit instead of over-generalizing
- separate wallet signing from submission concerns
- lock scope before implementing passkeys
- keep normative OWS changes separate from non-normative CLI, SDK, and Channels transport work
- keep the first upstream PR focused on feature addition, not opportunistic cleanup

## Testing Strategy

## Unit Tests

- chain parsing
- derivation paths
- address generation
- auth-entry signing
- malformed payload handling
- policy evaluation ordering for owner and agent credentials

## Integration Tests

- OWS signing against Stellar SDK verification
- smart account accepting OWS-generated signatures
- multisig threshold behavior
- enforcing-mode simulation flow
- linked `C...` account metadata surfacing in wallet output

## End-to-End Tests

- build and sign a smart-wallet invocation
- Channels assembles and submits
- transaction lands on testnet

## Conformance Artifacts

The official OWS conformance guidance expects machine-readable vectors. Stellar work should produce:

- one wallet derivation vector
- one API key resolution vector for a Stellar request
- one allow policy vector
- one deny policy vector
- one auth-entry signing vector
- one canonical id parsing vector
- negative vectors for unsupported chain, unsupported operation, malformed payload, expired API key, and denied policy

## Negative Tests

- wrong chain id
- wrong network passphrase
- bad signature
- expired auth entry
- wrong signer attached to contract
- Channels submission errors

## Open Decisions

- fork OWS versus build a wrapper package first
- whether to upstream Stellar support into OWS immediately
- whether classic G-account signing is required in MVP
- whether linked contract accounts are stored inside OWS wallet metadata
- whether Channels transport support belongs in this repo or only in companion integration code

## Immediate Next Actions

1. Confirm whether this repo should remain docs-only or become the implementation workspace.
2. Decide whether to fork OWS here or build a separate Stellar adapter package first.
3. Freeze the MVP to:
   - testnet
   - Ed25519
   - single-signer proof first
   - auth-entry signing
   - optional linked smart-account metadata
4. Draft or open the upstream design issue against OWS.
5. Start Phase 1 with chain and derivation support.

## Success Criteria

This project is successful when:

- OWS can manage a Stellar signing key locally
- OWS can sign Soroban auth entries
- an OZ smart account can validate those signatures
- optional later work can add Channels-backed submission without reworking the signing model
- the architecture is clear enough to extend later to passkeys without redoing the Ed25519 path
- the resulting implementation can honestly advertise a profile-scoped OWS conformance claim
