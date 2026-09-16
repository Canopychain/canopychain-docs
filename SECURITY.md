# Security Policy

This repository holds documentation rather than running code, so it does
not handle funds or keys directly. It can still cause harm: this site is
where operators and donors are told which contract ids to trust and how the
attestation model works, so wrong or tampered content here can lead someone
to trust the wrong thing.

## Scope

This policy covers the Docusaurus site in this repository, its build, and
its deployment workflow.

If you've found a vulnerability in the software itself rather than its
documentation, report it against the repository it lives in:

- [canopychain-contracts](https://github.com/Canopychain/canopychain-contracts) for the Soroban contracts.
- [canopychain-backend](https://github.com/Canopychain/canopychain-backend) for the API, indexer and attestor.
- [canopychain-frontend](https://github.com/Canopychain/canopychain-frontend) for the web app.

## Reporting a vulnerability

**Do not open a public GitHub issue for a security vulnerability.**

Instead, use GitHub's private vulnerability reporting:

1. Go to the [Security tab](https://github.com/Canopychain/canopychain-docs/security) of this repository.
2. Click "Report a vulnerability" to open a private advisory.
3. Describe the issue, including the affected page or workflow and the
   potential impact.

If you're unable to use GitHub's private reporting for any reason, contact
a maintainer directly rather than filing a public issue.

## What to expect

- We'll acknowledge new reports as soon as we can and work with you to
  understand and confirm the issue.
- We'll aim to keep you updated as a fix is developed and let you know
  before any public disclosure.
- Please give us a reasonable amount of time to address the issue before
  disclosing it publicly.

## What qualifies

Examples of in-scope issues:

- A documented contract id, address or endpoint that is wrong, so readers
  would be directed at something other than the real deployment.
- Script injection through page content or a theme configuration that would
  execute in a reader's browser.
- A weakness in the deploy workflow that would let someone publish content
  to the site without review.
- Documented instructions that would lead an operator or admin to expose a
  secret key, for example by putting one somewhere it would be committed or
  logged.

Out of scope: typos and factual errors with no security consequence, which
are welcome as ordinary pull requests or issues.
