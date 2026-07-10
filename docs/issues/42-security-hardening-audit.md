# Title

SECURITY.md & S1–S8 conformance audit

## Summary

Write `SECURITY.md` (reporting process + honest scope) and perform the
final security-invariant audit: verify each ADR-007 invariant (S1–S8) has a
named, passing test or configuration, fixing any gaps found.

## Context

ADR-007 requires every invariant be enforced by code + tests, not
convention. Individual issues added the enforcement piecemeal; this issue
is the completing sweep over the finished surface — the last gate before
release readiness.

## Scope

- `SECURITY.md`; an audit table committed at `docs/security-audit-v1.md`;
  gap fixes limited to test additions / small hardening patches (larger
  gaps become new issues — do not silently expand scope).

## Detailed Requirements

1. `SECURITY.md`:
   - supported versions table (0.x: latest minor only);
   - private reporting via GitHub Security Advisories; acknowledgment
     target: 7 days; no bounty;
   - **scope honesty**: what vetlock does (deterministic metadata signals,
     ledger enforcement) and does NOT do (no code/behavior analysis, no
     guarantee a `pass` package is safe; the report is evidence, not a
     verdict of innocence) — 1 short section, linked from README;
   - hardening facts users may rely on: never executes target packages,
     fixed host allowlist (list it), env-only token, no telemetry.
2. `docs/security-audit-v1.md` — the audit table:

   | Invariant | Enforcement point | Named test(s) | Status |
   |---|---|---|---|
   | S1 no child_process… | eslint rule + git.ts wrapper | lint-guards.test, git.test | ✅/❌ |
   | … all of S1–S8 … | | | |

   For each row the implementer runs the named tests and links the code.
   Any ❌ ⇒ fix within this issue if ≤ ~20 lines (test or patch), else file
   a new issue and mark the row with it. **v1 completion requires ALL
   S1–S8 rows ✅** (ISSUE_PLAN §1) — this issue therefore runs AFTER
   issues 43 and 44 land, so the release-workflow and governance halves of
   S8 are auditable, not merely referenced.
3. Adversarial spot-checks — all FIVE below performed and recorded
   (commands + results in the audit doc):
   1. `rg "child_process" src/ --glob '!src/infra/git.ts'` → empty;
   2. `rg "process\.env" src/core src/infra` → only the documented sites
      (issue 39 centralized env reads — verify);
   3. token-leak probe: with `VETLOCK_GITHUB_TOKEN=hunter2` and a FRESH
      temp cache, run a check whose fixtures include a cached-then-live
      GitHub request path (integration harness with injected fetch so an
      authenticated request actually fires), then `rg hunter2` over
      stdout, stderr, the JSON report, and every file in the cache dir →
      empty;
   4. hostile-manifest fixtures (issues 10/14) rerun via the e2e suite;
   5. `npm pack --dry-run` file list matches the allowlist (issue 02 job
      logic re-verified locally).
4. Dependency review: `npm ls --omit=dev --all` output committed to the
   audit doc; confirm exactly the four ADR-002 runtime deps (+ their
   transitive closure listed for the record).

## Acceptance Criteria

- [ ] SECURITY.md present, linked from README, factually consistent with
      the implementation (reviewer cross-checks the allowlist and scope
      statements).
- [ ] Audit table complete: ALL S1–S8 rows ✅ with named tests/config
      evidence (no open rows — v1 completion gate).
- [ ] All five spot-checks recorded with clean results.
- [ ] Runtime dependency tree recorded; no undeclared runtime deps.
- [ ] Any discovered gap either fixed (with test) or captured as a linked
      new issue in the table.

## Validation

- Full suite green (`npm run lint && npm run typecheck && npm test &&
  npm run test:e2e`); audit doc reviewed in PR.

## Dependencies

- 40 (finished surface), 43 (release workflow — S8), 44 (governance
  files — S8). (ISSUE_PLAN table: 40, 43, 44.)

## Non-goals

- No penetration testing, no third-party audit coordination, no new
  features; fixes beyond ~20 lines spawn issues instead.

## Design References

- ADR-007 (S1–S8 table); DESIGN.md §16; ADR-006 (allowlist text for
  SECURITY.md)
