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

1. `types.ts` — transcribe DESIGN §9.1 exactly (`SignalStatus`,
   `SignalCategory`, `Evidence`, `Signal`, `Collector`) plus:
   ```ts
   interface SignalContext {
     subject: { ecosystem: EcosystemId; name: string; version: string };
     facts: PackageFacts;
     infra: { depsdev: DepsDevClient; osv: OsvClient; github: GitHubClient;
              http: CachedHttp; offline: boolean; log: Logger };
     topPackages: TopPackagesIndex;       // defined in issue 27; use a
                                          // structural placeholder type
                                          // { has(name): boolean; nearest(name): {name, distance} | null }
   }
   const SIGNAL_CATALOG: readonly string[]; // the 22 ids from DESIGN §9.2
   ```
2. Orchestrator `collectSignals(collectors, ctx, opts?): Promise<Signal[]>`:
   - concurrency limit 4 (simple semaphore; no new dependency);
   - per-collector timeout `opts.timeoutMs ?? 10_000` via `Promise.race` +
     AbortSignal passed in ctx for cooperative cancellation (collectors may
     ignore it; race result decides);
   - collector throws / times out ⇒ every id in `collector.produces` becomes
     `{ status: "unavailable", unavailableReason: <ErrorClass or "timeout">, evidence: [] }`;
   - collector resolves but omits a declared id ⇒ orchestrator fills the
     gap the same way (`unavailableReason: "collector-gap"`) and logs warn
     (bug indicator);
   - collector emits an undeclared id ⇒ dropped + warn (contract
     enforcement);
   - duplicate id across collectors ⇒ `InternalError` at registration time
     (validate the collector set before running);
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
- [ ] A run with zero collectors yields all-catalog `unavailable` output
      (bootstrap behavior) — proves completeness invariant independent of
      collectors.
- [ ] `SIGNAL_CATALOG` matches DESIGN §9.2 exactly (22 ids; test pins the
      list literally so any change is a conscious diff).
- [ ] No collector error can reject `collectSignals` (fuzz: collectors that
      throw strings, Errors, reject after resolve-race, return garbage).

## Validation

- `npm test -- signals/orchestrator`.

## Dependencies

- 07 (types), 03 (errors/log). Clients (16–18) referenced as types only —
  use interfaces, not constructors.

## Non-goals

- No concrete collectors (20–29), no rule evaluation (30), no rendering.

## Design References

- DESIGN.md §9.1, §9.2 (catalog), P4; ADR-007 (honest degradation)
