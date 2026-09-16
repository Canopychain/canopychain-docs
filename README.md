# Canopychain — Docs

Documentation site for Canopychain, a milestone-verified reforestation
funding platform on Stellar. Built with [Docusaurus](https://docusaurus.io/).

## Local development

```
npm install
npm start
```

## Build

```
npm run build
```

Deploys automatically to GitHub Pages on every push to `main` (see
`.github/workflows/deploy.yml`) — but only once, one time, someone enables
it in the repo: **Settings → Pages → Source → GitHub Actions**. The
workflow won't publish anything until that's set.

## Versioning

Docusaurus's versioning is set up but not yet used — there's only ever
been one release, so there's nothing to freeze a snapshot of yet. Cut a
version once these docs would otherwise need to diverge for two audiences
at once — the clearest trigger is a mainnet deployment, where "current"
docs start describing mainnet but testnet-era docs are still worth
keeping around:

```
npm run version 1.0.0
```

This snapshots everything currently in `docs/` into `versioned_docs/` and
`versioned_sidebars/`, and adds `1.0.0` to `versions.json`. From then on,
`docs/` is always "next" (unreleased/in-progress) documentation, and the
versioned snapshot is what most readers see by default.

## Related repositories

- [canopychain-contracts](https://github.com/canopychain/canopychain-contracts) — Soroban smart contracts
- [canopychain-backend](https://github.com/canopychain/canopychain-backend) — indexer & API
- [canopychain-frontend](https://github.com/canopychain/canopychain-frontend) — donor & operator web app

## Status

Early development.

## Contributing

Issues and pull requests are welcome. [CONTRIBUTING.md](./CONTRIBUTING.md)
covers the toolchain and the checks CI runs, and
[CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md) covers how we expect people to
treat each other here.

Security vulnerabilities should be reported privately rather than in a public
issue. [SECURITY.md](./SECURITY.md) explains how.

## License

Apache-2.0 — see [LICENSE](./LICENSE).
