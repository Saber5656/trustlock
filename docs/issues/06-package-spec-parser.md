# Title

Package spec parser & ecosystem resolution

## Summary

Implement `src/core/spec.ts`: parse the user-facing package spec grammar
(`[ecosystem:]name[@version | ==version]`), resolve which ecosystem applies
(prefix → flag → project detection → error), and return a validated,
normalized `ParsedSpec`. This is the untrusted-input front door of the CLI.

## Context

DESIGN §6 defines the grammar and resolution order; name validation is
security boundary S3. The parser delegates ecosystem-specific parts (body
parsing, name validation) to adapters (issue 07) so new ecosystems need no
changes here.

## Scope

- `src/core/spec.ts` + unit tests. Uses the adapter registry interface from
  issue 07 (can be developed against a stub registry).

## Detailed Requirements

1. Types:
   ```ts
   interface ParsedSpec { ecosystem: EcosystemId; name: string; version?: string }
   interface SpecInput { raw: string; ecosystemFlag?: EcosystemId; cwd: string }
   async function parseSpec(input: SpecInput, registry: EcosystemRegistry): Promise<ParsedSpec>
   ```
2. Grammar (DESIGN §6.1):
   - Optional prefix `npm:` / `pypi:` (exact, lowercase; unknown prefix that
     matches `/^[a-z][a-z0-9]*:/` ⇒ `UsageError` listing supported
     ecosystems; note: a leading `@` never starts a prefix — `@scope/x` is
     an npm scoped name).
   - Remainder passed to `adapter.parseSpecBody(body)`; the adapter splits
     name vs version (`@` for npm respecting the scope `@`; `==` or `@` for
     PyPI).
3. Ecosystem resolution order (DESIGN §6.2): prefix → `ecosystemFlag` → 
   project detection: call `adapter.detectProject(cwd)` for every registered
   adapter; exactly one true ⇒ that adapter; zero or >1 ⇒ `UsageError`
   with message showing both explicit forms
   (`vetlock check npm:<name>` / `--ecosystem pypi`).
   Conflict rule: if prefix and flag are both present and disagree ⇒
   `UsageError` (never silently prefer one).
4. After the split: `adapter.validateName(name)` must pass; result carries
   the **normalized** name (PEP 503 for pypi, unchanged-but-checked for
   npm). Version, when present, is syntax-checked by the adapter
   (`UsageError` if a range/non-exact version is supplied — message: exact
   versions only, show example).
5. Error messages must include the offending input **escaped** via the
   sanitizer once it exists; until issue 33 lands, use `JSON.stringify`
   (leave a `TODO(sanitize)` comment referencing issue 33).
6. Empty string / whitespace / >300-char raw specs ⇒ `UsageError` before any
   further processing.

## Acceptance Criteria

- [ ] Table-driven tests cover at minimum:
      `express`, `express@5.1.0`, `@types/node`, `@types/node@24.0.1`,
      `npm:@scope/name@1.0.0`, `pypi:requests`, `pypi:requests==2.32.4`,
      `requests@2.32.4` (with flag pypi), `Django` (normalizes to `django`),
      prefix+flag conflict, unknown prefix `cargo:foo`, range `express@^5`,
      `foo@`, `@`, empty, 301-char spec.
- [ ] Zero-manifest cwd without prefix/flag ⇒ `UsageError` naming both
      explicit options; two-manifest cwd likewise (distinct message).
- [ ] No URL or filesystem access happens in this module (pure).
- [ ] All returned names are normalized (asserted per adapter).

## Validation

- `npm test -- spec`; property-style fuzz test: 1,000 random ASCII strings
  must either parse or throw `UsageError` — never any other error class.

## Dependencies

- 03 (UsageError), 07 (adapter interface + registry; a test stub is fine if
  07 is in progress).

## Non-goals

- No registry lookups, no "did you mean" suggestions, no ranges.

## Design References

- DESIGN.md §6; ADR-007 S3
