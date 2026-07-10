# Title

Popularity collector (download counts)

## Summary

Implement `src/core/signals/collectors/popularity.ts`: produce
`popularity.downloads` from the npm downloads API (last week) or pypistats
(last month), via the registry clients' download methods.

## Context

Very low download counts on a package a user is about to trust is a useful
caution (R-POP-001 notice) and, combined with the typosquat signal, a
classic squat indicator. pypistats is best-effort (U2).

## Scope

- One collector file + unit tests with faked registry clients. `produces`:
  `popularity.downloads`.

## Detailed Requirements

1. The collector calls `ctx.infra.downloads.fetch()` — the
   `DownloadsFacade` type is declared in `signals/types.ts` (issue 19):
   `{ fetch(): Promise<{ period: string; count: number; evidenceUrl: string } | null> }`.
   The facade is IMPLEMENTED in the check-command wiring (issue 39), which
   adapts the ecosystem-specific client shapes — npm
   `fetchDownloads → { weekly }` becomes
   `{ period: "last-week", count, evidenceUrl: "https://www.npmjs.com/package/<name>" }`;
   PyPI `fetchDownloads → { lastMonth }` becomes
   `{ period: "last-month", count, evidenceUrl: "https://pypistats.org/packages/<name>" }`.
   The collector therefore contains no ecosystem knowledge and no URL
   construction.
2. `null` from the facade (no stats / community-service failure) ⇒
   `unavailable(no-download-data)` — evidence sentence notes that brand-new
   packages have no stats yet.
3. Evaluated: value `{ period, count }`; evidence
   `"<count> downloads in the <period>."` with `evidence.url` =
   `evidenceUrl` from the facade.
4. Counts are absolute integers; no bucketing/judgement here (rule R-POP-001
   owns the threshold).

## Acceptance Criteria

- [ ] Facade returning `{period:"last-week"}` and `{period:"last-month"}`
      shapes each produce the matching evaluated value + evidence URL.
- [ ] null facade result ⇒ unavailable with the documented reason.
- [ ] Facade type is ecosystem-agnostic (compile-level: collector file
      contains no `"npm"`/`"pypi"` literals — extends the issue-07
      architecture guard to `collectors/` and verify it catches this file
      if violated).
- [ ] Exactly one downloads fetch per collect.

## Validation

- `npm run lint && npm run typecheck && npm test -- collectors/popularity`; architecture guard updated & green.

## Dependencies

- 19 (facade type + framework). The concrete facade wiring lands in 39
  (with 09/13 supplying the underlying client methods).
  (ISSUE_PLAN table: 19.)

## Non-goals

- No dependents counts (deferred — DESIGN §3.3), no trend analysis, no
  version-level downloads.

## Design References

- DESIGN.md §9.2 row 18; U2
