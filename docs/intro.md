---
slug: /
sidebar_position: 1
---

# What is Canopychain?

Canopychain is an open-source reforestation funding platform on
[Stellar](https://stellar.org) that releases donor funds to a project
**in tranches, only once satellite data confirms the forest on the
specific GPS-bounded plot being funded is still standing** — not on a
manual review cycle, and not as one lump sum up front.

A donor picks a registered project, deposits into its escrow vault, and
waits. A backend service checks the plot against Global Forest Watch on a
schedule and records how much of its forest is still standing. Once a
plot has held above its milestone's floor for long enough — months, by
design — the service attests that on-chain, releasing the tranche to the
project's recipient. Nothing releases without an attestation; nothing
attests without an unbroken record of satellite checks behind it.

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
- **The MVP metric is forest *retention*, not certified carbon tons —
  and not confirmed regrowth either.** The free satellite data available
  on a rolling basis records forest loss, not new growth, so what
  Canopychain can honestly pay for is a plot staying intact over time.
  Real carbon-credit accounting additionally requires accredited
  registries (Verra, Gold Standard) that this project doesn't integrate.
  Claiming either would overstate what's actually being measured — see
  [Forest Retention Metric & Limitations](./forest-cover-metric.md).

## How it works

1. **An operator registers a project** — a name, a GPS-bounded polygon,
   a recipient address, and an attestor address, submitted on-chain via
   `project-registry`.
2. **An admin approves it**, opening it up for funding.
3. **Donors fund the project** — deposits go into `milestone-vault`'s
   pooled escrow for that project, not directly to the operator.
4. **The backend polls Global Forest Watch** for each active project's
   polygon on a schedule, recording how much of the plot's baseline
   forest is still standing each time.
5. **Once the plot has held above the next milestone's floor** for that
   milestone's full sustain period, with no gaps in the record, the
   backend signs and submits an attestation on-chain, releasing that
   tranche to the recipient.
6. **If a project stalls**, an admin can cancel it; donors can then
   reclaim their proportional share of whatever hasn't been released yet.

Everything — the registry, every deposit, every attestation — lives
on-chain and is independently verifiable; the backend indexer and
frontend are just a convenient way to read and interact with it.

## Where to start

The sidebar lists every page in one flat order, but most of them only
matter to one of four audiences. Pick the one that's you:

- **Donor** — funding a project and tracking what you've given. Start
  with the [Donor guide](./donor-guide.md), then
  [What happens to a stalled project](./stalled-projects.md) to
  understand what a refund actually depends on, and
  [Verifying a project's polygon yourself](./verifying-polygons.md) if
  you want to check a project's claims independently rather than take
  them on trust.
- **Operator** — registering a project to receive milestone-verified
  funding. Start with the
  [Operator guide](./operator-guide.md), then
  [What happens to a stalled project](./stalled-projects.md) for what
  happens if your plot stops clearing its retention floor.
- **Admin** — running an instance: approving registrations, configuring
  milestone schedules, and cancelling stalled projects. Start with the
  [Admin runbook](./admin-runbook.md), which walks through those in the
  order that avoids leaving a project depositable with no schedule, then
  [What happens to a stalled project](./stalled-projects.md) and
  [The Milestone Attestation Model](./attestation-model.md#what-limits-the-blast-radius)
  for what cancellation and attestor rotation actually do.
- **Contributor / developer** — building, deploying, or extending
  Canopychain itself. Start with [Architecture](./architecture.md), then
  the [Contracts reference](./contracts.md),
  [Deploying the contracts](./deploying-contracts.md), and
  [Contributing](./contributing.md).

Whichever you are, the [FAQ](./faq.md) and [Glossary](./glossary.md) are
useful references throughout, not just for one audience.

## Repositories

Canopychain is split across four repos:

- **[canopychain-contracts](https://github.com/canopychain/canopychain-contracts)** — the Soroban smart contracts: a project registry and a milestone-release escrow vault.
- **[canopychain-backend](https://github.com/canopychain/canopychain-backend)** — the Global Forest Watch polling worker and attestation submitter, an indexer that watches the contracts for events, and the API that serves that data.
- **[canopychain-frontend](https://github.com/canopychain/canopychain-frontend)** — the donor and operator web app.
- **canopychain-docs** — this site.
