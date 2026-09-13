---
sidebar_position: 8.2
---

# Troubleshooting local development

[Local development](./local-development.md) ends with one line of advice
— check the backend's terminal output — which is right as far as it
goes, but doesn't say what you're looking for. These are the failure
modes someone actually hits working through that setup, in the order
they tend to show up.

## Port already in use

**Symptom**: `npm run dev` in `canopychain-backend` or
`canopychain-frontend` fails immediately with `EADDRINUSE` (or the
process starts but nothing answers on the port you expected).

The backend defaults to port `3000`; the frontend runs on `3001`
specifically because `3000` is taken by the backend — see
[Local development](./local-development.md#2-start-the-backend). If
either port is already held by a previous `npm run dev` you forgot to
stop, or by something else entirely, the new process can't bind it.

**Fix**: find and stop whatever's holding the port, or point the backend
at a different one — it reads `PORT` from `.env` (see the
[API reference](./api-reference.md)) — and update the frontend's
`NEXT_PUBLIC_API_URL` to match.

## An unset contract id

**Symptom**: registering, approving, or funding a project fails right
away — before any transaction even reaches the network — often with an
error about an invalid or missing contract address, rather than
something that looks like a rejected transaction.

Both apps need the same two contract ids from
[Deploying the contracts](./deploying-contracts.md), copied into two
different places under two different names — `PROJECT_REGISTRY_CONTRACT_ID`
/ `MILESTONE_VAULT_CONTRACT_ID` in the backend's `.env`, and
`NEXT_PUBLIC_PROJECT_REGISTRY_CONTRACT_ID` /
`NEXT_PUBLIC_MILESTONE_VAULT_CONTRACT_ID` in the frontend's. It's easy to
set one pair and forget the other, or to copy a stale id after
redeploying the contracts.

**Fix**: check both `.env` files against the same `deployments.json`
from your deploy step, and restart both `npm run dev` processes —
neither app picks up a changed `.env` while running.

## Postgres not yet accepting connections

**Symptom**: `npm run db:push` (or the backend's first `npm run dev`)
fails with a connection error — Prisma reporting it can't reach the
database, or a raw `ECONNREFUSED` — immediately after
`docker compose up -d postgres`.

`docker compose up -d` returns as soon as the container has *started*,
not once Postgres inside it is actually ready to accept connections,
which can take a couple of seconds on a cold volume. Running the next
command immediately after loses that race.

**Fix**: `docker compose logs postgres` and wait for "database system is
ready to accept connections" before running `db:push`, or just retry the
failed command once — by then the race has resolved itself.

## The indexer logs nothing

**Symptom**: the backend starts cleanly, `GET /health` returns
`{"status":"ok"}`, but nothing you do on-chain ever shows up in
`/dashboard` or any `GET /projects` response — and the backend's logs
have no indexer errors, or any indexer output at all.

This isn't the indexer failing — it's [deliberately a no-op](./deploying-contracts.md)
when `PROJECT_REGISTRY_CONTRACT_ID` and `MILESTONE_VAULT_CONTRACT_ID`
aren't both set, precisely so it doesn't spam errors trying to watch
contracts it doesn't have ids for. Silence here means "not configured,"
not "configured and stuck."

**Fix**: same as the unset-contract-id case above — set both ids in the
backend's `.env` and restart it. If both are set and it's still silent,
check they're the ids the indexer actually picked up (a value changed in
`.env` after the process started doesn't take effect until restart).

## Still stuck?

If none of these match, the backend's own terminal output is still the
first place to look — indexer and GFW-poll errors log there, never to
the frontend's console or terminal.
