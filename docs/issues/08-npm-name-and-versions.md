# Title

npm name validation & version ordering

## Summary

Implement the npm-specific primitives: package-name validation (including
scoped names), spec body parsing (`name[@version]` with scope-aware `@`
handling), and semver-based exact-version checking and ordering.

## Context

These are the npm halves of security boundary S3 and of the adapter
contract's `validateName` / `parseSpecBody` / `compareVersions` (DESIGN
§6.3, §7.1). Kept separate from the registry client so the pure logic is
table-testable in isolation.

## Scope

- `src/core/ecosystems/npm/name.ts`, `src/core/ecosystems/npm/versions.ts`
  + unit tests.

## Detailed Requirements

1. `name.ts` — `validateNpmName(raw: string): NameValidation`, rules
   (DESIGN §6.3 — deliberately slightly stricter than npm's legacy
   grammar, documented there):
   - total length 1–214 chars;
   - either unscoped `name` or scoped `@scope/name` (exactly one `/`);
   - each part matches `/^[a-z0-9][a-z0-9\-._~]*$/` — no uppercase, cannot
     start with `.` or `_` or `-`;
   - no URL-encoded sequences (`%`), no whitespace, no path tricks (`..`
     rejected by the charset rule but add an explicit test);
   - normalized form = input unchanged (npm names are canonical already).
   Failure reasons are specific ("uppercase not allowed", "scope missing
   name part", …).
2. `name.ts` — `encodeNpmNameForUrl(name): string`: **first re-validates
   via `validateNpmName` and throws `InternalError` on failure**
   (defense-in-depth at the URL boundary, S3); then scoped names encode the
   single `/` as `%2F` (`@scope%2Fname`); unscoped returned as-is. This is
   the ONLY place npm names are URL-encoded.
3. `versions.ts` using the `semver` package (implements the adapter's
   `validateExactVersion` — DESIGN §7.1):
   - `parseExactVersion(raw): { ok: true; version: string } | { ok: false; reason: string }`
     — pre-check BEFORE any cleaning: reject raw input containing
     whitespace or any range/operator token (`^ ~ > < = x * ||`); then
     allow an optional leading `v` (`semver.clean`) and require
     `semver.valid` on the result. The precheck exists because
     `semver.clean` is lenient (trims/normalizes) and must not launder
     non-exact input.
   - `compareNpmVersions(a, b): -1|0|1` via `semver.compare`.
   - `isPrerelease(v): boolean` via `semver.prerelease(v) !== null`.
4. `parseNpmSpecBody(body: string): { name: string; version?: string }` —
   exported from `src/core/ecosystems/npm/name.ts` (the adapter delegates
   `parseSpecBody` to it): split on the **last** `@` that is not the
   leading scope `@`. Required cases: `express` → no version;
   `express@5.1.0`; `@types/node` → no version; `@types/node@1.2.3`;
   `express@` → throws `UsageError("empty version")`; `a@b@c` → name `a`,
   version `b@c` (version validation then fails).

## Acceptance Criteria

- [ ] Table tests (≥ 25 cases) covering: valid unscoped/scoped, 214-char
      boundary (valid) and 215 (invalid), uppercase, leading `.`/`_`/`-`,
      `..`, `%2e`, empty scope, missing name after scope, double `/`,
      whitespace, emoji.
- [ ] Spec-body tests: the six required cases in Detailed Requirement 4.
- [ ] `encodeNpmNameForUrl` throws on an invalid name (never encodes it).
- [ ] Version tests: `1.2.3` ok; `v1.2.3` ok (cleaned to `1.2.3`); `^1.2.3`,
      `1.2`, `1.2.x`, `latest` rejected; prerelease `1.2.3-rc.1` ok and
      flagged by `isPrerelease`.
- [ ] `encodeNpmNameForUrl("@scope/name") === "@scope%2Fname"`.

## Validation

- `npm run lint && npm run typecheck && npm test -- npm/name npm/versions`; 100 % branch coverage on `name.ts`
  (it is a security boundary).

## Dependencies

- 07 (types).

## Non-goals

- No network, no packument logic (09), no PyPI (12).

## Design References

- DESIGN.md §6.3 (S3 rules), §7.2 (ordering); ADR-007 S3
