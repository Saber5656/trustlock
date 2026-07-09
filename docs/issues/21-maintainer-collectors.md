# Title

Maintainer collectors (2 signals)

## Summary

Implement `src/core/signals/collectors/maintainers.ts`: produce
`maintainers.count` and `maintainers.publisher-change` from PackageFacts
(npm), with honest `skipped` for PyPI where the registry exposes no
structured maintainer data.

## Context

Single-maintainer packages and first-time publishers of a new version are
classic compromise/handover indicators (npq's marshalls, R-MNT-001/002).
PyPI's JSON API only has free-text author/maintainer strings — vetlock does
not pretend otherwise (ecosystem-honest reporting, ADR-004 consequence).

## Scope

- One collector file + unit tests. `produces`:
  `maintainers.count`, `maintainers.publisher-change`.

## Detailed Requirements

1. `maintainers.count`:
   - npm: from `facts.maintainers` ⇒ `{ count, names }` (names capped at 10
     for the value; full count kept). Missing fact ⇒
     `unavailable(no-registry-data)`.
   - PyPI: `skipped(not-applicable)` with evidence summary explaining "PyPI
     does not expose a structured maintainer list".
2. `maintainers.publisher-change`:
   - npm: from `facts.latestPublisher` ⇒
     `{ latestPublisher, priorPublishCount }`. Note the semantics
     (computed in issue 11): priorPublishCount = publishes by this user to
     THIS package before the subject version.
   - PyPI: `skipped(not-applicable)`.
3. Evidence: registry URL; sentences like
   `"3 maintainers: alice, bob, carol."` /
   `"Version 5.1.0 was published by dougwilson (42 prior releases of this package)."`
   — names flow through the renderer sanitizer later; the collector stores
   raw values.

## Acceptance Criteria

- [ ] npm fixture with 3 maintainers and a veteran publisher ⇒ both
      evaluated with exact values.
- [ ] npm fixture where the latest publisher has `priorPublishCount: 0` ⇒
      value preserved (rule R-MNT-001 will trigger on it — no judgement in
      the collector).
- [ ] PyPI facts ⇒ both signals `skipped(not-applicable)`.
- [ ] Missing `maintainers`/`latestPublisher` facts on npm ⇒ `unavailable`.
- [ ] Names list capped at 10 in value; count remains true count (fixture
      with 12 maintainers).

## Validation

- `npm test -- collectors/maintainers`.

## Dependencies

- 19; facts from 11/15.

## Non-goals

- No npm user-profile fetches, no email-domain analysis (v2 candidate), no
  PyPI author-string heuristics.

## Design References

- DESIGN.md §9.2 rows 7–8; research/data-sources.md §2.2 caveats
