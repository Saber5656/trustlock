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

1. Source selection by ecosystem:
   - npm: `npmRegistry.fetchDownloads(name)` ⇒
     `{ period: "last-week", count }`;
   - pypi: `pypiRegistry.fetchDownloads(name)` ⇒
     `{ period: "last-month", count }`.
   The collector receives the download-capable client through
   `SignalContext.infra` — add a narrow
   `downloads: { fetch(ecosystem, name): Promise<{period, count} | null> }`
   facade in the check-command wiring (issue 39) so the collector stays
   ecosystem-agnostic; define the facade type HERE and let 39 implement the
   wiring (put the type next to the collector).
2. `null` from the source (no stats / community-service failure) ⇒
   `unavailable(no-download-data)` — evidence sentence notes that brand-new
   packages have no stats yet.
3. Evaluated evidence: `"<count> downloads in the <period>."` with URL
   `https://www.npmjs.com/package/<name>` or
   `https://pypistats.org/packages/<name>`.
4. Counts are absolute integers; no bucketing/judgement here (rule R-POP-001
   owns the threshold).

## Acceptance Criteria

- [ ] npm path: `{period:"last-week"}` with fixture count; pypi path:
      `{period:"last-month"}`.
- [ ] null source result ⇒ unavailable with the documented reason (both
      ecosystems).
- [ ] Facade type is ecosystem-agnostic (compile-level: collector file
      contains no `"npm"`/`"pypi"` literals — extends the issue-07
      architecture guard to `collectors/` and verify it catches this file
      if violated).
- [ ] Exactly one downloads fetch per collect.

## Validation

- `npm test -- collectors/popularity`; architecture guard updated & green.

## Dependencies

- 19; 09, 13 (download methods), wiring finalized in 39.

## Non-goals

- No dependents counts (deferred — DESIGN §3.3), no trend analysis, no
  version-level downloads.

## Design References

- DESIGN.md §9.2 row 18; U2
