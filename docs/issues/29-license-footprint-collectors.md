# Title

License & footprint collectors (3 signals)

## Summary

Implement `src/core/signals/collectors/license.ts` (`license.declared`) and
`src/core/signals/collectors/footprint.ts` (`footprint.dependencies`,
`footprint.install-size`): licensing hygiene and dependency-weight signals.

## Context

Missing/invalid licenses are an adoption and legal-hygiene flag (R-LIC-001
notice); a large transitive footprint is a risk-surface note (R-FOOT-001)
that also explains WHY vetlock's direct-only ledger still shows users what
they are pulling in.

## Scope

- Two collector files + a minimal SPDX-expression checker + tests.
  `produces`: `license.declared` | `footprint.dependencies`,
  `footprint.install-size`.

## Detailed Requirements

1. `license.declared`:
   - from `facts.license` ⇒ `{ license: string | null, spdxValid: boolean }`.
   - **Absent vs null (DESIGN §7.3)**: `facts.license` fact missing
     entirely ⇒ `unavailable(no-registry-data)`; fact present with
     `value: null` ⇒ evaluated `{ license: null, spdxValid: false }`
     ("no license declared" is a real answer).
   - SPDX validity: implement `isLikelySpdx(expr)` — tokenizes the
     expression (`AND`, `OR`, `WITH`, parentheses) and checks each license
     token against a bundled list of the ~80 most common SPDX ids
     (committed as `src/core/signals/spdx-common.json`; include the id list
     in the PR for review). Unknown-but-well-formed tokens ⇒
     `spdxValid: false` (conservative; evidence shows the raw string).
     This is a *hygiene* check, not legal advice — the evidence sentence
     must say so.
   - null license ⇒ evaluated with `{ license: null, spdxValid: false }`.
2. `footprint.dependencies`:
   - `ctx.infra.depsdev.getDependencies(ctx.subject.depsDevSystem,
     ctx.subject.name, ctx.subject.version)` ⇒
     `{ directCount, transitiveCount }`;
   - `"not-indexed"` ⇒ `unavailable(not-indexed)`; `"invalid-response"` ⇒
     `unavailable(source-error)`; thrown `NetworkError` family (incl.
     `OfflineMissError`) ⇒ `unavailable(<error class name>)` — caught in
     the collector (honest degradation, tests for 5xx-exhausted and
     offline-miss).
3. `footprint.install-size`:
   - from `facts.distribution` ⇒ `{ unpackedSize?, fileCount? }`; fact
     absent ⇒ `unavailable(no-registry-data)`; present ⇒ evaluated (value
     may carry only one field). For PyPI this signal is in the adapter's
     `notApplicableSignals` — the orchestrator pre-marks it `skipped`
     (issue 19), so this collector contains NO ecosystem branching (P6
     guard applies to it like any core module).
4. Evidence: deps.dev URL
   (`https://deps.dev/<system-lowercase>/<name>/<version>/dependencies`)
   for footprint; registry URL for license/size; sentences with concrete
   numbers ("Resolves to 3 direct and 24 transitive dependencies.").

## Acceptance Criteria

- [ ] License matrix: `MIT` valid; `(MIT OR Apache-2.0)` valid;
      `SEE LICENSE IN license.txt` invalid; `WTFPL` (not in common list)
      invalid-but-shown; `{value: null}` evaluated-null; fact-missing
      unavailable.
- [ ] Footprint happy path from deps.dev fixture; not-indexed,
      invalid-response, thrown-network, offline-miss variants each mapped.
- [ ] install-size evaluated / partial / fact-missing-unavailable (skip
      behavior belongs to the orchestrator).
- [ ] deps.dev called once per collect for footprint (spy); license
      collector makes zero network calls.

## Validation

- `npm run lint && npm run typecheck && npm test -- collectors/license collectors/footprint`.

## Dependencies

- 19, 16, 11, 15. (ISSUE_PLAN table lists the same.)

## Non-goals

- No full SPDX grammar/spec conformance, no license *compatibility*
  analysis, no PyPI size aggregation.

## Design References

- DESIGN.md §9.2 rows 19–22 (license, footprint)
