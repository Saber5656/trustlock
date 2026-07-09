# Title

verify engine & command

## Summary

Implement `src/core/verify/engine.ts` and `src/cli/cmd-verify.ts`: the pure
dependency-versus-ledger state machine (DESIGN §13.2) and the CI-facing
command with terminal/JSON output and exit-code semantics. verify performs
**zero network I/O**.

## Context

This is the enforcement half of the product loop (ADR-001): CI runs
`npx vetlock verify` and any direct dependency without an approved exact
version fails the build with actionable remediation output.

## Scope

- Engine + command + tests (fixture projects from issues 10/14 reused).

## Detailed Requirements

1. Engine:
   ```ts
   interface VerifyInput { deps: DirectDependency[]; ledger: LedgerFile }
   type VerifyStatus = "ok" | "unapproved" | "version_drift" | "rejected"
                     | "integrity_mismatch" | "lock_missing";
   interface VerifyResultEntry {
     dep: DirectDependency; status: VerifyStatus;
     approvedVersions?: string[];        // for version_drift hints
     ledgerEntry?: LedgerEntry;          // matching entry when relevant
   }
   interface VerifyResult {
     entries: VerifyResultEntry[];       // sorted: violations first (status
                                         // order as listed above, reversed
                                         // ok-last), then ecosystem, name
     stale: LedgerEntry[];               // ledger entries not in any manifest
     summary: Record<VerifyStatus, number> & { total: number };
     hasViolations: boolean;
   }
   verify(input): VerifyResult           // pure
   ```
   Status decision per dep — first match wins (DESIGN §13.2 order):
   1. `lockPresent === false` ⇒ `lock_missing`;
   2. exact entry with `decision: "rejected"` ⇒ `rejected`;
   3. exact approved entry, both integrities present and different ⇒
      `integrity_mismatch`;
   4. exact approved entry ⇒ `ok`;
   5. ≥1 approved entry for (ecosystem,name) at other versions ⇒
      `version_drift` (carry sorted `approvedVersions`);
   6. none ⇒ `unapproved`.
   Violations = every status except `ok`. `stale` computed by set-diff on
   (ecosystem, name) — version-level staleness is NOT stale (an approved
   old version of a still-present dep stays useful history).
2. Command flow:
   1. resolve project dir (`[dir]` arg, default cwd; must exist ⇒ else
      `UsageError`);
   2. detect ecosystems via registry (`--ecosystem` restricts); zero
      detected ⇒ `ProjectError` ("no supported manifests found: expected
      package.json or pyproject.toml");
   3. `readDirectDependencies` per adapter (warnings from readers pass
      through to stderr); `--prod-only` filters npm `group !== "prod"`
      after reading (PyPI optional groups included unless `--prod-only`,
      which keeps only `prod` for it too — uniform semantics);
   4. `findLedger`/`loadLedger` — missing ledger ⇒ exit 2 with hint
      ("run `vetlock check` + `vetlock approve` first" — DESIGN §12.4);
   5. run engine; render; exit `hasViolations ? 1 : 0`.
3. Terminal output (uses `sanitize` from issue 33 for all dynamic text):
   - violations table first: columns `status | dependency | resolved |
     hint`; hints per status:
     `unapproved` → `vetlock check <name>@<version>`;
     `version_drift` → `approved: <versions>; vetlock check <name>@<v>`;
     `rejected` → `rejected by <reviewedBy> on <date>: <reason>`;
     `integrity_mismatch` → `approved integrity differs — investigate before trusting`;
     `lock_missing` → the reader's guidance (regenerate lockfile);
   - then `ok` count line (not per-row unless `--verbose`);
   - stale list (informational): `stale ledger entries: name@version …`;
   - summary line:
     `14 direct dependencies: 11 ok, 2 unapproved, 1 version drift`.
   - `integrity_mismatch` styled red/critical (it is the tampering signal).
4. `--format json`: `{ schemaVersion: 1, results: [...], stale: [...],
   summary: {...} }` with fixed key order (same discipline as issue 32);
   `markdown` format ⇒ `UsageError` ("markdown is check-only in v1").
5. Determinism: same inputs ⇒ byte-identical output (no clock, no network).

## Acceptance Criteria

- [ ] Engine unit tests: one test per status including the precedence
      overlaps (rejected beats drift; lock_missing beats everything;
      integrity_mismatch requires both sides present — absent either side
      ⇒ `ok` path 4).
- [ ] Stale detection: entry for a removed dep listed, entry for an old
      version of a present dep NOT listed.
- [ ] Command e2e (fixture npm project + ledger fixtures): all-approved ⇒
      exit 0; one unapproved ⇒ exit 1 + remediation hint; mixed npm+pypi
      project verifies both ecosystems in one run.
- [ ] `--prod-only` excludes dev groups both ecosystems.
- [ ] No network: engine+command tests run with a throwing HttpClient
      injected (any call fails the test).
- [ ] JSON golden snapshot; terminal golden snapshot.

## Validation

- `npm test -- verify cmd-verify`.

## Dependencies

- 35, 10, 14, 33 (sanitize), 03.

## Non-goals

- No auto-approval, no network revalidation of ledger integrity values, no
  workspaces/monorepo sub-packages (U8), no non-registry dep approval
  (documented limitation from issues 10/14 — they surface as `unapproved`).

## Design References

- DESIGN.md §13 (statuses, output, exit codes), §12.4; ADR-001
