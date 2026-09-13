---
sidebar_position: 13
---

# FAQ

### Is Canopychain audited?

No. See [Security and audit status](./security.md) for exactly what
protections do and don't exist.

### Does a milestone attestation mean the forest actually grew?

Not exactly — it means the backend's satellite check found no *further
loss* against the year-2000 baseline within the plot, which is the
closest honest signal available from free, near-real-time data. It's not
a certified confirmation of new growth. See
[Forest-Cover Metric & Limitations](./forest-cover-metric.md) for the
full explanation.

### Who verifies the attestor is telling the truth?

No one, cryptographically — that's the honest limitation described in
[The Milestone Attestation Model](./attestation-model.md). Trusting an
attestation means trusting that specific key's holder. The contract does
bound what a compromised or dishonest attestor can do: it can only affect
the one project it's configured for, only up to that project's own
pre-configured schedule, and an admin can rotate the key or cancel the
project outright if something looks wrong.

### What token can I donate in?

Native XLM, or any Soroban token contract address you paste into the
custom-asset field. The amount math assumes 7 decimal places, which is
fixed by the Stellar Asset Contract standard for every asset, not
something per-token to configure.

### Is there a minimum donation?

Not a fixed one — any positive amount is accepted by the contract. An
extremely small deposit is still valid, though it may end up contributing
close to nothing to any individual tranche payout, since payouts are
computed as a percentage of the whole pool.

### Can I get my donation back?

Only if the project you funded gets cancelled. Unlike a payment stream
you control directly, a deposit joins a shared pool with every other
donor's contribution — there's no way to unilaterally pull your own share
back out of it. If an admin cancels a stalled project, you can then claim
a proportional refund of whatever hasn't already been released. See the
[donor guide](./donor-guide.md).

### What happens if I lose access to my wallet?

A project you funded keeps working exactly as the contract defines it —
losing your wallet doesn't affect the pool or any project's ability to
receive future attestations. What you lose is the ability to claim a
refund if that project is later cancelled, since that requires your
signature. For operators, the stakes are higher: the recipient address
you register is where every tranche pays out, with no recovery mechanism
if you lose access to it — see the [operator guide](./operator-guide.md).

### Can a project receive funding before being approved?

On-chain, technically yes — `milestone-vault` doesn't check
`project-registry`, so a deposit can target any project id. In the app,
no: the project explorer only ever lists approved projects, so there's no
path to fund an unapproved one without directly calling the contract
yourself.

### Why does the dashboard look out of date right after I do something?

Actions confirm on-chain immediately, but the numbers you see come from a
backend indexer that polls the chain on an interval (a few seconds by
default), not from the chain directly. Give it a moment before assuming
something's wrong.

### Does it cost anything to use Canopychain?

Not a platform fee, but yes — every on-chain action is a Stellar
transaction, and Stellar's network fee (typically a fraction of a cent,
paid in XLM) is charged to whoever signs it, not to Canopychain:

- A **donor** pays it when depositing into a project, and again if they
  later claim a refund.
- An **operator** pays it when registering a project.
- An **admin** pays it when approving a registration, configuring a
  milestone schedule, or cancelling a project.
- The **backend's attestor** pays it when submitting `attest_milestone` —
  see [The Milestone Attestation Model](./attestation-model.md). Neither
  the donor whose funds are released nor the operator who receives them
  pays for that transaction.

That's separate from the fee question below, which is about a
Canopychain-specific cut of funds — there isn't one.

### What fees does Canopychain take?

None — `milestone-vault` has no protocol-fee mechanism at all in this
MVP. Every attested tranche pays out in full to the project's recipient.
This is about Canopychain's own cut of funds; see the question above for
the Stellar network fees every on-chain action still costs.

### Can I run my own instance of Canopychain?

Yes — it's open source across four repos: [contracts](https://github.com/canopychain/canopychain-contracts),
[backend](https://github.com/canopychain/canopychain-backend),
[frontend](https://github.com/canopychain/canopychain-frontend), and this
docs site. See [Deploying the contracts](./deploying-contracts.md) and
[Local development](./local-development.md) to get started.

### What network does Canopychain run on?

[Stellar](https://stellar.org), via Soroban smart contracts. Everything
in this documentation defaults to testnet; running on mainnet is a
deliberate, manual step an operator takes — see [Deploying the contracts](./deploying-contracts.md#mainnet).
