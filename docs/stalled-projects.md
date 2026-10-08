---
sidebar_position: 10.5
---

# What happens to a stalled project

A project can be fully approved, funded, and given a milestone schedule,
and then never earn its next tranche. Nothing about that state is a
contract error: `attest_milestone` only fires when the evaluator finds a
milestone's conditions met, and conditions that are never met just mean
it's never called again. The pieces of what happens next — cancellation,
refunds, and who actually decides to pull the trigger — are documented
separately today. This page is the single place that ties them together.

## A stall has three different shapes

Because a milestone is a retention floor *and* a sustain period, there's
more than one way to be stuck, and they call for different responses:

- **The plot is below its floor.** Forest was cleared. Every check below
  the floor restarts the sustain clock from zero rather than pausing it,
  so a plot that dips repeatedly can stay short of a milestone
  indefinitely even if it spends most of its time compliant. This is the
  case that usually means the project has actually failed.
- **The plot is holding, but the checks stopped.** A gap in the record
  longer than `GFW_MAX_OBSERVATION_GAP_SECONDS` breaks a streak even
  though the forest is fine, because an outage isn't evidence of
  compliance. A backend that was down, an expired GFW key, or a polygon
  GFW keeps rejecting (which puts the project on an escalating per-project
  backoff) all land here. This is an infrastructure problem wearing a
  stalled project's clothes, and cancelling is the wrong response to it.
- **Nothing is wrong and it's simply slow.** Sustain periods are months
  long by design, and they increase across a schedule. A healthy project
  mid-sustain-period is indistinguishable from a stalled one by status
  alone.

## Nothing detects a stall automatically

Worth stating plainly: there is no timeout, no "no progress in N checks"
rule, and no automatic cancellation anywhere in this system. The
evaluator does exactly one thing per check — decide whether the next
pending milestone's floor has held long enough, and attest if it has
([`src/milestones/evaluator.ts`](https://github.com/canopychain/canopychain-backend/blob/main/src/milestones/evaluator.ts)).
It has no concept of "this project hasn't moved in a while," and nothing
in the system is watching the clock on a project's behalf.

What you *can* see is which of the three shapes you're in.
`GET /projects/:id` returns a `nextMilestoneProgress` object (see the
[API reference](./api-reference.md#get-projectsid)) that distinguishes
them without needing database access:

| What it shows | Which shape |
| --- | --- |
| `currentRetentionBps` below the floor | Below its floor — forest was lost |
| At or above the floor, but `sustainedSeconds` keeps resetting to 0 | Either a dipping plot or a broken check record |
| At or above the floor, `sustainedSeconds` climbing steadily | Healthy, just slow |

The one thing it won't tell you apart is a dipping plot from a stalled
poll worker, since both surface as a streak that won't accumulate. The
backend's own `/gfw/status` endpoint is what separates those: if the poll
worker hasn't completed a run recently, the problem is the checks, not
the forest.

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
