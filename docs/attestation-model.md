---
sidebar_position: 4
---

# The Milestone Attestation Model

Canopychain releases funds when "satellite data confirms real forest-cover
growth." It's worth being precise about what actually confirms that,
because the honest answer is a person's key, informed by a machine
process — not a trustless, on-chain fact the way a token transfer is.

## Why not fully trustless?

A fully trustless design would need the forest-cover check itself to
happen verifiably on-chain, or through a decentralized oracle network that
can prove it queried Global Forest Watch honestly and reported the result
unmodified. **Soroban has no mature decentralized oracle network** — no
Chainlink equivalent — as of this writing. Building one from scratch was
out of scope for this project; pretending the gap doesn't exist would have
been worse than disclosing it.

## The semi-trusted attestor

Instead, Canopychain uses a **semi-trusted attestor**: a backend service
computes the milestone condition from real data, and a single trusted key
signs the result on-chain. Concretely:

1. `canopychain-backend`'s GFW polling worker queries Global Forest Watch
   for a project's polygon on a schedule and computes forest-cover change
   since the last check (see
   [Forest-Cover Metric & Limitations](./forest-cover-metric.md) for what
   that number does and doesn't mean).
2. The milestone evaluator compares the project's cumulative change
   against its next pending milestone's `threshold_bps`.
3. If it's crossed, the attestation submitter signs and submits
   `attest_milestone` using the backend's configured attestor key — see
   [Architecture](./architecture.md#the-backend) for where this sits
   relative to the rest of the backend, which otherwise never signs
   anything.

The contract itself (`milestone-vault`) has no way to verify satellite
data — it only verifies that whoever called `attest_milestone` controls
the `attestor` address recorded against that project's vault. **Trusting a
milestone attestation on Canopychain means trusting that key's holder,**
full stop. That's a real, disclosed limitation, not a detail to gloss
over.

## What limits the blast radius

A compromised or dishonest attestor can't do arbitrary damage — the
contract narrows what an attestation can actually do:

- **Only the current project's attestor can act**, and only on that one
  project. `set_attestor` lets an admin rotate a project's attestor key at
  any time, without needing anything from the recipient or donors — the
  intended response to a suspected compromise.
- **Payout is capped by the project's own schedule**, configured once by
  an admin and immutable after (`configure_milestones` — see the
  [contracts reference](./contracts.md#milestone-vault)). An attestor
  can't invent a new tranche or exceed the schedule's total.
- **Attestation only ever advances the next pending milestone** — a
  compromised key can trigger a release early, but can't replay old
  milestones or release more than one schedule's worth of funds.
- **`cancel_project` is the circuit breaker.** An admin who loses
  confidence in a project — a compromised attestor, an operator gone
  dark, or a project that's simply stalled — can halt it outright.
  Cancelling blocks all further deposits and attestations, and donors can
  then reclaim their proportional share of whatever hasn't been released
  yet via `refund`.

## What this means for a donor

Funding a project on Canopychain means trusting three things, not one:
the Soroban contracts (open-source, [reference here](./contracts.md)),
the platform admin's `approve_project` decision, and the specific
project's attestor key. That's a meaningfully different trust model from
"the blockchain guarantees this," and this doc exists so nobody has to
take that on faith or discover it by reading the contract source
themselves.
