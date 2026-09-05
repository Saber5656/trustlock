# Review resolution record

- Repository: `Saber5656/trustlock`
- Pull request: #1
- Parent head observed before this addendum: `5353c372cc771d9325066ffb2b12d100accbe549`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNkJa86P2C7h`

### Preserve all uv.lock versions per package

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C7h` identifies this contract gap.
- Normative resolution: Represent all matching uv registry entries, including marker/platform variants, rather than indexing by name alone; verification must evaluate every applicable direct version or report an explicit multi-version violation.
- Focused verification before resolving this thread: Use a universal lock fixture with two versions of one direct package and assert neither applicable version is silently discarded.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C7o`

### Return source URLs from PyPI provenance fetch

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C7o` identifies this contract gap.
- Normative resolution: Standardize the PyPI provenance client result as `{ data: { present, publisherIdentity?, kinds }, sourceUrl }` and pass that URL into the citable attestation signal.
- Focused verification before resolving this thread: Run a provenance fetch fixture and assert the report contains the Integrity API source URL alongside the data.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C7t`

### Allow build/local version metadata in ledgers

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C7t` identifies this contract gap.
- Normative resolution: Validate exact versions with the ecosystem grammar and permit valid npm semver build metadata and PyPI local versions; do not reject `+` through a generic character ban.
- Focused verification before resolving this thread: Load ledger entries such as `1.0.0+build.1` and `1.0+local.1` and assert they round-trip and remain exact-matchable.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C7x`

### Don't reject exact prereleases containing x

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C7x` identifies this contract gap.
- Normative resolution: Distinguish wildcard/range tokens from characters inside an exact semver token; call the semver parser for exact versions so prereleases such as `1.0.0-next.0` remain valid.
- Focused verification before resolving this thread: Validate exact prereleases containing x and wildcard/range inputs separately, asserting only the latter take wildcard policy.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C70`

### Skip drift for unresolved dependencies

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C70` identifies this contract gap.
- Normative resolution: Enter the version-drift branch only when `dep.resolvedVersion` is present; unresolved non-registry dependencies use the explicit unapproved/non-approvable result and remediation.
- Focused verification before resolving this thread: Verify a VCS/path dependency with no resolved version and an unrelated registry ledger entry, asserting it is not reported as version drift.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C75`

### Reject prerelease latest tags before resolving

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C75` identifies this contract gap.
- Normative resolution: When `dist-tags.latest` is a prerelease, ignore it for omitted-version resolution and choose the highest eligible non-prerelease; if none exists, return the defined no-eligible-version error.
- Focused verification before resolving this thread: Run registry fixtures with prerelease latest, stable versions, and prerelease-only versions and assert the documented selection/error.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C78`

### Honor OSV pagination before reporting completeness

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C78` identifies this contract gap.
- Normative resolution: Read per-result `next_page_token`, follow pages within the hydration/resource cap, and mark the affected advisory set truncated whenever additional pages remain or the cap is reached.
- Focused verification before resolving this thread: Return a paginated OSV fixture and assert later advisories are fetched when allowed and completeness is false/truncated when the cap stops pagination.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C7-`

### Do not exempt separator-only typosquats

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C7-` identifies this contract gap.
- Normative resolution: Use exact raw/top-list membership only for the popularity exemption; evaluate separator-stripped equality separately so `python_dateutil` versus `python-dateutil` still produces the typosquat finding.
- Focused verification before resolving this thread: Run the separator-only pair and an exact popular-name fixture, asserting only the exact member receives the popularity exemption.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C8C`

### Reject non-PyPI registry entries in uv.lock

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C8C` identifies this contract gap.
- Normative resolution: Accept a uv registry source only when its normalized URL is the public PyPI simple/API origin; alternate or private indexes become non-registry/unapprovable and cannot inherit a public PyPI approval.
- Focused verification before resolving this thread: Load lock entries from a private/alternate registry and assert verification refuses approval even when name and version match.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C8E`

### Reject duplicate ledger entries on load

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C8E` identifies this contract gap.
- Normative resolution: Enforce uniqueness of `(ecosystem, name, version)` during ledger load and fail with a deterministic corruption error before any order-dependent lookup can occur.
- Focused verification before resolving this thread: Load conflicting duplicate approval/rejection entries and assert the ledger is rejected rather than selecting the first entry.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkJa86P2C8G`

### Avoid authenticated probes of package-declared repos

- Finding: The existing review thread `PRRT_kwDOTNkJa86P2C8G` identifies this contract gap.
- Normative resolution: Treat package repository metadata as untrusted and perform repository existence/provenance probes without bearer credentials; a private-only authenticated 200 must not become public evidence or cached data.
- Focused verification before resolving this thread: Point package metadata at a private repository and assert no GitHub token is sent and the result is not treated as public repository evidence.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.