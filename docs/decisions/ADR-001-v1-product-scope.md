# ADR-001: v1 product scope — research CLI + approval ledger

Status: accepted · 2026-07-08
Deciders: repository owner (human), Fable (design agent)

## Context

The one-line concept ("a dependency trust research tool for reviewing
packages before adding them") admits several v1 shapes: a stateless report
CLI, a report CLI plus a committable approval ledger, the same plus a
published GitHub Action, or a report CLI plus an MCP server for AI agents.

Competitive research ([research/competitive-landscape.md](../research/competitive-landscape.md))
shows the stateless-report niche is partially served (npq) while **no OSS
tool records dependency review decisions in a repo-owned artifact enforced by
CI** — that gap is the differentiator.

## Decision

v1 ships the three-verb loop:

1. `check` — deterministic, evidence-based pre-add research report;
2. `approve` / `reject` — record the human decision in a committable ledger
   (`vetlock.json`);
3. `verify` — offline comparison of the project's **direct dependencies
   (exact versions)** against the ledger, exit 1 on violations, designed to
   run as a plain CI step (`npx vetlock verify`).

The ledger records direct dependencies at exact-version granularity
(user-confirmed 2026-07-08). Transitive enforcement and semver-range
approvals are explicitly out of v1.

A dedicated GitHub Action and an MCP server are deferred to v2; `verify`'s
plain-CLI ergonomics and `check --json` keep both integrations thin wrappers
later.

## Consequences

- The ledger file format (§12 of DESIGN.md) becomes a compatibility surface
  from day one; it is versioned (`version: 1`) with additive evolution.
- Exact-version approvals mean routine version bumps surface as
  `version_drift` violations — this is the intended review-forcing behavior,
  and the primary UX cost accepted.
- Direct-only scope keeps approval volume human-sized; transitive risk is
  still *reported* by `check` (`footprint.dependencies`) but not enforced.

## Alternatives considered

- **Stateless report only** — weak differentiation vs npq/Socket; rejected.
- **Ledger + GitHub Action in v1** — Action packaging/permissions add a
  second product surface before the core loop is proven; deferred.
- **Transitive ledger** — hundreds of approvals per project would cause
  rubber-stamping; safety theater; rejected for v1.
- **Semver-range approvals** — silently trusts future releases, defeating
  the compromised-maintainer defense; rejected for v1 (may return as opt-in
  policy in v2).
