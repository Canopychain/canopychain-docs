---
sidebar_position: 15
---

# Roadmap and changelog

## Roadmap

Known gaps, roughly in the order they'd matter to someone deploying a
real instance:

- **No external security audit yet.** See [Security and audit status](./security.md).
  This is the top item for a reason.
- **No mainnet deployment.** Everything documented here defaults to
  testnet; going to mainnet is a deliberate manual step for whoever
  operates an instance, not something the project has done itself.
- **Milestone attestation is semi-trusted, not trustless.** Soroban has
  no mature decentralized oracle network today, so a backend-held key
  attests satellite-derived results on-chain. This is a disclosed,
  bounded design decision, not an oversight — see
  [The Milestone Attestation Model](./attestation-model.md) — but it's
  the single biggest thing that would change if the Soroban oracle
  ecosystem matures.
- **The forest-cover metric can't positively confirm regrowth**, only the
  absence of further loss against a year-2000 baseline. See
  [Forest-Cover Metric & Limitations](./forest-cover-metric.md) for what
  a production-grade version of this would actually need.
- **No rate limiting on any backend endpoint.** A known, un-mitigated gap
  in the current MVP — see [Security and audit status](./security.md).
- **The Freighter-driven end-to-end tests are documented stubs, not
  verified tests.** The navigation-only end-to-end tests are real and run
  in CI; the wallet-signing flows need a real browser and a real
  extension to actually verify, which isn't something CI does today.
- **No real-time push.** The dashboard, explorer, milestone timeline, and
  impact page all poll on an interval rather than updating the moment
  something changes on-chain — and the GFW polling worker itself only
  checks each project every `GFW_POLL_INTERVAL_MS` (6 hours by default),
  so "milestone-verified" is not the same as "instant."

## Changelog

### 0.1.0 — initial release

The first complete pass across all four repos:

- **Contracts** — `project-registry` (operator registration, admin
  approval) and `milestone-vault` (deposit, configure a milestone
  schedule, attest a milestone and release its tranche, pause/unpause,
  rotate an attestor, cancel a project, claim a proportional refund),
  with unit, integration, and tranche/refund-math edge-case tests.
- **Backend** — a Global Forest Watch polling worker and milestone
  evaluator, an attestation submitter, an event indexer with a durable,
  per-event checkpoint, a REST API for projects, donations, registration
  intake, and SEP-53-signature-gated admin review/oversight, retry with
  backoff for GFW and chain calls, and Docker Compose for local
  development.
- **Frontend** — wallet connection, project discovery with a polygon map,
  a live-polling milestone timeline, the full donor flow (fund a project,
  track donations, claim a refund after cancellation), an operator
  registration flow (polygon upload, on-chain register, backend intake),
  admin panels for project approval and ongoing oversight
  (attestor rotation, cancellation), and a public impact page.
- **Docs** — this site.
