# Upstreaming Plan: Adding Stellar Support to OWS

## Status

Draft

## Last Updated

2026-03-24

## Purpose

This document breaks down how to add Stellar support to the Open Wallet Standard (OWS) implementation and how to divide that work into upstreamable pull requests versus separate integration work.

Companion docs:

- `docs/specs/ows-stellar-smart-wallet-spec.md`
- `docs/specs/ows-stellar-smart-wallet-plan.md`
- `docs/specs/ows-stellar-implementation-surface-audit.md`
- `docs/specs/ows-stellar-pr1-checklist.md`
- `docs/specs/ows-stellar-first-class-parity-plan.md`

Primary OWS sources:

- https://docs.openwallet.sh/
- https://github.com/open-wallet-standard/core

Primary Stellar / OZ sources:

- https://developers.stellar.org/docs/build/guides/transactions/signing-soroban-invocations
- https://developers.stellar.org/docs/learn/fundamentals/contract-development/authorization
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/README.md
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/mod.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/storage.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/ed25519.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/examples/multisig-smart-account/ed25519-verifier/src/contract.rs
- https://docs.openzeppelin.com/relayer/1.4.x/guides/stellar-channels-guide

## Executive Summary

The recommended strategy is:

1. Add Stellar chain support and signer derivation to OWS.
2. Add a Stellar smart-wallet profile that can surface a linked OZ `C...` account as the primary wallet identity while keeping derived `G...` signers in storage.
3. Add a Stellar-specific signing primitive for Soroban auth-entry signing.
4. Upstream those changes to OWS in focused PRs.
5. Keep OZ smart-account orchestration outside OWS core, but plan for Channels as the preferred send transport once the signing surface exists.

The goal is to make OWS capable of being a Stellar wallet substrate inside the repo's current architecture, without turning the work into a generic provider framework or a cleanup project.

## Official OWS Framing That Matters

The upstream strategy should follow the official OWS document boundaries:

- normative core docs define storage, signing, policy, lifecycle, supported chains, and conformance
- optional docs define access-layer and key-isolation profiles
- CLI and SDK docs are non-normative reference implementation documents

That means upstream PRs should lead with normative behavior and artifact changes first, then follow with CLI and SDK updates.

It also means the work should fit the current reference implementation model:

- one universal wallet
- one closed set of chain families
- one shared signing core

This repo does not currently have a wallet-provider abstraction, so the Stellar contribution should be expressed as:

- one more supported chain family
- one smart-account-specific signing operation
- one metadata-backed smart-wallet profile that makes the linked `C...` account first-class without redefining how every OWS account entry works

## Source of Truth for the Stellar / OZ Side

This upstreaming plan is written against:

- the official Stellar docs for Soroban authorization and auth-entry signing
- the OpenZeppelin `stellar-contracts` `main` branch source and package README

Important caveat:

- some published OpenZeppelin docs pages still describe older rule-selection and policy-hook behavior
- the linked repository target is `main`, whose current source expects explicit `context_rule_ids` and no longer relies on rule auto-discovery

For PR design, the repo source should win over lagging docs pages whenever they conflict.

## What Belongs in OWS

These are good upstream candidates because they are generic wallet-layer capabilities.

### 1. Stellar Chain Family Support

- add `stellar` as a chain family
- add Stellar chain ids for testnet and pubnet
- add Stellar derivation support
- add G-address derivation from Ed25519 keys
- make wallet creation and import include Stellar accounts
- keep the stored account model aligned with existing `accounts[]` entries

### 1a. Linked Smart-Account Profile

- add metadata conventions for linked OZ smart accounts
- allow a Stellar wallet to designate a primary `C...` account
- surface that primary smart account through CLI and SDK output
- keep the derived `G...` signer as the custody anchor underneath

### 2. Stellar Signer Support

- add a Stellar signer implementation
- support auth-entry signing as a first-class capability
- optionally support classic Stellar transaction-envelope signing for G-account flows
- keep the upstream primitive at preimage-signing level rather than building Soroban auth trees in OWS core
- keep the first smart-account implementation focused on OZ `Signer::External`

### 3. CLI and SDK Surface

- support Stellar in wallet info and account listing
- expose `signAuthEntry` or equivalent in CLI
- expose Stellar signing in SDK bindings
- expose linked smart-account metadata and primary-account selection clearly

### 3a. Send Transport

- do not hardcode a bespoke fee-payer protocol into OWS
- reserve a later path for Channels-backed `signAndSend`
- fail clearly for unsupported Stellar smart-wallet send requests until that transport exists

### 4. Documentation and Tests

- supported-chains docs
- signing-interface docs
- storage-format docs if Stellar affects artifacts or audit logging
- conformance-and-security docs if Stellar adds a new chain profile
- CLI docs
- test fixtures for Stellar derivation and signing
- only the touched tests and docs that need to move because Stellar is now supported

## What Should Stay Outside OWS

These are product-level or application-level integrations and should stay in a separate package, example, or companion repo.

### 1. OpenZeppelin Smart Account Deployment Helpers

- deploy verifier contracts
- deploy smart accounts
- configure context rules and policies
- manage signer metadata

### 2. Soroban Invocation Assembly Helpers

- simulate in recording mode
- extract auth-entry preimages
- build OpenZeppelin `AuthPayload`
- choose one `context_rule_id` per auth context
- attach signatures
- re-simulate in enforcing mode

### 2a. OZ-Specific Signer Orchestration

- delegated-signer auth-entry crafting
- verifier-specific key-data handling
- signer-registry and policy-management workflows

### 3. Fee Payer / Relayer Infrastructure

- self-hosted fee sponsorship
- transaction assembly with separate source account
- RPC submission service
- abuse controls and rate limits
- security checks on untrusted incoming transaction XDR

Using the managed OZ Channels service should be documented and surfaced, but operating relayer infrastructure should stay outside OWS core.

### 4. Opinionated Wallet UX

- user-facing smart-wallet creation flows
- passkey onboarding UX
- specific multisig templates
- treasury policy presets

### 5. Unrelated Repo Cleanup

- family-count consistency work outside touched Stellar code paths
- refactors not required for the Stellar feature to function
- opportunistic API or storage redesign

## Why This Split Is Correct

OWS is a wallet standard and signer substrate. It should own:

- key custody
- signing
- wallet storage
- wallet API surface
- wallet identity surfacing for linked smart accounts where the storage model already supports it through metadata

It should not automatically own:

- every chain-specific smart-account UX
- self-hosted relayer infrastructure
- a full app-specific authorization orchestration stack

For Stellar specifically, the sharp edge is that smart-wallet usage requires more than raw signing. The simplification from the OZ source is that the smart account itself can still be the primary wallet identity while OWS remains only the signer substrate underneath it.

## Workstreams

The work should be divided into four workstreams.

## Exact Upstream Touchpoints

The current OWS repo is not pluginized enough for Stellar to be a one-file addition. The likely upstream touchpoints are:

### Normative OWS Docs

- `docs/00-specification.md`
- `docs/01-storage-format.md`
- `docs/02-signing-interface.md`
- `docs/03-policy-engine.md`
- `docs/06-wallet-lifecycle.md`
- `docs/07-supported-chains.md`
- `docs/08-conformance-and-security.md`

### Rust Core And Signer Surface

- `ows/crates/ows-core/src/chain.rs`
- `ows/crates/ows-core/src/lib.rs`
- `ows/crates/ows-core/src/config.rs`
- `ows/crates/ows-signer/src/traits.rs`
- `ows/crates/ows-signer/src/chains/mod.rs`
- `ows/crates/ows-signer/src/chains/stellar.rs`
- `ows/crates/ows-signer/src/lib.rs`
- `ows/crates/ows-lib/src/ops.rs`
- `ows/crates/ows-lib/src/key_ops.rs`

### CLI, SDK, And Binding Surface

- `ows/crates/ows-cli/src/main.rs`
- `ows/crates/ows-cli/src/commands/info.rs`
- `ows/crates/ows-cli/src/commands/derive.rs`
- `ows/crates/ows-cli/src/commands/sign_transaction.rs`
- `ows/crates/ows-cli/src/commands/send_transaction.rs`
- `ows/crates/ows-cli/src/commands/sign_message.rs`
- `ows/crates/ows-cli/src/commands/wallet.rs`
- `bindings/node/src/lib.rs`
- `bindings/node/__test__/index.spec.mjs`
- `bindings/python/src/lib.rs`
- `bindings/python/python/ows/__init__.py`
- `bindings/python/tests/test_bindings.py`
- `docs/sdk-cli.md`
- `docs/sdk-node.md`
- `docs/sdk-python.md`
- `bindings/node/README.md`
- `bindings/python/README.md`

## Existing OWS Drift To Clean Up

The upstream plan should account for current repo inconsistencies only where the Stellar feature directly touches them:

- `ChainType` and the supported-chains docs are not perfectly aligned with the universal-wallet derivation allowlist
- some tests and docs still hard-code an 8-family model
- the binding tests and signer coverage do not fully agree on the current family set

PR 1 should update the touched assertions and docs required for Stellar to land, but it should not broaden into a general cleanup pass.

## Workstream A: Core Chain Support

### Scope

- `ChainType::Stellar`
- chain parsing
- known chain registration
- derivation-path support
- wallet account generation
- linked smart-account metadata shape
- primary `C...` account surfacing

### Files Likely Touched in OWS

- chain model and known chains
- account derivation logic
- wallet lifecycle flows
- CLI parsing and display

### Deliverable

OWS can create and import wallets that contain Stellar signer accounts and can round-trip linked smart-account metadata.

### Upstream Suitability

High. This should definitely go upstream.

## Workstream B: Stellar Signing Primitives

### Scope

- Stellar signer implementation
- Ed25519 signing path for Stellar
- `signAuthEntry`
- optional classic transaction signing for G-addresses
- no delegated-signer orchestration
- no OpenZeppelin `AuthPayload` construction
- no Channels transport in the first upstream PR

### Deliverable

OWS can sign Stellar payloads in a way other tools can consume.

### Upstream Suitability

High, with one caveat:

- if maintainers are hesitant about adding `signAuthEntry` directly, propose it as a chain-specific signing extension under the signing-interface rules rather than as a one-off hack

## Workstream C: OpenZeppelin Smart Account Integration

### Scope

- map OWS-derived keys to OZ external signers
- build `AuthPayload.signers` and `context_rule_ids`
- deploy verifier and policy contracts
- prove `__check_auth` works with OWS signatures
- prove the linked `C...` account can be treated as the primary wallet identity in practice

### Deliverable

An end-to-end demo that shows OWS-managed keys authorizing an OZ smart account.

### Upstream Suitability

Low to medium.

This is better as:

- an `examples/` package in a companion repo
- a separate adapter package
- or a docs/example contribution if OWS maintainers want ecosystem examples

It should not block the core Stellar PR.

## Workstream D: Channels Send Transport

### Scope

- Channels-backed submission
- separate source account managed by the transport
- enforcing-mode simulation before submit

### Deliverable

A complete testnet smart-wallet flow using OZ Channels.

### Upstream Suitability

Low.

This should stay out of OWS core unless OWS explicitly wants a send transport for Stellar smart wallets after the signing layer is merged.

## Recommended PR Sequence

Do not open one large PR. Use a sequence of focused PRs.

## PR 0: Upstream Design Note or Issue

### Contents

- short explanation of Stellar support scope
- explanation that Soroban smart-wallet flows require auth-entry signing
- explanation that the OZ `C...` account is the primary wallet identity even though OWS stores a derived `G...` signer underneath
- note that OWS will sign serialized auth preimages, not construct auth trees
- note that operation naming and policy-context shape need maintainer agreement before implementation lands
- explicit statement that OZ orchestration stays out of OWS core and Channels transport is a later discussion
- note the OpenZeppelin docs/source drift if the target is `main`
- explicit statement that the proposal fits the existing universal-wallet model and does not introduce a provider abstraction

### Goal

Reduce surprise before interface-affecting work lands.

### Risk

Low.

## PR 1: Stellar Chain Model and Derivation

### Contents

- add Stellar chain family
- add testnet and pubnet ids
- add derivation path and account generation
- update the touched hard-coded family-count assertions caused by the new Stellar family
- add wallet lifecycle tests
- add supported-chain docs
- update any storage or conformance docs needed for the Stellar chain profile
- preserve current artifact invariants without unnecessary schema changes
- use canonical Stellar identifiers in artifacts and normalize aliases at the CLI boundary
- keep the stored wallet account model centered on derived `G...` signer accounts
- add metadata conventions for linked OZ smart accounts and primary-account surfacing if maintainers accept that in PR 1

### Goal

Get maintainers comfortable with Stellar as a supported chain before introducing new signing surface area.

### Risk

Low.

This is the easiest PR to review and has the cleanest boundaries.

### Acceptance Criteria

- wallet create and mnemonic import expose a deterministic Stellar account
- lifecycle tests cover create, import, list, export, and delete with Stellar present
- the touched docs, CLI output, and binding tests no longer assume the old family count
- no artifact schema bump is introduced unless strictly necessary
- linked `C...` account metadata round-trips cleanly if included
- no unrelated repo cleanup is bundled into the PR

## PR 2: Stellar Signing Support

### Contents

- add Stellar signer implementation
- add auth-entry signing support
- update normative signing, policy, and conformance docs as needed
- preserve owner-versus-agent policy behavior
- add a Stellar-specific policy context rather than overloading generic transaction fields
- add SDK and CLI support
- add signing tests and fixtures
- update signing-interface docs
- add conformance-oriented vectors
- add Node and Python parity coverage for the same fixtures
- keep nonce, expiration, invocation-tree construction, and `context_rule_ids` selection outside OWS core
- make unsupported `signAndSend`, `signTypedData`, and malformed-input paths fail explicitly under the Stellar smart-wallet profile
- if metadata was deferred from PR 1, add linked smart-account metadata and primary-account display here

### Goal

Make OWS useful for Stellar beyond address derivation.

### Risk

Medium.

This PR likely triggers interface discussion because `signAuthEntry` is not identical to existing transaction signing.

### Acceptance Criteria

- one successful auth-entry signing vector is shared across Rust, Node, and Python
- allow and deny policy fixtures exist for Stellar signing requests
- expired API key and unsupported-operation behavior are covered
- docs make clear that OWS signs preimages and does not build Soroban auth trees

## PR 3: Docs and Examples for Generic Stellar Usage

### Contents

- usage examples
- examples for wallet create and auth-entry sign
- docs on what is supported versus unsupported

### Goal

Reduce friction for adoption and clarify that Stellar smart wallets use a different submission model.

### Risk

Low.

## PR 4: Optional Classic Stellar Envelope Signing

### Contents

- classic G-account transaction signing, if still desired after MVP

### Goal

Support non-smart-wallet Stellar use cases without complicating the smart-wallet MVP.

### Risk

Medium.

This may be worth deferring until after auth-entry support is merged.

## Separate Adapter Repo or Package

In parallel with the OWS PRs, build a separate integration layer for the smart-wallet stack.

Recommended contents:

- OZ smart-account deployment helpers
- auth-entry extraction and attachment helpers
- multisig example flows
- Channels client and optional self-hosted relayer examples

Suggested package names:

- `ows-stellar-oz-adapter`
- `stellar-smart-wallet-client`
- `ows-stellar-channels-example`

## Execution Order

The recommended order is:

1. Read and align with the official OWS normative docs before touching code.
2. Build a local proof of concept off a fork or branch of OWS.
3. Validate derivation and auth-entry signing end to end.
4. Split the proof of concept into upstreamable commits.
5. Open PR 0 or an equivalent design issue.
6. Open PR 1 to OWS.
7. While PR 1 is under review, keep developing OZ integration outside OWS.
8. Open PR 2 once the core chain model lands or maintainers approve the direction.
9. Publish the adapter/examples repo after the signing surface is stable.

## Branch Strategy

Use at least three branches or work areas:

- `ows-stellar-core`
- `ows-stellar-signing`
- `ows-stellar-oz-integration`

This avoids mixing upstreamable generic work with application-level integration.

## Review Strategy

## For OWS Maintainers

Frame the proposal as:

- a new supported chain family
- minimal new abstractions
- no requirement to own self-hosted relayer infrastructure
- no requirement to own OpenZeppelin-specific wallet UX
- full respect for OWS's storage, policy, access-layer, and conformance model

Key message:

"This adds Stellar as a first-class wallet/signing target while letting OWS surface linked OZ smart-account identities without absorbing OZ orchestration into core."

Likely design objections to address directly:

- why `signAuthEntry` is not just `signMessage`
- why auth-entry signing is not the same as classic transaction-envelope signing
- why a Stellar-specific policy context is needed for correct audit and conformance semantics
- why `signAndSend` should be deferred or explicitly unsupported for the smart-wallet MVP
- why the linked `C...` account is user-facing while the derived `G...` account remains the custody anchor in storage

## For Stellar / OpenZeppelin Reviewers

Frame the proposal as:

- OWS becomes the local signer vault
- OZ smart accounts remain the on-chain policy engine
- Channels or another relayer remains a separate trust boundary

Key message:

"OWS is the signer substrate, not a replacement for Soroban auth mechanics or OZ account logic."

## Definition of Success for Each Layer

## Success for OWS Upstream

- Stellar chain family is accepted
- derivation works
- auth-entry signing works
- CLI and SDK support exist

## Success for Companion Integration

- OWS-derived keys can authorize an OZ smart account
- Channels can submit the resulting transaction
- a 2-of-3 multisig demo works on testnet

## Suggested Ownership Split

If multiple people are working on this, split ownership like this:

### Owner 1: OWS Core

- chain model
- derivation
- wallet lifecycle
- signer plumbing

### Owner 2: Stellar Signing Semantics

- auth-entry fixtures
- signature verification tests
- CLI and SDK wiring for signing
- conformance test vectors

### Owner 3: OZ Integration

- verifier deployment
- smart-account setup
- multisig example flows

### Owner 4: Channels / Relayer Transport

- Channels client integration
- optional self-hosted relayer service
- submission and enforcing-mode checks

## Risks and Failure Modes

## Risk 1: OWS Maintainers Reject `signAuthEntry`

Mitigation:

- propose it as a chain-specific signing extension
- keep the API small and explicit
- show that Stellar smart wallets cannot be supported cleanly without it

## Risk 2: Too Much Logic Leaks into OWS

Mitigation:

- keep OZ orchestration and self-hosted relayer flow outside core
- avoid adding contract-deployment or policy-management logic to OWS

## Risk 3: Classic and Smart-Wallet Flows Get Mixed

Mitigation:

- document the two flows separately
- treat auth-entry signing as the smart-wallet path
- treat classic tx-envelope signing as optional

## Risk 4: Passkey Scope Creep

Mitigation:

- explicitly lock MVP to Ed25519
- defer secp256r1 and WebAuthn to a later phase

## Immediate Next Steps

1. Decide whether this repo will host:
   - only docs
   - a fork of OWS
   - or a companion adapter implementation
2. Draft the upstream design issue using the scope in PR 0.
3. Stand up a working branch that modifies OWS for:
   - Stellar chain support
   - derivation
   - auth-entry signing
4. Build a minimal local proof of concept.
5. Slice that proof of concept into PR 1 and PR 2.
6. Open the first upstream PR once the diff is clean.

## Recommended Near-Term Deliverables

- a fork or working branch of OWS with Stellar support
- a short upstream RFC or issue summary for OWS maintainers
- PR 1 covering chain support and derivation
- a separate integration package for OZ examples

## Source Checklist for Upstream Work

Before opening upstream PRs, confirm the changes have been checked against:

- `docs/00-specification.md`
- `docs/01-storage-format.md`
- `docs/02-signing-interface.md`
- `docs/03-policy-engine.md`
- `docs/04-agent-access-layer.md`
- `docs/05-key-isolation.md`
- `docs/07-supported-chains.md`
- `docs/08-conformance-and-security.md`
- `README.md`

Also confirm the PR includes:

- a deterministic derivation fixture shared across Rust, Node, and Python
- at least one successful Stellar auth-entry signing vector
- at least one allow and one deny policy vector for a Stellar request
- negative coverage for malformed payload, unsupported operation, and expired API key

## Bottom Line

The idea is:

- extend OWS with Stellar
- upstream the generic Stellar wallet, linked smart-account profile, and signing support to OWS
- keep OZ smart-account orchestration as separate integration work
- treat Channels-backed send transport as a later integration or follow-on upstream discussion, not a blocker for the first signing PRs

That gives the cleanest review path, the best chance of landing upstream, and the least architectural confusion.
