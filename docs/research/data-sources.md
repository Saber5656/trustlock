# Research: Data Sources for Deterministic Trust Signals

Status: verified 2026-07 (all endpoints exercised or checked against primary documentation)
Related: [DESIGN.md](../DESIGN.md), [ADR-006](../decisions/ADR-006-data-sources-and-network-posture.md)

This document records the external data sources vetlock v1 relies on, the exact
endpoints, what each provides, auth/rate-limit characteristics, and the design
impact. All sources are public, read-only HTTP APIs. vetlock never executes
target package code and never calls any endpoint outside this list.

## 1. Source catalog

| # | Source | Base host | Auth | Rate limits | Used for |
|---|--------|-----------|------|-------------|----------|
| 1 | npm registry | `registry.npmjs.org` | none | undocumented, generous | package metadata, versions, maintainers, dist integrity, install scripts, deprecation, provenance attestations |
| 2 | npm downloads API | `api.npmjs.org` | none | undocumented | download counts (popularity) |
| 3 | PyPI JSON API | `pypi.org` | none | undocumented, generous | package metadata, releases, yanked status, requires_dist, project URLs |
| 4 | PyPI Integrity API | `pypi.org` | none | undocumented | PEP 740 attestations / Trusted Publishing provenance |
| 5 | pypistats | `pypistats.org` | none | undocumented (community service) | PyPI download counts (last 180 days only) |
| 6 | deps.dev v3 | `api.deps.dev` | none | undocumented, generous | dependency graphs, advisory keys, OpenSSF Scorecard, project stars/forks, licenses |
| 7 | OSV | `api.osv.dev` | none | "currently no limits"; 32 MiB response cap on HTTP/1.1 | known vulnerabilities and malicious-package advisories |
| 8 | GitHub REST | `api.github.com` | optional `GITHUB_TOKEN` | 60 req/h unauthenticated; 5,000 req/h with token | repository existence, archived flag, activity, stars |

## 2. Endpoint details

### 2.1 npm registry

| Endpoint | Notes |
|---|---|
| `GET https://registry.npmjs.org/{name}` | Full packument. Includes `time` (publish dates per version), `maintainers`, `dist-tags`, per-version `dist.integrity`, `hasInstallScript`, `deprecated`, `repository`, `license`. Abbreviated form (`Accept: application/vnd.npm.install-v1+json`) is smaller but drops `maintainers`/`time`, so vetlock needs the **full packument**. |
| `GET https://registry.npmjs.org/{name}/{version}` | Single version manifest (fallback / smaller fetch). |
| `GET https://registry.npmjs.org/-/npm/v1/attestations/{name}@{version}` | Provenance + publish attestations (Sigstore-backed). `404` when the version has no attestations. |
| `GET https://api.npmjs.org/downloads/point/{period}/{name}` | `period` ∈ `last-day`, `last-week`, `last-month`, `last-year`. Returns `{downloads, start, end, package}`. |

Scoped package names must be URL-encoded (`@scope%2Fname`) in registry paths.

### 2.2 PyPI

| Endpoint | Notes |
|---|---|
| `GET https://pypi.org/pypi/{name}/json` | Project metadata: `info` (author/maintainer strings, `project_urls`, license, `requires_dist`, `yanked`), `releases` (all versions with per-file `upload_time`, `digests.sha256`, `packagetype`). Names should be PEP 503-normalized before the request. |
| `GET https://pypi.org/pypi/{name}/{version}/json` | Version-specific metadata. |
| `GET https://pypi.org/integrity/{name}/{version}/{filename}/provenance` | PEP 740 provenance. `200` with `attestation_bundles` (includes publisher identity, e.g. GitHub repo + workflow for Trusted Publishing) or `404` when the file has no provenance. JSON is the default representation. |
| `GET https://pypistats.org/api/packages/{name}/recent` | `{last_day, last_week, last_month}` download counts. Community-run; data covers the trailing 180 days. Treat as best-effort. |

Caveats:
- PyPI's JSON API exposes maintainer info only as free-text `info.maintainer` /
  `info.author` strings — no structured maintainer list or history (unlike npm).
- Attestation adoption is meaningful but partial (defaults on for Trusted
  Publishing via `pypa/gh-action-pypi-publish` >= 1.11.0; a minority of the
  long tail has them). Missing attestations must be a *notice*, not a failure.

### 2.3 deps.dev v3

Verified live 2026-07:

| Endpoint | Verified response fields |
|---|---|
| `GET https://api.deps.dev/v3/systems/{system}/packages/{name}` | version list, default version |
| `GET https://api.deps.dev/v3/systems/{system}/packages/{name}/versions/{version}` | `publishedAt`, `isDefault`, `isDeprecated`, `licenses[]`, `advisoryKeys[]`, `links[]` (HOMEPAGE, ISSUE_TRACKER, SOURCE_REPO) |
| `GET .../versions/{version}:dependencies` | resolved dependency graph (npm, PyPI, Cargo, Maven supported) |
| `GET https://api.deps.dev/v3/projects/{urlencoded-project-id}` | `starsCount`, `forksCount`, `openIssuesCount`, `license`, OpenSSF `scorecard` (verified: returns current Scorecard v5 data) |

`system` ∈ `GO`, `RUBYGEMS`, `NPM`, `CARGO`, `MAVEN`, `PYPI`, `NUGET` — this
enum shape is why the vetlock adapter interface must carry a per-ecosystem
deps.dev system id.

Design note: deps.dev lets vetlock get GitHub stars + Scorecard **without
GitHub auth**, so the GitHub REST client is only needed for freshness signals
(recent pushes, archived flag) and as a fallback.

### 2.4 OSV

| Endpoint | Notes |
|---|---|
| `POST https://api.osv.dev/v1/query` | one package/version query |
| `POST https://api.osv.dev/v1/querybatch` | up to 1,000 queries per call; returns vulnerability **IDs only** |
| `GET https://api.osv.dev/v1/vulns/{id}` | full advisory record (hydrate top N) |

- Ecosystem values: `npm`, `PyPI`. Version comparison follows each ecosystem's
  scheme (semver for npm, PEP 440 for PyPI).
- A query item must use `version` or a versioned `purl`, **not both** (400
  otherwise).
- Malicious-package advisories appear with `MAL-` IDs — these are a distinct,
  stronger signal than ordinary CVEs and must map to `critical`.
- No API key; no documented request rate limit; 32 MiB response cap on
  HTTP/1.1.

### 2.5 GitHub REST

| Endpoint | Notes |
|---|---|
| `GET https://api.github.com/repos/{owner}/{repo}` | `archived`, `pushed_at`, `stargazers_count`, `open_issues_count`, `default_branch`, `license` |

- Unauthenticated limit is 60 req/h per IP → vetlock must treat GitHub data as
  **optional enrichment**: read `VETLOCK_GITHUB_TOKEN` or `GITHUB_TOKEN` from
  env when present, and degrade to "signal unavailable (rate limited)"
  otherwise. Never persist or log the token.

## 3. Popular-package datasets (typosquat reference lists)

| Ecosystem | Source | Notes |
|---|---|---|
| PyPI | [hugovk/top-pypi-packages](https://github.com/hugovk/top-pypi-packages) | monthly dump of the 15,000 most-downloaded PyPI packages, JSON/CSV; widely used in academic supply-chain research |
| npm | [`npm-high-impact`](https://www.npmjs.com/package/npm-high-impact) (verified: latest 1.13.0) | curated list of high-impact/popular npm package names, distributed as an npm package (data importable at build time) |
| cross-ecosystem ground truth | [ecosyste-ms/typosquatting-dataset](https://github.com/ecosyste-ms/typosquatting-dataset) | curated known-typosquat pairs (PyPI 95, npm 35, …) — useful as a **test fixture** for the typosquat collector |

Design impact: vetlock bundles a generated `data/top-packages/{npm,pypi}.json`
snapshot (top N names only) produced by a maintenance script, so the typosquat
check runs **offline and deterministically** at runtime. The update script must
record dataset provenance and check the upstream licenses when implemented.

## 4. Manifest / lockfile formats parsed in v1

| Format | What vetlock reads | Stability notes |
|---|---|---|
| `package.json` | `dependencies`, `devDependencies` (direct deps) | stable |
| `package-lock.json` v2/v3 | resolved exact version + `integrity` for each direct dep (`packages[""].dependencies` → `packages["node_modules/{name}"]`) | lockfileVersion 2/3 only; v1 rejects v1 lockfiles with guidance |
| `pyproject.toml` | PEP 621 `[project.dependencies]`, `[project.optional-dependencies]` (direct deps; PEP 508 strings) | stable standard |
| `uv.lock` | TOML; `[[package]]` entries with `name`, `version`, `source`; maps direct deps to resolved exact versions | **uv-specific format, explicitly not standardized**; has a top-level `version`/`revision` — parser must check it and degrade gracefully on unknown revisions. PEP 751 (`pylock.toml`) is the standardized alternative → deferred to v2. |

## 5. Cross-cutting design impact

1. **Everything is unauthenticated-friendly.** Only GitHub benefits from a
   token; all primary signals work with zero configuration.
2. **Caching is mandatory.** Every source is cacheable JSON over GET (except
   OSV querybatch). A file cache with per-source TTL + ETag revalidation keeps
   repeated `check`/`verify` runs fast and enables `--offline`.
3. **Partial failure is normal.** pypistats and GitHub can be unavailable or
   rate-limited at any time; the signal framework needs an explicit
   `unavailable` state so reports say "could not evaluate X" instead of
   silently passing (fail-closed reporting, fail-open execution).
4. **Freshness lag exists.** deps.dev indexes with delay; a brand-new package
   may 404 there while existing on the registry. Collectors must distinguish
   "not indexed yet" from "does not exist".
5. **Names are untrusted input.** Package names arrive from CLI args *and from
   manifests of cloned repos*; both must be validated against ecosystem name
   grammar before URL interpolation (SSRF/path-traversal guard).

## Sources

- [deps.dev API v3 documentation](https://docs.deps.dev/api/v3/), [google/deps.dev api.proto](https://github.com/google/deps.dev/blob/main/api/v3/api.proto)
- [OSV API documentation](https://google.github.io/osv.dev/api/), [POST /v1/querybatch](https://google.github.io/osv.dev/post-v1-querybatch/), [OSV FAQ](https://google.github.io/osv.dev/faq/)
- [npm registry API](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md), [package metadata response](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md), [npm Registry API docs](https://api-docs.npmjs.com/), [Viewing package provenance](https://docs.npmjs.com/viewing-package-provenance/), [Generating provenance statements](https://docs.npmjs.com/generating-provenance-statements/)
- [PEP 740](https://peps.python.org/pep-0740/), [PyPI Integrity API](https://docs.pypi.org/api/integrity/), [PyPI attestations announcement](https://blog.pypi.org/posts/2024-11-14-pypi-now-supports-digital-attestations/), [Trail of Bits: Attestations](https://blog.trailofbits.com/2024/11/14/attestations-a-new-generation-of-signatures-on-pypi/)
- [pypistats.org About](https://pypistats.org/about)
- [uv lockfile concepts](https://docs.astral.sh/uv/concepts/projects/sync/), [uv export (PEP 751)](https://docs.astral.sh/uv/concepts/projects/export/)
- [hugovk/top-pypi-packages](https://github.com/hugovk/top-pypi-packages), [ecosyste-ms/typosquatting-dataset](https://github.com/ecosyste-ms/typosquatting-dataset), [npm-high-impact](https://www.npmjs.com/package/npm-high-impact)
