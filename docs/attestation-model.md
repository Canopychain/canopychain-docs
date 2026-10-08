---
sidebar_position: 4
---

# The Milestone Attestation Model

Canopychain releases funds when "satellite data confirms a plot's forest
is still standing." It's worth being precise about what actually confirms
that, because the honest answer is a person's key, informed by a machine
process — not a trustless, on-chain fact the way a token transfer is.

## Why not fully trustless?

A fully trustless design would need the forest check itself to happen
verifiably on-chain, or through a decentralized oracle network that can
prove it queried Global Forest Watch honestly and reported the result
unmodified. **Soroban has no mature decentralized oracle network** — no
Chainlink equivalent — as of this writing. Building one from scratch was
out of scope for this project; pretending the gap doesn't exist would have
been worse than disclosing it.

## The semi-trusted attestor

Instead, Canopychain uses a **semi-trusted attestor**: a backend service
computes the milestone condition from real data, and a single trusted key
signs the result on-chain. Concretely:

1. `canopychain-backend`'s GFW polling worker queries Global Forest Watch
   for a project's polygon on a schedule and records how much of its
   baseline forest is still standing, in basis points (see
   [Forest Retention Metric & Limitations](./forest-cover-metric.md) for
   what that number does and doesn't mean).
2. The milestone evaluator checks the next pending milestone's two
   conditions against that history: has retention held at or above
   `retention_floor_bps`, continuously, for at least `sustain_seconds`?
   A reading below the floor breaks the run, and so does a gap in the
   record longer than the configured maximum — an outage doesn't count as
   compliance.
3. If both hold, the attestation submitter signs and submits
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

## Time is the attestor's main constraint

A retention milestone can't be satisfied in a hurry. `sustain_seconds` is
validated on-chain as non-zero and strictly increasing across a schedule,
so every tranche after the first requires a longer unbroken record than
the one before it. An honest attestor therefore can't release a schedule
early even if a project looks perfect on day one, and the structure makes
a dishonest early release conspicuous: the vault's attestation events are
timestamped, and the backend's recorded checks are what the claim rests
on.

## What limits the blast radius

A compromised or dishonest attestor can't do arbitrary damage — the
contract narrows what an attestation can actually do:

- **Only the current project's attestor can act**, and only on that one
  project. `set_attestor` lets an admin rotate a project's attestor key at
  any time, without needing anything from the recipient or donors — the
  intended response to a suspected compromise.
- **Payout is capped by the project's own schedule**, set by an admin and
  locked once the vault has taken its first deposit (`configure_milestones`
  — see the [contracts reference](./contracts.md#milestone-vault)). An
  attestor can't invent a new tranche or exceed the schedule's total, and
  nobody can move the goalposts under donors who have already funded.
- **A schedule can't be configured to pay for nothing.** The contract
  rejects a zero retention floor (which bare ground satisfies) and a zero
  sustain period (which anything satisfies instantly), so there's no
  legitimate-looking schedule that releases funds on no evidence.
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

One piece of this you don't have to take on faith at all: which plot a
project is actually being scored against. That's committed on-chain as a
hash specifically so it can't be silently swapped after you've funded a
project — see [Verifying a project's polygon yourself](./verifying-polygons.md)
for how to check it against the backend's copy independently.
