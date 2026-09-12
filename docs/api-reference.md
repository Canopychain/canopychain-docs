---
sidebar_position: 7
---

# Backend API reference

Base URL is whatever `PORT` [canopychain-backend](https://github.com/canopychain/canopychain-backend)
is running on (`http://localhost:3000` locally). All responses are JSON.
Every amount field (`totalDeposited`, `totalReleased`, `amount`,
`payoutAmount`) is a decimal **string**, not a number — these are `i128`
values on-chain and can exceed what a JS/JSON number can represent
exactly. `onChainId` is likewise a string, not a raw `BigInt`.

## Public, read-only

### `GET /health`

Pings the database. `{ "status": "ok" }` on success.

### `GET /projects`

Approved projects, newest first, capped at 100.

```json
[{
  "id": "...", "onChainId": "0", "operatorAddress": "G...", "name": "...",
  "approved": true, "cancelled": false,
  "totalDeposited": "1000000000", "totalReleased": "300000000",
  "createdAt": "...", "updatedAt": "..."
}]
```

### `GET /projects/:id`

One project's full profile: its milestone timeline plus summary stats.
`id` is the internal id from `GET /projects`, not the on-chain project id
or an operator address. `404` if it doesn't exist.

```json
{
  "id": "...", "onChainId": "0", "name": "...",
  "recipientAddress": "G...", "attestorAddress": "G...",
  "polygonGeoJson": { "type": "Polygon", "coordinates": [...] },
  "polygonHash": "...",
  "totalDeposited": "1000000000", "totalReleased": "300000000",
  "milestones": [
    { "id": "...", "index": 0, "thresholdBps": 500, "payoutBps": 3000, "status": "ATTESTED", "attestedAt": "...", "payoutAmount": "300000000" },
    { "id": "...", "index": 1, "thresholdBps": 1000, "payoutBps": 3000, "status": "PENDING", "attestedAt": null, "payoutAmount": null }
  ],
  "stats": { "donorCount": 2, "milestonesAttested": 1, "milestonesTotal": 3 }
}
```

`recipientAddress`, `attestorAddress`, `polygonGeoJson`, and
`polygonHash` are all `null` until the operator's registration intake
(`POST /projects/register`) attaches them — the on-chain `register` event
alone only carries the operator address and name. See
[Architecture](./architecture.md) for why the indexer and the intake
endpoint can each independently create this row, and
[The backend's data model](./data-model.md) for every field's exact
provenance.

### `GET /donations?donor=`

`donor` is required — a Stellar address (`^G[A-Z2-7]{55}$`). `400` if it
fails validation. A donor's own donations, newest first, capped at 100,
each with its project's summary attached.

```json
[{
  "id": "...", "amount": "600000000", "createdAt": "...",
  "project": {
    "id": "...", "onChainId": "0", "operatorAddress": "G...", "name": "...",
    "approved": true, "cancelled": false,
    "totalDeposited": "1000000000", "totalReleased": "300000000",
    "createdAt": "...", "updatedAt": "..."
  }
}]
```

## Public, write

### `POST /projects/register`

Off-chain intake — attaches a project's polygon and recipient/attestor
addresses to the on-chain project id an operator just got back from
calling `project-registry`'s `register`. Can arrive before or after the
indexer has processed that same `register` event; either order produces
the same final row (see [Architecture](./architecture.md)).

```json
// request body
{
  "onChainId": "0",
  "recipientAddress": "G...",
  "attestorAddress": "G...",
  "polygonHash": "...",
  "polygonGeoJson": { "type": "Polygon", "coordinates": [...] }
}
```

`201` with the resulting project row on success, `400` on validation
failure.

## Admin-only

These three require a signed request — see [Authentication](#authentication)
below. `503` if the backend has no `ADMIN_ADDRESS` configured; `401` if
the signature is missing, invalid, or stale.

### `GET /projects/pending`

Unapproved, non-cancelled projects — the review queue.

### `GET /projects/all`

Every project, regardless of status — the oversight view, for rotating an
attestor or cancelling a project that's already active, not just ones
still awaiting approval.

### `POST /projects/:id/reject`

Accepts an optional body `{ "reviewNote": "..." }` and returns the
updated project, now `cancelled: true`. There's no on-chain "reject" —
only "approve" — so this endpoint is the only place a project's rejection
is ever recorded; approving a project instead means an admin calling
`project-registry`'s `approve_project` directly (see the frontend's admin
panel), with this backend only ever mirroring that back via the indexer,
never triggering it.

## Authentication

Admin routes are gated by [SEP-53](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0053.md)
message signing, not a session or API key. To call one:

1. Build `payload = "${method}:${path}:${timestamp}"` — `path` is exactly
   what the server sees as the request path *and* query string (e.g.
   `/projects/pending`), with no scheme or host, and `timestamp` is the
   current time in epoch milliseconds.
2. Sign `payload` with your wallet's generic message-signing call (not
   transaction signing) — the wallet applies the SEP-53 prefix and SHA-256
   hash itself.
3. Send three headers: `x-admin-address` (your public key),
   `x-admin-signature` (the signature, base64), `x-admin-timestamp` (the
   same timestamp used above, as a string).

The server rejects the request if `x-admin-address` doesn't match the
configured `ADMIN_ADDRESS`, if the timestamp is more than 5 minutes old
(bounding replay of a captured header set), or if the signature doesn't
verify. See `canopychain-frontend`'s `src/lib/adminApi.ts` for a
complete, working implementation of this flow.
