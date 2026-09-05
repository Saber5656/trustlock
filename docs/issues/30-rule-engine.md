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
2. `resolvePolicy(ledgerPolicy: Partial<Policy> | undefined,
   cliFailOn: "critical" | "warn" | undefined,
   knownRuleIds: ReadonlySet<string>):
   { policy: Policy; unknownRuleIds: string[] }`
   - pure: precedence CLI flag > ledger file > default;
   - `unknownRuleIds` = policy rule ids ∉ `knownRuleIds`, kept in the
     returned policy (forward-compat) — the CALLER logs one warn listing
     them (the engine module does no logging);
   - invalid severity strings ⇒ `ProjectError` (the ledger is
     user-maintained; fail loud, not silent).
   - Policy zod schema (shared with issue 35): `.strict()` on the policy
     object and per-rule override objects; the `rules` record is parsed
     into a null-prototype map, and `__proto__`/`constructor`/`prototype`
     keys are rejected with `ProjectError` (hostile-ledger tests
     required — S6).
3. `evaluate(rules, signals, policy): Evaluation`:
   - signals indexed by id; for each **enabled** rule (policy
     `enabled: false` ⇒ rule excluded from findings entirely):
     - signal status `evaluated` ⇒ call `rule.evaluate(signal)`;
     - `unavailable` ⇒ outcome `not-evaluable`;
     - `skipped` ⇒ rule excluded from findings (not-applicable rules do not
       clutter reports — distinct from `not-evaluable`);
   - finding severity = policy override ?? rule.defaultSeverity;
   - `verdict`: any triggered finding with severity `critical` ⇒ `fail`;
     else any triggered `warn` ⇒ `warn`; else `pass`. Then, normatively,
     `failOn: "warn"` escalates a `warn` verdict to `fail` INSIDE the
     engine — the verdict itself changes (never a separate exit-code
     computation in callers), and the report shows the effective policy
     (DESIGN §5.4, §10.1);
   - `incomplete` = ≥1 `not-evaluable` finding;
   - findings sorted: triggered first (severity desc, then ruleId),
     then not-evaluable (ruleId), then pass (ruleId);
   - `info` severity can never affect the verdict regardless of overrides
     (severity remap to `info` effectively mutes a rule's verdict impact —
     allowed and documented).
   - a rule whose `signalId` matches no signal in the input ⇒
     `InternalError` (catalog and ruleset must agree; catches wiring bugs).
4. The whole module performs no I/O, no clock reads, and no logging —
   `resolvePolicy` reports unknown ids in its return value and `evaluate`
   is 100 % pure.

## Acceptance Criteria

- [ ] Verdict matrix tests: {no findings, info only, notice only, warn,
      critical} × {failOn critical, failOn warn} ⇒ documented verdicts.
- [ ] Severity override remaps a stub rule warn→critical and flips the
      verdict; `enabled: false` removes it entirely.
- [ ] `skipped` signal excludes the rule; `unavailable` yields
      not-evaluable + `incomplete: true`.
- [ ] Ordering test over a mixed evaluation snapshot.
- [ ] Policy precedence: default < ledger < CLI (three-layer test).
- [ ] Unknown rule ids returned in `unknownRuleIds` and preserved in the
      policy; hostile policy (`__proto__` key, extra fields, bad severity)
      ⇒ `ProjectError` (strict schema), no prototype mutation.
- [ ] Missing-signal wiring bug ⇒ `InternalError`.
- [ ] Purity: same inputs twice ⇒ deep-equal outputs (and no Date/Math.random
      usage — lint-level grep in test).

## Validation

- `npm run lint && npm run typecheck && npm test -- rules/engine`.

## Dependencies

- 19 (Signal types), 03 (errors).

## Non-goals

- No concrete rules (31), no report shaping (32), no ledger IO (35 — the
  policy *schema* defined here is imported there).

## Design References

- DESIGN.md §10.1, §10.3; ADR-003 (purity/reproducibility)
