# ADR-002: TypeScript on Node.js, distributed via npm; MIT license

Status: accepted · 2026-07-08
Deciders: repository owner (human: language choice), Fable (details)

## Context

The tool targets developers adding npm and PyPI dependencies. Candidate
stacks: TypeScript/Node (npx distribution), Go (single binary), Rust.
Implementation will be delegated to lower-capability coding agents, so
ecosystem maturity and unambiguous idioms matter.

## Decision

- **TypeScript, strict mode, ESM-only, Node.js ≥ 22.12.0** (Node 20 is EOL
  since April 2026; ≥ 22.12 guarantees global `fetch` and `util.styleText`).
- Distributed as npm package **`vetlock`** with `bin: vetlock`; primary
  invocation `npx vetlock …` for the npm-developer audience.
- Runtime dependency budget: exactly `commander`, `zod`, `semver`,
  `smol-toml` (justification table in DESIGN.md §4.1). Adding a runtime
  dependency requires a new ADR — a dependency-vetting tool must exemplify
  dependency discipline.
- No bundler in v1; `tsc` output shipped.
- License: **MIT** (maximally adoptable for a small CLI; consistent with the
  surrounding ecosystem). ⚠ Flagged for explicit owner confirmation in the
  design PR.

## Consequences

- PyPI-side users need Node/npx installed; accepted for v1 (documented in
  README). A brew formula / single-binary build is a v2 option.
- `util.styleText` replaces chalk; global fetch replaces axios/got — fewer
  deps, but contributors must know the stdlib equivalents (CONTRIBUTING.md
  notes this).

## Alternatives considered

- **Go**: best distribution story (single binary, brew), but npx-native
  reach to the primary npm audience is worse, and the user's agent tooling
  is strongest in TypeScript. Rejected for v1.
- **Rust**: highest robustness, highest implementation cost and slowest
  issue throughput for delegated agents. Rejected.
