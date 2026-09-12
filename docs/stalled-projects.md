---
sidebar_position: 10.5
---

# What happens to a stalled project

A project can be fully approved, funded, and given a milestone schedule,
and then simply never cross its next threshold — the plot's forest cover
stops changing enough, the operator goes dark, or the project just doesn't
work out. Nothing about that state is a contract error: `attest_milestone`
only ever fires when the evaluator finds the threshold crossed, and a
threshold that's never crossed just means it's never called again. The
pieces of what happens next — cancellation, refunds, and who actually
decides to pull the trigger — are documented separately today. This page is
the single place that ties them together.

## Nothing detects a stall automatically

Worth stating plainly: there is no timeout, no "N checks with no progress"
rule, and no automatic cancellation anywhere in this system. The backend's
milestone evaluator does exactly one thing on every check — compare a
project's cumulative forest-cover change to its next pending milestone's
`threshold_bps`, and attest if it's crossed
([`src/milestones/evaluator.ts`](https://github.com/canopychain/canopychain-backend/blob/main/src/milestones/evaluator.ts)).
It has no concept of "this project hasn't moved in a while" — a stalled
project just keeps failing that comparison, silently, forever, exactly like
a healthy project between two milestones does. The only difference between
"stalled" and "just slow" is time, and nothing in the system is watching
the clock.

That also means there's currently no dedicated signal to notice a stall
by. Each Global Forest Watch check is recorded as a
`ForestCoverSnapshot` (see [The backend's data model](./data-model.md)),
but that table isn't exposed by any public API endpoint — a donor or
operator watching for staleness today has only `GET /projects/:id`'s
milestone timeline to go on: the same `PENDING` milestone, unchanged, across
repeated checks over an unusually long time.

## The judgement call

Deciding a project has stalled — as opposed to just being slow, or between
two legitimately far-apart milestones — is entirely a platform admin's
manual call. There's no published SLA for it, the same way there's
[no published SLA for reviewing a new registration](./operator-guide.md#3-wait-for-review):
it depends on the instance operator's own judgement about the specific
project, its plot, and how long is reasonable for real-world forest-cover
change to show up.

## What an admin actually does

`/admin/projects` — the oversight view described in the
[architecture](./architecture.md) and used for [attestor rotation](./contracts.md#milestone-vault) —
is where this happens: a connected admin wallet can cancel any registered
project, approved or not, from that one screen. This is worth telling apart
from the *other* thing that looks similar, rejecting a pending
registration:

| | Rejecting a pending project | Cancelling a stalled project |
| --- | --- | --- |
| Where | `/admin`, or `POST /projects/:id/reject` | `/admin/projects`, calling `cancel_project` |
| On-chain? | No — there's no on-chain "reject" | Yes — `milestone-vault`'s `cancel_project` |
| Applies to | A project that was never approved | A project that's already approved, funded, and possibly mid-schedule |
| Reversible? | No on-chain record either way | No — cancellation is final for that vault |

Calling `cancel_project` **requires the vault admin's auth** and, per the
[contracts reference](./contracts.md#milestone-vault), immediately:

- Blocks any further `deposit` or `attest_milestone` call against that
  project's vault (`VaultCancelled`).
- Leaves whatever's already been released to the recipient exactly where it
  is — cancellation only affects what hasn't paid out yet.
- Opens `refund` for every donor who has an outstanding contribution.

This is also, by design, the circuit breaker for a *different* problem —
a compromised or unresponsive attestor — not just a stalled plot. See
[what limits the blast radius](./attestation-model.md#what-limits-the-blast-radius)
for that angle on the same function.

## What a donor gets back

Refunds are pull-based: each donor calls `milestone-vault`'s `refund`
themselves (their own wallet's auth, not something an admin or the
recipient can trigger on their behalf), and receives their proportional
share of whatever's left in the pool —

```
refund = donation * (total_deposited - total_released) / total_deposited
```

— not necessarily their full original deposit, since any tranches already
attested before cancellation have already left the pool. See the
[donor guide](./donor-guide.md#4-track-your-donations) for the donor-facing
walkthrough and the [contracts reference](./contracts.md#milestone-vault)
for the exact function signature and errors (`NotCancelled`,
`NothingToRefund`).

## A gap worth knowing about

The indexer that mirrors on-chain events into the backend's database
currently does **not** handle `milestone-vault`'s `cancelled` event — see
[The backend's data model](./data-model.md#the-cancelled-column-doesnt-yet-mirror-cancel_project).
In practice this means a project cancelled through `/admin/projects` changes
the vault's own on-chain state immediately, but the `GET /projects/:id`
API response and the frontend's `cancelled` badge may keep showing the
project as active until that's fixed. If you need to know for certain
whether a given project has actually been cancelled, `milestone-vault`'s
own `get_vault` is the source of truth — not the dashboard.
