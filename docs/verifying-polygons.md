---
sidebar_position: 10.7
---

# Verifying a project's polygon yourself

`project-registry` commits to a project's plot boundary as a 32-byte hash,
specifically so a donor doesn't have to take the backend's word for which
plot a project is actually being scored against — see
[Contracts reference](./contracts.md#storage) and
[The Milestone Attestation Model](./attestation-model.md#what-this-means-for-a-donor).
That guarantee only means something if you can actually perform the check.
This page is that check, step by step.

## What you're checking

The frontend computes a project's `polygon_hash` at registration time as
SHA-256 of the polygon geometry's canonical `JSON.stringify` form —
[`hashPolygon` in `canopychain-frontend`](https://github.com/canopychain/canopychain-frontend/blob/main/src/lib/polygon.ts):

```ts
async function hashPolygon(geometry: PolygonGeometry): Promise<Uint8Array> {
  const canonical = JSON.stringify(geometry);
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(canonical));
  return new Uint8Array(digest);
}
```

`geometry` here is a plain `{ type, coordinates }` object — not a full
GeoJSON `Feature`, and not the file the operator originally uploaded
verbatim if that file had extra whitespace or fields. This isn't a
general-purpose canonical-GeoJSON scheme (a different key order or added
whitespace hashes differently); it doesn't need to be, since this one
function is the only place that ever produces the hash an operator submits.
That also means your own recomputation has to match it byte for byte, not
just represent "the same polygon."

Verifying a project means recomputing that same hash from the same two
fields, and comparing it against the value actually committed on
`project-registry` — not against the backend's own copy of the hash, which
proves nothing beyond the backend being internally consistent with itself.

## 1. Get the project's on-chain id and its GeoJSON

```bash
curl -s https://<backend-host>/projects/<id> | jq '{onChainId, polygonGeoJson}'
```

`polygonGeoJson` is exactly the `{ type, coordinates }` shape `hashPolygon`
hashes — see [the API reference](./api-reference.md#get-projectsid) and
[the data model](./data-model.md#projects) for where this field comes from
(the operator's registration intake, not the on-chain `register` event).

## 2. Recompute the hash

The important part is reproducing `JSON.stringify`'s exact output — compact,
no extra whitespace, keys in `type, coordinates` order — before hashing.
In Node:

```bash
curl -s https://<backend-host>/projects/<id> | node -e '
  const data = JSON.parse(require("fs").readFileSync(0, "utf8"));
  const geometry = { type: data.polygonGeoJson.type, coordinates: data.polygonGeoJson.coordinates };
  const canonical = JSON.stringify(geometry);
  console.log(require("crypto").createHash("sha256").update(canonical).digest("hex"));
'
```

Or in a browser console, against the same response body, using the exact
function above (`crypto.subtle.digest` is available in any modern browser
without importing anything). Either way you should get a 64-character hex
string.

## 3. Read the hash actually committed on-chain

Not the backend's `polygonHash` field — `project-registry`'s own
`get_project`, which is what the contract itself stores:

```bash
stellar contract invoke \
  --id <PROJECT_REGISTRY_CONTRACT_ID> \
  --network testnet \
  --source-account <any-funded-identity> \
  -- \
  get_project --project_id <onChainId>
```

This is a read-only call — it simulates rather than submits, so any funded
identity works, including one that has nothing to do with the project. The
result includes `polygon_hash` as a hex-encoded 32-byte value. See
[Deploying the contracts](./deploying-contracts.md) for `PROJECT_REGISTRY_CONTRACT_ID`
if you don't already have it for the instance you're checking.

## 4. Compare

The hex string from step 2 should equal `polygon_hash` from step 3
(case-insensitive — hex casing isn't meaningful). If it doesn't, one of two
things is true: the backend is serving different GeoJSON than what the
operator actually committed to at registration, or the hash you recomputed
doesn't match the exact canonical form for some other reason (double-check
you hashed only `{ type, coordinates }`, with no added whitespace, and
that you're comparing against the right project's `onChainId`, not its
internal `id`).

Either way, this comparison never has to trust the backend for anything but
serving you *some* GeoJSON to check — the value it's checked against comes
straight from the contract.
