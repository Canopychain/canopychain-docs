---
sidebar_position: 8.5
---

# Full-stack deployment guide

[Deploying the contracts](./deploying-contracts.md) covers the contracts in
isolation, and [Local development](./local-development.md) covers running
everything on one laptop against testnet. Neither says what it takes to run
`canopychain-backend` and `canopychain-frontend` as a real, standing
deployment — what order to bring them up in, which values have to flow from
one piece to the next, and what has to be true before the first donor
arrives. This page is that missing middle step.

## The order

1. **Contracts** — deployed once, produce two contract IDs everything else
   depends on.
2. **Backend** — needs those two contract IDs, a Postgres database, and (to
   do anything beyond serve empty lists) a GFW API key and a funded attestor
   identity.
3. **Frontend** — needs the same two contract IDs, plus the backend's public
   URL.

Each step's output feeds the next; there's no way to usefully skip ahead.

## 1. Deploy the contracts

Follow [Deploying the contracts](./deploying-contracts.md). For a real
deployment you almost always want [Mainnet](./deploying-contracts.md#mainnet),
not testnet — read the [contracts reference](./contracts.md) for what
`init`, `pause`, `set_attestor`, and `cancel_project` do before running any
of them against real funds.

Keep the resulting `deployments.json` — you'll copy both contract IDs
(`project-registry`, `milestone-vault`) into both of the next two steps.
Also decide now who holds:

- The **contracts' admin identity** — whoever ran `init` on both contracts.
  Can call `approve_project`, `pause`, `configure_milestones`,
  `set_attestor`, and `cancel_project`.
- The **attestor identity** — a separate funded account whose secret key
  becomes the backend's `ATTESTOR_SECRET_KEY`. Keep this distinct from the
  admin identity; see [The Milestone Attestation Model](./attestation-model.md).

## 2. Deploy the backend

`canopychain-backend` ships a multi-stage Dockerfile (build TypeScript and
the generated Prisma client, then a slim non-root runtime image running
`node dist/index.js` on port `3000`) — build and run that image on whatever
container platform you use, pointed at a Postgres instance it can reach.

### Environment

Every variable is documented in the repo's own `ENVIRONMENT.md`; the ones
that matter specifically for a production deploy (rather than a laptop
against testnet):

| Variable | Production value |
| --- | --- |
| `DATABASE_URL` | Your production Postgres connection string — not the `docker-compose.yml` default. |
| `SOROBAN_RPC_URL` / `SOROBAN_NETWORK_PASSPHRASE` | A mainnet RPC endpoint and mainnet's passphrase, if you deployed the contracts to mainnet in step 1 — the testnet defaults will sign transactions the wrong network rejects. |
| `PROJECT_REGISTRY_CONTRACT_ID` / `MILESTONE_VAULT_CONTRACT_ID` | The two IDs from step 1's `deployments.json`. The indexer no-ops until both are set. |
| `ATTESTOR_SECRET_KEY` | The attestor identity's secret key from step 1. This is the one private key this whole system holds server-side — treat it like the production secret it is (a secrets manager, not a plain env file on disk). |
| `GFW_API_KEY` | A real [Global Forest Watch](https://developer.openepi.io/how-tos/getting-started-using-global-forest-watch-data-api) key — without it the polling worker's queries fail silently and no milestone ever attests. |
| `ADMIN_ADDRESS` | The Stellar address that must match the identity you'll use to sign admin requests — see the note on the three admins below. |
| `NODE_ENV` | The Docker image sets this to `production` itself; don't override it. |

### Database schema

The production image deliberately omits the Prisma CLI (only the generated
client ships in the runtime stage), so there's no `prisma db push` to run
*inside* the container. Sync the schema from a machine that has the repo and
`npm install` run, pointed at the production `DATABASE_URL`, before starting
the backend for the first time:

```bash
DATABASE_URL="<production connection string>" npm run db:push
```

Re-run this after pulling any release that changes `prisma/schema.prisma`,
before rolling out the new image.

### Health check

Once running, `GET /health` should return `{"status":"ok"}` — it pings the
database, so a failing health check almost always means `DATABASE_URL` or
network access to Postgres, not the app itself.

## 3. Deploy the frontend

`canopychain-frontend`'s own README covers two supported paths:

- **Vercel** — connect the repo, set the `NEXT_PUBLIC_*` variables below in
  the project settings, deploy. No Dockerfile involved.
- **Docker** — the repo's Dockerfile builds Next's `standalone` output and
  runs it as a non-root user on port `3001`.

Either way, all five `NEXT_PUBLIC_*` variables are **inlined into the
JavaScript bundle at build time**, not read at container startup — changing
one after the fact means rebuilding (a new Vercel deploy, or a new Docker
image), not just restarting with a different `--env-file` or updating the
platform's env settings in place.

| Variable | Production value |
| --- | --- |
| `NEXT_PUBLIC_API_URL` | The backend's real deployed origin from step 2 — not `http://localhost:3000`. |
| `NEXT_PUBLIC_SOROBAN_RPC_URL` / `NEXT_PUBLIC_NETWORK_PASSPHRASE` | Must match whatever network you deployed the contracts to in step 1, exactly — a mismatch here gets transactions rejected by the RPC endpoint, not silently sent to the wrong place. |
| `NEXT_PUBLIC_PROJECT_REGISTRY_CONTRACT_ID` / `NEXT_PUBLIC_MILESTONE_VAULT_CONTRACT_ID` | The same two IDs from step 1's `deployments.json`. |

## Values that flow across all three pieces

- **Both contract IDs** flow from step 1's `deployments.json` into the
  backend's `.env` and the frontend's `NEXT_PUBLIC_*` vars unchanged. A
  mismatch between what the frontend and backend point at (for example, a
  frontend redeployed against a fresh set of contracts while the backend
  still points at the old ones) means the dashboard and the contracts
  disagree about which project is which.
- **The registered attestor address must match `ATTESTOR_SECRET_KEY`'s
  public key.** `milestone-vault`'s `attestor` for a project isn't set by
  the backend's config directly — it's set by whichever `deposit` call
  happens first for that project, using the `attestor` address the
  frontend read from the project's own registration (the operator's
  `attestorAddress`, attached via `POST /projects/register` — see
  [The backend's data model](./data-model.md)). In a typical single-operator
  deployment, that registered value should simply be the backend's own
  attestor's public key; if it isn't, `attest_milestone` will keep failing
  auth against a key the backend doesn't hold, with nothing in the backend's
  own logs to explain why beyond a failed transaction.
- **"Admin" is three separate configuration values, not one** — see the
  [glossary](./glossary.md#canopychain-specific): `project-registry`'s admin,
  `milestone-vault`'s admin, and the backend's `ADMIN_ADDRESS`. Nothing
  technical forces them to be the same address, but a deployment that splits
  them across different keys should do so deliberately, not by accident of
  who ran which command.

## A gotcha specific to running frontend and backend on different origins

Most of the frontend's reads (`/projects`, a project's detail page,
`/donations`) run **server-side**, inside Next.js's own request handling —
there's no browser involved, so cross-origin restrictions don't apply.

Three flows are different: registering a project
(`POST /projects/register`), and every admin action (`GET /projects/pending`,
`GET /projects/all`, `POST /projects/:id/reject`) run **in the donor's or
admin's own browser**, because they need a wallet signature the server
can't produce. `canopychain-backend` doesn't set any CORS headers today — if
the frontend and backend are deployed on different origins (the common case:
frontend on Vercel, backend on its own host), those specific browser-side
requests will fail with a CORS error even though every server-rendered page
loads fine.

Until the backend adds CORS support itself, the practical fix is to avoid
the cross-origin browser call entirely — put both services behind one
public origin (for example, a reverse-proxy rule that routes
`yourdomain.com/api/*` to the backend, with `NEXT_PUBLIC_API_URL` set to
`https://yourdomain.com/api`) rather than pointing `NEXT_PUBLIC_API_URL` at
a separate host directly.

## Before the first donor arrives

- `GET /health` returns `{"status":"ok"}`.
- The backend's logs show the indexer picking up events (register a test
  project and confirm it shows up via `GET /projects/pending`, signed as the
  configured `ADMIN_ADDRESS`).
- At least one project has been approved (`approve_project` on-chain) and
  given a milestone schedule (`configure_milestones` — see the
  [operator guide](./operator-guide.md#4-youre-listed--set-a-milestone-schedule)),
  since an approved project with no schedule can accept deposits but can
  never attest.
- A test deposit against that project, from the deployed frontend, shows up
  in `GET /projects/:id` within one indexer poll interval.
- The GFW polling worker's logs show a successful check against that
  project's real polygon — confirming `GFW_API_KEY` actually works, not just
  that it's set.
