---
sidebar_position: 5
---

# Forest Retention Metric & Limitations

Canopychain's milestone schedules are expressed as a **retention floor** —
the share of a plot's baseline forest that must still be standing — held
for a **sustain period**. It's worth being precise about what that number
is, what it's derived from, and — just as important — what it is **not**,
because the gap between "measured" and "true carbon impact" is exactly
the kind of thing an environmental funding platform can accidentally
overstate.

## What's actually measured

`canopychain-backend`'s GFW polling worker
([`src/gfw/pollWorker.ts`](https://github.com/canopychain/canopychain-backend/blob/main/src/gfw/pollWorker.ts))
derives a project's standing forest from two Global Forest Watch
datasets, both real and queried live, not simulated:

- **`umd_tree_cover_density_2000`** — a static, year-2000 baseline of
  canopy density. A pixel counts as "forest" if its density is at least
  30%.
- **`umd_tree_cover_loss`** — annual tree-cover loss, updated yearly.

Each check records two figures from them:

```
standing_forest  = baseline_forest_area - cumulative_loss_since_baseline

retention_bps    = standing_forest / baseline_forest_area * 10,000
forest_cover_pct = standing_forest / total_polygon_area * 100
```

`retention_bps` is the one that decides milestones. `forest_cover_pct` is
kept alongside it for display — "how much of this plot is forest" is the
more intuitive number to read, but it says nothing on its own about
whether the plot is holding.

A polygon with no baseline forest at all records `retention_bps` of 0
rather than a vacuous 10,000. "All of nothing is still standing" would
otherwise clear a 99% retention floor on bare ground, which is precisely
the kind of milestone that releases money for nothing.

## Why retention, and not growth

**GFW's free, near-real-time data tracks loss, not regrowth.** There is no
continuously-updated "tree cover gain confirmed" layer this project
integrates — tree-cover *gain* products that do exist are one-time,
multi-year retrospectives (e.g. covering 2000–2012 or 2000–2020), not
something a polling worker can check on a rolling schedule the way it can
check loss.

That isn't only a caveat about precision; it fixes the **direction** of
the measurement. Standing forest is baseline minus cumulative loss, and
cumulative loss never goes down, so the measured figure can only stay
level or fall. Any milestone phrased as "forest cover increased by X"
would therefore be unreachable by construction — not hard to reach, but
impossible, no matter how well the project is run.

Retention is the question this data can actually answer: *is the forest
that was there still there, and has it stayed there?* Both halves of a
milestone are decidable from real observations, which is why the schedule
is shaped that way rather than around a growth figure the integration
can't support.

## What a retention milestone pays for

Each milestone has three fields (see the
[contracts reference](./contracts.md#milestone-vault)):

| Field | Meaning |
| --- | --- |
| `retention_floor_bps` | Share of baseline forest that must still be standing |
| `sustain_seconds` | How long it must continuously hold at or above that floor |
| `payout_bps` | Share of the vault released when it does |

So a three-tranche schedule at a 99% floor over 90, 180 and 365 days pays
an operator progressively for *keeping a plot intact for a year*. That is
stewardship and protection against clearing — not confirmed regrowth.

The honest consequence: a project that plants seedlings on bare ground
earns nothing here for the seedlings themselves. The new trees are
invisible to this metric until a dataset that can see them is wired in.
What Canopychain can fund honestly today is keeping standing forest
standing.

### Floors are relative to the year-2000 baseline

Because the baseline is fixed at year 2000, a plot cleared in 2015 starts
out with retention well below 10,000 and will never return to it under
this metric. Floors have to be set from the project's **first recorded
check**, not from 100% — see the
[operator guide](./operator-guide.md) for how a schedule gets picked.

## How a streak is judged

The evaluator
([`src/milestones/evaluator.ts`](https://github.com/canopychain/canopychain-backend/blob/main/src/milestones/evaluator.ts))
walks a project's checks backwards from the newest and stops at the first
one that breaks the run — either a reading below the floor, or a gap
longer than `GFW_MAX_OBSERVATION_GAP_SECONDS` (two days by default).
Whatever survives is credited from its oldest passing check, because
retention is only known at the moments it was measured.

The gap rule carries more weight than it looks. Without it, a poll worker
that was down for a month would come back and claim a month of unobserved
compliance, and a project whose checks had stopped entirely would keep
accruing credit forever. The same rule also catches a stale newest
reading, since the first gap measured is the one between now and the
latest check.

## Why this is still a defensible MVP metric

The project's own scope deliberately draws the line here rather than
overclaiming: real carbon-credit-grade accounting requires accredited
registries (Verra, Gold Standard) with their own certified measurement
methodologies, which is out of scope for this project entirely — see
[Security & Audit Status](./security.md) for the broader list of things
this platform does not claim to be. "This share of the plot's baseline
forest is still standing, and has been for this long," computed from real
satellite data and disclosed with this exact limitation, is a
meaningfully more honest MVP than a platform that implies certified
carbon impact without the accreditation to back it.

## What a production version would need

Paying for confirmed regrowth — not just the absence of further loss —
would need a dataset this integration doesn't reach for: canopy-height or
biomass data with enough temporal resolution to detect new growth on a
rolling basis (for example, higher-cadence optical or radar imagery
processed for vegetation height, rather than a once-per-decade gain
layer). A per-project baseline captured at approval, rather than the
shared year-2000 one, would go with it. That's real, substantial future
work, not a configuration change.
