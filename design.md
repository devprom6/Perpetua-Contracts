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
