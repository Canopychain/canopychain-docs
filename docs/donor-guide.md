---
sidebar_position: 10
---

# Donor guide

## 1. Connect a wallet

Any wallet [Stellar Wallets Kit](https://stellarwalletskit.dev/) supports —
Freighter is the most common. Click **Connect Wallet** in the top nav.

## 2. Find a project

Browse `/projects` for approved reforestation projects. Only approved
projects show up here — that's the platform's curation layer, not
something enforced by the contract itself. Each project's own page shows
its plot boundary on a map, its milestone timeline, and how much has been
deposited and released so far.

## 3. Fund it

From a project's page, **Fund this project** opens the funding form:

- **Token** — native XLM, or paste a custom asset's contract address.
- **Amount** — how much you're depositing.

Unlike a per-second payment stream, there's no rate or duration to set —
your deposit joins the project's pooled escrow immediately, and stays
there until a milestone releases a tranche of the **whole pool** (not
just your share) to the project's recipient.

**Review & Sign** builds the transaction and asks your wallet to sign it.
Once confirmed, your contribution is part of the project's pool
immediately, on-chain.

## 4. Track your donations

`/dashboard` lists every project you've funded, with your totals across
all of them. There's deliberately no way to cancel or withdraw a deposit
the way you could stop a payment stream — your funds are pooled with
every other donor's, held in escrow until a milestone earns their
release, and there's no way to unilaterally pull your share back out of a
pool other people also contributed to.

The one exception: if a project stalls and an admin cancels it, a
**Claim refund** button appears next to that project on your dashboard.
Refunds are proportional — you get back your share of whatever hasn't
been released yet (`your donation / total deposited` of the remainder),
not necessarily your full original amount, since some tranches may have
already paid out before the project was cancelled. See
[The Milestone Attestation Model](./attestation-model.md#what-limits-the-blast-radius)
for why cancellation exists and what triggers it.

Both funding and refunding confirm on-chain right away, but the numbers
on `/dashboard` and a project's own page come from an indexer that polls
on an interval — a just-confirmed change can take a few seconds to show
up. If it's been longer than that, something's actually wrong; otherwise,
it's just catching up.
