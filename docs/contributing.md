---
sidebar_position: 11
---

# Contributing

Canopychain is split across four repos — pick whichever matches what you
want to work on:

- **[canopychain-contracts](https://github.com/canopychain/canopychain-contracts)** — Rust, Soroban.
- **[canopychain-backend](https://github.com/canopychain/canopychain-backend)** — TypeScript, Fastify, Prisma.
- **[canopychain-frontend](https://github.com/canopychain/canopychain-frontend)** — TypeScript, Next.js, React.
- **canopychain-docs** — this site.

Each has its own README with setup instructions; [Local development](./local-development.md)
covers running all three app repos together.

## Issue labels

| Label | Meaning |
| --- | --- |
| `good first issue` | Self-contained, doesn't require prior context on the rest of the codebase, has a clear acceptance criterion. Start here. |
| `help wanted` | Open for anyone, not necessarily small. |
| `bug` | Something doesn't behave as documented or intended. |
| `enhancement` | A new capability or improvement, not a fix. |
| `docs` | This site, or README/inline documentation in any repo. |

Each repo also labels issues with which part of the system they touch
(`contracts`, `backend`, `frontend`) when it's not obvious from the repo
itself — useful mainly on cross-cutting issues.

## Stellar Wave

Canopychain's repos participate in [Drips Wave](https://www.drips.network/wave/stellar).
If you found this project through a Wave cycle, `good first issue` is the
label to filter on; point values are set per-issue by maintainers, not
something this page controls.

## Making a change

1. Fork the relevant repo, branch off `main`.
2. Match the existing commit style — a short, present-tense, conventional
   prefix (`feat:`, `fix:`, `docs:`, `test:`, `chore:`, `ci:`, `style:`)
   describing *why* the change matters, not a restatement of the diff.
3. Before opening a PR, run what that repo's own CI runs — lint, typecheck,
   test, build. Check the repo's `.github/workflows/ci.yml` if you're not
   sure what that is; it's the same commands CI will run against your PR.
4. Open the PR against `main`, describing what changed and why. Link the
   issue it closes if there is one.

## What reviewers look for

- Does it do what the issue asked, without quietly expanding scope?
- Does it match the surrounding code's existing patterns rather than
  introducing a new one for no reason?
- For contract changes: are there tests covering the new behavior,
  including the failure paths, not just the happy path?
- For backend/frontend changes: does it handle the case where the thing
  it depends on (the chain, the indexer, GFW, the wallet) is slow, down,
  or returns something unexpected — not just the case where everything
  works?
- For anything touching the milestone/attestation flow specifically: does
  it stay consistent with the semi-trusted-attestor model described in
  [The Milestone Attestation Model](./attestation-model.md), rather than
  quietly implying stronger trustlessness than the system actually has?

## What makes a good `good first issue`

If you're a maintainer labeling issues (including for Wave contributors):
a good one is scoped to one file or one clearly-bounded feature, states
its acceptance criteria explicitly rather than leaving them implicit, and
doesn't require understanding how the other three repos work to complete.
"Add a loading skeleton to the project explorer grid" is a good first
issue; "improve the indexer" is not.
