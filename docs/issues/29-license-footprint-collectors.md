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
   - `depsdev.getDependencies(system, name, version)` ⇒
     `{ directCount, transitiveCount }`;
   - `"not-indexed"` ⇒ `unavailable(not-indexed)`; `"invalid-response"` ⇒
     `unavailable(source-error)`.
3. `footprint.install-size`:
   - npm: from `facts.distribution` ⇒ `{ unpackedSize?, fileCount? }`;
     both absent ⇒ `unavailable(no-registry-data)`; present ⇒ evaluated
     (value may carry only one field).
   - PyPI: `skipped(not-applicable)` (wheel sizes are per-file; deferred).
4. Evidence: deps.dev URL
   (`https://deps.dev/<system-lowercase>/<name>/<version>/dependencies`)
   for footprint; registry URL for license/size; sentences with concrete
   numbers ("Resolves to 3 direct and 24 transitive dependencies.").

## Acceptance Criteria

- [ ] License matrix: `MIT` valid; `(MIT OR Apache-2.0)` valid;
      `SEE LICENSE IN license.txt` invalid; `WTFPL` (not in common list)
      invalid-but-shown; null.
- [ ] Footprint happy path from deps.dev fixture; both unavailable variants.
- [ ] install-size npm evaluated / partial / unavailable; PyPI skipped.
- [ ] deps.dev called once per collect for footprint (spy); license
      collector makes zero network calls.

## Validation

- `npm test -- collectors/license collectors/footprint`.

## Dependencies

- 19, 16; facts from 11/15.

## Non-goals

- No full SPDX grammar/spec conformance, no license *compatibility*
  analysis, no PyPI size aggregation.

## Design References

- DESIGN.md §9.2 rows 19–22 (license, footprint)
