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
   a new issue and mark the row with it (v1 does not ship with open ❌ on
   S1–S6; S7/S8 rows may reference issue 43 for release-workflow bits).
3. Adversarial spot-checks to perform and record (commands + results in
   the audit doc):
   - `rg "child_process" src/ --glob '!src/infra/git.ts'` → empty;
   - `rg "process\.env" src/core src/infra --glob '!src/cli/**'` → only the
     documented sites (issue 39 centralized env reads — verify);
   - run the CLI with `VETLOCK_GITHUB_TOKEN=hunter2` + `--verbose` against
     seeded cache and `rg hunter2` over all outputs and the cache dir →
     empty;
   - hostile-manifest fixtures (issues 10/14) rerun via the e2e suite;
   - `npm pack --dry-run` file list matches the allowlist (issue 02 job
     logic re-verified locally).
4. Dependency review: `npm ls --omit=dev --all` output committed to the
   audit doc; confirm exactly the four ADR-002 runtime deps (+ their
   transitive closure listed for the record).

## Acceptance Criteria

- [ ] SECURITY.md present, linked from README, factually consistent with
      the implementation (reviewer cross-checks the allowlist and scope
      statements).
- [ ] Audit table complete: all S1–S8 rows with named tests, S1–S6 all ✅.
- [ ] All four spot-check commands recorded with clean results.
- [ ] Runtime dependency tree recorded; no undeclared runtime deps.
- [ ] Any discovered gap either fixed (with test) or captured as a linked
      new issue in the table.

## Validation

- Full suite green (`npm run lint && npm run typecheck && npm test &&
  npm run test:e2e`); audit doc reviewed in PR.

## Dependencies

- 40 (finished surface to audit); effectively all implementation issues.

## Non-goals

- No penetration testing, no third-party audit coordination, no new
  features; fixes beyond ~20 lines spawn issues instead.

## Design References

- ADR-007 (S1–S8 table); DESIGN.md §16; ADR-006 (allowlist text for
  SECURITY.md)
