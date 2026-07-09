# Title

E2E CLI test suite & fixture projects

## Summary

Build the end-to-end test suite that executes the **built** CLI
(`node dist/cli/index.js`) as a child process against fixture projects and
a local fixture HTTP server, covering the full check → approve → verify
loop, every verify status, both ecosystems, hostile fixtures, and the
performance budgets.

## Context

Unit/integration tests fake modules; this suite proves the assembled
product: flag parsing, stream separation, exit codes, file writes, and
cross-platform behavior on the CI matrix (DESIGN §17 e2e row).

## Scope

- `test/e2e/*.test.ts`, fixture server harness, fixture projects under
  `test/fixtures/projects/` (extending those from issues 10/14), npm script
  `test:e2e` (vitest project or separate config) wired into CI (issue 02's
  workflow gets the step added here).

## Detailed Requirements

1. Harness:
   - start an in-process HTTP server serving recorded fixtures by
     URL-path lookup (reuse `test/fixtures/http/**`);
   - run the CLI via `execFile(node, [dist/cli/index.js, ...args])` with
     env `VETLOCK_CACHE_DIR=<tempdir>`;
   - network strategy (normative): the shipped code gets NO test-only host
     override — that would weaken S2. Instead, a fixture-cache builder
     pre-seeds the temp cache dir with entries (issue-05 format) for every
     URL each scenario needs, and all e2e scenarios run with `--offline`.
     This keeps S2 intact (zero network in e2e) and exercises the
     cache+offline path as a bonus. Scenarios that must test live-fetch
     behavior stay at the integration layer with injected fetch (and the
     in-process fixture server mentioned above is used only there, not by
     the spawned CLI).
2. Scenarios (each asserts stdout, stderr, exit code, and file effects):
   1. `check express` (seeded cache) → exit 0, terminal report contains
      verdict and ≥ 20 check lines; `--json` parses against `reportSchema`;
   2. `check <MAL-fixture>` → exit 1, `FAIL`;
   3. `approve express@<v> --by "CI <ci@x>"` → ledger created, golden
      content; re-approve prints replacement note;
   4. `verify` on npm fixture project: all six statuses exercised across
      scenario variants (approved / drift / rejected / missing-lock /
      integrity-mismatch / unapproved) → exit codes and remediation hints;
   5. mixed npm+pypi project full loop;
   6. `list --json` golden;
   7. hostile: project with ANSI-laden package fields → terminal outputs
      contain no ESC bytes; corrupt ledger → exit 2 and file untouched
      (byte-compare);
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

- 36, 37, 38, 39 (all commands), 05 (cache format for seeding), 02.

## Non-goals

- No live-API tests (issue 45 covers that manually), no benchmark rigor
  (budgets are smoke-level), no mutation testing.

## Design References

- DESIGN.md §17 (e2e row), §18, §5.4; ADR-007 S2 (why offline-seeded)
