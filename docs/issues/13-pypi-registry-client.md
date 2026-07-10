# Title

PyPI registry client (JSON API, Integrity API, pypistats)

## Summary

Implement `src/core/ecosystems/pypi/registry.ts`: typed, cached access to
the PyPI JSON API (project + version), the PEP 740 Integrity API
(provenance), and pypistats download counts, with zod validation and
committed fixtures.

## Context

PyPI is the registry of record for Python facts (ADR-006). Endpoints per
research/data-sources.md §2.2. pypistats is community-run and best-effort
(U2). Mapping to `PackageFacts` happens in issue 15.

## Scope

- `src/core/ecosystems/pypi/registry.ts`, zod schemas, fixtures under
  `test/fixtures/http/pypi/`, unit tests.

## Detailed Requirements

1. All requests use PEP 503-normalized names — every public method
   re-validates via `validatePypiName` (S3) and throws `InternalError` on
   un-normalized or invalid input before building any URL/cache key (tests:
   zero `CachedHttp` calls for `Django` (un-normalized) and `../evil`).
   **Return convention**: like issue 09, every method resolves to
   `{ data, sourceUrl }`. **S6 hardening**: `releases` and `project_urls`
   records are copied into null-prototype objects after zod validation
   (prototype-pollution fixtures required).
2. `fetchProject(name): Promise<PypiProject>`
   - `GET https://pypi.org/pypi/{name}/json`
   - zod (passthrough) for consumed fields: `info` (`name`, `version`
     (latest), `summary`, `license`, `license_expression?`, `author`,
     `author_email`, `maintainer`, `maintainer_email`, `project_urls`
     (record|null), `home_page`, `requires_dist` (array|null), `yanked`,
     `yanked_reason`), `releases` (record: version → array of file objects
     with `upload_time_iso_8601`, `packagetype` (`sdist`|`bdist_wheel`),
     `filename`, `digests.sha256`, `yanked`).
   - 404 ⇒ `RegistryError("package not found: <name>")`.
3. `fetchVersion(name, version): Promise<{ data: PypiVersionInfo; sourceUrl: string }>`
   - `GET https://pypi.org/pypi/{name}/{version}/json`; same info schema
     scoped to that version (`info.yanked` refers to the release); `urls`
     array = that version's files.
   - 404 ⇒ `RegistryError("version not found: <name>@<version>")` (issue 15
     relies on this exact semantic).
4. `fetchProvenance(name, version, filename): Promise<{ present: boolean; publisherIdentity?: string; kinds: string[] }>`
   - `GET https://pypi.org/integrity/{name}/{version}/{filename}/provenance`
     with `allow404`.
   - 200 ⇒ `present: true`; extract from the first
     `attestation_bundles[].publisher`: `kind` (e.g. `GitHub`) and
     repository-ish identity if present → `publisherIdentity`
     (`github:owner/repo` form when derivable); `kinds` = attestation
     predicate types found (defensive parsing, U1-style).
   - 404 ⇒ `{ present: false, kinds: [] }`.
   - Helper `pickProvenanceFile(files)`: prefer the first wheel, else the
     sdist — one file's provenance decides the signal in v1 (documented).
5. `fetchDownloads(name): Promise<{ data: { lastMonth: number } | null; sourceUrl: string }>`
   - `GET https://pypistats.org/api/packages/{name}/recent`
   - zod shape: `{ data: { last_month: number } }` (pypistats nests under
     a `data` key) → map `data.last_month` to `lastMonth`.
   - any error/404/shape mismatch ⇒ `data: null` (+ debug log) — never
     throws (U2).
6. Fixtures: `requests` (popular, wheel+sdist), `flask` or similar with
   Trusted-Publishing provenance (200 integrity fixture), one yanked-release
   fixture (may be synthetic but shape-accurate), one sdist-only package.
   Trim `releases` maps to ≤ 4 versions.

## Acceptance Criteria

- [ ] Project/version happy paths parse all fixtures; `project_urls: null`,
      `requires_dist: null`, empty `releases` entries tolerated.
- [ ] 404 project ⇒ `RegistryError("package not found…")`; 404 version ⇒
      `RegistryError("version not found…")`; provenance 200/404 both
      mapped; publisherIdentity extracted from the Trusted-Publishing
      fixture.
- [ ] S3: un-normalized/invalid names throw before any request (spy).
- [ ] S6: prototype-pollution keys in `releases`/`project_urls` fixtures
      cause no prototype mutation.
- [ ] pypistats failure modes (404, 5xx after retries, bad JSON) all ⇒
      `null`, never a throw (three tests).
- [ ] All request URLs captured in tests use normalized names.
- [ ] TTLs: project/version/integrity use `CACHE_TTLS.registry`; pypistats
      uses `CACHE_TTLS.downloads` (asserted via fake CachedHttp).

## Validation

- `npm run lint && npm run typecheck && npm test -- pypi/registry`.

## Dependencies

- 04, 05, 12 (name validation). (ISSUE_PLAN table lists the same.)

## Non-goals

- No facts mapping (15), no XML-RPC/legacy APIs, no BigQuery downloads, no
  multi-file provenance aggregation (v2).

## Design References

- DESIGN.md §7.3, §14; research/data-sources.md §2.2; ADR-006; U1/U2
