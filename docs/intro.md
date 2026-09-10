---
slug: /
sidebar_position: 1
---

# What is Canopychain?

Canopychain is an open-source reforestation funding platform on
[Stellar](https://stellar.org) that releases donor funds to a project
**in tranches, only when satellite data confirms real forest-cover
change on the specific GPS-bounded plot being funded** — not on a manual
review cycle, and not as one lump sum up front.

A donor picks a registered project, deposits into its escrow vault, and
waits. As the project's plot shows measurable forest-cover change, a
backend service checks that against the project's milestone schedule and,
when a threshold is crossed, attests to it on-chain — releasing that
tranche to the project's recipient. Nothing releases without an
attestation; nothing attests without the underlying satellite check.

## The problem

Most on-chain reforestation and environmental funding today is a one-off
transaction: donors send funds once, and the operator receives a lump sum
immediately, with no on-chain mechanism tying payout to outcome.
Milestone-based crowdfunding is the more honest pattern, but it usually
means a slow, centralized, manual review cycle — someone has to actually
go check the plot.

Canopychain automates that check against a real, independent data source
(the Global Forest Watch API) instead of a person's judgment call, and
ties fund release directly to the result.

## An honest architecture, not an overclaimed one

Two constraints shaped this project deliberately, and both are worth
stating up front rather than glossing over:

- **Soroban has no mature decentralized oracle network** — there is no
  trustless way today to get satellite data on-chain the way a Chainlink
  feed would on other ecosystems. Canopychain uses a **semi-trusted
  attestor**: a backend service computes the milestone condition from
  real satellite data, and a single trusted key attests it on-chain. This
  is a deliberate, disclosed tradeoff, not a hidden one — see
  [The Milestone Attestation Model](./attestation-model.md).
- **The MVP metric is "% forest-cover change in a polygon," not
  certified carbon tons.** Real carbon-credit accounting requires
  accredited registries (Verra, Gold Standard) that this project doesn't
  integrate. Claiming otherwise would overstate what's actually being
  measured — see [Forest-Cover Metric & Limitations](./forest-cover-metric.md).

## How it works

1. **An operator registers a project** — a name, a GPS-bounded polygon,
   a recipient address, and an attestor address, submitted on-chain via
   `project-registry`.
2. **An admin approves it**, opening it up for funding.
3. **Donors fund the project** — deposits go into `milestone-vault`'s
   pooled escrow for that project, not directly to the operator.
4. **The backend polls Global Forest Watch** for each active project's
   polygon on a schedule, computing forest-cover change since the last
   check.
5. **When a milestone threshold is crossed**, the backend signs and
   submits an attestation on-chain, releasing that tranche to the
   recipient.
6. **If a project stalls**, an admin can cancel it; donors can then
   reclaim their proportional share of whatever hasn't been released yet.

Everything — the registry, every deposit, every attestation — lives
on-chain and is independently verifiable; the backend indexer and
frontend are just a convenient way to read and interact with it.

## Repositories

Canopychain is split across four repos:

- **[canopychain-contracts](https://github.com/canopychain/canopychain-contracts)** — the Soroban smart contracts: a project registry and a milestone-release escrow vault.
- **[canopychain-backend](https://github.com/canopychain/canopychain-backend)** — the Global Forest Watch polling worker and attestation submitter, an indexer that watches the contracts for events, and the API that serves that data.
- **[canopychain-frontend](https://github.com/canopychain/canopychain-frontend)** — the donor and operator web app.
- **canopychain-docs** — this site.

See [Architecture](./architecture.md) for how these fit together,
[Deploying the contracts](./deploying-contracts.md) to run your own
instance, or [Contributing](./contributing.md) to work on Canopychain
itself.
