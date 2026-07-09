# Title

Top-packages dataset & update script

## Summary

Create the committed `data/top-packages/{npm,pypi}.json` name lists and the
maintenance script `scripts/update-top-packages.ts` that regenerates them
from public datasets, including license verification of the upstream
sources.

## Context

The typosquat check (28) must be deterministic and offline (ADR-006 #5), so
reference lists ship inside the package. Sources: `npm-high-impact` (npm)
and hugovk/top-pypi-packages (PyPI). Upstream licensing is known unknown U6
— this issue resolves it.

## Scope

- `scripts/update-top-packages.ts`, `data/top-packages/npm.json`,
  `data/top-packages/pypi.json`, loader `src/core/signals/top-packages.ts`,
  tests.

## Detailed Requirements

1. Data file format (both ecosystems):
   ```json
   {
     "ecosystem": "npm",
     "generatedAt": "2026-07-08T00:00:00Z",
     "source": { "name": "npm-high-impact", "version": "x.y.z", "url": "…", "license": "…" },
     "count": 5000,
     "names": ["…", "…"]
   }
   ```
   `names`: normalized (npm as-is, PyPI PEP 503), deduped, sorted, capped
   at 5,000.
2. Update script (dev-only; runs under `tsx`, may use dev-deps and plain
   `fetch` — it is NOT part of the shipped CLI and not subject to the
   runtime allowlist, but must state this in a header comment):
   - npm: import the name list from the `npm-high-impact` package (add as a
     **devDependency**) — use its published data export; record its version.
   - PyPI: fetch the latest 30-days JSON from the canonical
     top-pypi-packages location (the repo's published JSON; take the top
     5,000 rows' `project` fields).
   - **License verification step (U6)**: the script fetches/records each
     upstream's declared license into `source.license` and FAILS if the
     license is missing or forbids redistribution. Document findings in the
     PR (expected: npm-high-impact MIT; top-pypi-packages data is
     CC0/public-domain-ish — verify and cite).
   - Deterministic output: stable sort, fixed key order, LF, trailing
     newline (so reruns produce minimal diffs).
3. Loader `top-packages.ts`:
   - `loadTopPackages(ecosystem): TopPackagesIndex` reading the packaged
     JSON (resolved relative to the module, works from `dist/`);
   - `TopPackagesIndex` = `{ has(name): boolean; all(): readonly string[]; count: number }`
     (nearest-name search lives in issue 28, not here);
   - zod-validates the file; corrupt data file ⇒ `InternalError` (ships
     broken = bug).
4. Commit the two generated files (initial generation run by the
   implementer; sizes expected ≲ 150 KB each).

## Acceptance Criteria

- [ ] Both data files committed, valid against the loader schema,
      `count === names.length`, sorted/deduped (tests assert invariants,
      not specific names — except spot-checks: `lodash` ∈ npm list,
      `requests` ∈ pypi list).
- [ ] `npm run build` ships the files (pack-audit from issue 02 includes
      `data/`).
- [ ] Loader works from built output (test against `dist/` path resolution
      or an equivalent import.meta.url strategy).
- [ ] Update script rerun produces zero diff when upstream unchanged
      (deterministic serialization).
- [ ] `source.license` fields populated with verified values; PR documents
      the verification evidence (links).

## Validation

- `npm test -- top-packages`; run the update script once and commit output;
  include script run log in PR.

## Dependencies

- 01.

## Non-goals

- No runtime fetching of lists, no scheduled refresh workflow (v2), no
  similarity search (28).

## Design References

- DESIGN.md §9.3, §20.3; ADR-006 #5; research §3; U6
