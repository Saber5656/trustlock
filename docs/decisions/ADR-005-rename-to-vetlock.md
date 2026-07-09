# ADR-005: Rename product from "trustlock" to "vetlock"

Status: accepted · 2026-07-08
Deciders: repository owner (human)

## Context

Verified 2026-07 (see [research/competitive-landscape.md](../research/competitive-landscape.md) §5):

- npm package `trustlock` has been owned by a third party since 2026-04-14
  (latest 0.1.1, MIT): *"A Git-native dependency admission controller.
  Evaluates trust signals on every dependency change."* — the **same niche**,
  and it installs a `trustlock` binary, so even scoped publication
  (`@owner/trustlock`) would collide on the bin name.
- Candidate replacement `deptrust` is also occupied on GitHub by same-domain
  tools (clidey/deptrust ~47★; kriskimmerle/deptrust).
- `vetlock` verified available: npm 404, PyPI 404, no significant GitHub
  namesake.

## Decision

The product, npm package, CLI binary, ledger filename, and env-var prefix
are all **vetlock** (`vetlock`, `vetlock`, `vetlock.json`, `VETLOCK_*`).
Naming rationale: *vet* (to examine — cf. `go vet`) + *lock* (decisions
locked in a ledger) states the product loop in one word.

Operational sequence (owner-executed):

1. This design PR is authored inside the still-named `trustlock` repo; all
   docs already use `vetlock`.
2. **After the PR merges**, the owner renames the GitHub repo to `vetlock`
   (GitHub auto-redirects the old URL), updates the repo description, and
   renames the local directory at their convenience.
3. The owner registers the npm name (manual, per secret-handling rules)
   before the first release; PyPI registration is optional and deferred
   until a Python-side distribution actually exists (registering an empty
   placeholder would conflict with PyPI's PEP 541 squatting policy).

## Consequences

- Zero migration cost now (nothing published yet); maximal confusion cost
  avoided later.
- All identifiers in DESIGN.md and issues use `vetlock` exclusively;
  `trustlock` appears only in historical/naming context.
- Until step 2 executes, the GitHub repo name and its description disagree
  with the docs — accepted, time-boxed to the PR review window.
