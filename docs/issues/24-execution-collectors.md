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

1. Ecosystem applicability is owned by the orchestrator via adapter
   `notApplicableSignals` (issue 19): install-scripts/bin-entries are
   pre-marked `skipped` for PyPI, sdist-only for npm. This collector
   contains NO ecosystem logic — it implements only evaluated/unavailable
   paths from facts.
2. `execution.install-scripts`: from `facts.installScripts` ⇒
   `{ present, names }`. The names-empty-but-present form
   (hasInstallScript fallback, issue 11) yields
   `{ present: true, names: [] }` with evidence noting the exact script
   names are unavailable. Missing fact ⇒ `unavailable(no-registry-data)`.
3. `execution.sdist-only`: from `facts.pythonDistribution` ⇒
   `{ sdistOnly, hasWheel }`; when sdistOnly the evidence explains the
   implication ("sdist-only releases require building from source and may
   execute package build code during installation" — PEP 517 backends, not
   only `setup.py`). Missing fact ⇒ `unavailable(no-registry-data)`.
4. `execution.bin-entries`: from `facts.binEntries` ⇒ `{ bins }` (empty
   array = evaluated, no bins). Missing fact ⇒
   `unavailable(no-registry-data)`.
5. Evidence URLs: each consumed fact's `sourceUrl` (packument / PyPI JSON).

## Acceptance Criteria

- [ ] npm fixture with `postinstall` ⇒ install-scripts
      `{present: true, names:["postinstall"]}`.
- [ ] hasInstallScript-only fixture ⇒ `{present: true, names: []}` +
      distinct evidence sentence.
- [ ] npm without scripts ⇒ `{present: false, names: []}` evaluated.
- [ ] PyPI wheel+sdist ⇒ `{sdistOnly: false, hasWheel: true}`; sdist-only
      fixture ⇒ `{sdistOnly: true, hasWheel: false}`.
- [ ] Missing-fact ⇒ `unavailable(no-registry-data)` for each signal
      (skips are the orchestrator's, tested in issue 19).
- [ ] S1/no-network: no-arg factory; the module imports no infra/client
      modules and no `child_process` (grep-level test as in issue 23); it
      never installs, builds, or executes anything — only `ctx.facts`
      reads (spying ctx proves zero `ctx.infra` access).

## Validation

- `npm run lint && npm run typecheck && npm test -- collectors/execution`.

## Dependencies

- 19, 11, 15. (ISSUE_PLAN table lists the same.)

## Non-goals

- No tarball inspection (would require downloading the artifact — v2
  discussion at best), no script *content* analysis (GuardDog territory,
  ADR-003).

## Design References

- DESIGN.md §9.2 rows 13–15; ADR-003 (why no content analysis)
