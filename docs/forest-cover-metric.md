---
sidebar_position: 5
---

# Forest-Cover Metric & Limitations

Canopychain's milestone schedules are expressed as "% forest-cover
change" in basis points. It's worth being precise about what that number
actually is, what it's derived from, and — just as important — what it
is **not**, because the gap between "measured" and "true carbon impact"
is exactly the kind of thing an environmental funding platform can
accidentally overstate.

## What's actually measured

`canopychain-backend`'s GFW polling worker
([`src/gfw/pollWorker.ts`](https://github.com/canopychain/canopychain-backend/blob/main/src/gfw/pollWorker.ts))
estimates a project's current forest-cover percentage from two Global
Forest Watch datasets, both real and queried live, not simulated:

- **`umd_tree_cover_density_2000`** — a static, year-2000 baseline of
  canopy density. A pixel counts as "forest" if its density is at least
  30%.
- **`umd_tree_cover_loss`** — annual tree-cover loss, updated yearly.

The estimate is:

```
standing_forest = baseline_forest_area - cumulative_loss_since_baseline
forest_cover_pct = standing_forest / total_polygon_area
```

Each check ([`src/gfw/forestCoverChange.ts`](https://github.com/canopychain/canopychain-backend/blob/main/src/gfw/forestCoverChange.ts))
then compares that percentage to the project's previous check and reports
the signed difference, in basis points, as `changeBps`. The milestone
evaluator sums `changeBps` across all of a project's checks since baseline
to get the cumulative figure a milestone's `threshold_bps` is compared
against.

## The limitation this doesn't paper over

**GFW's free, near-real-time dataset tracks loss, not regrowth.** There is
no continuously-updated "tree cover gain confirmed" layer this project
integrates — tree-cover *gain* products that do exist are one-time,
multi-year retrospectives (e.g. covering 2000–2012 or 2000–2020), not
something a polling worker can check on a rolling schedule the way it can
check loss.

The practical consequence: this metric can only trend toward "less
forest" over time. On its own, it measures **how much of the original,
year-2000 forest still stands** — a real, useful, honestly-computed
number — but it cannot *positively confirm* that new trees have grown on
a previously bare plot. A reforestation project that starts from bare
land and successfully regrows a forest would not show that growth in this
metric the way a true regrowth-detection system would.

## Why this is still a defensible MVP metric

The project's own scope deliberately draws the line here rather than
overclaiming: real carbon-credit-grade accounting requires accredited
registries (Verra, Gold Standard) with their own certified measurement
methodologies, which is out of scope for this project entirely — see
[Security & Audit Status](./security.md) for the broader list of things
this platform does not claim to be. "% forest-cover change in a polygon,"
computed from real satellite data and disclosed with this exact
limitation, is a meaningfully more honest MVP than a platform that implies
certified carbon impact without the accreditation to back it.

## What a production version would need

Confirming actual regrowth — not just the absence of further loss — would
need a dataset this integration doesn't reach for: canopy-height or
biomass data with enough temporal resolution to detect new growth on a
rolling basis (for example, higher-cadence optical or radar imagery
processed for vegetation height, rather than a once-per-decade gain
layer). That's real, substantial future work, not a configuration change.
