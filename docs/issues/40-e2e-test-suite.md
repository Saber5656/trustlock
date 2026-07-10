# Title

E2E CLI test suite & fixture projects

## Summary

Build the end-to-end test suite that executes the **built** CLI
(`node dist/cli/index.js`) as a child process against fixture projects
with a pre-seeded offline cache (no network, no server for the spawned
CLI), covering the full check → approve → verify loop, every verify
status, both ecosystems, hostile fixtures, and the performance budgets.

## Context

Unit/integration tests fake modules; this suite proves the assembled
product: flag parsing, stream separation, exit codes, file writes, and
cross-platform behavior on the CI matrix (DESIGN §17 e2e row).

## Scope

- `test/e2e/*.test.ts`, the fixture-cache seeding harness, fixture projects
  under `test/fixtures/projects/` (extending those from issues 10/14), npm
  script `test:e2e` (vitest project or separate config) wired into CI
  (issue 02's workflow gets the step added here).

## Detailed Requirements

1. Harness:
   - run the CLI via `execFile(node, [dist/cli/index.js, ...args])`
     (argv-array, no shell — the DESIGN §16.3 dev-code rule) with env
     `VETLOCK_CACHE_DIR=<tempdir>`;
   - network strategy (normative): the shipped code gets NO test-only host
     override — that would weaken S2. A fixture-cache builder pre-seeds
     the temp cache dir with entries (issue-05 on-disk format) for every
     URL each scenario needs (reusing `test/fixtures/http/**` bodies), and
     ALL e2e scenarios run with `--offline`. This keeps S2 intact (zero
     network in e2e — no fixture server exists in this suite) and
     exercises the cache+offline path as a bonus. Live-fetch behavior is
     covered at the integration layer with injected fetch, not here.
2. Scenarios (each asserts stdout, stderr, exit code, and file effects):
   1. `check express` (seeded cache) → exit 0; the JSON report parses
      against `reportSchema` with all 22 catalog signal ids present; the
      terminal report shows the verdict badge and the footer counts line;
   2. `check malicious-fixture-pkg@1.0.0` → exit 1, `FAIL` (the seeded
      fixture package is defined by this issue with an OSV `MAL-` response
      cached for it — name it exactly `malicious-fixture-pkg` in
      `test/fixtures/http/osv/`);
   3. `approve express@<v> --by "CI <ci@example.com>"` → ledger created;
      assert content with `reviewedAt` normalized by pattern
      (`"reviewedAt": "<ISO-8601>"` regex replacement before golden
      compare — timestamps are runtime-generated); re-approve prints
      replacement note;
   4. `verify` on npm fixture projects: all six DESIGN §13.2 status
      literals exercised — `ok`, `unapproved`, `version_drift`,
      `rejected`, `integrity_mismatch`, `lock_missing` — with exit codes
      and remediation hints;
   5. mixed npm+pypi project full loop;
   6. `list --json` golden;
   7. hostile: project with ANSI-laden package fields → terminal outputs
      contain no ESC bytes; corrupt ledger → exit 2 and file untouched
      (byte-compare); hostile manifests from issues 10/14 fixtures
      (path-traversal names, `__proto__` keys, malformed lockfiles) run
      through `verify` end-to-end → documented warns, correct exit codes,
      no crash, no file modification;
   8. `--quiet`, `--no-color`, `NO_COLOR=1` variants;
   9. spec-error paths (`check foo@^1`, unknown prefix) → exit 2, usage
      messages.
3. Performance smoke (budget §18, generous CI multiplier ×4): warm
   (seeded) `check` < 12 s, `verify` (20-dep fixture) < 4 s wall.
4. Stream discipline assertion helper: stdout must parse as the expected
   payload alone; all diagnostics on stderr (applied in every scenario).
5. Windows: paths via `node:path` everywhere in the harness; the suite runs
   on the full CI matrix (no skips without a linked issue).

## Acceptance Criteria

- [ ] All scenarios green on ubuntu/macos/windows × Node 22/24 in CI.
- [ ] Six verify statuses each asserted at least once end-to-end.
- [ ] Zero live-network attempts: harness runs with no fixture-server at
      all (offline+seeded-cache design) — any network attempt fails the
      test by `OfflineMissError` visibility or timeout.
- [ ] CI workflow updated with `test:e2e` step (build first).
- [ ] Flake check: suite passes 3 consecutive local runs (log in PR).

## Validation

- `npm run test:e2e` locally + CI matrix links in PR.

## Dependencies

- 36, 37, 38, 39 (all commands), 05 (cache on-disk format for the seeding
  harness), 02 (workflow to extend). (ISSUE_PLAN table lists the same.)

## Non-goals

- No live-API tests (issue 45 covers that manually), no benchmark rigor
  (budgets are smoke-level), no mutation testing.

## Design References

- DESIGN.md §17 (e2e row), §18, §5.4; ADR-007 S2 (why offline-seeded)
