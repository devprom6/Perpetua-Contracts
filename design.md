# Design — multisig-threshold hardening (#434, #432, #430) and oracle-router (#423)

Status: **Draft** · Date: 2026-11-19 · Author: Kiro
Related issues: #434 (refactor error types), #432 (Byzantine fault-tolerance tests),
#430 (granular RBAC), #423 (oracle-router BFT tests).

---

## 0. TL;DR

- The four issues name contracts that **do not exist at the paths they cite**.
  `contracts/multisig-threshold` and `contracts/oracle-router` are absent on both
  `main` and `origin/issues-98-102-107-101-security-docs-sim`, confirmed by
  directory listing, recursive filename search, and `git ls-tree`.
- `contracts/governance` **is** the threshold multisig the "multisig-threshold"
  issues describe (README: "timelocked multi-sig"; code: `Threshold`,
  `MAX_SIGNERS`, quorum, timelock, emergency guardians). Issues #434/#432/#430
  are mapped onto `governance`.
- `oracle-router` (#423) has **no analog** in this repository and is **out of
  scope** for this design. It would require a net-new contract, which is a
  separate decision (see §6).
- This document is design-only, per the user's direction. Implementation and PR
  submission are intentionally not included here.

## 1. Findings (read before reviewing the rest)

### 1.1 The named targets are absent

`contracts/` contains only:

```
archival-probe   factory   governance   stream   stream-nft
```

No `multisig*` or `oracle*` path exists anywhere in the tree (recursive `find`,
both branches). The README's repository-layout section enumerates the same five
contracts and its non-goals list nothing resembling a multisig-threshold module
or an oracle router. So the issue paths are not a rename hiding in another
module; the contracts simply are not part of Perpetua-Contracts.

### 1.2 `multisig-threshold` ≅ `governance`

`contracts/governance` (`fluxora-governance`, v1.0.0) is a timelocked threshold
multisig for protocol parameters. The mapping is exact on every axis the issues
imply:

| Issue concept | Governance implementation |
|---|---|
| threshold multisig | `DataKey::Threshold`, `1 <= threshold <= signers.len()`, `MAX_SIGNERS = 20` |
| co-signers | `DataKey::Signers`, `DataKey::SignerIndex` (O(1) membership), per-signer `SignerWeights` (#48) |
| proposal lifecycle | `ProposalStatus` {Proposed, Approved, Queued, Executed, Cancelled}; `propose`/`approve`/`execute`/`cancel_proposal` |
| timelock | `GOVERNANCE_TIMELOCK_SECONDS = 172_800` (48h); `eta = quorum_reached_at + timelock` |
| grace / expiry | `DEFAULT_GRACE_PERIOD_SECONDS = 2_419_200`; `MAX_PROPOSAL_AGE_SECONDS = 2_592_000` |
| emergency cancel | `DataKey::EmergencyGuardians` (bounded by `MAX_SIGNERS`) |
| batch proposals | `CallData::Batch(Vec<ExecutionCall>)`, `MAX_BATCH_CALLS = 5` (#51) |
| reentrancy guard | `DataKey::Executing` + CEI in `execute` |
| retention | #52 policy: active vs terminal proposal storage TTL |

**Decision:** treat #434, #432, #430 as applying to `contracts/governance`.
This is an assumption that should be confirmed before implementation, but it is
the only interpretation consistent with the issues' "refactor"/"code review"
wording (a non-existent contract cannot be refactored or reviewed).

### 1.3 `oracle-router` has no home

Nothing in the repo reads external prices, aggregates feeds, or routes
off-chain data. The factory is a policy gate (cap/duration/rate/allowlist/pause);
the stream is the product; governance is parameter tuning. #423 cannot be
mapped to existing code. See §6.

### 1.4 The issues are templated

All four share boilerplate ("contract security, gas efficiency, and robustness
in the [X] subsystem", "Soroban SDK best practices, ledger footprint
optimization, and formal verification standards", and an identical Requirements
block). Combined with §1.1, this is consistent with auto-generated issues. The
design below treats them as a *brief* and fills in the specifics the brief omits;
it does not assume the issues' wording is precise.

## 2. Scope

**In scope** (this design):

- #434 — modularize/clarify error types + custom error messages for client
  observability, applied to `governance`.
- #430 — granular role-based authorization to prevent unauthorized configuration
  updates, applied to `governance`.
- #432 — Byzantine fault-tolerance test scenarios under simulated network
  partitions, applied to `governance`.

**Out of scope:**

- #423 (oracle-router). No target contract exists. §6.
- Any change to `stream`, `factory`, `archival-probe`, or `stream-nft`.
- New deployable contracts, upgrade paths, or fee mechanisms (README non-goals).
- Changing existing `GovernanceError` discriminant values — they are ABI.
- Submitting a PR (paused per user direction; see §8 for the blocker anyway).

## 3. Constraints that bound every option below

1. **Error discriminants are ABI.** `contracts/stream/src/error.rs` states the
   idiom explicitly: *"Discriminants are part of the public ABI. Never renumber
   an existing variant; only append."* `GovernanceError` follows the same idiom
   (`#[contracterror] #[repr(u32)]` with explicit codes 1–27, including a
   *documented* renumbering of `InvalidProposalState` off `21` to `26` because
   `21` collided with `TimelockNotMet`). Any "modularize" change must be
   **append-only and non-renumbering**. Splitting `GovernanceError` into
   sub-enums is ABI-breaking and is rejected.
2. **The repo's error idiom is a single flat, section-commented enum**, not a
   hierarchy. "Modularize" in this codebase means *logical sectioning with clear
   doc comments* — which `GovernanceError` already largely has. The real
   observability gap is runtime messages, not enum shape (see §4.1).
3. **Soroban has no scheduler and the SDK test host has no network.** "Network
   partition" cannot be simulated at a transport layer; it must be modeled as
   *signer availability across ledgers* (§5).
4. **Immutability is a product guarantee** (README: no admin key, no upgrade
   path for the product). `governance` is the *config* layer, so it intentionally
   *does* have an admin — but adding RBAC must not introduce a backdoor that
   bypasses the timelock/threshold proposal flow for parameter changes.
5. **Gas/footprint discipline.** Per-call cost is a first-class concern
   (governance already documents `MAX_PAGE_SIZE = 100` with a DoS rationale).
   New storage keys or per-call scans must be bounded and motivated.

## 4. Design: #434 — error types and client observability

### 4.1 What "modularize" means here (and what it does not)

The issue asks to "modularize error types and custom error messages for enhanced
client observability." Two readings; only one is safe:

- **(rejected) Split `GovernanceError` into sub-enums** (`AuthError`,
  `ProposalError`, `ConfigError`, …). Breaks Constraint 1: it renumbers the
  discriminant space that indexers and the TypeScript SDK already decode against.
  The stream contract — the project's reference for error design — deliberately
  keeps a *single* flat enum and forbids renumbering. Governance should match.
- **(accepted) Keep one flat enum; sharpen the grouping and the messages.**
  `GovernanceError` is already ordered logically (init → auth → proposal →
  calldata → arithmetic → batch). The actionable work is **observability**: a
   client that calls a view and gets back discriminant `16` should be able to
   resolve it to a human-readable name *without* the contract ABI in hand.

Note: `#[contracterror]` already emits variant names into the contract spec JSON,
so SDK clients that ingest the ABI *can* already map code→name. The marginal
value of a runtime catalog is for dashboards, explorers, and ad-hoc
`soroban-sdk` simulations that do not load the ABI, plus a single source of
truth for the human-readable string.

### 4.2 Recommended change (additive, ABI-safe)

1. **Do not alter `GovernanceError` discriminants.** Optionally append new
   variants only if #430/#432 require them (e.g., a role error) — appended at
   the end (28+), never inserted.
2. **Add a read-only error catalog.** Two viable shapes; pick one:

   a. `pub fn error_name(error: GovernanceError) -> Symbol` — maps each variant
      to a short `Symbol` (e.g. `Unauthorized -> "Unauthorized"`). Cheapest in
      footprint (a `match`, no storage). The return is a `Symbol`, which is
      what the host already serializes cheaply and what the SDK decodes as a
      string.

   b. A `#[contracttype]` `ErrorEntry { code: u32, name: Symbol, category:
      Symbol }` plus a `pub fn error_catalog() -> Vec<ErrorEntry>` returning the
      full table in one call. Larger event/return budget but lets a dashboard
      enumerate every code in a single simulation. Bounded: 27 entries today.

   Recommendation: **(a)** as the primary, with the catalog exposed as a
   `Vec<ErrorEntry>` view only if a concrete client requests enumeration. (a)
   is the smaller blast radius and matches "client observability" for the
   common case (decode one error). Add `category` to the `Symbol`-returning form
   by returning e.g. `"Auth/Unauthorized"` so a client gets grouping for free
   without a second type.

3. **Group the existing variants with section comments** mirroring `stream`'s
   idiom (`// --- Auth ---`, `// --- Proposal lifecycle ---`, `// --- Calldata
   ---`, `// --- Arithmetic ---`, `// --- Batch ---`). This is the "modularize"
   half that is genuinely a no-op on the wire but improves reviewability and
   matches the repo's own pattern. It touches only comments.

4. **Keep `ArithmeticOverflow` as an alias** the way `stream` does
   (`pub const ArithmeticOverflow: Error = Error::Overflow;`). Governance
   already names its variant `ArithmeticOverflow` directly, so no alias is
   needed — called out only so the implementer does not "fix" this
   inconsistently.

### 4.3 Test plan (#434)

- A test that, for every variant in `GovernanceError`, `error_name(v)` returns a
  non-empty `Symbol` (exhaustive over the enum — a `match` with no `_` arm forces
  the compiler to keep this exhaustive when variants are appended later).
- A round-trip test: each code round-trips through `error_name` and a future
  `error_from_name` inverse (if (b) is chosen) without collision.
- A "no discriminant changed" guard test: assert the numeric value of a sample
  of variants (e.g. `Unauthorized = 3`, `BatchTooLarge = 27`) equals the
  documented constant, so a future renumbering fails CI. Mirrors the spirit of
  `stream`'s ABI-freeze comment.

## 5. Design: #432 — Byzantine fault-tolerance tests

### 5.1 What "network partition" can mean on Soroban

There is no network layer in the SDK test host, and a contract cannot observe
one on chain. A partition is modeled purely as **which signers can submit
authenticated transactions in a given ledger window** — i.e., which
`require_auth` calls succeed. The contract's safety properties under a partition
reduce to properties of quorum, snapshots, expiry, and the reentrancy guard that
already exist in code. So #432 is really: *targeted tests that exercise those
properties under signer-subset availability*, not a transport simulation.

### 5.2 Mapped BFT concerns → existing mechanisms

| Byzantine concern | Existing mechanism | What the test must show |
|---|---|---|
| Double-vote / equivocation | per-proposal `ProposalApprovalIdx` → `AlreadyApproved` | a signer approving twice is rejected; quorum is never reached by replaying one signer |
| Signer unavailability (partition) | `MAX_PROPOSAL_AGE_SECONDS` → `ProposalExpired` | a proposal that cannot reach quorum before max-age expires, not hangs |
| Quorum reached in a partition minority | `Threshold` is a count, not a ratio — minority < threshold cannot queue | a 2-of-3 set split 1|2 cannot queue from the 1 side |
| Admin censoring via threshold bump | `QuorumInfo::threshold` snapshot at quorum; `SignerGeneration` invalidates stale proposals | an admin `set_threshold` after quorum does not retro-actively block or enable an in-flight proposal |
| Emergency guardian over-reach | `EmergencyGuardians` is a fixed, bounded set; cancel is the only power | a non-guardian cannot cancel; a guardian cannot execute or alter config |
| Reentrant execution during partition heal | `DataKey::Executing` + CEI in `execute` | a target that calls back into `execute` is rejected with `ReentrancyGuard` |
| Timelock bypass on heal | `eta` + `TimelockNotMet` | executing immediately after quorum (before `eta`) is rejected even if the partition just healed |
| Grace-window expiry | `DEFAULT_GRACE_PERIOD_SECONDS` | a proposal past `eta + grace` returns `ProposalExpired`/`InvalidProposalState` on execute |

### 5.3 Recommended test module

Add `#[cfg(test)] mod test_byzantine` inside `contracts/governance/src/lib.rs`
(the crate keeps all tests inline — `contracts/governance/src/` has only
`lib.rs`). Scenarios, each named for the property:

1. `partition_minority_cannot_queue` — 3 signers, threshold 2; only the isolated
   signer approves; assert status stays `Proposed`/`Approved`, never `Queued`.
2. `partition_quorum_then_heal_executes` — quorum reached on the majority side,
   timelock elapses, the isolated signer rejoins and executes; assert success
   and exactly one `proposal_executed` event.
3. `no_quorum_proposal_expires_under_partition` — quorum never reached; ledger
   advanced past `MAX_PROPOSAL_AGE_SECONDS`; `approve` and `execute` both return
   `ProposalExpired`; no state transition to `Queued`.
4. `equivocation_rejected` — same signer approves twice; second returns
   `AlreadyApproved`; weight counted once.
5. `threshold_change_mid_flight_does_not_alter_quorum` — proposal reaches
   quorum at threshold 2; admin bumps to 3; `execute` still succeeds using the
   `QuorumInfo::threshold` snapshot.
6. `signer_generation_invalidates_stale_proposals` — proposal created under
   generation G; admin adds/removes a signer (bumps to G+1); behavior of the
   stale proposal follows the existing `InvalidSignerGeneration` semantics.
7. `emergency_guardian_cancel_during_partition` — majority unreachable, proposal
   stuck; a guardian cancels; assert `Cancelled`, terminal retention applied, no
   later approve/execute.
8. `reentrant_execute_rejected` — a malicious target that re-enters `execute`
   (same or other proposal) returns `ReentrancyGuard`; original dispatch unwinds.
9. `timelock_not_met_on_heal` — quorum reached, partition heals, `execute`
   called before `eta` → `TimelockNotMet`; called after → success.
10. `grace_window_expiry` — `execute` after `eta + grace` →
    `ProposalExpired`/`InvalidProposalState`.

The partition itself is modeled with the SDK auth testutils: a signer "in the
partition" is one whose `require_auth` is simply not invoked that ledger (we
advance `env.ledger().set_timestamp`) and/or whose mock auth is deliberately
withheld for the duration, then restored at heal. No new contract code is
required for #432 unless a scenario reveals a gap — which is the point.

### 5.4 Test plan summary

10 scenarios above; each asserts both the *safety* outcome (no unauthorized
state change, no double execution) and the *liveness* outcome (correct proposals
still execute once the partition heals within the grace window). All gated
behind `#[cfg(test)]` with `testutils`.

## 6. Design: #430 — granular role-based authorization

### 6.1 Current authorization model

- **Admin** (single `Address`, `DataKey::Admin`): all config — `set_admin`,
  `set_threshold`/`update_threshold`, `add_signer`, `remove_signer`,
  `set_vote_weight`, `set_emergency_guardians`, `set_grace_period`.
- **Signers** (`DataKey::Signers`, weighted): `propose`, `approve`.
- **Emergency guardians** (`DataKey::EmergencyGuardians`): `cancel_proposal`
  only.
- **Executor**: any address (`require_auth` but no role check) — execution is
  gated by quorum + timelock, not by who calls it.

So configuration updates are **already** admin-gated. The issue's "prevent
unauthorized configuration updates" is therefore satisfied today by a single
all-powerful admin. The meaningful improvement is **granularity**: letting the
admin delegate *specific* config powers without handing over full admin
(which today also rotates admin and the guardian set).

### 6.2 Options

- **(A) Status quo.** Argue #430 is already met; close with a test that
  documents the existing admin-gating. Lowest risk, least value.
- **(B) Role enum + per-address role map.** Introduce
  `#[contracttype] enum Role { Admin, SignerManager, ThresholdManager,
  GuardianManager, ... }` and `DataKey::Roles Map<Address, Role>`. Each config
  entry point checks a *capability* rather than "is admin". Admin retains all
  capabilities (backward compatible). Admin can grant a subset to another
  address. **ABI-additive** (new storage key, new type; existing functions
  unchanged in signature). This is the design that matches "granular
  role-based authorization."
- **(C) Capability bitset.** `Map<Address, u32>` bitmask of capabilities. More
  compact, less self-describing than (B); harder to extend without renumbering
  the bitmask (which is itself a small ABI surface).

### 6.3 Recommendation

**(B)**, with these rules to stay inside the constraints:

- **Centralize authorization** in one helper, e.g.
  `fn require_role(env, role: Role) -> Result<(), GovernanceError>`, replacing
  the scattered `get_admin()?.require_auth()` calls in config entry points. This
  is the single chokepoint a reviewer audits — which is itself the security
  value of RBAC.
- **Admin is the implicit superset.** Any address holding `Role::Admin`
  satisfies every capability check, so existing deployments that only ever set
  an admin keep identical behavior. `init` assigns the admin `Role::Admin`.
- **Role grants go through the proposal flow** for non-admin roles (the admin
  granting `SignerManager` to X is itself a config change → propose/approve/
  timelock), **except** the admin granting/revoking roles, which stays
  admin-direct — otherwise there is no way to bootstrap delegation. This is the
  one judgment call; document it. Alternative: *all* role changes go through
  governance proposals, admin can do nothing directly except propose — purer,
  but then "admin" is just another signer for config and #430 collapses into
  the existing multisig. State that alternative in the doc and let the reviewer
  choose.
- **New error variants (appended only):** `UnauthorizedRole = 28`,
  `UnknownRole = 29` if (B) needs them. Never renumber.
- **Footprint:** one new instance key (`DataKey::Roles`). Role lookup is O(1).
  No per-call scan added. Bounds documented as for `MAX_PAGE_SIZE`.

### 6.4 Test plan (#430)

- Admin can perform every config op (backward compatibility).
- A `SignerManager`-only address can `add_signer`/`remove_signer` but *not*
  `set_threshold` or `set_admin` (returns `UnauthorizedRole`).
- A non-role address cannot perform any config op.
- Admin retains the ability to grant/revoke roles; a delegated role cannot
  escalate itself or grant roles to others.
- Granting a role emits an event (`RoleGranted { addr, role }`) for indexer
  observability, consistent with the existing `SignerAdded`/`SignerRemoved`
  pattern.
- The single `require_role` helper is the only authorization path — asserted by
  a test that greps the binary for direct `require_auth` in config entry points
  (or by code review gate).

## 7. Design: #423 — oracle-router (excluded)

`oracle-router` does not exist and has no analog. Implementing #423 would mean
scaffolding a new oracle-aggregation contract (price feeds, quorum across
oracles, staleness/decay, deviation thresholds). That is a substantial new
subsystem, explicitly outside this repo's stated scope (README non-goals:
"no cross-chain anything"; the project is a single-token streaming primitive).
It also does not belong in a PR titled as a multisig-threshold refactor.

**Recommendation:** do not implement #423 here. If it is real, it needs its own
spec (contract purpose, oracle model, liveness/safety under byzantine oracles,
integration with `stream`/`factory`/`governance`) and confirmation that
Perpetua-Contracts is even the right repository. Flagged back to the issue
author / user.

## 8. Implementation & PR notes

- This document is design-only, per the user's direction ("just draft the
  design").
- For completeness on the PR step the user mentioned earlier: `gh` is not
  installed in this environment (`gh: command not found`), and the configured
  remote is an SSH alias (`git@github-devprom6:devprom6/Perpetua-Contracts.git`)
  rather than `github.com`, so `gh pr create` is not available as-is. The
  repository identity is also inconsistent across sources (issues avatar
  `releaseken`, git remote `devprom6`, `Cargo.toml` `hannyloveworld`) — a
  sandboxed clone. PR submission will need either `gh` installed and
  authenticated, the platform's own API/CLI, or manual creation; confirm the
  target before pushing.
- When implementation proceeds, the order should be #434 (smallest, ABI-safe,
  unblocks nothing else) → #430 (RBAC, the one real behavior change) → #432
  (tests, some of which assert #430's new auth paths). Each as its own commit on
  a feature branch.
- Before implementing #430/#432, read the remainder of `governance/src/lib.rs`
  not yet reviewed here (`execute`, `cancel_proposal`, the view functions, and
  the existing inline `#[cfg(test)]` modules) to match test idioms and confirm
  the exact set of admin-gated entry points. This design is based on the error
  enum, storage keys, `init`/`set_*`/`propose`/`approve` paths, the events, the
  retention policy, and the reentrancy guard; the execute/cancel paths were not
  read in full.

## 9. Open questions (require confirmation before implementation)

1. Is mapping `multisig-threshold` → `contracts/governance` correct, or are
   #434/#432/#430 meant for a different repository?
2. For #430, is the intent (B) granular delegation, or (A) "already met,
   document it"? The issues are too vague to tell.
3. Should #423 be dropped, or is there an oracle-router contract elsewhere that
   should be the real target?
4. How should the PR be submitted given no `gh` and a sandbox remote?

---

# Design: Oracle-Router & Multisig-Threshold Performance and Fuzz Hardening

**Status:** Draft
**Issues:** #422, #431, #427, #428
**Scope:** `contracts/oracle-router`, `contracts/multisig-threshold`

## Prerequisite and scope note

The four issues target `contracts/oracle-router` and `contracts/multisig-threshold`.
Neither contract is present in this checkout — the workspace currently contains only
`archival-probe`, `factory`, `governance`, `stream`, and `stream-nft` under `contracts/`,
and a repo-wide search finds no references to either contract name.

This design therefore specifies the **target behavior and performance characteristics**
for those contracts rather than a retrofit of existing code. Implementation requires
first importing or scaffolding `oracle-router` and `multisig-threshold` into the
workspace. The optimization and fuzz strategies below are defined against that target
so they apply directly once the contracts land. Open question Q1 tracks this.

## Goals

1. Minimize Soroban host CPU instructions in `oracle-router` and `multisig-threshold` by
   eliminating redundant deserialization in loops (#422, #431).
2. Streamline cross-contract invocation (CCI) overhead in `oracle-router` during
   high-frequency batch claims (#427).
3. Add fuzz coverage for edge-case timestamps near the `u64` maximum boundary in
   `oracle-router` (#428).
4. Defensive error handling and checked arithmetic throughout, with no panics on
   untrusted input.

## Non-goals

- Changing external contract APIs or on-wire storage formats beyond what the
  optimizations require.
- Treating before/after instruction counts as a deliverable in themselves; they are
  verification evidence, not a goal.
- Gas tuning unrelated to deserialization, CCIs, or timestamp arithmetic.

## Background: cost model

- **Deserialization.** Reading a collection from durable storage via
  `Env::storage().get()` returns a `Val` that is converted to the target type through
  `TryFromVal`. Re-reading the same key inside a loop, or re-deriving an element per
  iteration, repeats host work proportional to `N`. Binding the result to a local
  once and iterating the local avoids the repeat.
- **Cross-contract invocation.** Each `invoke_contract` / CCI carries fixed host
  overhead (args serialization, dispatch, result conversion). In a per-claim loop this
  cost is multiplied by the batch size.
- **Timestamps.** Soroban ledger timestamps are `u64`. Expiry, staleness-window, and
  next-update arithmetic can overflow near `u64::MAX`; `checked_add`/`checked_sub`
  return a typed `Error` instead of panicking or silently wrapping.

## Design

### 1. Loop deserialization (#422 oracle-router, #431 multisig-threshold)

**Problem.** Iterating a stored collection by calling `storage().get()` per element —
or holding a `Val` and re-converting each iteration — re-enters the host deserializer
`N` times.

**Approach.**

- **Deserialize once, iterate the local.** Pull the collection from storage into a
  strongly-typed local (`Vec<T>` / `Map<K, V>`) once per call, then iterate the bound
  local. Do not call `storage().get()` inside the loop.
- **Iterate, don't index-fetch.** Prefer `iter()` over the bound collection over
  repeated `Vec::get(i)` when the element is consumed once; materialize the whole
  collection up front rather than touching storage per index.
- **Nested collections.** Ensure the outer deserialize fully materializes nested
  structures (e.g. `Vec<Signer>` whose `Signer` holds a nested `Vec`); do not lazily
  re-read inner data per outer element.
- **multisig-threshold shape.** Store signers as a single `Map<Address, u32>`
  (address → weight) or a single `Vec<Signer>`, deserialized once per verification
  pass, instead of per-signer storage reads. Threshold checks then iterate the bound
  local.
- **oracle-router shape.** Cache the active price feed map deserialized once per
  routing decision; route lookups iterate the local rather than re-reading storage.

**Verification.**

- Unit tests at large `N` (256 signers; 256 price entries) assert correct aggregates
  with zero per-element storage reads (a test storage hook counts reads).
- Before/after host CPU instruction counts captured in test output to confirm the
  reduction.

### 2. Cross-contract invocation overhead in batch claims (#427 oracle-router)

**Problem.** In high-frequency batch claims, issuing one CCI per claim to fetch oracle
price(s) multiplies fixed CCI overhead by batch size.

**Approach.**

- **Single-read fan-out.** Fetch the relevant price feed(s) once at the start of the
  batch, bind to a local, and reuse for every claim that consumes the same feed. Only
  re-invoke when a claim requires a different feed/asset.
- **Group claims by feed.** Bucket claims by the price feed they consume; fetch each
  distinct feed at most once, then process each bucket. CCI count becomes
  `O(distinct feeds)`, not `O(claims)`.
- **Compact CCI args.** Pass minimal `Val`-typed arguments across the CCI boundary
  (a feed id / index) rather than large structs.
- **Cache-first reads.** Have the router pull prices into its own storage on update;
  claims read the cache, so a batch claim triggers zero CCIs in the common case and
  falls back to a single CCI per missing feed.

**Verification.**

- Unit/fuzz tests with batch sizes 1, 16, 256 assert CCI count scales with distinct
  feeds, not claims, via a test double that counts invocations.

### 3. Edge-case timestamp fuzzing (#428 oracle-router)

**Problem.** Timestamp arithmetic (expiry, staleness window, next-update) can overflow
near `u64` boundaries, causing panics or silent wraparound.

**Approach.**

- Add a `proptest` fuzz target generating timestamps across boundary regions: `0`,
  `1`, near ledger close time, near `u32::MAX`, near `u64::MAX`, and `u64::MAX` itself;
  deltas include `0`, `1`, large values, and values that force `t + delta` and
  `t - delta` to overflow.
- All timestamp arithmetic uses checked operations; overflow maps to a single typed
  `Error` variant rather than panicking or wrapping.
- **Property under test:** for any `(timestamp, delta)` the router never panics and
  returns a deterministic `Ok`/`Err`; every overflow maps to the same error variant.

**Verification.**

- `cargo test` fuzz target runs many shrinkable cases; CI gate prevents regressions.

### 4. Cross-cutting: defensive error handling and checked arithmetic

- Arithmetic on amounts, weights, timestamps, and counts uses checked operations; on
  overflow return a contract `Error` variant. No `unwrap`/`expect`/unvalidated
  indexing on untrusted input.
- Validate at trust boundaries: batch length, feed id, signer address, threshold
  range.
- Storage reads validated against expected types; missing keys return typed errors.

## Risks and trade-offs

- **Footprint.** Caching deserialized collections increases transient memory within a
  call, bounded by batch size — acceptable for the batch sizes in scope.
- **Failure atomicity.** Batching CCIs changes failure semantics: a single feed fetch
  failure now fails the whole batch. Mitigation: per-feed error isolation where the
  spec allows partial success; otherwise documented as intentional all-or-nothing
  (see Q2).

## Open questions

- **Q1.** Are `oracle-router` and `multisig-threshold` to be scaffolded in this repo, or
  imported from `releaseken/Perpetua-Contracts`? Neither exists in this checkout; this is
  blocking for implementation.
- **Q2.** Should batch claims support partial success, or remain all-or-nothing?

## Out of scope for this draft

Implementation code, the actual contract scaffolding, and measured before/after
numbers — all pending resolution of Q1 and the contracts landing in the workspace.

---

# Design — Hardening the multisig/rent/archival tranche

Status: **Draft — pending scope confirmation** (see §0).
Scope: issues #429, #440, #441, #426.

---

## 0. Scope caveat — read this first

The four issues name three contracts that **do not exist in this workspace**:

| Issue | Names this contract | Present in this repo? |
|-------|--------------------|------------------------|
| #429 | `contracts/multisig-threshold` | ❌ no such crate |
| #440 | `contracts/rent-manager` | ❌ no such crate |
| #441 | `contracts/rent-manager` | ❌ no such crate |
| #426 | `contracts/oracle-router` | ❌ no such crate |

The actual contracts in this workspace are `stream`, `stream-nft`, `factory`,
`governance`, and `archival-probe`. The issues are also filed against
`releaseken/Perpetua-Contracts`, while this checkout is `aristotle224/Perpetua-Contracts`
— a different fork.

This design therefore **translates each issue's substantive theme onto the real
contracts that already carry that responsibility here**, rather than inventing
three new contract crates from a one-line issue title. The mapping is explicit
below and is the single most important assumption to confirm before any code is
written. If the intent was instead to scaffold `multisig-threshold`,
`rent-manager`, and `oracle-router` as brand-new contracts, §8 lists what that
path requires and this document does not design it.

### Mapping

| Issue | Theme | Real contract(s) this design targets | Why this is the right home |
|-------|-------|---------------------------------------|----------------------------|
| #429 | Precision loss in fractional stream withdrawal under a multisig threshold | `governance` (the threshold layer) + `stream` (the withdrawal math) | `governance` **is** a multisig-threshold contract (co-signers, `Threshold`, weighted `approval_weight`, timelock, quorum) and already dispatches operations into `stream` via `CallData` — but has no withdrawal operation today. |
| #440 | Minimize host CPU instructions by optimizing loop deserialization | `stream` `batch_withdraw` + `storage` | The rent/TTL management in this repo lives inside `stream::storage`; `batch_withdraw` is the loop that deserializes many stream entries and is the realistic optimization target. |
| #441 | Byzantine fault tolerance test scenarios under simulated network partitions | `stream` `accrual` read-freshness/replay primitives + `archival-probe` + `governance` | `accrual` already ships `classify_read` / `ReadFreshness`, `recovery_for` / `Recovery`, and `remaining_to_apply` — the exact primitives a BFT-under-partition test suite would exercise. |
| #426 | Automated entry lifetime extension for archival resilience | `stream::storage` TTL engine + `archival-probe` | `storage` already implements schedule-aware TTL targets and a permissionless keeper path; `archival-probe` is the live-network canary for the archival failure mode. |

Everything below is written against the contracts in the left-hand real-contract
column. Where an issue's literal wording implies a new contract, the design
notes that explicitly rather than silently inventing one.

---

## 1. Goals

1. **#429** — Make a threshold-gated, *fractional* stream withdrawal
   numerically exact: no stroop is created or destroyed by the fraction→amount
   conversion, residue is accounted for deterministically, and every arithmetic
   step is checked.
2. **#440** — Reduce host CPU instructions in the multi-entry deserialization loop
   (`batch_withdraw`) by removing redundant deserialization and consolidating the
   storage footprint, without changing observable behaviour or the public ABI.
3. **#441** — Add a deterministic BFT test matrix for the rent/TTL subsystem that
   simulates ledger skew, stale reads, replayed/duplicate operations, and
   archive→restore transitions, asserting the invariants the production code
   already claims.
4. **#426** — Specify an automated, idempotent entry-lifetime-extension routine
   that guarantees no live stream entry archives, and degrades gracefully
   (detectable restore) if it ever does.

## 2. Non-goals

- Designing new standalone `multisig-threshold`, `rent-manager`, or
  `oracle-router` crates. (See §8 if that is actually wanted.)
- Changing the stream's core accrual rounding policy (round-down, residue to
  sender). That policy is load-bearing for conservation (I1, I4, I6) and stays.
- Renumbering any `Error` / `GovernanceError` discriminant — both are frozen
  public ABI, append-only.
- Adding an on-chain scheduler/cron. Stellar has none; "automated" in #426 means
  a deterministic keeper *routine* plus an off-chain trigger, not a contract that
  wakes itself.

## 3. Background (grounding in the current code)

### 3.1 Stream accrual (`contracts/stream/src/accrual.rs`)

The withdrawal pipeline is:

```
vested(t)        = 0                                  if before cliff
                 = deposited                         if duration==0 or elapsed>=duration
                 = deposited * elapsed / duration     otherwise  (integer div, rounds DOWN)
withdrawable(t)  = max(0, vested(t) - withdrawn)
refundable(t)    = deposited - vested(t)              (exact; I4: vested+refundable==deposited)
```

Rounding is always **down**, by design: truncation residue stays in the pool and
returns to the sender at settlement, so the pool can never owe more than it
holds (`accrual.rs`, `vested` doc + final clamp `if raw > deposited { deposited
} else { raw }`). The only arithmetic error is `Error::Overflow` (alias
`ArithmeticOverflow`), used at the `checked_mul` in `vested` and the
`checked_sub` in `withdrawable`/`refundable`/`liability`.

Three primitives already exist that a threshold-gated withdrawal and the BFT
tests will lean on:

- `remaining_to_apply(requested_total, withdrawn) = (requested_total - withdrawn).max(0)`
  — idempotent replay guard: replaying a withdrawal whose effect is already in
  `withdrawn` yields zero, not a double payout.
- `classify_read(observed_ledger, required_ledger) -> ReadFreshness::{Fresh, Stale{behind_by}}`
  — deterministic freshness verdict for a read that may lag the ledger the
  caller requires; a stale read is discarded and re-read, never combined.
- `recovery_for(present, stream_id, next_stream_id) -> Recovery::{None, RestoreFootprint, NeverExisted}`
  — distinguishes an archived-but-issued entry from one that was never created,
  the signal a keeper/SDK keys on.

### 3.2 Stream storage / TTL engine (`contracts/stream/src/storage.rs`)

The rent model in this repo:

- **Instance** entry `DataKey::NextStreamId` is pinned to `max_ttl()` on every
  mutating call (`extend_instance`), so the id counter can never archive before
  the streams it issued.
- **Persistent** entry `DataKey::Stream(id)` is extended to
  `ttl_target_ledgers` = `seconds_to_ledgers(remaining_life + TTL_BUFFER_SECONDS)`,
  floored at `MIN_STREAM_TTL_LEDGERS` (~30 days × 1.1 drift headroom), clamped to
  the network `max_entry_ttl` (queried dynamically, never baked in).
- `seconds_to_ledgers` rounds **up** (ceiling) so the ledger count never
  undershoots the requested lifetime.
- `load_stream` extends TTL on read; `peek_stream` is view-only (no write) — used
  by batch preflight and views.
- A permissionless keeper path (`extend_stream_ttl`) keeps a stream readable
  without the sender's cooperation; it bumps both the persistent stream entry
  and the instance counter together (the ordering guarantee).

### 3.3 Governance = the multisig-threshold layer (`contracts/governance/src/lib.rs`)

`governance` is a co-signer multisig with a configurable threshold, per-signer
voting weight, a timelock, quorum snapshots, a grace window, and a reentrancy
guard. It executes by decoding a `CallData` variant and dispatching a
cross-contract call to a `target`. The stream-related `CallData` variants today
are `StreamSetAdmin`, `StreamSetMaxRate`, `StreamGlobalResume`, and
`StreamBulkResumeAsAdmin`. **There is no variant that performs a withdrawal.**
That gap is exactly where #429 lands.

### 3.4 Archival probe (`contracts/archival-probe/src/lib.rs`)

A non-product canary that writes one persistent entry and deliberately never
extends its TTL, so it archives on the network's minimum schedule. It proves the
live-network sequence `read archived → tx FAILS → resubmit with
RestoreFootprint → read succeeds`. The restore mechanism it exercises is a
ledger property, so proving it for the probe proves it for `DataKey::Stream(id)`.

---

## 4. Design — #429: exact fractional withdrawal under a multisig threshold

### 4.1 Problem

A threshold-gated withdrawal lets governance move a *fraction* of a stream's
withdrawable balance — e.g. "pay the recipient one-third of what they have
earned" — without the proposer precomputing the stroop amount off-chain (which
would race the accrual clock between propose and execute). Converting a fraction
to stroops is where precision loss enters: `withdrawable * num / denom` is
integer division, and if `denom` does not divide the product cleanly the residue
has to go somewhere deterministic, or stroops are silently created or destroyed.

### 4.2 Shape

Add one stream entry point and one `CallData` variant, both additive (no ABI
renumber, no existing behaviour change).

**Stream side — new function** (additive, behind admin auth, mirroring the
existing `bulk_resume_streams_as_admin` admin path):

```
FluxoraStream::withdraw_fraction_as_admin(
    env, stream_id, recipient, numerator, denominator
) -> Result<i128, Error>
```

- `numerator`/`denominator` are positive `i128`, `denominator > 0`, `numerator <= denominator`
  (a fraction in `[0,1]`; `numerator == denominator` is "withdraw everything").
- Auth: the stream **admin** (`require_auth_for_args`). Governance becomes the
  stream admin (it already supports `StreamSetAdmin`), so a passed proposal is
  what authorizes this call. This reuses the existing admin-auth pattern rather
  than introducing a second authorization surface.
- Computes `payout = withdrawable(stream, now) * numerator / denominator` with the
  same rounding discipline as `vested`: **round down**, residue stays in the pool.

**Governance side — new `CallData` variant** (append-only, wire form is the
variant *name* not its position, so existing encoded proposals stay decodable):

```
CallData::StreamWithdrawFraction { stream_id: u64, recipient: Address, numerator: i128, denominator: i128 }
```

Dispatched via `dispatch_operation` to `withdraw_fraction_as_admin` on the
proposal's `target`. Reaches threshold → waits the timelock (`eta`) → executes
inside the existing reentrancy guard.

### 4.3 Exact arithmetic (the core of #429)

The conversion must be overflow-safe and conserving. The design:

```
let available = accrual::withdrawable(&stream, now)?;            // i128, >= 0
// overflow-safe product: available * numerator
let scaled = available.checked_mul(numerator).ok_or(Error::Overflow)?;
let payout = scaled.checked_div(denominator).ok_or(Error::DivisionByZero)?;
```

Conservation and residue handling, matching the stream's existing invariant set
(I1/I4/I6 in `accrual.rs`):

1. **Rounding direction = down.** Identical to `vested`. The residue
   `available - payout - (already withdrawn delta)` stays in the pool and is
   refundable to the sender at settlement. The pool can never owe more than it
   holds.
2. **Conservation is exact across the call.** `withdrawable` is already defined
   as `vested - withdrawn`, and `apply_withdrawal` (the existing helper used by
   `withdraw` and `batch_withdraw`) bumps `stream.withdrawn` by exactly `payout`
   and transfers exactly `payout`. So `vested + refundable == deposited` (I4) is
   preserved by construction — the fractional path reuses `apply_withdrawal`
   rather than carrying its own transfer.
3. **Idempotency against replay.** A proposal that executes, then is replayed
   (e.g. an ambiguous tx result under a partition — see #441) must not pay twice.
   `withdraw_fraction_as_admin` applies `remaining_to_apply` semantics: the
   *intended* total is bounded by the current `withdrawable`, which already
   subtracts `withdrawn`. A replay computes `payout` against a `withdrawn` that
   has already absorbed the first execution, so a replayed identical fraction
   pays strictly less (the remaining fraction of the *newly accrued* balance
   only) — never a duplicate of the original amount. This is the property
   `remaining_to_apply` was written to guarantee; the fractional path inherits it
   by reading `withdrawable` fresh rather than caching an amount.
4. **Fraction normalization is the caller's job, not the contract's.** `1/3` and
   `2/6` are different inputs that happen to produce the same `payout` only when
   `denominator` divides `available*numerator` identically; the contract does
   not reduce fractions on chain (reduction is a pure function the proposer can
   do off-chain). This keeps the on-chain path one division, not a GCD + division.
5. **Overflow is a typed error, never a trap.** `available.checked_mul(numerator)`
   maps to `Error::Overflow`, consistent with `vested`'s guard. Because
   `available <= deposited <= i128::MAX` and `numerator <= denominator`, the
   product is bounded by `deposited * denominator`; `create_stream` already
   guarantees `deposited * duration` fits `i128`, but `denominator` is
   caller-chosen and unbounded, so the checked_mul is **not** defence-in-depth
   here — it is the primary guard. The design rejects `denominator` values that
   would make `available * numerator` overflow by returning `Error::Overflow`
   rather than truncating.

### 4.4 Edge cases explicitly handled

| Edge case | Handling |
|-----------|----------|
| `denominator == 0` | New `Error::DivisionByZero` (append to `Error`, e.g. `= 35`). Distinct from `Overflow` so a client can tell "bad fraction" from "too big". |
| `numerator == 0` | `payout == 0`; the call is a no-op that still pays gas. Reject with `Error::InvalidAmount` (existing variant) to match the zero-amount policy in `error.rs`. |
| `numerator > denominator` | Reject — a fraction > 1 would let governance over-draw. New `Error::InvalidFraction` (append) or reuse `InsufficientWithdrawable` semantics. Design prefers a dedicated variant for clarity. |
| `available == 0` (pre-cliff / pre-start / fully drawn) | Existing `NothingToWithdraw` / `StreamTerminated` paths apply unchanged; the fractional call routes through the same `available == 0` branch as `withdraw`. |
| Stream terminal | `StreamTerminated` before computing the fraction, matching `withdraw`. |
| Replay under partition | `withdrawable` is recomputed from fresh `withdrawn`; see §4.3(3). |
| Maturity mid-proposal | If the stream matures between propose and execute, `withdrawable` grows to the full deposit and the fraction pays out of the larger pool — correct, and exactly why the amount is computed at execute time, not propose time. |

### 4.5 Tests / fuzz (#429)

- **Unit:** for each `(available, num, denom)` in a boundary set
  (`available = 0`, `1`, `i128::MAX`, `denom = 1`, `2`, `available`, prime
  denominators), assert `payout == available*num/denom` (floor), `payout <=
  available`, and that after applying, `vested + refundable == deposited` still
  holds (reuse the existing `Harness::assert_invariants`).
- **Conservation fuzz:** random `deposited`, `elapsed`/`duration` pair, random
  fraction; assert no stroop created/destroyed across a sequence of fractional
  withdrawals ending in a `cancel`.
- **Replay fuzz:** apply a fractional withdrawal, then replay the identical
  signed payload; assert the second pays at most the first's accrual delta, never
  a duplicate.
- **Governance path:** propose → reach threshold → wait timelock → execute
  dispatches `StreamWithdrawFraction` and moves the expected stroops; a
  sub-threshold or pre-timelock execution reverts (reuses governance's existing
  `QuorumNotReached` / `TimelockNotElapsed` guards).

---

## 5. Design — #426: automated entry lifetime extension for archival resilience

### 5.1 Problem

A stream that lives longer than the network's `max_entry_ttl` *must* be
periodically re-extended or its accounting entry archives and becomes unreadable
until someone pays to restore it. The contract already has a permissionless
`extend_stream_ttl` keeper path and schedule-aware targets; #426 asks for the
**automated, resilience-guaranteeing** layer on top: a routine that provably
keeps every live entry alive and degrades detectably if it ever fails.

### 5.2 Two components

**A. On-chain: a single deterministic `sweep_ttl` routine** (additive view/keeper
entry point on `stream`), not a new contract:

```
FluxoraStream::sweep_ttl(env, stream_ids: Vec<u64>) -> Result<u32, Error>
```

- Permissionless (any caller — keeper, recipient, passer-by), mirroring
  `extend_stream_ttl`.
- For each id: load via `peek_stream` (view-only, no TTL bump on the *read*),
  then `extend_stream` to `ttl_target_ledgers`. Returns the count extended.
- **Idempotent:** extending an already-fresh entry to the same target is a
  no-op at the storage level (Soroban `extend_ttl` to a TTL ≥ current is a
  no-op), so a keeper may re-sweep safely and at low cost.
- **Bounds:** reuses `MAX_BATCH_SIZE` (16) and the same `BatchTooLarge` /
  `DuplicateStreamId` / `MalformedStreamId` validation as `batch_withdraw`, so
  the footprint and event budget math already measured for batches applies
  unchanged. No new resource limit.

**B. Off-chain: the automation trigger** (specification, not contract code —
Stellar has no on-chain scheduler):

- A keeper watches TTLs (readable via `get_ttl` on the persistent entry) and
  calls `sweep_ttl` for any stream whose TTL falls inside `TTL_BUFFER_SECONDS`.
- The schedule-aware target already gives the keeper the full 30-day buffer, so
  "automated" means "well before expiry", not "at the last ledger".
- `archival-probe` is the regression canary: a CI job runs
  `script/archival-canary.sh` against quicktest/testnet to prove the
  archive→RestoreFootprint→read sequence still works, so the resilience
  guarantee is tested against a real network, not just the in-memory host (which
  auto-restores and would hide a regression).

### 5.3 The resilience guarantee

The guarantee has two tiers, both enforceable from existing primitives:

1. **Liveness (normal operation):** for any stream that is not terminal,
   `sweep_ttl` before the TTL drops below `MIN_STREAM_TTL_LEDGERS` keeps the
   entry readable. The instance counter is always at `max_ttl`, so it never
   archives before the streams it issued (the existing ordering guarantee).
2. **Detectable degradation (if the keeper misses):** if an entry does archive,
   `stream_recovery(env, id)` (already in `storage.rs`, wrapping
   `recovery_for`) returns `Recovery::RestoreFootprint` for an issued id whose
   entry is absent — distinct from `NeverExisted`. The caller resubmits with a
   `RestoreFootprint` and the accounting is intact (the probe proves this is a
   ledger property, not a contract one). So a missed sweep is a recoverable
   outage, not a fund loss: tokens sit in the pooled balance; only the
   accounting entry is temporarily unreadable.

### 5.4 Tests / fuzz (#426)

- **Unit:** seed a ledger state with a near-expiry stream (`testutils::Ledger`
  to advance the sequence), call `sweep_ttl`, assert the TTL is restored above
  `MIN_STREAM_TTL_LEDGERS` and below `max_entry_ttl`.
- **Idempotency:** sweep the same ids twice; assert the second is a no-op
  (TTL unchanged, no spurious events).
- **Archival round-trip (network-level, via the probe):** the probe's existing
  `the_probe_does_not_extend_its_own_ttl` test plus a canary run that archives,
  fails to read, restores, and reads — proving the degradation path.
- **Property:** over random schedules with random sweep cadence, no
  non-terminal stream's TTL ever reaches zero (monotonic-in-time TTL floor).

---

## 6. Design — #440: minimize host CPU instructions in the deserialization loop

### 6.1 Problem

`batch_withdraw` today runs **two passes** over the same stream entries:

1. A preflight loop: `peek_stream` (deserialize) for each id, validate
   recipient, compute `withdrawable`, accumulate `total`, and push the
   deserialized `Stream` into a `Vec<Stream>` (`streams`) and amounts into
   `payouts`.
2. An apply loop: pop each stored `Stream` back out of the `Vec`, and for
   non-zero payouts call `apply_withdrawal` — which itself loads/extends storage
   again.

Each `peek_stream`/`load_stream` is a host storage read that **deserializes** the
`Stream` from its ledger entry. Holding an entire `Vec<Stream>` in memory also
costs CPU to deserialize-and-copy every entry up front, even streams that will
pay zero and only need a TTL bump. The instruction cost scales with batch size
and with how many bytes each `Stream` serializes to. #440 asks to minimize that.

### 6.2 Optimizations (behaviour-preserving, ABI-stable)

1. **Load once, reuse.** Replace the two-pass `peek`-then-`load` with a single
   pass: `load_stream` (which extends TTL) on each id in the preflight loop, hold
   the **already-deserialized** `Stream` in the `Vec`, and reuse that same value
   in the apply loop — do not call `load_stream`/`peek_stream` a second time.
   `apply_withdrawal` already takes a `&mut Stream`; it should not re-load. This
   halves the deserialization count from `2*N` to `N`.
2. **Hoist the clock.** `now = env.ledger().timestamp()` is currently read once
   before the loop — keep that, and ensure no per-iteration `env.ledger()` call
   sneaks in via helpers. `accrual::withdrawable` takes `now` by value, so this
   is already satisfied; the change is to keep it that way and avoid any new
   per-iteration host call.
3. **Footprint preloading (the real CPU win).** Soroban charges per storage
   *deserialization*. If the batch's stream keys are declared in the transaction
   footprint up front, the host can carry the deserialized entries across the
   transaction rather than re-deserializing on each access. The contract cannot
   itself declare a footprint, but `sweep_ttl`/`batch_withdraw` can be paired
   with guidance (and a SDK helper) to preload `DataKey::Stream(id)` for the
   batch ids. Where the host supports it, this turns N repeated deserializations
   into N footprint-declared reads. The design documents this as the expected
   caller pattern; the on-chain code change is the single-pass load (#1) which
   is correct regardless of footprint.
4. **Skip the `Vec<Stream>` for zero-payout entries.** A stream whose
   `withdrawable == 0` only needs a TTL extension, not a payout. Its entry can
   be extended in the preflight pass (it is already deserialized) and not held
   in the payout `Vec`, cutting the memory/CPU of carrying it through the apply
   pass. Behaviour is unchanged: zero-payout entries are already "skipped" today,
   this just skips them earlier.
5. **No algorithmic change to validation.** Duplicate detection
   (`reject_duplicate_ids`), `MAX_BATCH_SIZE`, and recipient checks stay exactly
   as-is; they are not in the hot deserialization path.

### 6.3 What does not change

- **Atomicity.** The batch stays all-or-nothing: any error unwinds every staged
  write, including payouts already applied to earlier streams. Single-pass load
  does not affect this — the host still owns the transaction.
- **Event order and shape.** Per-stream `withdrawn` events still emit in batch
  order; the total returned is unchanged.
- **ABI.** No signature change. This is purely an internal loop restructure.

### 6.4 Measurement

- Add a `gas_benchmark`-style test (the factory crate already has
  `tests/gas_benchmark.rs` as a template) that records instruction count and
  `mem_bytes` for `batch_withdraw` at `MAX_BATCH_SIZE` before and after, asserting
  a strict reduction in instructions with no increase in byte reads. The
  existing `test::resource_limits` suite (which reports the 16-stream cap
  against protocol-27 limits) is the place to assert the optimization keeps 2x
  headroom on footprint/writes/reads/instructions.

---

## 7. Design — #441: BFT test scenarios under simulated network partitions

### 7.1 Problem

Under a network partition, a client may read stale ledger state, a transaction
may be replayed after an ambiguous result, and an entry may archive and need
restoration. The production code already has primitives for each of these
(`classify_read`, `recovery_for`, `remaining_to_apply`), but they are currently
`#[allow(dead_code)]` — present for defence in depth but not exercised by a
named scenario suite. #441 asks for that suite.

### 7.2 What "simulated partition" means on Soroban

Soroban has no network simulation primitive, so "partition" is modelled at the
deterministic inputs the contract already consumes:

- **Ledger skew** — an observer's `observed_ledger` is behind the
  `required_ledger` of a confirmed write. `classify_read` returns
  `Stale { behind_by }`; the suite asserts a stale read is discarded and
  re-read, never combined with a fresh one.
- **Replay** — the same signed operation resubmitted after an ambiguous result.
  `remaining_to_apply` asserts the replay pays only the un-applied delta, never
  a duplicate.
- **Archive/restore** — an entry's TTL elapses and the host refuses the read
  until a `RestoreFootprint` is supplied. `recovery_for` asserts the contract
  reports `RestoreFootprint` for an issued-but-absent id (vs `NeverExisted` for
  an unissued one); the `archival-probe` round-trip asserts the data is intact
  after restore.

### 7.3 Scenario matrix

Each scenario is a deterministic test (no real network except the archival
canary), grouped under `contracts/stream/src/test/` alongside the existing
`accrual_*` suites:

1. **stale-read-discard** — `classify_read(observed=99, required=100)` ⇒ `Stale
   { behind_by: 1 }`; caller re-reads at ledger 100 ⇒ `Fresh`. Assert no
   accounting is derived from the stale read.
2. **replay-no-double-pay** — withdraw `X`, then replay the identical withdraw
   request; assert the second pays `remaining_to_apply(X, withdrawn_after_first)`
   = the newly accrued delta, ≤ the first only if accrual advanced, never `X`
   again.
3. **duplicate-id-rejection-under-replay** — a replayed `batch_withdraw` whose
   ids include a duplicate is rejected with `DuplicateStreamId` on every
   attempt (deterministic, position-independent) — already an invariant, now
   asserted under the replay model.
4. **archive-then-restore-preserves-accounting** — advance the ledger past an
   entry's TTL, assert `stream_exists` is `false` and `stream_recovery` returns
   `RestoreFootprint`; restore the footprint, re-read, assert `vested`/`withdrawn`
   are unchanged. (Unit-level via `testutils`; network-level via
   `archival-probe` + `archival-canary.sh`.)
5. **threshold-under-partition (governance × stream)** — a proposal that
   dispatches `StreamWithdrawFraction` (§4) executes after a simulated
   partition gap (ledger advanced, stale reads discarded, timelock still
   enforced); assert the payout reflects the *execute-time* balance, not the
   propose-time balance, and that a replayed execute is rejected
   (`AlreadyExecuted`).
6. **keeper-miss-then-recover** — let a stream's TTL lapse (simulating a keeper
   that missed its window under partition), assert `sweep_ttl` (§5) restores
   readability without loss; this ties #441 to #426.

### 7.4 Fuzz

- **Replay fuzz:** random withdraw schedules, random replay points; invariant:
  total transferred across all (original + replayed) calls == total transferred
  by a single non-replayed run with the same ledger advances. Stronger than
  "no double pay" — it pins conservation under replay exactly.
- **Skew fuzz:** random `observed`/`required` ledger pairs; invariant:
  `classify_read` never returns `Fresh` when `observed < required`, and
  `Stale{behind_by}` equals the exact difference.
- **Restore fuzz:** random stream states, random archive points; invariant:
  post-restore `vested(t)` equals pre-archive `vested(t)` for all `t`.

### 7.5 What this does not claim

These are **tests**, not new on-chain partition-tolerance logic. Soroban's
ordering and atomicity are the host's; the contract cannot "survive" a
partition at the consensus level. What the suite proves is that the contract's
*accounting* is partition-safe: stale reads don't corrupt it, replays don't
enrich it, and archival doesn't lose it. That is the meaningful BFT property at
the contract layer.

---

## 8. Open questions / risks

1. **[Blocking] Contract-name mismatch (§0).** This design targets the real
   contracts (`stream`/`governance`/`archival-probe`). If the intent was to
   create new `multisig-threshold`, `rent-manager`, and `oracle-router` crates,
   this document is the wrong artifact and the work is substantially larger
   (three new crates, their ABIs, their tests, and integration with the existing
   factory). **Confirm the mapping before implementation.**
2. **`withdraw_fraction_as_admin` authorization surface.** Design reuses the
   existing stream-admin path (governance is set as admin). If the stream has no
   admin concept today beyond what `bulk_resume_streams_as_admin` uses, the
   exact auth wiring needs a read of the untruncated tail of
   `stream/src/lib.rs` before implementation.
3. **New `Error` variants.** §4.4 proposes appending `DivisionByZero` (= 35) and
   possibly `InvalidFraction`. Append-only is ABI-safe, but the exact
   discriminants and whether `InvalidFraction` is warranted vs reusing
   `InsufficientWithdrawable` is a review call.
4. **Footprint preloading (#440, #6.2-3).** The contract cannot declare its own
   transaction footprint; the preload is a caller/SDK pattern. If a purely
   on-chain instruction reduction is required, the single-pass load (#1) is the
   ceiling of what the contract alone can do, and the expected gain is ~halving
   deserialization, not eliminating it.
5. **`sweep_ttl` as a new public entry point** is an ABI addition (bumps
   `ABI_VERSION` only if it is *breaking*; a new method is additive per the
   `ABI_VERSION` comment in `lib.rs`, so no bump is needed). Confirm against
   `contracts/stream/abi/fluxora_stream.json`.
6. **No on-chain scheduler.** #426's "automated" is a keeper + spec, not a
   self-waking contract; this is a Stellar constraint, not a design choice.

---

## 9. Summary of concrete changes

| Issue | Contract | Change | Additive? |
|-------|----------|--------|-----------|
| #429 | `stream` | new `withdraw_fraction_as_admin`; reuse `apply_withdrawal` + `withdrawable` | new method (additive) |
| #429 | `stream` | append `Error::DivisionByZero` (+ optional `InvalidFraction`) | append-only |
| #429 | `governance` | new `CallData::StreamWithdrawFraction` + `dispatch_operation` arm | additive (name-keyed) |
| #440 | `stream` | single-pass load in `batch_withdraw`; skip zero-payout hold | internal, ABI-stable |
| #441 | `stream` (tests) | new `test/bft_partition.rs` scenario suite + fuzz | test-only |
| #426 | `stream` | new `sweep_ttl` keeper entry point | new method (additive) |
| #426 | off-chain | keeper spec + `archival-canary.sh` in CI | non-contract |
