# Title

Signal framework & orchestrator

## Summary

Implement `src/core/signals/types.ts` and
`src/core/signals/orchestrator.ts`: the Signal/Collector model, the
SignalContext, and the orchestrator that runs collectors concurrently with
timeouts, converts failures into `unavailable` signals, and produces a
complete, deterministically-ordered signal list.

## Context

DESIGN §9.1 defines the model; principle P4 (fail-open execution,
fail-closed reporting) lives here: a collector crash must never kill a
`check`, and every declared signal must appear in the output so reports are
honest about gaps.

## Scope

- `src/core/signals/types.ts`, `src/core/signals/orchestrator.ts`, test
  helpers (`makeStubCollector`), unit tests.

## Detailed Requirements

1. `types.ts` — transcribe DESIGN §9.1 exactly: `SignalStatus`,
   `SignalCategory`, `Evidence`, `Signal`, `Collector`, and the normative
   `SignalContext` from DESIGN §9.1 — including
   `subject: { ecosystem, name, version, osvEcosystem, depsDevSystem,
   registryPageUrl }`, `abortSignal: AbortSignal`, and
   `infra: { depsdev, osv, github, downloads, offline, log }` where the
   client fields are **structural interfaces declared in this file**
   (`DepsDevLike`, `OsvLike`, `GitHubLike`, `DownloadsFacade =
   { fetch(): Promise<{ period: string; count: number; evidenceUrl: string } | null> }`)
   so this issue has no dependency on issues 16–18 — the concrete clients
   satisfy the interfaces structurally. `TopPackagesIndex` uses the
   issue-27 shape:
   `{ has(name): boolean; all(): readonly string[]; count: number,
   source: { name: string; url: string } }` (nearest-name search lives in
   issue 28, not here). Plus:
   ```ts
   const SIGNAL_CATALOG: readonly string[]; // the 22 ids from DESIGN §9.2
   ```
2. Orchestrator `collectSignals(collectors, adapter, ctx, opts?): Promise<Signal[]>`:
   - **validates the collector set synchronously before starting any
     collector**: duplicate signal id across collectors ⇒ `InternalError`;
   - pre-marks every id in `adapter.notApplicableSignals` as
     `skipped(not-applicable)` and excludes those ids from collector
     routing (DESIGN §9.1) — collectors never see them;
   - concurrency limit 4 (simple semaphore; no new dependency);
   - per-collector timeout `opts.timeoutMs ?? 10_000` via `Promise.race`;
     each collector run gets a fresh `AbortController` whose signal is
     placed in `ctx.abortSignal` for cooperative cancellation (collectors
     may ignore it; the race result decides);
   - collector throws / times out ⇒ every id in `collector.produces` becomes
     `{ status: "unavailable", unavailableReason, evidence: [] }` with the
     deterministic reason mapping: timeout ⇒ `"timeout"`; a `VetlockError`
     ⇒ its class name (e.g. `"OfflineMissError"`); any other thrown value
     (strings, plain objects, non-VetlockError Errors) ⇒ `"unknown-error"`;
   - collector resolves but omits a declared id ⇒ orchestrator fills the
     gap the same way (`unavailableReason: "collector-gap"`) and logs warn
     (bug indicator);
   - collector emits an undeclared id ⇒ dropped + warn (contract
     enforcement);
   - output sorted by signal id (ascending, `localeCompare("en")`).
3. Status semantics (normative, for all collector issues):
   - `evaluated` — data obtained, value present, ≥1 evidence entry;
   - `unavailable` — should exist but couldn't be obtained (network, rate
     limit, offline-miss, not-indexed, source-error);
   - `skipped` — structurally not applicable to this ecosystem
     (`unavailableReason: "not-applicable"`).
4. Every `evidence.summary` is a complete English sentence with concrete
   values ("First published 2016-03-23 (3,760 days ago)."); URLs point at
   human-readable pages when they exist, else API URLs.
5. `validateSignalOutput(signals)` dev helper: asserts exactly the catalog
   ids, sorted, no dupes — reused by tests of every collector issue.

## Acceptance Criteria

- [ ] Stub-collector tests: happy path, throwing collector, timeout (fake
      timers), gap-fill, undeclared-id drop, duplicate-id registration
      error, ordering, concurrency ≤ 4 (max-in-flight counter assertion).
- [ ] A run with zero collectors yields all-catalog output: adapter
      `notApplicableSignals` as `skipped`, everything else `unavailable`
      (bootstrap behavior) — proves completeness independent of collectors.
- [ ] Pre-marking: a stub adapter declaring two not-applicable ids ⇒ those
      ids `skipped` even when a collector also declares them (collector
      not routed those ids).
- [ ] Reason mapping: timeout ⇒ "timeout"; `OfflineMissError` ⇒
      "OfflineMissError"; thrown string ⇒ "unknown-error".
- [ ] `SIGNAL_CATALOG` matches DESIGN §9.2 exactly (22 ids; test pins the
      list literally so any change is a conscious diff).
- [ ] No collector error can reject `collectSignals` (fuzz: collectors that
      throw strings, Errors, reject after resolve-race, return garbage).

## Validation

- `npm run lint && npm run typecheck && npm test -- signals/orchestrator`.

## Dependencies

- 07 (types), 03 (errors/log). Clients (16–18) referenced as types only —
  use interfaces, not constructors.

## Non-goals

- No concrete collectors (20–29), no rule evaluation (30), no rendering.

## Design References

- DESIGN.md §9.1, §9.2 (catalog), P4; ADR-007 (honest degradation)
