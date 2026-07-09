# Title

OSV client (querybatch + advisory hydration)

## Summary

Implement `src/infra/osv.ts`: query OSV.dev for vulnerabilities affecting an
exact package version, hydrate up to 10 advisory records, and classify
malicious-package advisories (`MAL-` ids) separately from ordinary
vulnerabilities.

## Context

OSV is the vulnerability source (ADR-006); `MAL-` advisories are the
strongest single signal vetlock has (rule R-MAL-001, critical). API shapes
per research/data-sources.md §2.4. POST responses are cached via the
issue-05 `postJson` cache path.

## Scope

- `src/infra/osv.ts`, zod schemas, fixtures under `test/fixtures/http/osv/`,
  unit tests.

## Detailed Requirements

1. `createOsvClient(infra)`; endpoints
   `POST https://api.osv.dev/v1/querybatch` and
   `GET https://api.osv.dev/v1/vulns/{id}`; TTL `CACHE_TTLS.osv` for both.
2. `queryPackage(osvEcosystem, name, version): Promise<OsvQueryResult>`:
   - body: `{ "queries": [{ "package": { "ecosystem": osvEcosystem, "name": name }, "version": version }] }`
     — exactly `version`, never purl+version together (400 otherwise).
   - response: `results[0].vulns[]?.id` list (may be absent ⇒ empty).
   - split ids: `maliciousIds` = ids starting `MAL-`; `vulnIds` = rest.
   - hydrate: `GET /v1/vulns/{id}` for the first 10 of `vulnIds` ∪ first 10
     of `maliciousIds` (cap total 20); consumed fields: `id`, `summary?`,
     `severity[]? ({type, score})`, `database_specific?.severity?`,
     `aliases[]?`.
   - severity mapping to `"low"|"medium"|"high"|"critical"|"unknown"`:
     prefer `database_specific.severity` (string, lowercased) else parse
     CVSS v3/v4 score from `severity[]` (`type` `CVSS_V3`/`CVSS_V4`, score
     string like `CVSS:3.1/...` — extract base score via the vector's
     numeric evaluation is NOT required: use `database_specific.severity`
     or, absent that, `"unknown"`; do not implement CVSS math in v1 —
     document).
   - result: `{ vulns: [{id, summary?, severity, url}], malicious: [{id, summary?, url}], truncated: boolean }`
     where `url = https://osv.dev/vulnerability/{id}`.
3. Unqueried ecosystems guard: `osvEcosystem` must be non-empty (adapters
   provide it); name passed as-is for npm, normalized for PyPI (OSV uses
   normalized PyPI names — matches adapter normalization; test it).
4. Hydration failures for individual ids ⇒ that entry keeps
   `severity: "unknown"` with id+url only (never fails the batch).
5. Empty result (no vulns) is the common path — zero extra requests.

## Acceptance Criteria

- [ ] Fixtures: lodash@4.17.19 (multiple GHSA vulns — real recorded,
      trimmed), a `MAL-` fixture (real MAL id for any npm malware package,
      e.g. from OSV search; content trimmed), empty-result fixture.
- [ ] `MAL-` ids land in `malicious`, others in `vulns`.
- [ ] Cap: fixture with 15 vuln ids hydrates exactly 10 and sets
      `truncated: true`.
- [ ] Individual hydration 500 ⇒ entry degraded, batch succeeds.
- [ ] querybatch request body matches the documented shape byte-for-byte
      (snapshot of captured body).
- [ ] POST cache: second identical query in a test hits cache (fake fetch
      call count = 1).

## Validation

- `npm test -- osv`.

## Dependencies

- 04, 05.

## Non-goals

- No CVSS vector math; no range queries (exact version only); no paging
  (`next_page_token` — with 1 query it is irrelevant; assert absent or
  ignore).

## Design References

- DESIGN.md §14.3, §9.2 (vulnerabilities.*); research §2.4; ADR-006
