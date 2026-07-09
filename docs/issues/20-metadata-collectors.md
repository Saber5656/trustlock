# Title

Metadata collectors (6 signals)

## Summary

Implement `src/core/signals/collectors/metadata.ts`: the collector producing
`metadata.package-age`, `metadata.version-age`, `metadata.release-cadence`,
`metadata.latest-drift`, `metadata.deprecated`, and `metadata.yanked` from
`PackageFacts` alone (no extra network).

## Context

These are the cheapest, highest-coverage signals (DESIGN §9.2) and feed six
rules (R-AGE-001/002, R-CAD-001, R-DRIFT-001, R-DEP-001). They must work
identically for npm and PyPI via the normalized facts.

## Scope

- One collector file + unit tests. `produces` = the six ids above.

## Detailed Requirements

1. Time handling: the collector receives `now: Date` via its constructor
   (`createMetadataCollector(now)`) — never reads the clock itself
   (deterministic tests; the CLI passes real now).
2. `metadata.package-age`: from `facts.firstPublishedAt` ⇒
   `{ firstPublishedAt, ageDays }` (floor of (now − ts)/86400s). Missing
   fact ⇒ `unavailable(no-registry-data)`.
3. `metadata.version-age`: same from `facts.publishedAt`.
4. `metadata.release-cadence`: from `facts.releaseDates` ⇒
   `{ releasesLast12mo, gapBeforeLatestDays, latestReleaseAgeDays }` where
   `gapBeforeLatestDays` = days between the latest release date and the one
   before it (0 or 1 release ⇒ gap null, other fields computed; empty ⇒
   unavailable) and `latestReleaseAgeDays` = days from the latest release
   date to `now` (consumed by rule R-CAD-001, which reads only this signal).
5. `metadata.latest-drift`: from `facts.latestVersion` + subject version ⇒
   `{ latestVersion, isLatest, behindCount }`; `behindCount` = number of
   release dates strictly after the subject version's date (date-based, so
   it works cross-ecosystem without version math); if the subject IS latest
   ⇒ `{ isLatest: true, behindCount: 0 }`.
6. `metadata.deprecated`: npm — from `facts.deprecated`
   (`{ deprecated, message? }`, message sanitizer-bound at render);
   PyPI — `skipped(not-applicable)`.
7. `metadata.yanked`: PyPI — from `facts.yanked`; npm —
   `skipped(not-applicable)`.
8. Evidence: each evaluated signal cites the registry page URL
   (`registryPageUrl` passed via ctx subject? — **normative**: evidence URL =
   the fact's `sourceUrl`) and a sentence with the concrete values, e.g.
   `"Latest release is 5.1.0 (published 2026-05-02); requested 4.18.2 has 14 newer releases."`.

## Acceptance Criteria

- [ ] Six signals emitted for an npm facts fixture; `metadata.yanked`
      skipped for npm, `metadata.deprecated` skipped for PyPI (and vice
      versa evaluated).
- [ ] Age math: fixed `now` fixture asserts exact `ageDays` values,
      including same-day (0) and leap-ish boundaries (floor semantics).
- [ ] Cadence: fixtures for 0/1/many releases; releasesLast12mo counts only
      releases within 365 days of `now`.
- [ ] Drift: subject==latest, subject older by N, subject newer than
      latest-tag (possible with dist-tags; ⇒ isLatest false, behindCount 0 —
      lock this edge in a test).
- [ ] Missing facts produce `unavailable`, never throws; validateSignalOutput
      helper passes.

## Validation

- `npm test -- collectors/metadata`.

## Dependencies

- 19; facts fixtures from 11/15.

## Non-goals

- No rule thresholds (issue 31 owns "what is *too* new"), no extra fetches.

## Design References

- DESIGN.md §9.2 rows 1–6
