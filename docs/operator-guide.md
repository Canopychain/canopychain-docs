---
sidebar_position: 9
---

# Operator guide: registering a project

This page is for people who want to register a reforestation project on
Canopychain to receive milestone-verified funding — not for developers,
see the [contracts](./contracts.md) and [API](./api-reference.md)
references for that.

## Before you start: three addresses, not one

Registering a project asks for three Stellar addresses, and it matters
that you think about each one separately rather than reusing the same
wallet for all three:

- **Your operator wallet** — the one you connect to submit the
  registration itself. It's recorded on-chain but doesn't receive or
  control funds.
- **The recipient address** — where released tranches actually get paid.
  This should be a wallet your organization controls long-term; there's
  no "change recipient" flow available to an operator after registration
  (an admin can reassign a project's attestor, but not its recipient).
- **The attestor address** — the identity authorized to attest milestones
  for your project. In practice this is almost always the platform's own
  configured attestor key (see
  [The Milestone Attestation Model](./attestation-model.md)) — not
  something you generate yourself, unless you're running your own
  Canopychain instance end to end. Check with whoever operates the
  instance you're registering on for the correct value.

## 1. Draw your plot boundary

Your project's polygon is a real, binding part of the registration — it's
committed on-chain as a hash, and it's what the backend actually queries
Global Forest Watch against for every milestone check. Draw it at
[geojson.io](https://geojson.io) (or export one from GIS software you
already use) and download it as a `.geojson` file. It needs to be a
`Polygon` or `MultiPolygon` geometry.

## 2. Register

Go to `/register`, connect your operator wallet, and fill in your
project's name, the recipient and attestor addresses, and upload your
polygon file. Submitting does two things in sequence:

1. Calls `project-registry`'s `register` on-chain — this is what actually
   creates your project's on-chain entry and costs a transaction fee.
2. Submits your polygon and addresses to the backend, which attaches them
   to that same on-chain project id.

Your project starts **unapproved** — donors can't fund it yet.

## 3. Wait for review

An admin reviews newly registered projects and either approves or rejects
them from `/admin`. There's no fixed SLA published here since it depends
entirely on who's running a given instance of Canopychain — check with
whoever operates the instance you registered on.

Approval calls `project-registry`'s `approve_project` on-chain directly;
rejection is backend-only (there's no on-chain "reject" — see the
[API reference](./api-reference.md#post-projectsidreject)) and simply
keeps your project out of the public explorer.

## 4. You're listed — set a milestone schedule

Once approved, your project appears on `/projects` and donors can start
funding it. Before that's useful, an admin needs to configure your
project's tranche schedule (`configure_milestones` — see the
[contracts reference](./contracts.md#milestone-vault)). Each tranche says
three things: how much of your plot's baseline forest must still be
standing (`retention_floor_bps`), how long it must stay there
(`sustain_seconds`), and what share of the pool that earns
(`payout_bps`). The schedule can be corrected until your vault takes its
first deposit, and is locked from then on — so it's worth agreeing on
with whoever operates the instance before donors start funding.

**Set the floor from your plot's first measurement, not from 100%.** The
baseline is Global Forest Watch's year-2000 canopy layer, so a plot that
was cleared at any point since then starts out well below 100% retention
and will never return to it under this metric. A floor picked without
looking at your plot's actual first reading is either trivially met or
permanently unreachable. See
[Forest Retention Metric & Limitations](./forest-cover-metric.md#floors-are-relative-to-the-year-2000-baseline).

## 5. Receiving funds

You don't do anything to receive a tranche — there's no withdraw button
to click. Once the backend's satellite checks show your plot has held
above the next milestone's floor for that milestone's full sustain
period, the attestor signs an attestation on-chain and the tranche
transfers straight to your recipient address automatically. Watch your
project's own page for its live milestone timeline, which shows both how
much is standing now and how far into the sustain period you are.

A few things worth knowing:

- **Tranches are slow by design.** A sustain period is measured in
  months, and it only ever runs forward while the plot is above the
  floor. A single check below the floor restarts that clock from zero —
  it doesn't pause it — so a plot that dips repeatedly can stay short of
  a milestone indefinitely.

- Funds are pooled from every donor into one on-chain vault per project —
  there's no per-donor tracking on the payout side, and the tranche size
  is a percentage of the **total pool**, not any one donor's contribution.
- If your project stalls and an admin cancels it, no further tranches
  release, and donors can reclaim their proportional share of whatever
  hasn't been paid out yet. Whatever already released to you stays
  released.
