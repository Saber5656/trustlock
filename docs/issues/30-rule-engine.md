# Title

Rule engine & policy resolution

## Summary

Implement `src/core/rules/types.ts` and `src/core/rules/engine.ts`: the
Rule/Finding model, policy resolution (defaults ← ledger policy ← CLI
flag), and the pure evaluation function producing findings, verdict, and
the `incomplete` flag.

## Context

DESIGN §10.1/§10.3. The engine is deliberately a pure function so reports
are reproducible (ADR-003) and every rule is table-testable. The concrete
default ruleset lands separately (issue 31).

## Scope

- `src/core/rules/types.ts`, `src/core/rules/engine.ts`, policy schema
  (zod, shared with the ledger schema in issue 35), unit tests with stub
  rules.

## Detailed Requirements

1. `types.ts` — transcribe DESIGN §10.1 (`Severity`, `Verdict`, `Rule`,
   `Finding`, `RuleOutcome = "pass" | "triggered" | "not-evaluable"`) plus:
   ```ts
   interface Policy {
     failOn: "critical" | "warn";
     rules: Record<string, { severity?: Severity; enabled?: boolean }>;
   }
   const DEFAULT_POLICY: Policy = { failOn: "critical", rules: {} };
   interface Evaluation { findings: Finding[]; verdict: Verdict; incomplete: boolean }
   ```
2. `resolvePolicy(ledgerPolicy?: Partial<Policy>, cliFailOn?: "critical"|"warn"): Policy`
   - precedence: CLI flag > ledger file > default;
   - unknown rule ids in ledger policy ⇒ keep them (forward-compat) but
     emit one `warn` log listing them;
   - invalid severity strings ⇒ `ProjectError` (the ledger is
     user-maintained; fail loud, not silent).
3. `evaluate(rules, signals, policy): Evaluation`:
   - signals indexed by id; for each **enabled** rule (policy
     `enabled: false` ⇒ rule excluded from findings entirely):
     - signal status `evaluated` ⇒ call `rule.evaluate(signal)`;
     - `unavailable` ⇒ outcome `not-evaluable`;
     - `skipped` ⇒ rule excluded from findings (not-applicable rules do not
       clutter reports — distinct from `not-evaluable`);
   - finding severity = policy override ?? rule.defaultSeverity;
   - `verdict`: any triggered finding with severity `critical` ⇒ `fail`;
     else any triggered `warn` ⇒ `warn`; else `pass`; then `failOn: "warn"`
     escalates a `warn` verdict to `fail` (implemented as: exit-relevant
     verdict computed by caller? — **No**, normative: `failOn` changes the
     VERDICT itself; report shows the effective policy);
   - `incomplete` = ≥1 `not-evaluable` finding;
   - findings sorted: triggered first (severity desc, then ruleId),
     then not-evaluable (ruleId), then pass (ruleId);
   - `info` severity can never affect the verdict regardless of overrides
     (severity remap to `info` effectively mutes a rule's verdict impact —
     allowed and documented).
   - a rule whose `signalId` matches no signal in the input ⇒
     `InternalError` (catalog and ruleset must agree; catches wiring bugs).
4. Engine performs no I/O, no clock reads, no logging except via an
   injected logger for the unknown-rule-id warn (pass log through
   `resolvePolicy` caller instead — keep `evaluate` 100 % pure).

## Acceptance Criteria

- [ ] Verdict matrix tests: {no findings, info only, notice only, warn,
      critical} × {failOn critical, failOn warn} ⇒ documented verdicts.
- [ ] Severity override remaps a stub rule warn→critical and flips the
      verdict; `enabled: false` removes it entirely.
- [ ] `skipped` signal excludes the rule; `unavailable` yields
      not-evaluable + `incomplete: true`.
- [ ] Ordering test over a mixed evaluation snapshot.
- [ ] Policy precedence: default < ledger < CLI (three-layer test).
- [ ] Missing-signal wiring bug ⇒ `InternalError`.
- [ ] Purity: same inputs twice ⇒ deep-equal outputs (and no Date/Math.random
      usage — lint-level grep in test).

## Validation

- `npm test -- rules/engine`.

## Dependencies

- 19 (Signal types), 03 (errors).

## Non-goals

- No concrete rules (31), no report shaping (32), no ledger IO (35 — the
  policy *schema* defined here is imported there).

## Design References

- DESIGN.md §10.1, §10.3; ADR-003 (purity/reproducibility)
