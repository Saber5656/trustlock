# ADR-004: Multi-ecosystem adapter architecture; npm + PyPI in v1

Status: accepted · 2026-07-08
Deciders: repository owner (human), Fable (design agent)

## Context

v1 targets npm and PyPI (owner decision 2026-07-08), with the explicit
instruction that **issues and architecture must assume many ecosystems**,
so later additions (Cargo, Go, RubyGems, Maven…) do not force a redesign.

## Decision

- Every ecosystem-specific behavior sits behind the `EcosystemAdapter`
  interface (DESIGN.md §7): name validation/normalization, spec body
  parsing, version ordering, registry-of-record access, manifest reading,
  and the per-ecosystem ids used by shared enrichment sources
  (`depsDevSystem`, `osvEcosystem`).
- Core modules (`spec`, `signals`, `rules`, `report`, `ledger`, `verify`)
  are ecosystem-agnostic; an architectural test asserts no ecosystem-id
  conditionals exist outside `core/ecosystems/`.
- Cross-ecosystem enrichment (deps.dev, OSV, GitHub, download stats) lives
  in collectors, parameterized by adapter-provided ids — deps.dev's own
  system enum (`NPM`, `PYPI`, `CARGO`, …) confirms this split maps cleanly
  to at least seven ecosystems.
- npm is the reference implementation; PyPI is implemented in the same wave
  specifically to keep the interface honest (two concrete implementations
  before the interface freezes).
- The ledger schema carries `ecosystem` on every entry from v1 — no
  migration needed when ecosystems are added.

## Consequences

- Slightly more indirection than an npm-only tool; accepted.
- Ecosystem-asymmetric facts (npm maintainer lists vs PyPI free-text
  strings) must be expressed as *optional* facts with honest `skipped` /
  `unavailable` signals rather than papered over.
- Adding an ecosystem later = new adapter directory + top-packages data
  file + fixtures; the issue plan reserves this as a repeatable template.
