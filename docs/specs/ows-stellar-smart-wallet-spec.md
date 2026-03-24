# OWS for Stellar Smart Wallets with OpenZeppelin Smart Accounts

## Status

Draft

## Last Updated

2026-03-24

## Summary

This document specifies how to extend the Open Wallet Standard (OWS) for use with a Stellar-style wallet built on Soroban and OpenZeppelin smart accounts.

The core conclusion from the research is:

- OWS is a good fit for the local key vault, policy-gated signing boundary, and multi-tool wallet access model.
- OWS is not yet a direct fit for Stellar smart wallets because Stellar contract accounts do not sign transaction envelopes. They authorize via Soroban auth entries and require a separate G-account fee payer for submission.
- The correct integration is an OWS-backed Stellar signing adapter plus an OpenZeppelin smart account, not a naive reuse of OWS's current `signTransaction` flow.

This spec defines the MVP architecture, required OWS extensions, wallet behavior, threat model assumptions, API surface, implementation boundaries, and acceptance criteria.

## Background

OWS v1.0.0 currently supports the following chain families in its released spec and reference implementation:

- EVM
- Solana
- Bitcoin
- Cosmos
- Tron
- TON
- Sui
- Spark
- Filecoin

Stellar is not currently included in the OWS supported-chain specification or the Rust reference implementation. The released OWS implementation hard-codes supported chain families, known chains, derivation defaults, and signer dispatch in Rust.

At the same time, Stellar smart wallets use a distinct authorization model:

- classic G-accounts can sign transaction envelopes
- contract accounts starting with `C` cannot sign transaction envelopes
- contract accounts authorize through signed Soroban auth entries
- a separate G-account must pay fees and submit the transaction
- auth flows require simulation in recording mode before signing, then enforcing mode before submission

OpenZeppelin's Stellar smart account framework provides a modular account abstraction layer composed of:

- context rules
- signers
- verifiers
- policies

That framework supports both:

- delegated signers, which are Soroban addresses
- external signers, which route signature verification through verifier contracts

The built-in verifier building blocks include:

- Ed25519
- WebAuthn / passkey verification

## Problem Statement

We need a specification for using OWS as the wallet substrate for a Stellar smart wallet that uses OpenZeppelin smart accounts.

The design must answer:

- how Stellar keys are stored and derived in OWS
- how Stellar auth-entry signing fits into OWS
- how an OWS-managed signer maps into OpenZeppelin smart account signers
- how fee payment and submission work when the user wallet is a contract account
- what the MVP should support first
- which features are explicitly deferred

## Goals

- Add Stellar as a first-class chain family in the OWS implementation.
- Support local-first Stellar signing with private keys never exposed upstream.
- Support OpenZeppelin smart accounts using OWS-managed Ed25519 signers in the MVP.
- Support Soroban auth-entry signing as the main smart-wallet authorization flow.
- Support relayed or fee-sponsored submission for contract-account transactions.
- Preserve OWS's local vault and policy-gated access model.
- Define a path to future passkey support without blocking the MVP.

## Non-Goals

- Replacing Stellar wallet UX broadly across classic and Soroban flows in one release.
- Supporting every OpenZeppelin signer mode in the MVP.
- Supporting WebAuthn / secp256r1 in the MVP.
- Defining a production relayer protocol beyond the minimum wallet-facing contract.
- Making OWS's current `signAndSend` semantics work unchanged for Stellar contract accounts.
- Solving general cross-chain account abstraction across non-Stellar smart-wallet systems.

## Key Findings from Research

### Finding 1: OWS spec is chain-extensible, but the reference implementation is not pluginized enough yet

The OWS supported-chains spec says new chains can be added without changing the standard, but the released Rust implementation still requires explicit additions in:

- `ChainType`
- namespace parsing
- known chain registry
- derivation defaults
- signer registry
- chain-specific signer implementation

Implication:

- Stellar can be added cleanly at the spec level.
- Stellar still requires code changes across the Rust implementation.

### Finding 2: Stellar smart wallets are auth-entry native, not tx-envelope native

For contract accounts:

- authorization happens through `__check_auth`
- clients sign auth entries, not the transaction envelope
- the fee payer rebuilds or signs the final envelope and submits it

Implication:

- OWS needs a Stellar-specific authorization operation
- `signTransaction(txXdr)` is insufficient for smart-wallet usage

### Finding 3: OpenZeppelin smart accounts map well to OWS-managed signers

OpenZeppelin smart accounts separate:

- who can sign
- what they can do
- how policy is enforced

This matches OWS conceptually:

- OWS manages key custody and local signing
- the smart account manages on-chain authorization policy

Implication:

- OWS should be the off-chain signer vault
- OpenZeppelin contracts should remain the on-chain policy and account abstraction layer

### Finding 4: Ed25519 is the right MVP

OWS currently supports:

- secp256k1
- ed25519

OWS does not currently support:

- secp256r1
- WebAuthn credential structures

OpenZeppelin's Stellar smart accounts support both Ed25519 and WebAuthn verifier patterns.

Implication:

- MVP should use OWS-managed Ed25519 signers with OZ external verifier contracts
- passkeys should be Phase 2

## Proposed Architecture

## High-Level Design

The system has four distinct layers:

1. Local vault layer
2. Stellar signing adapter layer
3. OpenZeppelin smart account layer
4. Fee payer / relayer layer

### 1. Local Vault Layer

OWS remains responsible for:

- encrypted wallet storage
- chain account derivation
- API key scoped access
- local policy-gated signing
- preventing raw key material from leaving the signer boundary

### 2. Stellar Signing Adapter Layer

A new Stellar-specific adapter extends OWS with:

- Stellar account derivation
- G-address generation
- auth-entry preimage signing
- optional classic message signing
- optional classic transaction-envelope signing for G-account workflows

This adapter is the core new component required by this spec.

### 3. OpenZeppelin Smart Account Layer

The on-chain smart wallet is an OpenZeppelin smart account contract configured with:

- one or more external Ed25519 signers
- context rules
- optional threshold policy
- optional spending-limit policy

The contract account address is the user wallet for Soroban contract authorization.

### 4. Fee Payer / Relayer Layer

A separate G-account service:

- receives a user-prepared invocation with signed auth entries
- verifies the request
- simulates in enforcing mode
- signs the transaction envelope as fee payer
- submits to Stellar RPC

This can be:

- a backend service
- a relayer
- a sponsor
- a wallet-owned fee payer

## System Boundaries

### OWS Responsibilities

- derive and store Stellar keys locally
- expose a Stellar signing API
- sign auth-entry preimages
- optionally sign classic Stellar transactions for G-address flows
- enforce local access control before signing

### Smart Account Responsibilities

- decide who is authorized
- decide what contexts are allowed
- enforce thresholds and spending limits
- validate submitted signatures in `__check_auth`

### Fee Payer Responsibilities

- pay XLM fees
- manage sequence number for submission account
- re-simulate in enforcing mode
- reject malformed or malicious payloads
- submit the final transaction envelope

## Wallet Models

This spec supports two wallet models.

### Model A: Classic Stellar Wallet

The user holds a G-account key derived from the OWS wallet.

Use cases:

- basic Stellar operations
- simpler integrations
- compatibility with tooling that expects classic signatures

This is not the primary target of this spec, but it should remain possible once Stellar support exists in OWS.

### Model B: Soroban Smart Wallet

The user controls a contract account backed by an OpenZeppelin smart account.

Use cases:

- programmable authorization
- multisig
- session or policy-based auth
- relayed submission
- spend limits and scoped permissions

This is the primary target of this spec.

## Chain and Addressing Specification

## Stellar Chain Family

Add a new OWS chain family:

- family name: `stellar`
- primary curve for MVP: `ed25519`
- SLIP-44 coin type: `148`

### Proposed Chain IDs

Use the following provisional CAIP-style identifiers:

- `stellar:pubnet`
- `stellar:testnet`

This should be treated as an implementation decision subject to confirmation against current chain-agnostic namespace guidance at implementation time.

### Account IDs

Use CAIP-10 style account identifiers:

- `stellar:pubnet:G...`
- `stellar:testnet:G...`

For contract accounts:

- `stellar:pubnet:C...`
- `stellar:testnet:C...`

### Derivation

For the MVP, Stellar key derivation should follow Stellar's existing HD path conventions:

- `m/44'/148'/{index}`

If OWS preserves a family-specific derivation path formatter, Stellar should use this path for account derivation.

## API Extensions

## New Core Operation

Add a new operation for Stellar smart-wallet support:

```ts
interface SignAuthEntryRequest {
  walletId: WalletId;
  chainId: ChainId;
  preimageXdr: string; // base64 XDR or hex, choose one canonical encoding
  signerAddress?: string;
  signerIndex?: number;
}

interface SignAuthEntryResult {
  signature: string;
  publicKey?: string;
  keyType: "ed25519";
}
```

Behavior:

- resolve wallet and chain
- authenticate caller
- enforce local policies
- derive or load Stellar signing key
- sign the auth-entry preimage hash according to Stellar expectations
- return signature bytes and optional public key metadata

## Optional Classic Operation

The following operation is optional but recommended:

```ts
interface SignStellarTransactionRequest {
  walletId: WalletId;
  chainId: ChainId;
  transactionXdr: string;
}
```

This is for G-account flows only and is separate from contract-account auth-entry signing.

## No Direct `signAndSend` for Contract Accounts

For Stellar contract accounts, OWS must not pretend to directly own submission.

Instead, the user-facing flow is:

1. build invocation
2. simulate in recording mode
3. extract auth-entry preimages
4. sign auth entries with OWS
5. re-simulate in enforcing mode
6. send to fee payer
7. fee payer signs envelope and submits

Because of this, Stellar smart-wallet support should expose:

- `signAuthEntry`
- optionally `prepareAuthorization`
- optionally `validateAuthorization`

The current OWS `signAndSend` shape should be marked unsupported for Stellar contract accounts in the MVP.

## Wallet Data Model Changes

## New ChainType

The OWS implementation must add:

- `ChainType::Stellar`

## Known Chains

The implementation must add:

- Stellar testnet
- Stellar public network

## Account Descriptors

Wallet descriptors should include Stellar accounts the same way other chains do:

- chain id
- address
- derivation path

For MVP:

- store G-address derived from the Ed25519 public key

Contract accounts are not derivable from the seed phrase alone and should not be auto-created at wallet-creation time. They should instead be stored as metadata or linked accounts once deployed.

### Linked Smart Account Metadata

Add optional wallet metadata for deployed smart accounts:

```json
{
  "stellar": {
    "linkedSmartAccounts": [
      {
        "network": "stellar:testnet",
        "contractAddress": "C...",
        "deploymentType": "oz-smart-account",
        "signerRefs": [
          {
            "derivationPath": "m/44'/148'/0",
            "verifierType": "ed25519"
          }
        ]
      }
    ]
  }
}
```

This metadata is implementation-specific and not required for the first cut if it slows execution.

## OpenZeppelin Smart Account Integration

## MVP Contract Shape

The target contract shape is:

- OpenZeppelin smart account
- one default context rule
- one or more `External` Ed25519 signers
- optional simple threshold policy
- optional spending-limit policy

### Recommended MVP Profiles

#### Profile 1: Single-Signer Smart Wallet

- one Ed25519 external signer
- no threshold policy
- optional spending limit

#### Profile 2: 2-of-3 Treasury Smart Wallet

- three Ed25519 external signers
- simple threshold policy set to 2
- optional spending-limit policy

#### Profile 3: Agent Session Wallet

- base signer set controlled by operators
- short-lived context rule
- restricted allowed target contracts
- optional per-period spend cap

## Signer Mapping

Each OWS-managed Stellar signing key maps to an OpenZeppelin signer entry:

- signer kind: `External`
- verifier contract: deployed Ed25519 verifier
- key data: raw Ed25519 public key bytes

OWS is off-chain and does not know policy semantics beyond local pre-sign checks.

OZ remains the source of truth for:

- threshold logic
- context restrictions
- policy enforcement

## Relayer / Fee Payer Interface

## Submission Contract

The fee payer interface is out of scope as a public standard, but the implementation must support the following wallet-side contract:

### Request Payload

- target network
- serialized invocation XDR
- signed auth entries
- declared smart account address
- optional simulation snapshot
- expiration parameters

### Fee Payer Behavior

- parse and validate payload
- ensure fee payer is not accidentally being authorized as user
- rebuild or assemble transaction with fee payer as source
- simulate in enforcing mode
- reject if auth fails or simulation fails
- sign transaction envelope
- submit to RPC

## Security Model

## Threat Model Assumptions

Assume:

- the local machine is trusted enough to hold encrypted keys
- the agent or calling tool is less trusted than the OWS signer boundary
- the fee payer is a semi-trusted infrastructure component
- on-chain contracts are the final source of authorization truth

## Security Goals

- raw private keys never leave the OWS signing boundary
- auth-entry payloads are explicit and inspectable
- local policy checks can refuse suspicious signing requests before any key is used
- fee payer cannot arbitrarily spend on behalf of the user without valid auth-entry signatures
- smart-account contracts remain replay-safe under Soroban host semantics

## Local Policy Opportunities

Local OWS policies should support Stellar-specific checks over time:

- allowed contract address allowlist
- allowed function allowlist
- max token spend per asset
- max number of auth entries
- signature expiration bounds
- network allowlist

These are advisory or pre-sign controls only. The smart account still enforces the authoritative policy on-chain.

## MVP Scope

## Included

- Stellar chain family in OWS implementation
- G-address derivation from Ed25519 keys
- `signAuthEntry` support
- OpenZeppelin Ed25519-based smart account integration
- testnet-first implementation
- relayed submission flow for contract accounts
- example single-signer and 2-of-3 wallet profiles

## Excluded

- WebAuthn / passkey support
- secp256r1 support in OWS
- hardware signer support
- direct `signAndSend` for contract accounts
- broad UI/UX standardization
- multi-network automatic discovery beyond testnet and pubnet

## Deferred Features

- passkey support via secp256r1
- OWS policy simulation aware of Soroban auth context
- contract deployment helpers
- contract-account metadata sync and discovery
- classic+contract hybrid wallet orchestration
- fee abstraction integration with OZ fee-forwarder modules

## Developer Experience

## Expected Wallet Flow for Smart Accounts

1. Create OWS wallet.
2. Derive Stellar Ed25519 signer(s).
3. Deploy OZ verifier contract(s) if needed.
4. Deploy OZ smart account configured with derived public keys.
5. Build Soroban invocation.
6. Simulate in recording mode.
7. Extract auth-entry preimages.
8. Call OWS `signAuthEntry`.
9. Attach signatures.
10. Simulate in enforcing mode.
11. Send to fee payer.
12. Fee payer signs envelope and submits.

## Example Pseudocode

```ts
const authPreimages = await stellarAdapter.prepareAuthEntries(invocation);

const signatures = await Promise.all(
  authPreimages.map((preimage) =>
    ows.signAuthEntry({
      walletId,
      chainId: "stellar:testnet",
      preimageXdr: preimage.xdr,
      signerIndex: preimage.signerIndex,
    }),
  ),
);

const signedInvocation = await stellarAdapter.attachAuthSignatures(
  invocation,
  authPreimages,
  signatures,
);

await stellarAdapter.validateEnforcingMode(signedInvocation);

await feePayer.submit(signedInvocation);
```

## Implementation Strategy

## Why Not Reuse `signTransaction`

`signTransaction` in OWS assumes:

- the signable object is the transaction payload
- the same signer concept works across chain families
- `signAndSend` can optionally own submission

Those assumptions break for Stellar smart wallets:

- the signable object is the auth-entry preimage
- the transaction source and authorizer are decoupled
- submission is normally delegated to a fee payer

Therefore the implementation should extend OWS with a first-class Stellar authorization operation rather than overloading existing transaction signing semantics.

## Compatibility Notes

## Spec Compatibility

This design is compatible with OWS's stated philosophy:

- chain-specific signing behavior is allowed
- local encrypted storage remains unchanged conceptually
- policy engine remains pre-sign
- account descriptors still fit CAIP-style identifiers

## Implementation Compatibility

This design is not backward-compatible with the current Rust implementation without source changes.

Expected code areas affected:

- OWS core chain enum and known chains
- OWS signer curve and signer registry
- HD derivation logic for Stellar
- wallet account derivation loops
- library operations for Stellar auth-entry signing
- CLI and SDK bindings

## Open Questions

- What exact CAIP-2 identifier should Stellar use in OWS if chain-agnostic namespace guidance changes?
- Should Stellar classic and Soroban smart-wallet support ship together or separately?
- Should `signAuthEntry` take raw preimage XDR, raw hash bytes, or both?
- Should contract-account metadata live in the wallet descriptor or a separate local mapping?
- How much of the fee payer flow belongs in OWS versus a separate relayer package?
- Should local OWS policies understand Soroban function names and assets at MVP time, or only raw payload metadata?

## Acceptance Criteria

The implementation satisfies this spec when:

- a wallet can derive a Stellar Ed25519 key locally from OWS
- the implementation can expose a Stellar testnet G-address
- a client can call `signAuthEntry` and receive a valid Ed25519 signature
- an OpenZeppelin smart account configured with the matching public key accepts the signature in `__check_auth`
- a relayed testnet transaction can be simulated, signed, fee-paid, and submitted successfully
- a 2-of-3 Ed25519 OZ smart account can authorize a transaction using OWS-managed keys
- no raw key material is exposed outside the local signer boundary

## References

- `docs/research/ows-open-wallet-standard.json`
- `docs/research/stellar-smart-wallet-auth-entry.json`
- https://github.com/open-wallet-standard/core
- https://docs.openwallet.sh/
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/packages/accounts
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/examples/multisig-smart-account
- https://docs.openzeppelin.com/stellar-contracts/accounts/smart-account
- https://developers.stellar.org/docs/build/guides/transactions/signing-soroban-invocations
- https://developers.stellar.org/docs/build/guides/freighter/sign-auth-entries
- https://developers.stellar.org/docs/build/guides/contract-accounts
- https://developers.stellar.org/docs/build/guides/contract-accounts/smart-wallets
