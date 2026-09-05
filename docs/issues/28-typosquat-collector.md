# Title

Typosquat collector

## Summary

Implement `src/core/signals/collectors/typosquat.ts` plus the
Damerau-Levenshtein similarity module: produce `name.typosquat` by comparing
the subject name against the bundled top-packages list, offline and
deterministically.

## Context

Typosquatting is a critical-severity, high-precision check when done
conservatively (R-TYPO-001). Algorithm is fixed in DESIGN §9.3; ground-truth
test vectors come from the ecosyste-ms typosquatting dataset
(research/data-sources.md §3).

## Scope

- `src/core/signals/collectors/typosquat.ts`,
  `src/core/signals/similarity.ts` (hand-rolled Damerau-Levenshtein,
  no new dependency), tests with real-world typosquat vectors.
  `produces`: `name.typosquat`.

## Detailed Requirements

1. `similarity.ts`:
   - `damerauLevenshtein(a, b): number` — optimal string alignment variant
     (substitution, insertion, deletion, adjacent transposition), O(len a ×
     len b), early-exit when a length difference alone exceeds the max
     relevant distance (2);
   - pure, fully unit-tested (`kitten→sitting = 3`, transposition
     `ab→ba = 1`, empty strings, unicode passthrough by code unit —
     document that comparison operates on normalized ASCII-ish names).
2. Normalization for comparison (applied to subject and list names):
   lowercase → strip npm scope (`@scope/name` → `name`, scope recorded) →
   remove all `-`, `_`, `.` characters. The raw normalized name (without
   separator-stripping) is also compared, and the flag fires if EITHER
   representation matches the criteria (catches `python-dateutil` vs
   `python_dateutil` style squats and `requests` vs `requsts` style typos).
3. Decision (DESIGN §9.3; value shape per DESIGN §9.2 is
   `{ suspect, isPopular, nearest?, distance? }`):
   - if subject name ∈ top list (either representation, exact) ⇒
     `{ suspect: false, isPopular: true }` — popular packages are exempt;
   - else compute min distance over the list:
     threshold = 1 for stripped-length ≤ 7, else 2;
     distance ≤ threshold ⇒
     `{ suspect: true, nearest: <top name>, distance }`;
   - separator-stripped equality (distance 0 on stripped form, different
     raw name) is always suspect regardless of length;
   - else `{ suspect: false, isPopular: false }`.
4. Performance: full scan of 5,000 names with early-exit must stay < 50 ms
   (perf smoke test with generous bound; no index structure needed in v1).
5. Evidence: suspect ⇒
   `"Name is within edit distance <d> of popular package '<nearest>'."`;
   non-suspect popular ⇒
   `"Name is itself among the top <count> packages."`. In both cases
   `evidence.url` = `ctx.topPackages.source.url` (attribution provided by
   the issue-27 index shape).
6. The collector consumes `ctx.topPackages`, which is guaranteed valid:
   loading happens once in the check-command wiring (issue 39) and a
   corrupt shipped data file fails fast there with `InternalError`
   (issue 27). This collector has no dataset-missing path.

## Acceptance Criteria

- [ ] Ground-truth vectors (≥ 10 from the ecosyste-ms dataset, committed as
      a fixture with attribution): each known squat flags against its
      legitimate target when the target is in the bundled list (inject a
      test list containing the targets).
- [ ] Exemption: `lodash` itself ⇒ suspect false, isPopular true.
- [ ] `lodahs` (transposition) ⇒ suspect, nearest lodash, distance 1.
- [ ] `python-dateutil` vs `python_dateutil` separator squat caught.
- [ ] Short-name threshold: distance-2 match on a 5-char name does NOT
      flag; distance-2 on a 12-char name does.
- [ ] Scoped subject `@evil/lodahs` compares scope-stripped and flags.
- [ ] Perf smoke: < 50 ms over 5,000 names.

## Validation

- `npm run lint && npm run typecheck && npm test -- typosquat similarity`.

## Dependencies

- 19, 27.

## Non-goals

- No homoglyph/keyboard-adjacency models (v2 candidates), no embedding
  search, no online lookups, no flag on packages that are IN the list.

## Design References

- DESIGN.md §9.3; research §3; R-TYPO-001 (§10.2)
