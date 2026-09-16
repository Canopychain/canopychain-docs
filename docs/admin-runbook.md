---
sidebar_position: 10.8
---

# Admin runbook

This page is for whoever operates a Canopychain instance — the person
holding the admin keys, not a donor or a project operator. Each action
here is already described where it's defined, in the
[contracts reference](./contracts.md) or the
[API reference](./api-reference.md); this page is the sequence an admin
actually works through, and the order-of-operations mistakes that aren't
obvious from either reference alone.

## Which admin are you?

"Admin" is three separate configuration values, not one — see the
[glossary](./glossary.md#canopychain-specific). Before doing anything
below, know which of these you're acting as, since each uses a different
key and a different signing mechanism:

| Identity | Set via | Can do | Signs with |
| --- | --- | --- | --- |
| `project-registry`'s admin | that contract's `init` | `approve_project` | A direct on-chain transaction |
| `milestone-vault`'s admin | that contract's `init` | `configure_milestones`, `set_attestor`, `pause`/`unpause`, `cancel_project` | A direct on-chain transaction |
| Backend `ADMIN_ADDRESS` | backend `.env` | `GET /projects/pending`, `GET /projects/all`, `POST /projects/:id/reject` | A [SEP-53 signed header](./api-reference.md#authentication), not a transaction |

In a typical single-operator deployment all three are the same address,
but nothing enforces that. If your instance split them across different
keys, keep that mapping written down somewhere — a failed call here gives
no hint beyond "auth failed," not which of the three keys it wanted.

## The order that matters: schedule before approval

Approving a project and configuring its milestone schedule are two
independent calls, on two different contracts, and nothing forces one to
happen before the other. But `milestone-vault` doesn't check
`project-registry` at all — `deposit` opens a vault for any project id,
approved or not — so **the moment a project is approved and listed,
donors can start depositing into it**, whether or not a schedule exists
yet.

A deposit into a project with no configured schedule isn't rejected, and
it isn't lost — it just sits in the pool, uncounted toward any tranche,
because `attest_milestone` fails with `ScheduleNotFound` until
`configure_milestones` is called. Nothing warns you about this at the
time; the donor's transaction succeeds either way.

To avoid depending on how quickly you get around to the second call,
do them in this order:

1. Review the registration (below).
2. Call `configure_milestones` for the project **first**, while it's
   still unapproved and off the public explorer.
3. Call `approve_project` only once the schedule is in place.

This is the reverse of the order the
[operator guide](./operator-guide.md#4-youre-listed--set-a-milestone-schedule)
describes from the operator's side — that page shows approval and
scheduling as two separate steps an operator waits through, without
telling you which order minimizes risk. From the admin side, doing the
schedule first costs nothing (the operator doesn't need to be able to see
the schedule before it's set) and closes the gap entirely.

## Reviewing and approving a project

New registrations land in the review queue:

```
GET /projects/pending
```

(signed per [Authentication](./api-reference.md#authentication) — this is
a backend call, not a contract call). For each one, decide from its
polygon and details whether to approve or reject it — see
[Verifying a project's polygon yourself](./verifying-polygons.md) if you
want to check the submitted plot independently rather than trusting the
frontend's rendering of it.

- **Reject**: `POST /projects/:id/reject`, optionally with
  `{ "reviewNote": "..." }`. Backend-only — there's no on-chain "reject,"
  so this is the only record of it. Reversible in the sense that nothing
  on-chain has happened yet.
- **Approve**: call `project-registry`'s `approve_project` directly
  on-chain, from the admin panel. Only do this after the schedule is
  configured — see above. There's no on-chain "un-approve"; if you
  approve by mistake, [cancelling the project](#cancelling-a-stalled-project)
  is the only way back, and that's final for its vault.

## Configuring a milestone schedule

`configure_milestones(project_id, milestones)` on `milestone-vault` — see
the [contracts reference](./contracts.md#milestone-vault) for the exact
shape of a `Milestone` and its validation rules
(`threshold_bps` strictly increasing, `payout_bps` summing to at most
10,000).

This can only be called **once** per project — there's no "update
schedule" function, deliberately, so a schedule donors already funded
against can't be changed underneath them. Agree on the thresholds and
payout split with the project operator before calling this; there's no
fixing a typo afterward short of cancelling the whole project.

## Rotating an attestor

`set_attestor(project_id, new_attestor)` on `milestone-vault` replaces
the address authorized to attest milestones for one project — see the
[contracts reference](./contracts.md#milestone-vault). Reach for this
when an attestor key is retiring or you suspect it's been compromised;
see [what limits the blast radius](./attestation-model.md#what-limits-the-blast-radius)
for why a compromised attestor is bounded to begin with and doesn't
always require an emergency rotation.

Rotating doesn't require anything from the donor or the recipient — it
only changes who can call `attest_milestone` going forward, and takes
effect immediately for the next attestation.

## Pausing the vault

`pause()` / `unpause()` on `milestone-vault` halt or restore `deposit`
and `attest_milestone` **contract-wide**, across every project — there's
no per-project pause. Reach for this for a suspected systemic issue (a
compromised admin or attestor key, a bug found in production) rather than
a single project's problem, which `cancel_project` handles instead
without affecting anyone else's vault.

`refund` is deliberately exempt from pause — see
[Security and audit status](./security.md#contracts). Pausing stops new
activity; it never traps a donor's funds that are already eligible for
refund from a separately cancelled project.

Check current state before acting on a report of "deposits are failing"
— `paused(env)` is a read-only call, and a stuck deposit is just as
likely to be a wrong contract id or an unfunded testnet account as an
actual pause.

## Cancelling a stalled project

Deciding a project has stalled — rather than just being slow between two
legitimately far-apart milestones — is a manual judgement call with no
published SLA. [What happens to a stalled project](./stalled-projects.md)
covers this in full, including how it differs from rejecting a pending
registration, what `cancel_project` actually does on-chain, and a known
gap where the indexer doesn't yet mirror the `cancelled` event into the
dashboard. Read that page before cancelling anything; the summary here is
only: it's final for that project's vault, it doesn't touch tranches
already released, and it opens `refund` for every donor with an
outstanding contribution.
