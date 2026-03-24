# Implementation Plan: OWS for Stellar Smart Wallets

## Status

Draft

## Last Updated

2026-03-24

## Purpose

This document turns the companion spec into an execution plan. It breaks the work into phases, defines deliverables, dependencies, verification strategy, risks, and decision gates.

Companion document:

- `docs/specs/ows-stellar-smart-wallet-spec.md`

Research artifacts:

- `docs/research/ows-open-wallet-standard.json`
- `docs/research/stellar-smart-wallet-auth-entry.json`

## Executive Plan

Deliver the work in four phases:

1. Foundation and chain support
2. Stellar signing adapter
3. OpenZeppelin smart-account integration
4. Relayed end-to-end flow and hardening

The recommended MVP is:

- Stellar testnet only
- Ed25519 only
- OWS `signAuthEntry`
- OpenZeppelin single-signer and 2-of-3 multisig examples
- fee-payer submission flow

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
- decide the initial implementation language and packaging strategy
- choose whether this repo will host:
  - only specs and experiments
  - a fork or wrapper around OWS
  - a standalone Stellar adapter package

## Deliverables

- spec document
- implementation plan
- initial repo layout

## Exit Criteria

- scope is explicit
- MVP is narrowed to Ed25519 and testnet
- implementation ownership is clear

## Phase 1: OWS Stellar Foundation

## Goals

- add Stellar to the OWS data model
- support key derivation and address generation
- make the core library understand Stellar as a chain family

## Work Items

### 1. Chain Model

- add `Stellar` to OWS chain enums
- add namespace parsing for Stellar
- register known Stellar testnet and pubnet chains
- add coin type `148`

### 2. Derivation

- implement Stellar HD derivation using `m/44'/148'/{index}`
- derive Ed25519 keys from mnemonic
- generate classic Stellar G-addresses from public keys

### 3. Wallet Serialization

- ensure Stellar accounts are persisted as normal wallet accounts
- verify account id formatting
- define provisional CAIP-style chain ids

### 4. CLI and SDK Plumbing

- allow `stellar:testnet` and `stellar:pubnet` selection in CLI and SDK
- surface Stellar wallet info in account lists

## Deliverables

- Stellar chain type support in core
- Stellar account derivation in wallet creation and import flows
- CLI and SDK recognition of Stellar accounts

## Verification

- unit tests for chain parsing
- unit tests for derivation path generation
- golden tests for known mnemonic to G-address derivation
- wallet lifecycle test including create, import, export, and list

## Exit Criteria

- a new OWS wallet can produce a Stellar account entry
- a wallet imported from a mnemonic reproduces the same Stellar address

## Phase 2: Stellar Signing Adapter

## Goals

- implement the right signing primitive for Stellar smart wallets
- support auth-entry signing as a first-class operation

## Work Items

### 1. Define API Surface

- add `signAuthEntry` request/response types
- decide canonical payload encoding
- decide whether raw hash signing is exposed or kept internal

### 2. Implement Signing Logic

- build Stellar signing adapter
- sign `HashIdPreimageSorobanAuthorization` payloads with Ed25519
- return signature bytes in a binding-friendly format

### 3. Optional Classic Support

- optionally support classic transaction-envelope signing for G-accounts
- keep this separate from auth-entry signing

### 4. CLI Support

- add a command for auth-entry signing
- support input through file, stdin, or inline payload

### 5. SDK Bindings

- expose `signAuthEntry` in Node bindings
- expose `signAuthEntry` in Python bindings if required for scope

## Deliverables

- library-level `signAuthEntry`
- CLI support for auth-entry signing
- Node SDK support

## Verification

- unit tests that verify signature correctness against Stellar SDK validation logic
- fixture-based tests using known auth-entry preimages
- error handling tests for malformed payloads
- local policy branch tests

## Exit Criteria

- a client can ask OWS to sign a Stellar auth entry and receive a valid Ed25519 signature

## Phase 3: OpenZeppelin Smart Account Integration

## Goals

- prove that OWS-managed Stellar keys work with OpenZeppelin smart accounts
- establish deployable example contracts and client flows

## Work Items

### 1. Single-Signer Example

- deploy OZ Ed25519 verifier
- deploy OZ smart account with one external signer
- configure signer key from OWS-derived public key

### 2. Multisig Example

- deploy threshold policy
- deploy smart account with 3 Ed25519 signers
- require 2-of-3 approvals

### 3. Client Integration

- build helper code to:
  - simulate in recording mode
  - extract auth-entry preimages
  - sign with OWS
  - attach signatures
  - re-simulate in enforcing mode

### 4. Account Metadata

- decide whether to persist linked smart-account addresses locally
- if yes, define minimal metadata format

## Deliverables

- testnet deployment scripts or examples
- client helper layer
- end-to-end single-signer example
- end-to-end multisig example

## Verification

- contract tests using OZ example patterns
- integration tests against Stellar testnet or local environment
- successful `__check_auth` validation using OWS signatures
- multisig threshold validation

## Exit Criteria

- the OZ smart account accepts OWS-generated signatures
- both single-signer and multisig examples are reproducible

## Phase 4: Fee Payer and End-to-End Submission

## Goals

- complete the full smart-wallet flow
- support real submission with a G-account fee payer

## Work Items

### 1. Fee Payer Interface

- define request payload to a fee payer
- decide trust boundaries and validation requirements

### 2. Rebuild and Submit

- rebuild transaction with fee payer as source
- simulate in enforcing mode
- sign envelope
- submit to Stellar RPC

### 3. Safety Checks

- ensure fee payer is never accidentally added to auth tree
- reject expired or malformed auth payloads
- verify network passphrase and target RPC consistency

### 4. Example Relayer

- implement a minimal fee payer service or script
- keep scope small and testnet-only for MVP

## Deliverables

- fee payer request/response contract
- working submission example
- end-to-end integration script

## Verification

- end-to-end transaction success on testnet
- negative tests for malformed auth entries
- negative tests for replay or expiration failures
- relayer-side simulation rejection tests

## Exit Criteria

- a contract-account transaction can be:
  - built
  - simulated
  - auth-signed by OWS
  - re-simulated
  - fee-paid
  - submitted

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

### 3. DX Improvements

- scaffold scripts for smart-account deployment
- docs for local development
- examples for Node integration

### 4. Security Review

- review key handling and zeroization assumptions
- review fee payer trust boundaries
- review auth-entry payload handling
- review replay and expiration handling

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

## Dependencies

## External Dependencies

- OWS codebase access and willingness to modify or fork
- Stellar SDK support for auth-entry assembly and validation
- Stellar testnet access
- OpenZeppelin Stellar smart account contracts
- a fee payer or relayer implementation

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
  fee-payer/
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

Stellar chain support lands in OWS data model and wallet derivation.

Definition of done:

- wallet creation shows Stellar account
- derivation tests pass

## Milestone 2

`signAuthEntry` works locally.

Definition of done:

- signatures verify against Stellar tooling
- SDK and CLI support exist

## Milestone 3

OpenZeppelin single-signer smart account works with OWS key.

Definition of done:

- successful testnet auth flow

## Milestone 4

2-of-3 multisig smart account works with OWS keys.

Definition of done:

- threshold policy path validated

## Milestone 5

Full fee-payer submission succeeds on testnet.

Definition of done:

- end-to-end relayed invocation confirmed on chain

## Risks

## Technical Risks

- CAIP identifier choice for Stellar may need adjustment.
- OWS's current architecture may require more invasive refactoring than expected.
- Stellar auth-entry signing may require tighter coupling with Stellar SDK types than ideal.
- fee payer flow may introduce hidden complexity in simulation and transaction rebuilding.
- contract-account UX may differ enough from other OWS chains to create abstraction leakage.

## Product Risks

- supporting both classic and smart-wallet Stellar flows at once may dilute the MVP
- early passkey pressure could expand scope prematurely
- relayer assumptions may become product decisions before the wallet layer is stable

## Mitigations

- keep MVP testnet-only
- ship Ed25519 only first
- make auth-entry signing explicit instead of over-generalizing
- separate wallet signing from submission concerns
- lock scope before implementing passkeys

## Testing Strategy

## Unit Tests

- chain parsing
- derivation paths
- address generation
- auth-entry signing
- malformed payload handling

## Integration Tests

- OWS signing against Stellar SDK verification
- smart account accepting OWS-generated signatures
- multisig threshold behavior
- enforcing-mode simulation flow

## End-to-End Tests

- build and sign a smart-wallet invocation
- fee payer assembles and submits
- transaction lands on testnet

## Negative Tests

- wrong chain id
- wrong network passphrase
- bad signature
- expired auth entry
- wrong signer attached to contract
- fee payer rebuild errors

## Open Decisions

- fork OWS versus build a wrapper package first
- whether to upstream Stellar support into OWS immediately
- whether classic G-account signing is required in MVP
- whether linked contract accounts are stored inside OWS wallet metadata
- whether fee payer tooling belongs in this repo

## Immediate Next Actions

1. Confirm whether this repo should remain docs-only or become the implementation workspace.
2. Decide whether to fork OWS here or build a separate Stellar adapter package first.
3. Freeze the MVP to:
   - testnet
   - Ed25519
   - single-signer plus 2-of-3 multisig
   - auth-entry signing
   - fee-payer submission
4. Start Phase 1 with chain and derivation support.

## Success Criteria

This project is successful when:

- OWS can manage a Stellar signing key locally
- OWS can sign Soroban auth entries
- an OZ smart account can validate those signatures
- a fee payer can submit the resulting transaction
- the architecture is clear enough to extend later to passkeys without redoing the Ed25519 path
