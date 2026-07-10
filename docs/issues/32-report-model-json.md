# Title

Report model, canonical digest & JSON renderer

## Summary

Implement `src/core/report/model.ts`, `src/core/report/digest.ts`, and
`src/core/report/render-json.ts`: the versioned Report type assembled from
subject + signals + evaluation, the canonical sha256 digest (volatile fields
excluded), and the stable JSON serialization used by `--format json`.

## Context

DESIGN §11.1. The JSON report is a public machine interface (AI agents and
CI consume it) and the digest anchors `approve --report` (§12.3) — both
require byte-level discipline.

## Scope

- Three modules + unit tests with golden fixtures.

## Detailed Requirements

1. `model.ts`:
   ```ts
   interface Report {
     schemaVersion: 1;
     tool: { name: "vetlock"; version: string };
     generatedAt: string;              // ISO 8601 UTC
     subject: { ecosystem: EcosystemId; name: string;
                requestedVersion: string | null; resolvedVersion: string;
                registryUrl: string };
     verdict: Verdict;
     incomplete: boolean;
     findings: Finding[];              // engine order (issue 30): triggered
                                       // (severity desc, ruleId), then
                                       // not-evaluable, then pass — buildReport
                                       // asserts (dev-mode) rather than re-sorts
     signals: Signal[];                // ordering from orchestrator (19)
     policy: { failOn: "critical" | "warn";
               overrides: Record<string, { severity?: Severity; enabled?: boolean }> };
     durationMs: number;
   }
   buildReport(inputs): Report          // pure assembly, no clock: generatedAt
                                        // and durationMs are passed in
   ```
2. `render-json.ts`: `renderJson(report): string` — `JSON.stringify` with
   **fixed key order** (serialize via an explicit key-ordered replacer or
   construct objects in declared order and rely on insertion order — choose
   the explicit-order approach with a unit test pinning the first N bytes),
   2-space indent, LF, trailing newline. Output goes to stdout verbatim.
   **Sanitization stance (DESIGN §16.6, normative)**: JSON output is NOT
   run through `sanitize()` — machine consumers need faithful values, and
   `JSON.stringify` escapes control bytes (ESC becomes \u001b, etc.) so the emitted
   byte stream cannot carry raw terminal escapes. A test feeds
   ANSI/C0-laden strings through a report and asserts the output bytes
   contain no raw `\x1b`/`\x9b`/C0 (only their `\uXXXX` escaped forms).
3. `digest.ts`:
   - `canonicalReportDigest(report): string` returning `sha256-<hex>`;
   - canonicalization: deep-clone report, delete `generatedAt`,
     `durationMs`, and `tool.version`; serialize with sorted keys at every
     level (recursive), no whitespace; sha256 over UTF-8 bytes.
   - Property: two runs over the same package data at different times ⇒
     identical digest (test with two reports differing only in volatile
     fields).
4. Schema-stability guard: a test pins the full key structure of a golden
   report (deep key paths snapshot) — any addition shows up as a conscious
   snapshot update; removals/renames fail review by policy (note in test
   header comment).
5. zod schema `reportSchema` exported for consumers (approve --report uses
   it in issue 36 to validate user-supplied files — untrusted input, S6):
   top-level and nested objects `.strict()` except `Signal.value` (typed
   `z.unknown()` but re-serialized only through the key-sorting canonical
   path); records parsed into null-prototype maps;
   `__proto__`/`constructor`/`prototype` keys anywhere ⇒ parse failure
   (hostile-report fixtures required).

## Acceptance Criteria

- [ ] Golden JSON fixture for a fully-populated npm report and a PyPI
      report with `skipped`/`unavailable` signals — byte-identical output
      across runs.
- [ ] Digest invariant test (volatile-fields-only diff ⇒ same digest;
      any finding change ⇒ different digest).
- [ ] Key order pinned; trailing newline; LF-only (assert no `\r`).
- [ ] `reportSchema.parse(JSON.parse(renderJson(r)))` round-trips.
- [ ] `buildReport` is pure (no Date.now/Math.random — grep test as in 30).

## Validation

- `npm run lint && npm run typecheck && npm test -- report/model report/digest report/json`.

## Dependencies

- 30 (Finding/Verdict), 19 (Signal).

## Non-goals

- No terminal/markdown rendering (33/34), no JSON Schema file publication
  (v2), no report persistence.

## Design References

- DESIGN.md §11.1, §12.3 (digest use); ADR-003 (reproducibility)
