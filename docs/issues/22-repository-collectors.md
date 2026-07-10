# Title

Repository collectors (3 signals)

## Summary

Implement `src/core/signals/collectors/repository.ts`: produce
`repository.declared`, `repository.status`, and `repository.scorecard` by
combining the declared repository URL from PackageFacts with the GitHub
client (18) and deps.dev project data (16).

## Context

"Declared repo doesn't exist / is archived / is dormant" are strong
pre-add signals (rules R-REPO-001..004). Data strategy (ADR-006): GitHub
API when reachable (token-optional), deps.dev as the no-auth fallback for
stars/scorecard; both can be independently unavailable.

## Scope

- One collector file + unit tests with faked clients. `produces`:
  `repository.declared`, `repository.status`, `repository.scorecard`.

## Detailed Requirements

1. `repository.declared`: from `facts.repositoryUrl` ⇒ `{ url | null }`,
   always `evaluated` (null is a real answer). Evidence: the registry
   sourceUrl.
2. Parse the declared URL with `parseGitHubRepo` (18):
   - null URL ⇒ `repository.status` and `repository.scorecard` are
     `unavailable(no-repository-declared)`.
   - non-GitHub URL ⇒ both `unavailable(unsupported-host)` (U7) with the
     host named in the reason.
3. `repository.status` (GitHub repos only; value shape per DESIGN §9.2:
   `{ exists, archived?, lastPushAt?, lastPushAgeDays?, stars?,
   openIssues?, forks? }`):
   - primary: `github.getRepo(owner, repo)`:
     - repo data ⇒ `{ exists: true, archived, lastPushAt, lastPushAgeDays,
       stars, openIssues }` — `lastPushAgeDays` computed against an injected
       `now: Date` (constructor parameter, same pattern as issue 20; rules
       stay pure);
     - `"not-found"` ⇒ `{ exists: false }` (evaluated — this is the
       R-REPO-002 trigger);
     - `"invalid-response"` ⇒ `unavailable(source-error)`;
     - `"rate-limited"` ⇒ fallback: `depsdev.getProject(projectId)`; project
       data ⇒ `{ exists: true, stars, forks }` with `archived`/`lastPushAt`
       omitted (partial data is still `evaluated`; evidence notes the
       fallback); fallback `"not-indexed"` ⇒
       `unavailable(github-rate-limited)` with hint to set
       `VETLOCK_GITHUB_TOKEN`; fallback `"invalid-response"` ⇒
       `unavailable(source-error)`;
     - thrown `NetworkError`/`OfflineMissError` from either client ⇒
       `unavailable(<error class name>)` — caught HERE per signal (partial
       results stay possible: status may evaluate while scorecard is
       unavailable, and vice versa).
4. `repository.scorecard`: `depsdev.getProject(projectId)` ⇒
   `{ score, date }` when scorecard present; `"not-indexed"` ⇒
   `unavailable(not-indexed)`; `"invalid-response"` ⇒
   `unavailable(source-error)`; project exists but no scorecard field ⇒
   `unavailable(no-scorecard)`; thrown network errors as in 3.
5. Only ONE `getProject` call per run (share the promise between 3 & 4).
6. Evidence URLs: `https://github.com/{owner}/{repo}` for status; for
   scorecard, build from the shared `projectId(owner, repo)` helper
   (issue 18): `https://deps.dev/project/<urlencoded projectId>`.

## Acceptance Criteria

- [ ] Split tests: (a) declared-null and non-GitHub inputs ⇒ immediate
      unavailable with ZERO client calls (spies); (b) GitHub inputs only:
      matrix {getRepo ok/not-found/rate-limited/invalid-response/throws} ×
      {getProject ok/not-indexed/invalid-response/throws} produces exactly
      the documented statuses/values (table-driven).
- [ ] Rate-limited + deps.dev-ok yields evaluated status with partial
      fields and fallback evidence sentence.
- [ ] `getProject` called at most once per collect (spy assertion).
- [ ] Archived + stale-push fixture carries both fields for rules
      R-REPO-003/004 to consume.
- [ ] Hostile declared URL (`javascript:…`, `github.com.evil.com`) ⇒
      unsupported-host unavailable (no client call made — spy).

## Validation

- `npm run lint && npm run typecheck && npm test -- collectors/repository`.

## Dependencies

- 19, 16, 18.

## Non-goals

- No package-name-appears-in-repo verification (v2 known unknown), no
  GitLab/Codeberg (U7), no commit-history analysis.

## Design References

- DESIGN.md §9.2 rows 9–11; ADR-006; U7
