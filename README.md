# OWS Stellar

Workspace for upstreaming Stellar support into OWS.

This repository is not a standalone wallet SDK or app. It is the coordination repo for the Stellar workstream around `open-wallet-standard/core`, with:

- design and planning docs at the top level
- a pinned `ows-core/` submodule where the actual implementation work happens

## What This Repo Is For

The goal is to make OWS a usable wallet substrate for Stellar, especially for OpenZeppelin-based Soroban smart wallets, without turning OWS into a generic provider framework or a broad cleanup project.

In practical terms, the work is split into two layers:

- this repo defines the scope, architecture, upstream PR plan, and review framing
- `ows-core/` contains the forked OWS codebase where Rust, CLI, and SDK changes are implemented and tested

## Relationship Between This Repo And `ows-core`

`ows-core/` is a git submodule pointing at the `stellar-experimental/ows-core` fork. That fork also keeps an `upstream` remote pointing at `open-wallet-standard/core`.

The submodule is configured to follow the `feat/stellar-smart-wallet` branch when using `git submodule update --remote`, while the parent repo still pins an exact submodule commit for reproducibility.

The intended flow is:

1. use the top-level docs in this repo to keep scope tight and explain the why
2. implement the actual feature work inside `ows-core/`
3. cut focused upstream PRs from the fork into `open-wallet-standard/core`

This means the top-level repo is the project wrapper, and the submodule is the code that will eventually be upstreamed.

## Project Goal

The current project goal is:

- add Stellar as a first-class OWS chain family
- preserve OWS's local vault and policy-gated signing model
- support Stellar smart-wallet flows based on Soroban auth-entry signing
- make OpenZeppelin smart-account `C...` identities first-class at the wallet surface while keeping derived `G...` signer accounts as the custody anchor underneath
- upstream the result in small, reviewable PRs

The docs in this repo consistently frame the work in phases:

1. Stellar chain support and wallet profile
2. Stellar auth-entry signing surface
3. OpenZeppelin smart-account integration
4. Channels-backed send flow and hardening

## Current Status

At the top level, this repo currently contains the design and upstreaming documents for the Stellar effort.

Inside `ows-core/`, the checked-out implementation branch already contains active Stellar work, including:

- Stellar chain registration and wallet/account support
- Stellar signer implementation
- CLI and binding updates
- test vectors and fixtures for Stellar signing flows

The top-level repo exists to keep that implementation tied to a clear spec and an upstreaming plan instead of letting the effort drift into a one-off fork.

## Repo Layout

```text
.
├── README.md
├── docs/
│   ├── specs/
│   │   ├── ows-stellar-smart-wallet-spec.md
│   │   ├── ows-stellar-smart-wallet-plan.md
│   │   ├── ows-stellar-upstreaming-plan.md
│   │   ├── ows-stellar-implementation-surface-audit.md
│   │   ├── ows-stellar-pr1-checklist.md
│   │   └── ows-stellar-first-class-parity-plan.md
│   └── research/
└── ows-core/
    └── ... upstream-target implementation fork as a submodule
```

## Recommended Reading Order

If you are new to the project, start here:

1. [smart wallet spec](docs/specs/ows-stellar-smart-wallet-spec.md)
2. [implementation plan](docs/specs/ows-stellar-smart-wallet-plan.md)
3. [upstreaming plan](docs/specs/ows-stellar-upstreaming-plan.md)
4. [implementation surface audit](docs/specs/ows-stellar-implementation-surface-audit.md)
5. [PR1 checklist](docs/specs/ows-stellar-pr1-checklist.md)

Then review the implementation in [`ows-core/`](ows-core).

## Working In This Repo

Initialize the submodule:

```bash
git submodule update --init --recursive
```

Check the implementation repo status:

```bash
git -C ows-core status
git -C ows-core remote -v
git -C ows-core branch --show-current
```

If you want to move the submodule forward to the latest commit on the tracked feature branch:

```bash
git submodule update --remote ows-core
```

That updates the checked-out submodule commit in your working tree, after which the parent repo can record the new pinned SHA in a normal commit.

Run the main validation commands from inside the submodule:

```bash
cd ows-core/ows
cargo test --workspace

cd ../bindings/node
npm test

cd ../bindings/python
pytest
```

## What This Repo Is Not

- Not a replacement for `open-wallet-standard/core`
- Not a standalone distributable package
- Not the final upstream home of the code
- Not a generic Stellar wallet product repo

It is the staging ground for getting Stellar support into OWS cleanly.
