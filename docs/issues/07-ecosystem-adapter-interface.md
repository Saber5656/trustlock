# Title

Ecosystem adapter interface & registry

## Summary

Define the normative `EcosystemAdapter` TypeScript interface, the shared
data types (`PackageFacts`, `Fact<T>`, `ResolvedVersion`,
`DirectDependency`), the adapter registry with ecosystem detection, and the
architectural guard test that keeps core modules ecosystem-agnostic.

## Context

This is the P6 (multi-ecosystem first) contract from DESIGN §7 / ADR-004.
npm (issues 08–11) and PyPI (12–15) implement it; core modules depend only
on these types. Freezing precise types here lets the npm and PyPI waves
proceed in parallel.

## Scope

- `src/core/ecosystems/types.ts`, `src/core/ecosystems/index.ts`,
  `test/architecture.test.ts`. No concrete adapters (stubs in tests only).

## Detailed Requirements

1. `types.ts` — transcribe **exactly** the interfaces from DESIGN §7.1,
   §7.3, §8 (`EcosystemId`, `EcosystemAdapter`, `Fact<T>`, `PackageFacts`,
   `DirectDependency`), plus:
   ```ts
   interface ResolvedVersion {
     version: string;                     // exact, normalized
     resolvedFrom: "requested" | "latest";
     registryPageUrl: string;             // human page, e.g. npmjs.com/package/x
   }
   interface NameValidation { ok: true; normalized: string } | { ok: false; reason: string }
   interface InfraContext { http: CachedHttp; offline: boolean; log: Logger }
   ```
   All types exported; JSDoc on every field citing the DESIGN section.
2. `index.ts`:
   - `class EcosystemRegistry { register(a: EcosystemAdapter): void; get(id: EcosystemId): EcosystemAdapter; all(): EcosystemAdapter[]; detect(cwd): Promise<EcosystemAdapter[]> }`
   - `get` on unknown id throws `InternalError` (registration happens at
     startup; user-facing unknown-ecosystem errors are `UsageError` raised
     in spec parsing).
   - `detect(cwd)` runs `detectProject` on all adapters concurrently and
     returns matches (order = registration order: npm, then pypi).
   - `createDefaultRegistry()` factory — initially registers nothing;
     issues 11/15 add their adapters here (leave a clearly marked
     registration block).
3. `test/architecture.test.ts` (the P6 guard):
   - Reads all `src/core/**/*.ts` and `src/cli/**/*.ts` files **excluding**
     `src/core/ecosystems/**`, and asserts none contains the substrings
     `"npm"` / `"pypi"` as string literals or identifiers in code
     (implementation: regex on stripped-of-comments source for
     `/(["'`])(npm|pypi)\1/` and `/\b(npm|pypi)[A-Z_]/`). Allowlist file
     paths may be added ONLY with a comment justifying each (expected to
     stay empty through v1).
   - Asserts `EcosystemAdapter` has no optional methods (interface
     completeness — compile-time via a type-level test).
4. Stub adapter helper for tests:
   `makeStubAdapter(partial): EcosystemAdapter` in `test/helpers/` so issues
   06/19 can test against fakes.

## Acceptance Criteria

- [ ] Types compile under `exactOptionalPropertyTypes` and are imported by a
      compile-only test exercising every field.
- [ ] Registry register/get/all/detect covered by unit tests with stubs,
      including duplicate-registration error and unknown-id error.
- [ ] Architecture test passes and demonstrably fails when a synthetic file
      `src/core/rules/_bad.ts` containing `"npm"` is present (do this as an
      in-test temp write to a fixture dir mimicking the check, or by testing
      the checker function directly against inline source strings).
- [ ] No concrete network or fs calls in this issue's modules.

## Validation

- `npm test -- ecosystems architecture`.

## Dependencies

- 01, 03 (errors), 05 (CachedHttp type import).

## Non-goals

- No concrete adapters (08–15), no ecosystem detection from lockfiles alone
  (manifest files decide, per adapters).

## Design References

- DESIGN.md §7 (contract), §8 (DirectDependency), §4.2 (layout); ADR-004
