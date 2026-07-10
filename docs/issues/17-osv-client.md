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
   - hydrate: `GET /v1/vulns/{id}` with a **single total cap of 10**
     (DESIGN §14.3), allocated malicious-first: all `maliciousIds` up to
     10, remaining budget to `vulnIds` in response order; consumed fields:
     `id`, `summary?`, `database_specific?.severity?`, `aliases[]?`.
   - severity mapping to `"low"|"medium"|"high"|"critical"|"unknown"` —
     ONE rule for v1: lowercase `database_specific.severity` when it is one
     of the four recognized strings; anything else (absent, other strings,
     CVSS vectors in `severity[]`) ⇒ `"unknown"`. No CVSS math in v1.
   - result: `{ vulns: [{id, summary?, severity, url}], truncatedVulns: boolean,
     malicious: [{id, summary?, url}], truncatedMalicious: boolean }`
     where `url = https://osv.dev/vulnerability/{id}` and each `truncated*`
     flag is true when that list had ids beyond its hydration allocation
     (un-hydrated ids still appear as `{id, url, severity: "unknown"}`
     entries — nothing is dropped, only detail).
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
- [ ] Cap: fixture with 15 vuln ids + 0 MAL hydrates exactly 10, all 15
      present in `vulns`, `truncatedVulns: true`; fixture with 3 MAL + 12
      vulns hydrates 3 MAL + 7 vulns (`truncatedVulns: true`,
      `truncatedMalicious: false`).
- [ ] Malformed querybatch 200 and malformed hydration 200 (zod failure) ⇒
      treated as source failure per S6: querybatch-level ⇒ throw
      `NetworkError("invalid response from api.osv.dev")`; per-id
      hydration-level ⇒ that entry degrades to `severity: "unknown"`.
- [ ] Individual hydration 500 ⇒ entry degraded, batch succeeds.
- [ ] querybatch request body matches the documented shape byte-for-byte
      (snapshot of captured body).
- [ ] POST cache: second identical query in a test hits cache (fake fetch
      call count = 1).

## Validation

- `npm run lint && npm run typecheck && npm test -- osv`.

## Dependencies

- 04, 05.

## Non-goals

- No CVSS vector math; no range queries (exact version only). Pagination:
  a single query CAN return `next_page_token` when a package has very many
  vulns — v1 does NOT follow pages; when the token is present, set the
  affected list's `truncated*` flag to true (test with a paginated
  fixture).

## Design References

- DESIGN.md §14.3, §9.2 (vulnerabilities.*); research §2.4; ADR-006
