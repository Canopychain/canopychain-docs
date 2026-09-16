# Contributing

This file covers the toolchain and checks for working in this repository.
The full contribution guide for the Canopychain project lives on this very
site, at [Contributing](https://canopychain.github.io/canopychain-docs/contributing);
its source is [docs/contributing.md](./docs/contributing.md).

## Toolchain

- Node.js 20 or newer, and npm.

## Getting set up

```sh
npm install
npm start
```

That serves the site locally with hot reload.

## Checks to run before opening a PR

CI (`.github/workflows/ci.yml`) builds the site on every pull request. Run
the same build locally first, because problems that hot reload tolerates
will still fail it:

```sh
npm run build
```

A Docusaurus build fails on broken internal links and reports broken
anchors, so a heading you renamed can break a link on a page you didn't
touch. The build output names both, and it's worth reading even when it
succeeds.

## Writing

- One page per topic, listed in `sidebars.ts`. A page that isn't in the
  sidebar is reachable only by URL.
- Link between pages with relative paths ending in `.md`, so Docusaurus can
  resolve and check them.
- Diagrams are Mermaid, rendered by `@docusaurus/theme-mermaid`. A fenced
  code block tagged `mermaid` renders directly in any page.
- Prefer describing behaviour that's actually implemented. Where something
  is planned rather than built, say so explicitly, and put it on the
  roadmap page.

## Deployment

Pushes to `main` deploy to GitHub Pages via
`.github/workflows/deploy.yml`. The workflow only publishes once someone
has enabled Pages for the repository under
Settings, then Pages, then Source, then GitHub Actions.

## Security

Please don't open a public issue for a security vulnerability. See
[SECURITY.md](./SECURITY.md) for how to report one privately.
