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

## Executive Summary

The recommended strategy is:

1. Add generic Stellar chain support to OWS.
2. Add a Stellar-specific signing primitive for Soroban auth-entry signing.
3. Upstream those changes to OWS in focused PRs.
4. Keep OpenZeppelin smart-account orchestration and fee-payer flow outside OWS unless maintainers explicitly want them in-core.

The goal is to make OWS capable of being a Stellar wallet substrate without forcing OWS to own the entire Stellar smart-wallet application stack.

## What Belongs in OWS

These are good upstream candidates because they are generic wallet-layer capabilities.

### 1. Stellar Chain Family Support

- add `stellar` as a chain family
- add Stellar chain ids for testnet and pubnet
- add Stellar derivation support
- add G-address derivation from Ed25519 keys
- make wallet creation and import include Stellar accounts

### 2. Stellar Signer Support

- add a Stellar signer implementation
- support auth-entry signing as a first-class capability
- optionally support classic Stellar transaction-envelope signing for G-account flows

### 3. CLI and SDK Surface

- support Stellar in wallet info and account listing
- expose `signAuthEntry` or equivalent in CLI
- expose Stellar signing in SDK bindings

### 4. Documentation and Tests

- supported-chains docs
- signing-interface docs
- CLI docs
- test fixtures for Stellar derivation and signing

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
- attach signatures
- re-simulate in enforcing mode

### 3. Fee Payer / Relayer Infrastructure

- fee sponsorship
- transaction assembly with separate source account
- RPC submission service
- abuse controls and rate limits

### 4. Opinionated Wallet UX

- user-facing smart-wallet creation flows
- passkey onboarding UX
- specific multisig templates
- treasury policy presets

## Why This Split Is Correct

OWS is a wallet standard and signer substrate. It should own:

- key custody
- signing
- wallet storage
- wallet API surface

It should not automatically own:

- every chain-specific smart-account UX
- relayer infrastructure
- a full app-specific authorization orchestration stack

For Stellar specifically, the sharp edge is that smart-wallet usage requires more than raw signing. That is exactly why the split matters.

## Workstreams

The work should be divided into four workstreams.

## Workstream A: Core Chain Support

### Scope

- `ChainType::Stellar`
- chain parsing
- known chain registration
- derivation-path support
- wallet account generation

### Files Likely Touched in OWS

- chain model and known chains
- account derivation logic
- wallet lifecycle flows
- CLI parsing and display

### Deliverable

OWS can create and import wallets that contain Stellar accounts.

### Upstream Suitability

High. This should definitely go upstream.

## Workstream B: Stellar Signing Primitives

### Scope

- Stellar signer implementation
- Ed25519 signing path for Stellar
- `signAuthEntry`
- optional classic transaction signing for G-addresses

### Deliverable

OWS can sign Stellar payloads in a way other tools can consume.

### Upstream Suitability

High, with one caveat:

- if maintainers are hesitant about adding `signAuthEntry` directly, propose it as a chain-specific signing extension under the signing-interface rules rather than as a one-off hack

## Workstream C: OpenZeppelin Smart Account Integration

### Scope

- map OWS-derived keys to OZ external signers
- deploy verifier and policy contracts
- prove `__check_auth` works with OWS signatures

### Deliverable

An end-to-end demo that shows OWS-managed keys authorizing an OZ smart account.

### Upstream Suitability

Low to medium.

This is better as:

- an `examples/` package in a companion repo
- a separate adapter package
- or a docs/example contribution if OWS maintainers want ecosystem examples

It should not block the core Stellar PR.

## Workstream D: Fee Payer Flow

### Scope

- relayed submission
- separate source account
- enforcing-mode simulation before submit

### Deliverable

A complete testnet smart-wallet flow.

### Upstream Suitability

Low.

This should stay out of OWS core unless OWS explicitly expands into relayer-owned chain flows.

## Recommended PR Sequence

Do not open one large PR. Use a sequence of focused PRs.

## PR 1: Stellar Chain Model and Derivation

### Contents

- add Stellar chain family
- add testnet and pubnet ids
- add derivation path and account generation
- add wallet lifecycle tests
- add supported-chain docs

### Goal

Get maintainers comfortable with Stellar as a supported chain before introducing new signing surface area.

### Risk

Low.

This is the easiest PR to review and has the cleanest boundaries.

## PR 2: Stellar Signing Support

### Contents

- add Stellar signer implementation
- add auth-entry signing support
- add SDK and CLI support
- add signing tests and fixtures
- update signing-interface docs

### Goal

Make OWS useful for Stellar beyond address derivation.

### Risk

Medium.

This PR likely triggers interface discussion because `signAuthEntry` is not identical to existing transaction signing.

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
- fee-payer client and service examples

Suggested package names:

- `ows-stellar-oz-adapter`
- `stellar-smart-wallet-client`
- `ows-stellar-relayer-example`

## Execution Order

The recommended order is:

1. Build a local proof of concept off a fork or branch of OWS.
2. Validate derivation and auth-entry signing end to end.
3. Split the proof of concept into upstreamable commits.
4. Open PR 1 to OWS.
5. While PR 1 is under review, keep developing OZ integration outside OWS.
6. Open PR 2 once the core chain model lands or maintainers approve the direction.
7. Publish the adapter/examples repo after the signing surface is stable.

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
- no requirement to own relayer infrastructure
- no requirement to own OpenZeppelin-specific wallet UX

Key message:

"This adds Stellar as a first-class wallet/signing target while keeping smart-account orchestration out of OWS core."

## For Stellar / OpenZeppelin Reviewers

Frame the proposal as:

- OWS becomes the local signer vault
- OZ smart accounts remain the on-chain policy engine
- relayer remains a separate trust boundary

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
- a fee payer can submit the resulting transaction
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

### Owner 3: OZ Integration

- verifier deployment
- smart-account setup
- multisig example flows

### Owner 4: Relayer / Fee Payer

- fee payer service
- transaction assembly
- submission and enforcing-mode checks

## Risks and Failure Modes

## Risk 1: OWS Maintainers Reject `signAuthEntry`

Mitigation:

- propose it as a chain-specific signing extension
- keep the API small and explicit
- show that Stellar smart wallets cannot be supported cleanly without it

## Risk 2: Too Much Logic Leaks into OWS

Mitigation:

- keep OZ orchestration and relayer flow outside core
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
2. Stand up a working branch that modifies OWS for:
   - Stellar chain support
   - derivation
   - auth-entry signing
3. Build a minimal local proof of concept.
4. Slice that proof of concept into PR 1 and PR 2.
5. Open the first upstream PR once the diff is clean.

## Recommended Near-Term Deliverables

- a fork or working branch of OWS with Stellar support
- a short upstream RFC or issue summary for OWS maintainers
- PR 1 covering chain support and derivation
- a separate integration package for OZ examples

## Bottom Line

The idea is:

- extend OWS with Stellar
- upstream the generic Stellar wallet and signing support to OWS
- keep OZ smart-account orchestration and fee-payer submission as separate integration work

That gives the cleanest review path, the best chance of landing upstream, and the least architectural confusion.
