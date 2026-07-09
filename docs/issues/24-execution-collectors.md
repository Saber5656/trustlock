# Title

Execution-surface collectors (3 signals)

## Summary

Implement `src/core/signals/collectors/execution.ts`: produce
`execution.install-scripts` (npm), `execution.sdist-only` (PyPI), and
`execution.bin-entries` (npm) — the signals describing what code paths a
package gets to run on the installing machine.

## Context

Install scripts are the #1 npm attack vector (R-EXEC-001 warn);
sdist-only Python releases execute `setup.py`-time code at install
(R-EXEC-002 notice); bin entries shadow commands (R-EXEC-003 info). All
derive from facts already fetched — no network.

## Scope

- One collector file + unit tests. `produces`: the three ids above.

## Detailed Requirements

1. `execution.install-scripts`:
   - npm: from `facts.installScripts` ⇒ `{ present, names }`. The
     names-empty-but-present form (hasInstallScript fallback, issue 11)
     yields `{ present: true, names: [] }` with evidence noting the exact
     script names are unavailable.
   - PyPI: `skipped(not-applicable)` (evidence: "npm-specific check; see
     execution.sdist-only for the Python equivalent").
   - missing fact on npm ⇒ `unavailable(no-registry-data)`.
2. `execution.sdist-only`:
   - PyPI: from `facts.pythonDistribution` ⇒ `{ sdistOnly, hasWheel }`;
     evidence explains the implication when sdistOnly
     ("installation will execute build code from the sdist").
   - npm: `skipped(not-applicable)`.
3. `execution.bin-entries`:
   - npm: from `facts.binEntries` ⇒ `{ bins }` (empty array = evaluated,
     no bins).
   - PyPI: `skipped(not-applicable)` (entry_points not exposed by the JSON
     API in a reliable way — documented gap).
4. Evidence URLs: the facts' sourceUrls (packument / PyPI JSON).

## Acceptance Criteria

- [ ] npm fixture with `postinstall` ⇒ install-scripts
      `{present: true, names:["postinstall"]}`.
- [ ] hasInstallScript-only fixture ⇒ `{present: true, names: []}` +
      distinct evidence sentence.
- [ ] npm without scripts ⇒ `{present: false, names: []}` evaluated.
- [ ] PyPI wheel+sdist ⇒ `{sdistOnly: false, hasWheel: true}`; sdist-only
      fixture ⇒ `{sdistOnly: true, hasWheel: false}`.
- [ ] Cross-ecosystem skips exactly as specified (all three signals appear
      for both ecosystems, with correct statuses).
- [ ] No network (compile-level: uses only ctx.facts).

## Validation

- `npm test -- collectors/execution`.

## Dependencies

- 19; facts from 11/15.

## Non-goals

- No tarball inspection (would require downloading the artifact — v2
  discussion at best), no script *content* analysis (GuardDog territory,
  ADR-003).

## Design References

- DESIGN.md §9.2 rows 13–15; ADR-003 (why no content analysis)
