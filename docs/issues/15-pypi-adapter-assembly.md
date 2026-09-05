# Title

PyPI adapter assembly (facts mapping, version resolution)

## Summary

Assemble `src/core/ecosystems/pypi/adapter.ts`: the concrete
`EcosystemAdapter` for PyPI wiring name/versions (12), registry client (13),
and manifest reader (14); implement `resolveVersion` and the PyPI →
`PackageFacts` mapping; register the adapter. This issue also serves as the
proof that the adapter interface holds for a second ecosystem.

## Context

Second concrete adapter (ADR-004): if anything in the interface only fits
npm, it surfaces here and must be fixed in `types.ts` (with issue-07 tests
updated) rather than worked around.

## Scope

- `src/core/ecosystems/pypi/adapter.ts`, registration, unit tests.

## Detailed Requirements

1. Constants: `id: "pypi"`, `displayName: "PyPI"`, `depsDevSystem: "PYPI"`,
   `osvEcosystem: "PyPI"`.
2. Delegations to 12 (`validateName`, `validateExactVersion` via
   `parseExactPypiVersion`, `parseSpecBody` via `parsePypiSpecBody`,
   `compareVersions`) and 14 (project/manifest). Adapter constants:
   `notApplicableSignals: ["metadata.deprecated", "maintainers.count",
   "maintainers.publisher-change", "execution.install-scripts",
   "execution.bin-entries", "footprint.install-size"]` (DESIGN §9.2
   footnote) and `lockfileGuidance: "run uv lock to generate uv.lock"`.
   **Boundary rule (S3)**: `resolveVersion` and `fetchPackageFacts` first
   run `validateName`/`validateExactVersion` and throw before any
   registry-client call on failure (tests assert zero client calls for
   `..`, `Django` un-normalized passthrough is normalized, `==1.*`).
3. `resolveVersion(name, requested, ctx)`:
   - requested: must exist as a key of `releases` with ≥1 file; a release
     whose files are all yanked resolves but the yanked flag will surface
     via facts (do not block resolution); missing ⇒
     `RegistryError("version <v> not found for <name>; latest is <info.version>")`.
   - undefined: `info.version` (PyPI's latest) — if that release is fully
     yanked or is a prerelease, fall back to the highest non-yanked,
     non-prerelease version by `comparePypiVersions` (display-order caveat
     accepted; deterministic given same data); `resolvedFrom` set
     accordingly.
   - `registryPageUrl: https://pypi.org/project/<name>/<version>/`.
4. `fetchPackageFacts(name, version, ctx)` mapping (project `P`, version
   info `V`, files `F` = that version's files). Source attribution rule:
   every fact carries `sourceUrl` — project-wide facts
   (firstPublishedAt, latestVersion, releaseDates) cite the project JSON
   URL, version-scoped facts cite the version JSON URL, and
   `attestations` cites the Integrity API URL (all taken from the
   clients' `{ data, sourceUrl }` results):

   | PackageFacts field | Source | Rule |
   |---|---|---|
   | `publishedAt` | min `upload_time_iso_8601` over `F` | no files ⇒ omit |
   | `firstPublishedAt` | min upload time across all releases' files | |
   | `latestVersion` | `P.info.version` | |
   | `releaseDates` | per release: min file upload time | sorted asc; releases without files skipped |
   | `yanked` | `V.info.yanked` / all files yanked | `{flag, reason?}` |
   | `license` | `V.info.license_expression ?? V.info.license` | empty string ⇒ null |
   | `description` | `V.info.summary` | |
   | `repositoryUrl` | first of `project_urls` keys (case-insensitive) in priority: `Repository`, `Source`, `Source Code`, `Code`, `Homepage` matching a forge URL | none ⇒ `{value:null}` |
   | `homepageUrl` | `project_urls.Homepage ?? info.home_page` | |
   | `maintainers` | — | **omit** (structurally unavailable; collectors emit `skipped`) |
   | `latestPublisher` | — | omit |
   | `installScripts` / `binEntries` / `distribution` | — | omit (npm-only) |
   | `pythonDistribution` | `F` packagetypes | `{hasWheel, sdistOnly}` |
   | `attestations` | integrity API via `pickProvenanceFile(F)` | `{present, kinds, publisherIdentity?}` |

5. Register in `createDefaultRegistry()` after npm.
6. Interface-fit report: if any workaround was needed, the PR description
   must list it and the corresponding `types.ts` change (or explicitly state
   "no interface changes required").

## Acceptance Criteria

- [ ] `resolveVersion`: requested-exists, requested-missing, latest-normal,
      latest-yanked-fallback, latest-prerelease-fallback covered.
- [ ] Full `PackageFacts` golden snapshots for the requests /
      trusted-publishing / yanked / sdist-only fixtures.
- [ ] `repositoryUrl` priority test with a `project_urls` containing
      `Homepage` + `Source` (Source wins).
- [ ] Registered adapter passes a shared adapter-contract test suite (write
      it here if issue 11 didn't: a parametrized test run over
      [npmAdapter, pypiAdapter] asserting interface behaviors — name
      validation rejects `..`, resolveVersion requested-missing throws
      `RegistryError`, detectProject false on empty dir).
- [ ] Architecture test still green.

## Validation

- `npm run lint && npm run typecheck && npm test -- pypi/adapter adapter-contract`.

## Dependencies

- 07, 12, 13, 14, 11 (shared adapter-contract suite location).
  (ISSUE_PLAN table lists the same five.)

## Non-goals

- No maintainer facts for PyPI (honest gap), no multi-file provenance
  aggregation, no signals.

## Design References

- DESIGN.md §7 (contract), §7.3 facts table; ADR-004; research §2.2
