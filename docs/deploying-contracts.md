---
sidebar_position: 6
---

# Deploying the contracts

## Prerequisites

- Rust with the `wasm32-unknown-unknown` target: `rustup target add wasm32-unknown-unknown`
- The [Stellar CLI](https://developers.stellar.org/docs/tools/cli): `winget install --id Stellar.StellarCLI` (or `cargo install --locked stellar-cli`)
- A funded identity on whichever network you're deploying to

## Testnet

Create and fund a testnet identity if you don't have one:

```bash
stellar keys generate canopychain-deployer --network testnet --fund
```

From the root of [canopychain-contracts](https://github.com/canopychain/canopychain-contracts):

```bash
STELLAR_SOURCE_ACCOUNT=canopychain-deployer ./scripts/deploy-testnet.sh
```

This builds both contracts, deploys them, calls `init` on each with the
deploying identity as admin, and writes a `deployments.json` at the repo
root:

```json
{
  "network": "testnet",
  "deployed_at": "...",
  "admin": "canopychain-deployer",
  "contracts": {
    "project-registry": "C...",
    "milestone-vault": "C..."
  }
}
```

Copy the two contract IDs from there into:

- `canopychain-backend`'s `.env` — `PROJECT_REGISTRY_CONTRACT_ID` and
  `MILESTONE_VAULT_CONTRACT_ID` (the indexer no-ops until both are set;
  the attestation submitter needs `MILESTONE_VAULT_CONTRACT_ID`
  specifically).
- `canopychain-frontend`'s `.env` —
  `NEXT_PUBLIC_PROJECT_REGISTRY_CONTRACT_ID` and
  `NEXT_PUBLIC_MILESTONE_VAULT_CONTRACT_ID`.

Each milestone-vault also needs an attestor identity — see
[The Milestone Attestation Model](./attestation-model.md) — funded and
set as `ATTESTOR_SECRET_KEY` in the backend's `.env`. It's a separate key
from the deploying/admin identity; keep it that way.

## Mainnet

There's deliberately no `deploy-mainnet.sh` script — a mainnet deploy is a
higher-stakes, one-time operation worth doing by hand rather than
scripting. The steps are the same shape as testnet, with `--network
mainnet` (or `--network public`, depending on your CLI's configured
network aliases) and a real funded account instead of a friendbot-funded
test identity. Review the [contracts reference](./contracts.md) for what
`init`, `pause`, `set_attestor`, and `cancel_project` actually do before
running any of them against real funds.

## Operational note: the vault reads the registry when a project's vault opens

`milestone-vault` is initialized with `project-registry`'s contract ID
(`scripts/deploy-testnet.sh` deploys the registry first and passes its ID
to the vault's `init`). A project's first deposit opens its vault only if
the registry has that project and has approved it, and the vault copies
the project's recipient and attestor from the registry at that moment.
Later deposits don't consult the registry again.

So approval is enforced on-chain, not only by the apps: a deposit made
directly against the contract, bypassing the frontend, still can't fund an
unregistered or unapproved project, or choose its recipient or attestor.
Rotating a vault's attestor afterwards is the admin's `set_attestor`.
