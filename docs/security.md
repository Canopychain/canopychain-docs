---
sidebar_position: 12
---

# Security and audit status

**Canopychain's contracts have not undergone an external audit.** Nothing
on this page is a substitute for one, and no one should deposit more than
they'd accept losing to an unaudited contract or an attestor-key
compromise (see [The Milestone Attestation Model](./attestation-model.md)
before assuming this system is trustless — it deliberately isn't, in one
specific, bounded way). This page exists to document what protections
exist, what's been deliberately left as an accepted risk, and how to
report a problem.

## What's in place

### Contracts

- Every mutating function requires the auth of a specific, named address
  (`require_auth`) — never inferred from who happened to call it.
- Tranche-payout and refund-share arithmetic use saturating operations —
  they can't overflow or panic, only cap out at what's actually available.
- A milestone schedule's payouts are hard-capped at 100% of the pool in
  the contract itself (`configure_milestones` rejects anything higher via
  `validate_schedule`) — an admin key compromise can't turn a schedule
  into a de facto over-payout.
- `pause` halts new activity (`deposit`, `attest_milestone`) without ever
  blocking `refund` — an admin can stop new deposits and attestations,
  but can never trap a donor's already-cancelled-project funds in the
  vault.
- Unit and integration tests cover the happy paths, the documented error
  paths, and — for the tranche and refund math specifically — a grid of
  edge-case inputs (zero/near-`i128::MAX` values, deposits that don't
  evenly divide a schedule's bps) as a stand-in for full property-based
  fuzzing.

### Backend and frontend

- With one deliberate exception (below), the backend never holds funds
  and never signs a transaction — every donor, operator, and admin
  contract call (`deposit`, `register`, `approve_project`,
  `cancel_project`, `set_attestor`, `refund`) is built and signed in the
  user's own browser. A compromised backend can serve wrong *data*; for
  all of those calls, it cannot move funds.
- **The one exception is `attest_milestone`.** The backend's attestor key
  is the single private key this system holds server-side, used for
  exactly one call. Its blast radius is bounded by design — see
  [what limits it](./attestation-model.md#what-limits-the-blast-radius) —
  and an admin can rotate a project's attestor via `set_attestor` at any
  time if that key is ever suspected compromised.
- Admin-only backend routes require a [SEP-53](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0053.md)-signed
  request bound to the specific method, path, and a timestamp — a
  captured request can't be replayed against a different endpoint, and is
  only valid for 5 minutes.
- There's no off-chain "approved" status that could ever drift from
  on-chain reality: approving a project calls `project-registry`'s
  `approve_project` directly from the admin's own wallet, and the
  backend's `approved` flag is only ever a mirror of that same event —
  never something the backend itself decides.

## Accepted risks and known gaps

- **Admin and attestor key custody is entirely the operator's
  responsibility.** Both contracts' admin address, the backend's
  `ADMIN_ADDRESS`, and `ATTESTOR_SECRET_KEY` are configuration, not
  something the protocol itself constrains beyond what's described above.
- **`milestone-vault` doesn't check `project-registry`.** A vault can be
  opened for any project id, approved or not; approval is enforced by the
  apps, not the chain. See [Deploying the contracts](./deploying-contracts.md#operational-note-the-registry-and-the-vault-dont-check-each-other)
  for the reasoning.
- **The forest-cover metric itself is a disclosed, bounded proxy, not a
  certified measurement.** See [Forest-Cover Metric & Limitations](./forest-cover-metric.md) —
  this isn't a bug to fix, it's a scope boundary to know about before
  trusting what a milestone actually confirms.
- **No rate limiting protects any endpoint** — public reads
  (`/projects`, `/donations`) or the registration intake
  (`POST /projects/register`) alike. This is a known, un-mitigated gap in
  the current MVP, not a deliberate design choice the way the points
  above are.
- **The indexer's read model can lag.** Dashboard and explorer values
  come from a database the indexer updates on a poll interval
  (`INDEXER_POLL_INTERVAL_MS`), not directly from the chain — a
  just-confirmed transaction can take a few seconds to show up. This is a
  **data-freshness issue in the UI only**, never a fund-safety one: the
  contracts' own state is never affected by what the indexer does or
  doesn't record.

## Reporting a vulnerability

Use GitHub's private vulnerability reporting on the relevant repo (the
**Security** tab → **Report a vulnerability**) rather than a public issue,
for anything that could put funds or user data at risk. For the contracts
specifically, that's [canopychain-contracts](https://github.com/canopychain/canopychain-contracts).
