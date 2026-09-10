---
sidebar_position: 14
---

# Glossary

## Stellar and Soroban

**Stellar** — the blockchain network Canopychain runs on.

**Soroban** — Stellar's smart contract platform. Canopychain's contracts
are Soroban contracts, written in Rust.

**Lumens (XLM)** — Stellar's native asset. One of the token choices when
funding a project.

**Stellar Asset Contract (SAC)** — the standard contract interface every
asset on Stellar exposes to Soroban, native XLM included. It's why "every
token uses 7 decimal places" is true universally rather than per-asset.

**Testnet / Mainnet** — testnet is Stellar's free, reset-able test
network (funded via a public faucet, no real value); mainnet (sometimes
called the "public network") is the real one. Everything in this
documentation defaults to testnet.

**G-address / C-address** — Stellar addresses starting with `G` identify
accounts (wallets); addresses starting with `C` identify contracts. A
donor, operator, or attestor's identity is a G-address; `project-registry`
and `milestone-vault` are each a C-address.

**SEP** — a Stellar Ecosystem Proposal, the mechanism Stellar uses to
standardize cross-wallet, cross-app behavior. Canopychain relies on two:
[SEP-43](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0043.md)
(a standard wallet interface — what lets any SEP-43-compatible wallet plug
into the same connect/sign flow) and
[SEP-53](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0053.md)
(signing arbitrary messages, not just transactions — what the backend's
admin authentication is built on).

**Freighter** — the most common Stellar browser wallet extension.

**Stellar Wallets Kit** — the library the frontend uses to talk to
Freighter and other SEP-43-compatible wallets through one interface,
rather than integrating each wallet separately.

## Canopychain-specific

**Project** — a single reforestation effort registered on
`project-registry`: a name, a GPS-bounded polygon, an operator, a
recipient, an attestor, and an approval status.

**Vault** — a project's pooled donor escrow on `milestone-vault`. Holds
every donor's deposit for that project until milestones release it.

**Milestone** — one tranche in a project's release schedule: a
forest-cover-change threshold (`threshold_bps`) and the percentage of the
pool it releases when attested (`payout_bps`). Set once, via
`configure_milestones`, and immutable after.

**Tranche** — the portion of a project's pool released by a single
milestone attestation — a percentage of `total_deposited`, computed fresh
against the pool's current size, not a snapshot from when the schedule
was configured.

**Attestation / attest** — the on-chain act of confirming a milestone's
threshold has been crossed, performed by calling `attest_milestone`. See
[The Milestone Attestation Model](./attestation-model.md) for who can do
this and what it actually verifies.

**Attestor** — the address authorized to attest milestones for one
specific project. In practice, almost always the backend's own
configured signing key — see
[The Milestone Attestation Model](./attestation-model.md).

**Basis points (bps)** — hundredths of a percent; 100 bps = 1%, 10,000
bps = 100%. How every threshold and payout percentage in a milestone
schedule is expressed, to avoid floating-point rounding in on-chain math.

**Operator** — the person or organization that registers and manages a
project. Distinct from the project's recipient address, which is where
funds actually pay out.

**Recipient** — the address a project's released tranches pay out to. Set
once at registration; there's no operator-facing way to change it
afterward.

**Refund** — a donor's proportional claim on whatever remains in a
cancelled project's pool, computed as `donation / total_deposited` of the
unreleased remainder. Only available after an admin cancels a project.

**Escrow** — the general pattern `milestone-vault` implements: funds are
held by a contract rather than sent directly, and released according to
rules the contract enforces rather than by manual transfer.

**GFW (Global Forest Watch)** — the satellite-data API the backend
queries for each project's forest-cover status. See
[Forest-Cover Metric & Limitations](./forest-cover-metric.md) for exactly
what it does and doesn't measure here.

**Approved project** — a project `project-registry` has marked
`approved: true` via `approve_project`. Only approved projects appear in
the app's explorer; the vault contract itself doesn't care whether a
project is approved.

**Indexer** — the backend process that watches the contracts' events and
mirrors them into a queryable database. Never a source of truth, only a
read-optimized copy of it.

**"Admin" — which one?** Three different things share this name, and
they don't have to be the same address:

- `project-registry`'s admin, set via that contract's own `init`, who can
  call `approve_project`.
- `milestone-vault`'s admin, set via *its* `init`, who can `pause`,
  `configure_milestones`, `set_attestor`, and `cancel_project`.
- The backend's `ADMIN_ADDRESS`, which gates the platform admin API
  routes and the frontend's `/admin` panel.

In a typical deployment these are all the same person's address for
simplicity, but nothing enforces that — see [Deploying the contracts](./deploying-contracts.md).
