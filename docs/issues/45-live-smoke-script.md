# Title

Live-API smoke script (manual pre-release QA)

## Summary

Implement `scripts/smoke-live.ts`: a manually-run script that exercises the
built CLI against the real registry/enrichment APIs for known-good subjects
and prints a pass/fail checklist. Never runs in CI.

## Context

CI is fixture-only by design (DESIGN §17), which means API drift (schema
changes, endpoint moves — known unknowns U1/U3/U5) would otherwise be
discovered by users. This script is the release-gate reality check
(RELEASING.md references it; ISSUE_PLAN §8).

## Scope

- `scripts/smoke-live.ts` (tsx, dev-only) + a line in RELEASING.md (added
  here if issue 43 already merged, or coordinated in whichever lands
  second).

## Detailed Requirements

1. Subjects: `npm:express` (popular, provenance expected),
   `npm:left-pad` (legacy), `pypi:requests` (popular),
   `pypi:flask` (trusted-publishing attestations expected).
2. For each subject the script runs the BUILT CLI
   (`node dist/cli/index.js check <subject> --json`) in a fresh temp cache
   dir (real network) and asserts structural expectations — NOT specific
   verdicts (real data drifts):
   - exit code ∈ {0, 1};
   - report parses against `reportSchema`;
   - `incomplete === false` (all sources reachable) — if true, list which
     signals were unavailable and mark the check ⚠ (network flakiness is
     distinguishable from schema breakage);
   - per-subject expectations: express/requests have
     `popularity.downloads` evaluated with count > 10_000;
     `vulnerabilities.*` evaluated; `repository.status.exists === true`;
     provenance `evaluated` (value may vary).
3. Also exercises: `verify` + `approve` round-trip in a temp fixture
   project (network only for approve's resolveVersion), and one deliberate
   failure (`check npm:this-package-should-not-exist-vetlock-smoke` ⇒ exit
   2 with RegistryError message).
4. Output: checklist table to stdout (`✓/⚠/✗ subject — detail`), exit 0
   only when all ✓ (⚠ exits 1 with a "rerun / investigate" note; ✗ exits 1).
5. Guardrails: refuses to run when `CI` env var is set (prints why);
   respects `VETLOCK_GITHUB_TOKEN` if present (documents that unauth runs
   may show GitHub signals ⚠ due to rate limits).
6. Runtime target < 90 s; sequential subjects (politeness; no hammering).

## Acceptance Criteria

- [ ] Script runs green locally against live APIs (paste the checklist
      output in the PR).
- [ ] `CI=1` refusal path works.
- [ ] Nonexistent-package expectation asserts exit 2 + message.
- [ ] No fixtures or CI wiring added; `npm run smoke:live` script alias
      registered in package.json (dev-only section comment).
- [ ] RELEASING.md references the script as a pre-publish gate.

## Validation

- Manual run log in PR (the point of the issue); lint/typecheck green.

## Dependencies

- 39 (check), 37 (verify), 36 (approve); 43 (RELEASING.md reference —
  soft dependency, coordinate ordering).

## Non-goals

- No scheduled/nightly automation (v2 decision), no fixture recording (a
  separate dev helper exists per issue 09), no performance measurement.

## Design References

- DESIGN.md §17 (live smoke row), §18; U1/U3/U5; ISSUE_PLAN §8
