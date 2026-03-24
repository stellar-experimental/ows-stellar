# OWS for Stellar Smart Wallets with OpenZeppelin Smart Accounts

## Status

Draft

## Last Updated

2026-03-24

## Summary

This document specifies a feature addition to the `open-wallet-standard/core` repository: Stellar wallet support in the existing OWS universal-wallet model, with Soroban smart-account signing support for OpenZeppelin smart accounts.

The core conclusion from the research is:

- OWS is a good fit for the local key vault, policy-gated signing boundary, and multi-tool wallet access model.
- OWS is not yet a direct fit for Stellar smart wallets because Stellar contract accounts do not sign transaction envelopes. They authorize via Soroban auth entries, and the wallet-facing account is an OZ smart-account contract rather than a raw keypair.
- The correct implementation in this repo is not a new wallet-provider plugin. It is one more chain backend in the current OWS model, plus a Stellar-specific smart-account signing operation.
- The correct integration is an OWS-managed Ed25519 signer under an OpenZeppelin smart account, with the `C...` smart-account address treated as the primary operational wallet identity for the Stellar smart-wallet profile.
- The cleanest send path is OZ Channels / Relayer, not a bespoke fee-payer protocol, because it already accepts Soroban function + auth payloads and pays resource fees through managed channel accounts.

This spec defines the MVP architecture, required OWS extensions, wallet behavior, threat model assumptions, API surface, implementation boundaries, and acceptance criteria.

## Alignment with Official OWS Documents

OWS's official documentation is split into:

- normative core documents for storage, signing, policy, lifecycle, supported chains, and conformance
- optional profile documents for agent access and key isolation
- non-normative reference implementation docs for CLI and SDK behavior

This spec is written against that structure.

Implications for Stellar support:

- chain behavior, wallet artifacts, signing semantics, and conformance claims belong in the normative OWS layer
- access-layer transport choices, CLI ergonomics, and SDK naming are implementation details
- relayer UX is outside the core OWS standard unless the OWS project explicitly chooses to expand scope, but the existing OWS `signAndSend` transport hook gives a natural place to surface a Channels-backed send path

## Repo Scope Clarification

This document is intentionally scoped to how `open-wallet-standard/core` works today.

That repo is currently:

- one local vault
- one universal wallet model
- one closed set of chain families wired into Rust
- one shared CLI and SDK surface over the same signing core

It is not currently organized around pluggable wallet providers.

So this proposal should be read as:

- add Stellar as another supported chain family in the existing model
- add the minimum smart-account-specific signing surface needed for OpenZeppelin-based Stellar wallets
- avoid unrelated cleanup or redesign unless a touched file must change for the feature to land

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
- a separate G-account or relayer transport must pay fees and submit the transaction
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

The simplest useful OZ wallet shape is therefore:

- one `ContextRuleType::Default` rule
- one `Signer::External(ed25519_verifier, pubkey)`
- no policy at all, or a later-added threshold/spend policy

That makes the `C...` smart account the user-facing wallet while OWS only needs to manage the Ed25519 signer key and produce valid auth-entry signatures.

## Problem Statement

We need a specification for using OWS as the wallet substrate for a Stellar smart wallet that uses OpenZeppelin smart accounts.

The design must answer:

- how Stellar keys are stored and derived in OWS
- how Stellar auth-entry signing fits into OWS
- how an OWS-managed signer maps into OpenZeppelin smart account signers
- how a `C...` smart-account address is represented and surfaced in OWS
- how fee payment and submission work when the user wallet is a contract account
- what the MVP should support first
- which features are explicitly deferred

## Goals

- Add Stellar as a first-class chain family in the current OWS implementation model.
- Support local-first Stellar signing with private keys never exposed upstream.
- Support OpenZeppelin smart accounts using OWS-managed Ed25519 signers in the MVP.
- Treat the OZ `C...` smart-account address as a first-class operational wallet identity in the Stellar smart-wallet profile.
- Support Soroban auth-entry signing as the main smart-wallet authorization flow.
- Preserve OWS's local vault and policy-gated access model.
- Fit Stellar into the existing universal-wallet, CLI, and SDK surfaces already used by other chains.
- Surface a Channels-backed submission path as the preferred way to send Stellar smart-wallet transactions once signing works.

## Non-Goals

- Turning OWS into a generic wallet-provider framework.
- Broad cleanup of unrelated OWS implementation drift or inconsistencies.
- Replacing Stellar wallet UX broadly across classic and Soroban flows in one release.
- Supporting every OpenZeppelin signer mode in the MVP.
- Supporting WebAuthn / secp256r1 in the MVP.
- Defining a new bespoke relayer protocol when OZ Channels already covers the primary send path.
- Making OWS's current raw-transaction `signAndSend` semantics work unchanged for Stellar contract accounts.
- Solving general cross-chain account abstraction across non-Stellar smart-wallet systems.

## Key Findings from Research

### Finding 0: Stellar support splits into two layers

There are really two different scopes:

- Stellar classic support as a new OWS chain profile
- Stellar smart-wallet support for Soroban contract accounts

The first scope mostly fits OWS's existing "add a new chain" model:

- canonical identifier
- derivation rules
- address encoding
- deterministic signing rules

The second scope does not fit as cleanly because Soroban contract accounts authorize through auth entries rather than ordinary transaction-envelope signing.

Implication:

- "add Stellar" should not be treated as synonymous with "fully support Stellar smart wallets"
- classic Stellar support can be framed as a chain-profile addition
- Soroban smart-wallet support likely requires a signing-interface extension or a clearly documented chain-specific signing profile

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
- the sender transport rebuilds or signs the final envelope and submits it

Implication:

- OWS needs a Stellar-specific authorization operation
- `signTransaction(txXdr)` is insufficient for smart-wallet usage

### Finding 3: OpenZeppelin smart accounts are more explicit and simpler than the earlier model assumed

OpenZeppelin smart accounts separate:

- who can sign
- what they can do
- how policy is enforced

Current OZ `main` also makes the client obligations explicit:

- `AuthPayload` must include `signers`
- `AuthPayload` must include one `context_rule_id` per auth context
- callers choose the rule to validate against
- `Signer::External(verifier, key_data)` is the natural fit for OWS-managed Ed25519 keys
- `ContextRuleType::Default` can authorize every context

Implication:

- the minimum viable OZ smart wallet is much simpler than a generalized policy-heavy account
- a single Ed25519 signer plus a default rule already gives a usable smart wallet
- the OZ `C...` account can be the first-class wallet identity at the product and SDK layer
- OWS core still only needs to manage keys and signatures, not policy semantics or rule discovery

### Finding 4: The `C...` account can be first-class even if OWS still stores derived signer accounts

OWS wallet files currently describe `accounts[]` as derived accounts, and Stellar smart-account contract addresses are not seed-derivable.

Implication:

- the derived `G...` signer still belongs in `accounts[]` for storage compatibility
- the linked `C...` smart account should be treated as the primary operational account in the Stellar smart-wallet profile
- the right place to express that in OWS is metadata and wallet-surface behavior, not a forced redefinition of how every OWS account entry works

### Finding 5: Ed25519 is the right MVP

This still matches OWS conceptually:

- OWS manages key custody and local signing
- the smart account manages on-chain authorization policy

Implication:

- OWS should be the off-chain signer vault
- OpenZeppelin contracts should remain the on-chain policy and account abstraction layer
- MVP signer support should be limited to the curves OWS already has today

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

### Finding 6: Soroban host owns replay prevention and invocation-tree semantics

Per Stellar's authorization model:

- signed auth entries are bound to an authorized invocation tree, not just a flat message
- the Soroban host validates signature expiration and nonce uniqueness
- the Soroban host computes the final signature payload hash from the authorization preimage
- `__check_auth` receives the final payload hash plus the auth contexts being authorized

Implication:

- OWS should sign a serialized authorization preimage or host-compatible payload, not invent its own higher-level Stellar auth format
- OWS should not own nonce management, expiration policy, or invocation-tree construction in core
- client or adapter code must preserve invocation structure exactly and still perform enforcing-mode simulation

### Finding 7: OpenZeppelin `main` has stricter client obligations than some docs pages suggest

The linked OpenZeppelin repository target is `main`, and the source there currently assumes:

- `AuthPayload` includes both `signers` and `context_rule_ids`
- the caller must provide exactly one context-rule id per auth context
- no rule auto-discovery or newest-first iteration occurs
- policy enforcement happens through `enforce()` in the current source and README, even though some published docs pages still describe an older `can_enforce()` pre-check flow

Implication:

- the source and package README are the authoritative assumptions for this integration
- any OWS + OZ adapter must explicitly build `AuthPayload` and select rule ids outside OWS core
- local docs should not rely on older OZ docs behavior when targeting `main`

### Finding 8: OZ Channels should be the default send story

The official OZ Stellar Channels service already provides:

- Soroban function + auth submission
- managed fee payment
- channel-account parallelism
- a pre-signed-XDR fallback mode

Implication:

- the architecture should not center on a custom fee-payer service
- the preferred transport story is OWS signing plus OZ Channels submission
- the existing OWS `signAndSend` transport concept can eventually surface this cleanly without pretending the contract account signs envelopes

## Proposed Architecture

## High-Level Design

The system has four distinct layers:

1. Local vault and signer layer
2. Stellar smart-wallet profile layer inside OWS
3. OpenZeppelin smart account layer
4. Channels / relayer transport layer

### 1. Local Vault and Signer Layer

OWS remains responsible for:

- encrypted wallet storage
- chain account derivation
- API key scoped access
- local policy-gated signing
- preventing raw key material from leaving the signer boundary

### 2. Stellar Smart-Wallet Profile Layer

A new Stellar-specific profile extends OWS with:

- Stellar account derivation
- G-address generation
- auth-entry preimage signing
- linked OZ smart-account metadata
- smart-account-aware wallet display and selection
- optional Channels-backed send support through the existing OWS send surface

This is the core new component required by this spec.

### 3. OpenZeppelin Smart Account Layer

The on-chain smart wallet is an OpenZeppelin smart account contract configured with:

- one or more external Ed25519 signers
- context rules
- optional threshold policy
- optional spending-limit policy

The contract account address is the user wallet for Soroban contract authorization.

### 4. Channels / Relayer Transport Layer

The preferred submission transport is OZ Channels:

- receives Soroban function + auth entries or a fully signed XDR
- builds the final transaction through a managed pool of channel accounts
- pays resource fees
- submits to Stellar
- returns transaction status and hash

Self-hosted relayers remain possible, but they are not the default story this spec should optimize around.

## System Boundaries

### OWS Responsibilities

- derive and store Stellar keys locally
- expose a Stellar signing API
- sign auth-entry preimages
- optionally sign classic Stellar transactions for G-address flows
- surface linked smart-account identities and the primary smart account for the wallet
- enforce local access control before signing

### Smart Account Responsibilities

- decide who is authorized
- decide what contexts are allowed
- enforce thresholds and spending limits
- validate submitted signatures in `__check_auth`

### Channels / Relayer Responsibilities

- pay XLM fees
- manage sequence number for the submission account
- build or accept the final transaction envelope
- perform submission-side validation and simulation
- submit the final transaction envelope

## Storage and Artifact Compatibility

The Stellar design should fit into OWS's existing vault model rather than invent a chain-specific layout.

That means reusing:

- wallet files in `~/.ows/wallets/`
- API key files in `~/.ows/keys/`
- policy files in `~/.ows/policies/`
- append-only audit records in `~/.ows/logs/audit.jsonl`

Important implications from the official OWS storage spec:

- wallet files currently use `ows_version = 2`
- wallet secrets are encrypted with AES-256-GCM
- API key files contain token-scoped encrypted copies of wallet secrets re-encrypted under HKDF-derived token keys
- policy files are not secret and can remain more broadly readable than wallet and key files

Artifact invariants that Stellar support should preserve:

- no schema bump unless strictly necessary
- no redefinition of existing required fields
- preservation of unknown metadata fields during non-destructive updates
- rejection of unknown required schema fields and unsupported schema versions

Stellar support should plug into those existing artifacts without changing the vault topology.

## Lifecycle Compatibility

Stellar support should preserve the existing OWS wallet lifecycle semantics:

- wallet create should auto-derive Stellar alongside the current auto-derived family set
- mnemonic import should deterministically reproduce the same Stellar account
- wallet info and list operations should expose Stellar without changing their core shape
- export and re-import flows should preserve Stellar accounts under the existing artifact model
- delete and rotate flows should not require Stellar-specific lifecycle exceptions

Lifecycle conformance should be treated as a first-class acceptance target, not a side effect of derivation.

## Access Model and Policy Semantics

OWS defines two access tiers:

- owner mode via passphrase
- agent mode via `ows_key_...` token

That distinction matters for Stellar:

- owner mode bypasses policy evaluation
- agent mode evaluates all attached policies before any token-backed secret is decrypted
- policies attach to API keys, not directly to wallets
- attached policies are AND-combined and fail closed
- unknown declarative rules deny
- executable-policy failures deny
- no Stellar-specific bypass flags should exist for token-backed signing

This complements OpenZeppelin smart-account policy enforcement:

- OWS policy is off-chain and pre-decrypt
- OpenZeppelin policy is on-chain and authorization-final

The two layers should remain distinct.

Stellar-aware policy support should extend the existing `PolicyContext` model rather than introduce a parallel policy model.

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

### Representation in OWS

Inside `open-wallet-standard/core`, the primary wallet representation should stay aligned with the current storage model:

- the OWS wallet stores the derived Stellar signer account in `accounts[]`
- the signer account is a classic `G...` account derived from the mnemonic or imported Ed25519 key
- an OpenZeppelin smart contract account `C...` is linked to that signer through metadata

Inside the Stellar smart-wallet profile, however, the primary operational account should be the linked `C...` address:

- CLI and SDK surfaces should be able to show a primary smart account for Stellar wallets
- signing requests should target the linked smart account context, even though the actual cryptographic signer is the derived Ed25519 key
- wallet UX should not force users to think of the `G...` signer as the wallet they spend from

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

Canonical identifiers must be used in:

- wallet files
- policy contexts
- audit logs
- API parameters

Any friendly aliases for Stellar should be treated as CLI-only conveniences and must be resolved to canonical identifiers before further processing.

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
  preimageXdr: string; // base64 XDR in the MVP
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
- sign the serialized `HashIdPreimageSorobanAuthorization` payload according to Stellar expectations
- return signature bytes and optional public key metadata

Non-behavior:

- OWS does not choose nonce values
- OWS does not choose signature-expiration ledgers
- OWS does not build or normalize the authorized invocation tree
- OWS does not construct OpenZeppelin `AuthPayload` in core

## Optional Classic Operation

The following operation is optional but recommended:

```ts
interface SignStellarTransactionRequest {
  walletId: WalletId;
  chainId: ChainId;
  transactionXdr: string;
}
```

This should be treated as either:

- a formal OWS signing-interface extension
- or a clearly documented chain-specific signing profile

It should not be disguised as ordinary `signTransaction` behavior.

This is for G-account flows only and is separate from contract-account auth-entry signing.

The first Stellar smart-account PR does not need this operation if it slows review. The core smart-wallet requirement is auth-entry signing.

## No Direct Raw-RPC `signAndSend` for Contract Accounts

For Stellar contract accounts, OWS must not pretend that raw RPC envelope submission is the right default.

Instead, the user-facing flow is:

1. build invocation
2. simulate in recording mode
3. extract auth-entry preimages
4. sign auth entries with OWS
5. re-simulate in enforcing mode
6. send via Channels or another relayer transport
7. transport builds, fee-pays, and submits

Notes:

- in auth-entry signing flows, the transaction source account spends the sequence number
- in the common sponsored flow, that means the channel or fee-payer G-account spends sequence and pays fees
- fee-bump transactions are a possible additional layer, but they are outside the MVP

Because of this, Stellar smart-wallet support should expose:

- `signAuthEntry`
- optionally a smart-account-aware `signAndSend` transport backed by Channels once the signing layer is stable

The current raw-transaction `signAndSend` shape should be marked unsupported for Stellar contract accounts in the MVP unless a relayer-backed transport is implemented.

Likewise, under the Stellar smart-wallet MVP profile:

- `signTypedData` is unsupported
- optional classic transaction-envelope signing is separate from contract-account auth signing

Unsupported operations must fail clearly and must not silently degrade into weaker or ambiguous behavior.

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
- keep the stored account shape aligned with the current `account_id`, `address`, `chain_id`, `derivation_path` pattern used by other families

Contract accounts are not derivable from the seed phrase alone and should not be auto-created at wallet-creation time. They should instead be stored as metadata or linked accounts once deployed.

### Linked Smart Account Metadata

Add optional wallet metadata for deployed smart accounts:

```json
{
  "stellar": {
    "primarySmartAccount": "stellar:testnet:C...",
    "linkedSmartAccounts": [
      {
        "accountId": "stellar:testnet:C...",
        "chainId": "stellar:testnet",
        "contractAddress": "C...",
        "kind": "oz-smart-account",
        "primary": true,
        "channels": {
          "transport": "openzeppelin-channels",
          "baseUrl": "https://channels.openzeppelin.com/testnet"
        },
        "signerRefs": [
          {
            "derivationPath": "m/44'/148'/0",
            "verifierType": "ed25519",
            "contextRuleType": "default"
          }
        ]
      }
    ]
  }
}
```

This metadata is implementation-specific, but it is the cleanest way to make the `C...` account a first-class citizen without breaking the current OWS storage model.

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

OpenZeppelin-specific client constraints:

- build `AuthPayload.signers` using the signer objects expected by the smart account
- provide exactly one `context_rule_id` per auth context
- prefer `Signer::External` for the MVP
- avoid `Signer::Delegated` in the MVP because delegated signers require manual auth-entry crafting and are not returned by normal simulation flows
- respect verifier-specific key-data rules and deduplication semantics

## Channels / Relayer Integration

## Submission Contract

The relayer interface is out of scope as a public standard, but the implementation should align with the wallet-side contract already exposed by OZ Channels.

### Request Payload

- target network
- Soroban function XDR
- signed auth entries
- declared smart account address
- optional simulation snapshot
- expiration parameters

### Channels / Relayer Behavior

- parse and validate payload
- ensure relayer account is not accidentally being authorized as user
- rebuild or assemble transaction with relayer source
- simulate in enforcing mode
- reject if auth fails or simulation fails
- sign transaction envelope
- submit to RPC

The transport must also treat the incoming transaction as untrusted input and verify that:

- the operation source is not silently shifted onto the relayer
- auth entries do not authorize the relayer's own account
- the rebuilt invocation tree matches what the client intended to authorize

## Security Model

## Threat Model Assumptions

Assume:

- the local machine is trusted enough to hold encrypted keys
- the agent or calling tool is less trusted than the OWS signer boundary
- the relayer transport is a semi-trusted infrastructure component
- on-chain contracts are the final source of authorization truth

## Security Goals

- raw private keys never leave the OWS signing boundary
- auth-entry payloads are explicit and inspectable
- local policy checks can refuse suspicious signing requests before any key is used
- relayer transport cannot arbitrarily spend on behalf of the user without valid auth-entry signatures
- smart-account contracts remain replay-safe under Soroban host semantics

## Key Isolation Alignment

The official OWS guidance describes the current implementation as:

- in-process
- memory-hardened where possible
- zeroizing key material immediately after use
- capable of short-lived key caching

It also describes a future subprocess-enclave model as an optional stronger isolation profile.

Implications for Stellar support:

- the first Stellar implementation should preserve the current in-process hardening model
- auth-entry signing must follow the same decrypt, derive, sign, and zeroize lifecycle as other chains
- a future subprocess model can be considered later if Stellar smart-wallet flows justify stronger isolation

## Security Invariants

The Stellar implementation should preserve the official OWS security invariants:

- verify policy-before-decrypt for token-backed requests
- fail closed on unknown declarative policy rules
- fail closed on executable-policy errors or malformed results
- clear `OWS_PASSPHRASE` or equivalent environment-carried credentials promptly after reading where relevant
- zeroize mnemonic, private-key, derived-key, and KDF output buffers after use
- avoid logging secrets in audit, debug, or telemetry output
- refuse to operate when secret-bearing directories have unsafe permissions

## Local Policy Opportunities

Local OWS policies should support Stellar-specific checks over time:

- allowed contract address allowlist
- allowed function allowlist
- max token spend per asset
- max number of auth entries
- signature expiration bounds
- network allowlist

These are advisory or pre-sign controls only. The smart account still enforces the authoritative policy on-chain.

## Audit Logging

OWS treats audit logs as append-only and excludes secrets from them.

Stellar support should add explicit audit events for:

- `sign_auth_entry`
- optional `sign_stellar_transaction`
- optional `submit_via_channels`

Recommended audit fields for Stellar operations:

- wallet id
- chain id
- API key id when applicable
- operation type
- allow or deny result
- target contract address when available
- timestamp

## MVP Scope

## Included

- Stellar chain family in OWS implementation
- G-address derivation from Ed25519 keys
- `signAuthEntry` support
- OpenZeppelin Ed25519-based smart account integration
- testnet-first implementation
- linked `C...` smart-account metadata and primary-account surfacing
- example single-signer and 2-of-3 wallet profiles

## Excluded

- WebAuthn / passkey support
- secp256r1 support in OWS
- hardware signer support
- broad UI/UX standardization
- multi-network automatic discovery beyond testnet and pubnet
- delegated-signer orchestration

## Deferred Features

- passkey support via secp256r1
- OWS policy simulation aware of Soroban auth context
- contract deployment helpers
- contract-account metadata sync and discovery
- classic+contract hybrid wallet orchestration
- full Channels-backed `signAndSend` transport
- delegated-signer support for OZ smart accounts

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
11. Send `func` + `auth` to Channels.
12. Channels builds, fee-pays, and submits.

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

await channels.submitSorobanTransaction({
  func: signedInvocation.funcXdr,
  auth: signedInvocation.authXdrs,
});
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
- submission is normally delegated to a relayer transport

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

## Conformance Position

OWS's conformance model is profile-based. The resulting implementation should not claim generic compliance if it only supports part of the system.

The right eventual claim is something like:

- `OWS Storage + Signing + Policy + Lifecycle + Stellar Chain Profile`

The implementation should also ship machine-readable vectors for:

- Stellar wallet derivation
- Stellar wallet and API key artifact handling
- Stellar auth-entry signing
- allow and deny policy fixtures for Stellar requests
- policy evaluation around Stellar requests
- negative error cases

## Open Questions

- What exact CAIP-2 identifier should Stellar use in OWS if chain-agnostic namespace guidance changes?
- Should Stellar classic and Soroban smart-wallet support ship together or separately?
- Should `signAuthEntry` take raw preimage XDR, raw hash bytes, or both?
- Should contract-account metadata live in the wallet descriptor or a separate local mapping?
- How much of the Channels-backed send flow belongs in OWS versus a separate transport package?
- Should local OWS policies understand Soroban function names and assets at MVP time, or only raw payload metadata?

## Acceptance Criteria

The implementation satisfies this spec when:

- a wallet can derive a Stellar Ed25519 key locally from OWS
- the implementation can expose a Stellar testnet G-address
- a client can call `signAuthEntry` and receive a valid Ed25519 signature
- an OpenZeppelin smart account configured with the matching public key accepts the signature in `__check_auth`
- a linked `C...` smart account can be surfaced as the primary Stellar wallet identity
- a Channels-backed testnet transaction can be simulated, signed, fee-paid, and submitted successfully
- a 2-of-3 Ed25519 OZ smart account can authorize a transaction using OWS-managed keys
- no raw key material is exposed outside the local signer boundary

## References

- `docs/research/ows-open-wallet-standard.json`
- `docs/research/stellar-smart-wallet-auth-entry.json`
- https://github.com/open-wallet-standard/core
- https://github.com/open-wallet-standard/core/blob/main/docs/00-specification.md
- https://github.com/open-wallet-standard/core/blob/main/docs/01-storage-format.md
- https://github.com/open-wallet-standard/core/blob/main/docs/03-policy-engine.md
- https://github.com/open-wallet-standard/core/blob/main/docs/04-agent-access-layer.md
- https://github.com/open-wallet-standard/core/blob/main/docs/05-key-isolation.md
- https://github.com/open-wallet-standard/core/blob/main/docs/08-conformance-and-security.md
- https://github.com/open-wallet-standard/core/blob/main/docs/sdk-cli.md
- https://github.com/open-wallet-standard/core/blob/main/docs/sdk-node.md
- https://docs.openwallet.sh/
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/packages/accounts
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/examples/multisig-smart-account
- https://docs.openzeppelin.com/stellar-contracts/accounts/smart-account
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/README.md
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/mod.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/storage.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/mod.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/ed25519.rs
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/webauthn.rs
- https://github.com/OpenZeppelin/stellar-contracts/tree/main/examples/multisig-smart-account/ed25519-verifier
- https://github.com/OpenZeppelin/stellar-contracts/blob/main/examples/multisig-smart-account/ed25519-verifier/src/contract.rs
- https://docs.openzeppelin.com/relayer/1.4.x/guides/stellar-channels-guide
- https://developers.stellar.org/docs/build/guides/transactions/signing-soroban-invocations
- https://developers.stellar.org/docs/build/guides/freighter/sign-auth-entries
- https://developers.stellar.org/docs/build/guides/contract-accounts
- https://developers.stellar.org/docs/build/guides/contract-accounts/smart-wallets
- https://developers.stellar.org/docs/learn/fundamentals/contract-development/authorization
