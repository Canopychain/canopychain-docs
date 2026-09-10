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

## Operational note: the registry and the vault don't check each other

Worth internalizing before deploying either contract for real:
`milestone-vault`'s `deposit` doesn't check `project-registry` at all —
it will happily open a vault for any project id, approved or not. This is
a deliberate contract-level choice (it keeps the vault simpler and avoids
a cross-contract call on every deposit), but it pushes the responsibility
onto whoever operates the frontend and backend:

- The frontend's project explorer only ever lists projects the backend
  has marked `approved` — it doesn't and shouldn't offer a way to fund an
  unreviewed project id directly.
- If a deposit ever happens directly (bypassing the frontend) against a
  project id that was never registered or approved, the indexer still
  records it — with a placeholder project entry — rather than dropping
  the data. That's intentional: the backend never discards real on-chain
  activity just because it doesn't fit the expected shape.

In short: approval is a curation layer the *apps* enforce, not something
the vault contract itself guarantees.
