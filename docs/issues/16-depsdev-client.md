# Title

deps.dev v3 client

## Summary

Implement `src/infra/depsdev.ts`: typed, cached client for the deps.dev v3
endpoints vetlock consumes — GetVersion, GetProject (OpenSSF Scorecard,
stars), and GetDependencies (dependency counts).

## Context

deps.dev provides cross-ecosystem enrichment without auth (ADR-006): stars
and Scorecard even when no GitHub token is configured, and resolved
dependency-graph sizes. Endpoints verified live 2026-07
(research/data-sources.md §2.3). Freshness lag for new packages is known
unknown U5 — the client must distinguish "not indexed" from errors.

## Scope

- `src/infra/depsdev.ts`, zod schemas, fixtures under
  `test/fixtures/http/depsdev/`, unit tests.

## Detailed Requirements

1. `createDepsDevClient(infra)`; base `https://api.deps.dev/v3`; all GETs
   through CachedHttp with `CACHE_TTLS.depsdev`.
2. `getVersion(system, name, version): Promise<DepsDevVersion | "not-indexed" | "invalid-response">`
   - `GET /systems/{system}/packages/{encodeURIComponent(name)}/versions/{encodeURIComponent(version)}`
   - consumed fields: `publishedAt?`, `isDefault?`, `isDeprecated?`,
     `licenses[]`, `advisoryKeys[] ({id})`, `links[] ({label, url})`.
   - 404 ⇒ `"not-indexed"` (U5), never an exception; malformed 200 ⇒
     `"invalid-response"` (requirement 6 applies to all three methods).
3. `getProject(projectId): Promise<DepsDevProject | "not-indexed" | "invalid-response">`
   - 404 ⇒ `"not-indexed"`; malformed 200 ⇒ `"invalid-response"`.
   - `GET /projects/{encodeURIComponent(projectId)}` where projectId is e.g.
     `github.com/expressjs/express` (encoded once, `/` → `%2F`).
   - consumed: `starsCount?`, `forksCount?`, `openIssuesCount?`, `license?`,
     `scorecard?.date`, `scorecard?.overallScore` — **inspect fixture**: the
     score field location must be taken from a recorded real response
     (`scorecard.overallScore` vs nested; record `express` live once and
     commit).
4. `getDependencies(system, name, version): Promise<{ directCount: number; transitiveCount: number } | "not-indexed" | "invalid-response">`
   - `GET .../versions/{version}:dependencies`; response `nodes[]` where
     `relation` ∈ {`SELF`, `DIRECT`, `INDIRECT`} — counts derived:
     directCount = #DIRECT, transitiveCount = #DIRECT + #INDIRECT.
   - 404 ⇒ `"not-indexed"`; malformed 200 ⇒ `"invalid-response"`.
5. This client takes fully-formed `projectId` strings from callers and
   performs NO repository-URL parsing — `parseGitHubRepo`/`projectId(...)`
   are owned by issue 18 (`infra/github.ts`) and used by the repository
   collector (issue 22).
6. Normative return convention for all three methods:
   `T | "not-indexed" (404) | "invalid-response" (200 with a shape that
   fails the zod schema — log at debug)`. Network-level failures (5xx after
   retries, timeouts, offline miss) still THROW (`NetworkError` family) —
   collectors convert those per DESIGN §15.

## Acceptance Criteria

- [ ] Fixtures: express GetVersion + GetProject (real recorded), lodash
      dependencies (real, trimmed), 404 case, malformed-200 case.
- [ ] Counts computed correctly from a fixture with SELF+3 DIRECT+5
      INDIRECT.
- [ ] URL encoding: scoped npm name `@types/node` produces
      `packages/%40types%2Fnode` (captured request assertion).
- [ ] 404 → `"not-indexed"`; malformed → `"invalid-response"`; 5xx (after
      http-layer retries) → throws `NetworkError` (collectors catch).
- [ ] No token ever attached (allowlist host but assert no auth header).

## Validation

- `npm run lint && npm run typecheck && npm test -- depsdev`.

## Dependencies

- 04, 05.

## Non-goals

- No GetRequirements, no batch endpoints, no advisory hydration (OSV owns
  advisories — deps.dev advisoryKeys are not consumed by any v1 signal).

## Design References

- DESIGN.md §14.3; research/data-sources.md §2.3; ADR-006; U5
