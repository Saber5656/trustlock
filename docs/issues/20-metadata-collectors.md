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
5. `metadata.latest-drift`: requires BOTH `facts.latestVersion` and
   `facts.releaseDates` ⇒ `{ latestVersion, isLatest, behindCount }`;
   `behindCount` = number of release dates strictly after the subject
   version's date (date-based, so it works cross-ecosystem without version
   math); subject IS latest ⇒ `{ isLatest: true, behindCount: 0 }`.
   Unavailable cases: `latestVersion` or `releaseDates` fact missing ⇒
   `unavailable(no-registry-data)`; subject version absent from
   `releaseDates` ⇒ `unavailable(subject-version-not-in-release-history)`.
6. `metadata.deprecated`: from `facts.deprecated` — value mapping
   `facts.deprecated.value.flag → value.deprecated`,
   `facts.deprecated.value.message → value.message` (message
   sanitizer-bound at render). For PyPI this signal is pre-marked
   `skipped` by the orchestrator (adapter `notApplicableSignals`, issue
   19) — the collector contains NO ecosystem logic and simply never
   receives the id there.
7. `metadata.yanked`: from `facts.yanked` — mapping
   `facts.yanked.value.flag → value.yanked`,
   `facts.yanked.value.reason → value.reason`. Pre-marked `skipped` for
   npm by the orchestrator, same mechanism.
8. Evidence: normative — each signal's `evidence.url` is the `sourceUrl` of
   the fact(s) it consumed (cadence and drift use
   `releaseDates.sourceUrl`); the summary is a sentence with the concrete
   values, e.g.
   `"Latest release is 5.1.0 (published 2026-05-02); requested 4.18.2 has 14 newer releases."`.

## Acceptance Criteria

- [ ] Six signals emitted for an npm facts fixture (yanked/deprecated
      skip behavior is owned and tested by the orchestrator — this
      collector's tests only cover the evaluated/unavailable paths of the
      signals it is routed).
- [ ] Age math: fixed `now` fixture asserts exact `ageDays` values,
      including same-day (0) and leap-ish boundaries (floor semantics).
- [ ] Cadence: fixtures for 0/1/many releases; releasesLast12mo counts only
      releases within 365 days of `now`.
- [ ] Drift: subject==latest, subject older by N, subject newer than
      latest-tag (possible with dist-tags; ⇒ isLatest false, behindCount 0 —
      lock this edge in a test), subject missing from release history ⇒
      the documented unavailable reason.
- [ ] Missing facts produce `unavailable`, never throws; validateSignalOutput
      helper passes.

## Validation

- `npm run lint && npm run typecheck && npm test -- collectors/metadata`.

## Dependencies

- 19, 11 (npm facts fixtures); PyPI-shaped cases additionally use 15
  fixtures. (ISSUE_PLAN table: 19, 11, 15.)

## Non-goals

- No rule thresholds (issue 31 owns "what is *too* new"), no extra fetches.

## Design References

- DESIGN.md §9.2 rows 1–6
