---
sidebar_position: 3
---

# Contracts reference

Two Soroban contracts, in [canopychain-contracts](https://github.com/canopychain/canopychain-contracts).
Every mutating function here requires the auth of whichever address is
named in its own doc comment below (checked via `require_auth`, not
inferred).

## project-registry

Owns one fact per project: whether it's approved to receive funding.

### Storage

| Type | Fields |
| --- | --- |
| `Project` | `operator: Address`, `recipient: Address`, `attestor: Address`, `polygon_hash: BytesN<32>`, `name: String`, `approved: bool` |

| `DataKey` variant | Points to |
| --- | --- |
| `Admin` | The registry admin `Address` (instance storage) |
| `NextProjectId` | The `u64` counter for the next project's id (instance storage) |
| `Project(u64)` | A `Project` record, keyed by its id (persistent storage) |

`polygon_hash` commits on-chain to the GPS-bounded plot boundary — the
actual GeoJSON polygon lives off-chain in the backend, keyed by this hash,
so the plot a project is scored against can't be silently swapped after
donors have funded it.

### Errors

| Code | Name | Meaning |
| --- | --- | --- |
| 1 | `AlreadyInitialized` | `init` called more than once |
| 2 | `NotInitialized` | Admin read before `init` |
| 3 | `ProjectNotFound` | No project with that id |

### Functions

```rust
fn init(env: Env, admin: Address) -> Result<(), Error>
```
Sets the admin and seeds the project-id counter. Once only.

```rust
fn admin(env: Env) -> Result<Address, Error>
fn get_project(env: Env, project_id: u64) -> Result<Project, Error>
```
Read-only. `get_project` returns the entry whether approved or not.

```rust
fn register(env: Env, operator: Address, recipient: Address, attestor: Address, polygon_hash: BytesN<32>, name: String) -> Result<u64, Error>
```
**Requires `operator`'s auth.** Creates an entry with `approved: false`
and returns its new id. Emits a `register` event — topics
`(Symbol("register"), project_id)`, data `(operator, name)`.

```rust
fn approve_project(env: Env, project_id: u64) -> Result<(), Error>
```
**Requires the admin's auth.** Flips `approved` to `true` on an existing
entry. Emits an `approved` event — topics `(Symbol("approved"), project_id)`,
no data.

## milestone-vault

Holds every project's pooled donor escrow. Notably, **this contract
doesn't check project-registry at all** — `deposit` opens a vault for any
project id, approved or not. [Deploying the contracts](./deploying-contracts.md#operational-note-the-registry-and-the-vault-dont-check-each-other)
covers why that's a deliberate choice and what it means operationally.

### Storage

| Type | Fields |
| --- | --- |
| `ProjectVault` | `recipient: Address`, `attestor: Address`, `token: Address`, `total_deposited: i128`, `total_released: i128`, `milestones_completed: u32`, `cancelled: bool` |
| `Milestone` | `threshold_bps: u32`, `payout_bps: u32` |

| `DataKey` variant | Points to |
| --- | --- |
| `Admin` | The vault admin `Address` (instance) |
| `Vault(u64)` | A `ProjectVault` record, keyed by project id (persistent) |
| `Donation(u64, Address)` | A donor's cumulative `i128` contribution to a project (persistent) |
| `Schedule(u64)` | A project's `Vec<Milestone>` release schedule (persistent) |
| `Paused` | `bool` — whether fund-moving actions are halted (instance) |

`Milestone.threshold_bps` is the cumulative forest-cover-change (in basis
points of plot area) the backend's satellite check must confirm before
the attestor will attest that milestone. It's recorded on-chain purely
for donor-facing transparency — **the contract has no way to verify
satellite data and does not check this value itself.** `payout_bps` is
the share of `total_deposited` released when that milestone is attested;
a schedule's `payout_bps` values must sum to at most 10,000 (100%), and
`threshold_bps` must strictly increase from one milestone to the next.

### Errors

| Code | Name | Meaning |
| --- | --- | --- |
| 1 | `AlreadyInitialized` | `init` called more than once |
| 2 | `NotInitialized` | Admin read before `init` |
| 3 | `VaultNotFound` | No vault open for that project id |
| 4 | `InvalidAmount` | A deposit amount was ≤ 0 |
| 5 | `TokenMismatch` | A deposit's token didn't match the vault's existing token |
| 6 | `VaultCancelled` | Called a fund-moving action on a cancelled vault |
| 7 | `ScheduleAlreadySet` | `configure_milestones` called twice for the same project |
| 8 | `InvalidSchedule` | Empty schedule, non-increasing thresholds, or payouts summing over 100% |
| 9 | `ScheduleNotFound` | `attest_milestone` called before a schedule was configured |
| 10 | `AllMilestonesComplete` | `attest_milestone` called with nothing left to attest |
| 11 | `ContractPaused` | Called a paused action while the vault is paused |
| 12 | `NotCancelled` | `refund` called on a project that hasn't been cancelled |
| 13 | `NothingToRefund` | Caller has no recorded donation left to refund |

### Tranche math

Every attestation pays out the same way:

```
payout = total_deposited * milestone.payout_bps / 10_000
```

computed against the pool's total-deposited-to-date, not a snapshot taken
when the schedule was configured — a donor who tops up after milestone 0
still contributes to milestone 1's payout. A refund after cancellation
splits whatever's left proportionally by each donor's own contribution:

```
refund = donation * (total_deposited - total_released) / total_deposited
```

so the order donors claim in never matters — the shares always add up to
exactly what's left in the pool.

### Functions

```rust
fn init(env: Env, admin: Address) -> Result<(), Error>
```
Sets the admin. Once only.

```rust
fn admin(env: Env) -> Result<Address, Error>
fn get_vault(env: Env, project_id: u64) -> Result<ProjectVault, Error>
fn get_donation(env: Env, project_id: u64, donor: Address) -> i128
fn get_schedule(env: Env, project_id: u64) -> Result<Vec<Milestone>, Error>
fn paused(env: Env) -> bool
```
Read-only.

```rust
fn pause(env: Env) -> Result<(), Error>
fn unpause(env: Env) -> Result<(), Error>
```
**Require the admin's auth.** Halt (or restore) `deposit` and
`attest_milestone`. `refund` is deliberately exempt — pausing stops new
activity, it never traps donor funds already in the vault. Emit `pause` /
`unpause` events (no data).

```rust
fn deposit(env: Env, donor: Address, project_id: u64, recipient: Address, attestor: Address, token: Address, amount: i128) -> Result<i128, Error>
```
**Requires `donor`'s auth.** The first deposit for a `project_id` opens
its vault, recording `recipient` and `attestor`; later deposits ignore
those two arguments and top up the existing vault instead, so they can't
redirect an already-funded project's payout or attestation rights.
Returns the vault's new `total_deposited`. Emits a `deposit` event —
topics `(Symbol("deposit"), project_id, donor)`, data
`(amount, total_deposited)`.

```rust
fn configure_milestones(env: Env, project_id: u64, milestones: Vec<Milestone>) -> Result<(), Error>
```
**Requires the admin's auth.** Sets a project's tranche schedule once —
the schedule donors funded against can't be quietly changed underneath
them afterward. Emits a `schedule` event — topics
`(Symbol("schedule"), project_id)`, data `milestones.len()`.

```rust
fn attest_milestone(env: Env, project_id: u64) -> Result<i128, Error>
```
**Requires the vault's `attestor`'s auth.** Releases the next tranche's
share of `total_deposited` to the recipient and advances
`milestones_completed` by one. Returns the payout amount released (which
may be `0` for a milestone configured with `payout_bps: 0`). Emits an
`attested` event — topics
`(Symbol("attested"), project_id, milestones_completed)`, data `payout`.

```rust
fn set_attestor(env: Env, project_id: u64, new_attestor: Address) -> Result<(), Error>
```
**Requires the admin's auth.** Rotates the attestor authorized to attest
milestones for a project, so a compromised or retiring attestor key can
be replaced without needing anything from donors or the recipient. Emits
an `attestor` event — topics `(Symbol("attestor"), project_id)`, no data.

```rust
fn cancel_project(env: Env, project_id: u64) -> Result<(), Error>
```
**Requires the admin's auth.** Marks a vault cancelled — for projects that
stall or fail to progress, not something a single donor can trigger
unilaterally against a pool other donors also contributed to. Blocks
further deposits and attestations; whatever hasn't been released becomes
claimable via `refund`. Emits a `cancelled` event — topics
`(Symbol("cancelled"), project_id)`, no data. See
[What happens to a stalled project](./stalled-projects.md) for when and how
this actually gets called.

```rust
fn refund(env: Env, project_id: u64, donor: Address) -> Result<i128, Error>
```
**Requires `donor`'s auth.** Pull-based — each donor claims their own
share of a cancelled project's remaining pool, rather than the contract
pushing to everyone at once, since there's no way to enumerate every donor
to a project in one call. Zeroes the donor's recorded donation on success,
so a second call has nothing left to pay. Emits a `refund` event — topics
`(Symbol("refund"), project_id, donor)`, data `refund_amount`.
