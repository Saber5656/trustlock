# Title

check command orchestration

## Summary

Implement `src/cli/cmd-check.ts`: wire spec parsing → adapter resolution →
facts fetch → signal collection → rule evaluation → report building →
rendering, with `--fail-on` exit-code semantics and the full infra context
(cache, offline, GitHub token, downloads facade).

## Context

`check` is the flagship command (workflow §2.2). Everything it uses exists
after waves 0–5; this issue is pure orchestration plus the flag/exit
contract, and it fixes the wiring decisions deferred by earlier issues
(downloads facade from issue 26, glyph mode from issue 33, token sourcing
from issue 18).

## Scope

- `src/cli/cmd-check.ts` + integration-level tests against the fixture
  HTTP layer (fake fetch, full pipeline).

## Detailed Requirements

1. Flow:
   1. `parseSpec` (ecosystem resolution per §6.2 using cwd);
   2. build `InfraContext`: `CachedHttp` (cache dir from context; offline
      flag), token = `VETLOCK_GITHUB_TOKEN ?? GITHUB_TOKEN` (env read
      happens HERE only), clients (depsdev/osv/github), downloads facade
      mapping ecosystem → the adapter registry-client method (issue 26's
      type), `topPackages` loaded for the subject ecosystem (issue 27);
   3. `resolveVersion` (RegistryError ⇒ exit 2; the "registry of record
      unreachable" rule from §15);
   4. `fetchPackageFacts`;
   5. `collectSignals` with all collectors registered (metadata(now),
      maintainers, repository(now), provenance, execution, vulnerabilities,
      popularity, typosquat, license, footprint);
   6. `resolvePolicy(ledgerPolicy, --fail-on, knownRuleIds)` — missing
      ledger ⇒ default policy (fine: check works without a ledger);
      **corrupt/invalid ledger ⇒ `ProjectError`, exit 2** (DESIGN §12.2 /
      S7: a ledger that exists but cannot be trusted must never be
      silently ignored); `unknownRuleIds` from the resolver are logged as
      one warn;
   7. `evaluate` → `buildReport` (now/durations measured here);
   8. render per `--format` (terminal default; asciiGlyphs on bare
      Windows per issue 33's option; markdown allowed for check);
   9. exit: verdict `fail` ⇒ 1; else 0 (the `--fail-on warn` escalation
      already happened inside policy/verdict — §10.1).
2. Concurrency/performance: facts fetch then signals; collectors already
   run concurrently (issue 19); end-to-end budget asserted loosely in tests
   (< 2 s with fake fetch — catches accidental serialization).
3. `--verbose` prints per-collector timing lines to stderr
   (`signals: repository 220ms, osv 180ms, …`).
4. Offline: `--offline` propagates. With a **totally cold cache** the
   registry of record itself is unreachable ⇒ `resolveVersion` throws
   `OfflineMissError` ⇒ exit 2 (DESIGN §15: no facts, no meaningful
   report). With registry facts cached but enrichment sources cold, the
   report is produced with `unavailable(OfflineMissError)` enrichment
   signals and `incomplete: true` — exit code follows the verdict.
5. Error taxonomy at this layer: spec/flag problems ⇒ `UsageError`;
   package/version not found ⇒ `RegistryError` message with both name and
   nearest context (from adapters); anything thrown by collectors is a bug
   (orchestrator guarantees containment) ⇒ let the shell's `InternalError`
   wrapper handle it.

## Acceptance Criteria

- [ ] Happy-path integration test (npm fixture package): full pipeline
      produces the issue-32 golden report; exit 0.
- [ ] `fail` fixture (MAL advisory) ⇒ exit 1; `--fail-on warn` with a
      warn-verdict fixture ⇒ exit 1; same fixture without the flag ⇒ 0.
- [ ] PyPI fixture end-to-end (proves adapter-agnostic wiring incl.
      normalized-name flow into OSV/typosquat).
- [ ] Ledger policy paths: missing ledger ⇒ default policy, report
      produced; corrupt ledger ⇒ exit 2 (`ProjectError`).
- [ ] Offline paths: totally cold cache ⇒ exit 2 (`OfflineMissError`);
      registry-seeded cache with cold enrichment ⇒ report with
      `incomplete: true`, exit per verdict.
- [ ] Env token test: with `VETLOCK_GITHUB_TOKEN` set, the github client
      receives it (spy); no other module reads env (grep test across
      `src/core/**` for `process.env` — must be zero matches).
- [ ] `--json` stdout is exactly the report document (no stray writes).

## Validation

- `npm run lint && npm run typecheck && npm test -- cmd-check`; manual smoke against real APIs via issue 45's
  script once available.

## Dependencies

- 06, 11, 15, 16, 17, 18, 19, 20–29, 30/31, 32, 33, 34, 35 (ledger/policy
  read). (ISSUE_PLAN table lists 06, 11, 15, 19, 31, 32, 33, 34, 35 as
  direct blockers; 16–18 and 20–29 arrive transitively through 31 but are
  listed here for the implementer's context.)

## Non-goals

- No `--approve` shortcut (explicit approve only, ADR-001), no multiple
  specs per invocation (v2), no report file output flag (shell redirection
  suffices).

## Design References

- DESIGN.md §4.3 (data flow), §5 (flags/exit), §10.3, §15, §18
