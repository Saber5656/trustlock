# Title

approve / reject commands

## Summary

Implement `src/cli/cmd-approve.ts` (serving both `vetlock approve` and
`vetlock reject`) and the `src/infra/git.ts` helper: resolve the spec,
confirm the version exists, capture npm integrity, determine the reviewer
identity, and upsert the decision into the ledger.

## Context

DESIGN §12.3. These are the only commands that write the ledger. The
reviewer identity default comes from git config — the sole permitted
child-process call in the codebase (S1 wrapper).

## Scope

- `src/cli/cmd-approve.ts`, `src/infra/git.ts`, command registration in
  `src/cli/index.ts`, unit + command-level tests.

## Detailed Requirements

1. `src/infra/git.ts`:
   - `getGitIdentity(cwd): Promise<{ name?: string; email?: string }>` via
     `execFile("git", ["config", "--get", "user.name"])` (and email) with
     2 s timeout, argv-array only (no shell), missing git / non-zero exit ⇒
     empty result (never throws).
   - This file is the single allowed `child_process` importer (issue 01's
     ESLint exception).
2. Command flow (shared; `decision` parameter distinguishes approve/reject):
   1. `parseSpec` (issue 06);
   2. resolve version via adapter (`resolveVersion`) — spec without version
      resolves latest and prints
      `resolved <name> latest → <version>` to stderr (visible decision);
   3. version-existence check is inherent in resolveVersion (`RegistryError`
      exit 2 when missing);
   4. npm only: fetch facts for integrity — call `fetchPackageFacts` and
      take `facts.distribution?.value.integrity` (`distribution` is a
      `Fact<…>` — DESIGN §7.3; absent ⇒ omit field, log debug); PyPI: no
      integrity in v1;
   5. `--report <path>`: read file (≤ 5 MiB), `reportSchema.parse` (issue
      32), require subject match — `subject.ecosystem` and normalized
      `subject.name` equal, and `subject.resolvedVersion` equals the
      version being approved (mismatch ⇒ `UsageError` naming both
      subjects); set `entry.reportDigest = canonicalReportDigest(report)`;
   6. reviewer: `--by` value, else git identity as `Name <email>` (name
      only if email missing), else `UsageError` with hint
      (`--by "Your Name <you@example.com>"` or set git config);
   7. `reviewedAt`: current time ISO 8601 UTC (the CLI layer is the only
      clock reader; injected `now()` for tests);
   8. ledger: `findLedger(cwd, --ledger)` (§12.4). Missing-target rules:
      explicit `--ledger <path>` given and file missing ⇒ create the
      empty ledger AT THAT PATH (+ notice); no flag and discovery finds
      nothing ⇒ create at `<cwd>/vetlock.json` (+ notice); corrupt ledger
      ⇒ exit 2 (never overwrite — S7);
   9. `upsertDecision`; when replacing, print previous decision line
      (`replacing: approved 4.17.20 by Alice on 2026-06-01`) — all
      ledger-originating strings in this and other messages pass through
      `sanitize()` (issue 33) before terminal output (§16.6);
   10. `saveLedger` (comparators from the adapter registry); print
       one-line confirmation to stdout:
       `approved npm:left-pad@1.3.0 (by Yasushi Takagi <ty@…>)`.
       With `--format json`: emit
       `{ "decision", "ecosystem", "name", "version", "reviewedBy", "reviewedAt", "replaced"? }`.
3. `reject` differences: `decision: "rejected"`; confirmation wording
   `rejected npm:foo@1.0.0`; everything else identical.
4. `--reason` stored verbatim (sanitized at render time, not at write —
   the ledger keeps user input faithfully; length cap 500 chars with
   `UsageError` beyond).
5. Offline behavior: `--offline` + cache miss on resolveVersion ⇒ exit 2
   with hint (approval requires knowing the version exists).
6. Exit codes: success 0; all failures per §5.4 (2 — approve/reject have
   no "policy outcome" exit 1 path).

## Acceptance Criteria

- [ ] approve with explicit version writes the documented entry (golden
      ledger diff) including integrity for npm fixture.
- [ ] approve without version resolves latest + prints resolution note.
- [ ] reject writes `decision: "rejected"`.
- [ ] Replacement prints previous entry summary; ledger ends with one entry
      per triple.
- [ ] `--report` happy path stores matching digest; subject-mismatch and
      malformed-report paths exit 2 with specific messages.
- [ ] Reviewer resolution precedence (`--by` > git > error) — git helper
      faked; the real `git.ts` has its own tests (missing git binary path
      included).
- [ ] Missing-ledger creation: explicit `--ledger` path honored; discovery
      miss creates at cwd; corrupt-ledger exit 2.
- [ ] PyPI approve stores normalized name (`Django` input → `django`
      entry).
- [ ] Hostile `reviewedBy`/`reason` in a replaced entry renders sanitized
      in the replacement notice.

## Validation

- `npm run lint && npm run typecheck && npm test -- cmd-approve git`.

## Dependencies

- 35, 06, 11, 15, 32 (report schema + digest), 33 (sanitize), 03.
  (ISSUE_PLAN table lists the same.)

## Non-goals

- No auto-check before approving (users run `check` themselves; `--report`
  is the optional link), no bulk approve, no interactive prompts.

## Design References

- DESIGN.md §12.3, §12.4, §5; ADR-007 S1/S7
