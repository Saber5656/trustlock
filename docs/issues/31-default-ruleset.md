# Title

Default ruleset (22 rules)

## Summary

Implement `src/core/rules/default-rules.ts`: all 22 v1 rules from DESIGN
§10.2 with their trigger predicates, threshold constants, titles, and
detail-sentence builders — each covered by trigger / pass / not-evaluable
tests.

## Context

This file IS vetlock's opinion. Every threshold must be a named constant in
one table so users (and the signals reference doc, issue 41) can see exactly
what the tool believes. Rule ids are a public, stable API (referenced in
ledger policy overrides).

## Scope

- `src/core/rules/default-rules.ts` + `test/rules/default-rules.test.ts`
  (table-driven).

## Detailed Requirements

1. Export `THRESHOLDS` (single frozen object):
   `packageAgeMinDays: 30`, `versionAgeMinDays: 7`,
   `dormancyGapDays: 540`, `dormancyRecentDays: 14`,
   `stalePushDays: 730`, `lowDownloads: 500`,
   `bigFootprintTransitive: 100`, `releasesBehindNotice: 2`.
2. Implement exactly the 22 rules of DESIGN §10.2. Normative trigger
   predicates (each rule reads exactly ONE signal; value shapes per issues
   20–29):

   | Rule | Signal | Triggered when |
   |---|---|---|
   | R-MAL-001 | vulnerabilities.malicious | `count > 0` |
   | R-TYPO-001 | name.typosquat | `suspect === true` |
   | R-VULN-001 | vulnerabilities.known | `count > 0` |
   | R-AGE-001 | metadata.package-age | `ageDays < packageAgeMinDays` |
   | R-AGE-002 | metadata.version-age | `ageDays < versionAgeMinDays` |
   | R-DEP-001 | metadata.deprecated | `deprecated === true` |
   | R-DEP-002 | metadata.yanked | `yanked === true` |
   | R-MNT-001 | maintainers.publisher-change | `priorPublishCount === 0` |
   | R-MNT-002 | maintainers.count | `count === 1` |
   | R-REPO-001 | repository.declared | `url === null` |
   | R-REPO-002 | repository.status | `exists === false` |
   | R-REPO-003 | repository.status | `archived === true` |
   | R-REPO-004 | repository.status | `lastPushAgeDays !== undefined && lastPushAgeDays >= stalePushDays` (missing field ⇒ pass — partial-data honesty) |
   | R-PROV-001 | provenance.attestation | `present === false` |
   | R-EXEC-001 | execution.install-scripts | `present === true` |
   | R-EXEC-002 | execution.sdist-only | `sdistOnly === true` |
   | R-EXEC-003 | execution.bin-entries | `bins.length > 0` |
   | R-CAD-001 | metadata.release-cadence | `gapBeforeLatestDays !== null && gapBeforeLatestDays >= dormancyGapDays && latestReleaseAgeDays <= dormancyRecentDays` |
   | R-POP-001 | popularity.downloads | `count < lowDownloads` |
   | R-LIC-001 | license.declared | `license === null \|\| spdxValid === false` |
   | R-FOOT-001 | footprint.dependencies | `transitiveCount > bigFootprintTransitive` |
   | R-DRIFT-001 | metadata.latest-drift | `isLatest === false && behindCount >= releasesBehindNotice` (release-count-based, cross-ecosystem) |

   Note: R-REPO-002 and R-REPO-003/004 share a signal — up to three rules
   may read `repository.status`; the one-signal-per-rule invariant is about
   each rule reading a single signal, not exclusivity.
3. Severities exactly as DESIGN §10.2 (R-MAL-001, R-TYPO-001 critical;
   R-VULN-001, R-AGE-001, R-DEP-001, R-DEP-002, R-REPO-002, R-REPO-003,
   R-EXEC-001, R-CAD-001 warn; R-MNT-002, R-EXEC-003 info; the rest notice).
4. Titles: short imperative-free noun phrases ("Known malicious version",
   "Very new package", "Install scripts present"). `detail(signal)` builds
   one sentence embedding concrete values; for R-MAL-001/R-VULN-001 include
   advisory ids (≤ 3, then "and N more").
5. Export `DEFAULT_RULES: Rule[]`. Static meta-tests assert: ids unique;
   every `signalId` ∈ `SIGNAL_CATALOG`; severity table matches DESIGN §10.2
   (pin literally); every rule id appears in the behavior-test table.

## Acceptance Criteria

- [ ] For EACH of the 22 rules: a triggering fixture, a passing fixture,
      and a not-evaluable case (signal `unavailable`) — table-driven,
      ≥ 66 cases.
- [ ] Threshold boundary tests: ageDays 29/30 and 6/7, downloads 499/500,
      gap 539/540 × latest-age 14/15, push-age 729/730, transitive 100/101,
      behindCount 1/2.
- [ ] R-REPO-004 with `lastPushAgeDays` absent ⇒ pass (not not-evaluable).
- [ ] R-DEP-001 never fires for PyPI subjects (its signal is skipped there —
      engine excludes it; integration-style test through `evaluate`).
- [ ] Meta-tests from requirement 5 all green.

## Validation

- `npm run lint && npm run typecheck && npm test -- rules/default`; paste the generated rule/severity table in
  the PR description for human review.

## Dependencies

- 30 (engine/types); 20–26, 28, 29 (signal value shapes).

## Non-goals

- No policy parsing, no doc generation (41 documents rules), no new
  signals, no threshold configurability beyond policy severity remaps.

## Design References

- DESIGN.md §10.2 (rule table), §9.2 (value shapes); ADR-003
