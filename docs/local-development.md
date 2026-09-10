---
sidebar_position: 8
---

# Local development

Each repo documents its own setup in its own README; this page is the
order to actually run them in, end to end, with one instance of everything
talking to the others correctly.

Clone all four repos as siblings — the paths below assume that layout:

```
Canopychain/
├── canopychain-contracts/
├── canopychain-backend/
├── canopychain-frontend/
└── canopychain-docs/
```

## 1. Deploy the contracts (once)

Follow [Deploying the contracts](./deploying-contracts.md) to get both
contracts onto testnet. Keep the resulting `deployments.json` — its two
contract IDs feed both of the next two steps. You don't need to repeat
this for every local session, only when you don't have a deployment yet
or want a fresh one.

## 2. Start the backend

```bash
cd canopychain-backend
cp .env.example .env
docker compose up -d postgres   # also creates a canopychain_test DB
npm install
```

Edit `.env`: set `PROJECT_REGISTRY_CONTRACT_ID` and
`MILESTONE_VAULT_CONTRACT_ID` from step 1's `deployments.json`,
`ADMIN_ADDRESS` to whichever Stellar address you want to act as platform
admin (it doesn't have to be the contracts' deploy admin, but it's
simplest if it is, locally), `GFW_API_KEY` (see
[Global Forest Watch's own API docs](https://developer.openepi.io/how-tos/getting-started-using-global-forest-watch-data-api)
for getting one), and `ATTESTOR_SECRET_KEY` to a funded testnet identity's
secret key — this is the key that will actually sign attestations, so it
needs to be separate from your admin identity.

```bash
npm run db:push
npm run dev
```

The API is now at `http://localhost:3000`. `GET /health` should return
`{"status":"ok"}`.

## 3. Start the frontend

```bash
cd canopychain-frontend
cp .env.example .env
npm install
```

Edit `.env`: `NEXT_PUBLIC_API_URL` should already default to
`http://localhost:3000`; set `NEXT_PUBLIC_PROJECT_REGISTRY_CONTRACT_ID`
and `NEXT_PUBLIC_MILESTONE_VAULT_CONTRACT_ID` to the same two ids from
step 1.

```bash
npm run dev
```

The app is now at `http://localhost:3001` (3000 is taken by the backend).

## 4. Smoke test

1. Visit `http://localhost:3001`, connect a testnet-funded wallet.
2. Visit `/register` and register a project — a name, recipient/attestor
   addresses, and a small GeoJSON polygon (draw one at
   [geojson.io](https://geojson.io) and download it).
3. Approve it from `/admin` — connected as the `ADMIN_ADDRESS` you set in
   step 2 — which calls `project-registry`'s `approve_project` directly.
4. The newly approved project should now show up on `/projects`. Fund it
   from its detail page, then check `/dashboard` — the indexer polls on
   an interval (`INDEXER_POLL_INTERVAL_MS`, default 5s), so a
   just-confirmed action can take a few seconds to show up.
5. Attestation is the one step that doesn't happen on a smoke-testable
   timescale by default: the GFW polling worker checks each active
   project on `GFW_POLL_INTERVAL_MS`, which defaults to six hours.
   Configuring your project's plot to match an already-known, real-world
   deforestation or reforestation site — or temporarily lowering that
   interval — is the practical way to see an attestation happen locally
   without waiting.

If something doesn't show up at all rather than just lagging, check the
backend's own terminal output first — indexer and GFW-poll errors log
there, not to the frontend.
