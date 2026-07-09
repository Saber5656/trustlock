# Title

npm registry client (packument, downloads, attestations)

## Summary

Implement `src/core/ecosystems/npm/registry.ts`: typed, cached access to the
npm registry packument, the downloads point API, and the attestations
endpoint, with zod validation and recorded fixtures for `express`,
`left-pad`, a scoped package, and a package with install scripts.

## Context

The npm registry is the registry of record for npm facts (ADR-006 #3).
Endpoints and field semantics are documented in
research/data-sources.md §2.1. This client returns raw-but-validated
registry data; mapping to `PackageFacts` happens in issue 11.

## Scope

- `src/core/ecosystems/npm/registry.ts`, zod response schemas, fixtures
  under `test/fixtures/http/npm/`, unit tests.

## Detailed Requirements

1. Constructor: `createNpmRegistryClient(infra: InfraContext)`; all calls go
   through `CachedHttp` with `CACHE_TTLS.registry` (packument, attestations)
   and `CACHE_TTLS.downloads` (downloads).
2. `fetchPackument(name): Promise<NpmPackument>`
   - `GET https://registry.npmjs.org/{encodeNpmNameForUrl(name)}` (full
     packument — **no** `Accept: application/vnd.npm.install-v1+json`).
   - zod schema (lenient: `.passthrough()`) validating the fields consumed:
     `name`, `dist-tags` (record), `time` (record: version → ISO, plus
     `created`/`modified`), `maintainers` (array of `{name, email?}`),
     `versions` (record: version → version manifest with `version`,
     `deprecated?` (string|boolean), `dist` (`integrity?`, `unpackedSize?`,
     `fileCount?`), `hasInstallScript?`, `scripts?` (record), `bin?`
     (string|record), `repository?` (string | {type?, url?, directory?}),
     `license?` (string|object), `description?`, `_npmUser?` ({name})).
   - 404 ⇒ throw `RegistryError("package not found: <name>")`.
3. `fetchDownloads(name): Promise<{ weekly: number } | null>`
   - `GET https://api.npmjs.org/downloads/point/last-week/{name}` (scoped
     names: path is `last-week/@scope%2Fname`).
   - 404/invalid shape ⇒ `null` (new packages have no stats — not an error).
4. `fetchAttestations(name, version): Promise<{ present: boolean; kinds: string[] } >`
   - `GET https://registry.npmjs.org/-/npm/v1/attestations/{encoded}@{version}`
     with `allow404`.
   - 200 ⇒ `present: true`, `kinds` = list of `predicateType` (or
     `attestationType` — inspect the real fixture; known unknown U1: write
     the schema defensively, collect whichever type-discriminator fields
     exist into strings, and add a fixture-based TODO comment if the shape
     surprises).
   - 404 ⇒ `{ present: false, kinds: [] }`.
5. Record real fixtures once (via a `scripts/record-fixture.ts` dev helper
   or manual curl, committed as JSON): `express` (popular, provenance),
   `left-pad` (legacy), one `@types/*` scoped package, one package with
   `hasInstallScript: true` (e.g. `esbuild`). Strip fixtures to the fields
   consumed + a size note (packuments are large; keep versions map trimmed
   to ≤ 5 versions in fixtures, preserving `dist-tags.latest`'s entry).
6. Every returned object carries `sourceUrl` fields as needed by issue 11
   (return the final URL used per call alongside data:
   `{ data, sourceUrl }`).

## Acceptance Criteria

- [ ] Packument happy path parses all four fixtures; missing-optional-field
      fixtures (no `maintainers`, boolean `deprecated`) parse.
- [ ] 404 packument ⇒ `RegistryError` with the package name in message.
- [ ] Downloads 404 ⇒ `null`; malformed JSON body ⇒ `null` + debug log.
- [ ] Attestations 200/404 paths tested; unknown extra fields tolerated.
- [ ] Scoped-name URLs contain `%2F` (asserted via fake-fetch capture) in
      all three endpoint families.
- [ ] No live network in tests.

## Validation

- `npm test -- npm/registry`; fixture files reviewed for secrets/PII (none
  should exist — public data only).

## Dependencies

- 04, 05 (CachedHttp), 08 (name encoding).

## Non-goals

- No PackageFacts mapping (11), no version resolution policy (11), no
  search/audit endpoints.

## Design References

- DESIGN.md §7.3, §14; research/data-sources.md §2.1; ADR-006; U1
