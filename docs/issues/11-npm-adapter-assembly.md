# Title

npm adapter assembly (facts mapping, version resolution)

## Summary

Assemble `src/core/ecosystems/npm/adapter.ts`: the concrete
`EcosystemAdapter` for npm wiring together name validation (08), the
registry client (09), and the manifest reader (10); implement
`resolveVersion` and the packument → `PackageFacts` mapping; register the
adapter in `createDefaultRegistry()`.

## Context

This issue completes the npm ecosystem behind the ADR-004 interface. The
facts mapping is the single place npm registry semantics are translated to
the cross-ecosystem model; collectors (wave 4) rely on its exact field
semantics.

## Scope

- `src/core/ecosystems/npm/adapter.ts`, registration line in
  `src/core/ecosystems/index.ts`, unit tests over fixtures from issue 09.

## Detailed Requirements

1. Interface constants: `id: "npm"`, `displayName: "npm"`,
   `depsDevSystem: "NPM"`, `osvEcosystem: "npm"`.
2. Delegations: `validateName` → 08; `validateExactVersion` → 08
   (`parseExactVersion`); `parseSpecBody` → 08; `compareVersions` → 08;
   `detectProject`/`readDirectDependencies` → 10. Adapter constants:
   `notApplicableSignals: ["metadata.yanked", "execution.sdist-only"]`
   (DESIGN §9.2 footnote) and
   `lockfileGuidance: "run npm install (npm >= 7) to generate a v2+ package-lock.json"`.
   **Boundary rule (S3)**: `resolveVersion` and `fetchPackageFacts` first
   run `validateName`/`validateExactVersion` on their inputs and throw
   before any registry-client call on failure (tests assert zero client
   calls for `../evil`, `UPPER`, `^1.2.3`).
3. `resolveVersion(name, requested, ctx)`:
   - fetch packument;
   - `requested` given: must exist as key in `versions` (else
     `RegistryError` "version <v> not found for <name>; latest is <latest>");
     `resolvedFrom: "requested"`.
   - `requested` undefined: use `dist-tags.latest`; if that tag is missing
     (pathological) ⇒ highest non-prerelease semver in `versions`; if no
     non-prerelease version exists either ⇒
     `RegistryError("no stable version found for <name>; specify an exact version")`;
     `resolvedFrom: "latest"`.
   - `registryPageUrl: https://www.npmjs.com/package/<name>` (scoped names
     unencoded in the human URL) + `/v/<version>`.
4. `fetchPackageFacts(name, version, ctx)` — `P.versions[version]` absent
   ⇒ `RegistryError("version <v> not found for <name>; latest is <latest>")`
   (same message family as resolveVersion). Mapping table (packument `P`,
   version manifest `V = P.versions[version]`; each fact's `sourceUrl` =
   the `sourceUrl` returned by the client call that produced it):

   | PackageFacts field | Source | Rule |
   |---|---|---|
   | `publishedAt` | `P.time[version]` | absent ⇒ omit fact |
   | `firstPublishedAt` | `P.time.created` | |
   | `latestVersion` | `P["dist-tags"].latest` | |
   | `releaseDates` | `P.time` entries excluding `created`/`modified` | sorted by date asc |
   | `deprecated` | `V.deprecated` | string ⇒ `{flag:true,message}`; `true` ⇒ `{flag:true}`; else `{flag:false}` |
   | `license` | `V.license` | object form (`{type}`) ⇒ its `type`; missing ⇒ `{value:null}` |
   | `description` | `V.description ?? P.description` | |
   | `repositoryUrl` | `V.repository ?? P.repository` | string ⇒ as-is; object ⇒ `.url`; missing ⇒ `{value:null}` |
   | `homepageUrl` | `V.homepage ?? P.homepage` | |
   | `maintainers` | `P.maintainers` | `{count, names}` |
   | `latestPublisher` | `V._npmUser.name` + count of earlier versions (by `P.time` date) whose `_npmUser.name` equals it | `priorPublishCount` |
   | `installScripts` | `V.scripts` keys ∩ {`preinstall`,`install`,`postinstall`} (fallback: `V.hasInstallScript === true` ⇒ `present:true, names:[]`) | |
   | `binEntries` | `V.bin` | string ⇒ `[name]`; record ⇒ its keys |
   | `distribution` | `V.dist` | `{integrity?, unpackedSize?, fileCount?}` |
   | `attestations` | attestations endpoint (09) | `{present, kinds}`; `publisherIdentity` omitted for npm in v1 |
   | `pythonDistribution` / `yanked` | — | never set for npm |

   Every fact's `sourceUrl` = the endpoint URL it came from.
5. Prerelease note: `resolveVersion` with `requested: undefined` never
   returns a prerelease; a *requested* prerelease is allowed (exact pin).
6. Register in `createDefaultRegistry()`.

## Acceptance Criteria

- [ ] `resolveVersion` covered: requested-exists, requested-missing (error
      message includes latest), latest-tag, latest-tag-missing fallback,
      no-stable-version error, prerelease exclusion.
- [ ] S3 boundary tests: invalid name and non-exact version each rejected
      with zero registry-client calls (spies).
- [ ] `fetchPackageFacts` with a version absent from the packument ⇒
      `RegistryError`.
- [ ] Facts mapping golden test per fixture package (express, left-pad,
      scoped, install-scripts) — full `PackageFacts` object snapshot.
- [ ] `latestPublisher.priorPublishCount` computed correctly on a fixture
      with mixed `_npmUser` values across versions.
- [ ] Architecture test (issue 07) still passes after registration.
- [ ] Boolean-`deprecated` and object-`license`/`repository` variants each
      exercised.

## Validation

- `npm run lint && npm run typecheck && npm test -- npm/adapter architecture`.

## Dependencies

- 07, 08, 09, 10.

## Non-goals

- No signals/rules; no PyPI; no publisherIdentity extraction (v2 with full
  Sigstore verification).

## Design References

- DESIGN.md §7 (contract, facts), §7.2; research/data-sources.md §2.1
