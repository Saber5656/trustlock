# Title

GitHub client & repository-URL parsing

## Summary

Implement `src/infra/github.ts`: parse `owner/repo` out of the repository
URL forms found in registry metadata, and fetch repository status (exists,
archived, last push, stars) from the GitHub REST API with an optional
env-provided token and graceful rate-limit degradation.

## Context

GitHub data enriches `repository.status` (DESIGN §9.2). Unauthenticated
callers get 60 req/h — hitting the limit must degrade the signal, not break
the run (ADR-006 #2). The token is the only secret vetlock ever touches
(S5). Repository URLs come from hostile metadata (S3): parse defensively.

## Scope

- `src/infra/github.ts`, fixtures under `test/fixtures/http/github/`, unit
  tests. Consolidate the `projectIdFromRepoUrl` helper started in issue 16.

## Detailed Requirements

1. `parseGitHubRepo(url: string): { owner: string; repo: string } | null`:
   - Accept forms: `https://github.com/o/r`, `http://github.com/o/r`,
     `git+https://github.com/o/r.git`, `git://github.com/o/r.git`,
     `ssh://git@github.com/o/r`, `git@github.com:o/r.git`,
     `github:o/r` (npm shorthand), trailing `.git`, trailing `/`, extra
     path segments (`/tree/main/...` → keep first two segments), and
     `owner/repo#hash` fragments (strip).
   - Validate `owner` and `repo` against `/^[A-Za-z0-9._-]{1,100}$/` after
     extraction; anything else ⇒ null. Non-GitHub hosts ⇒ null (U7).
   - `projectId(owner, repo)` = `github.com/{owner}/{repo}` lowercased —
     shared with the deps.dev client (issue 16 switches to this helper).
2. `createGitHubClient(infra, token?: string)`:
   - `getRepo(owner, repo): Promise<GitHubRepo | "not-found" | "rate-limited">`
     via `GET https://api.github.com/repos/{owner}/{repo}` with headers
     `Accept: application/vnd.github+json`, `X-GitHub-Api-Version:
     2022-11-28`; token (when present) via the http client's
     GitHub-host-only auth (issue 04).
   - consumed fields: `full_name`, `archived`, `pushed_at`,
     `stargazers_count`, `open_issues_count`, `default_branch`,
     `license?.spdx_id`.
   - 404 ⇒ `"not-found"`; 403/429 with `x-ratelimit-remaining: 0` or
     rate-limit message ⇒ `"rate-limited"` (log info suggesting
     `VETLOCK_GITHUB_TOKEN`); other 403 ⇒ `"rate-limited"` as well
     (conservative), other errors propagate `NetworkError`.
   - TTL `CACHE_TTLS.github`.
3. Token sourcing happens in the CLI wiring (issue 39):
   `VETLOCK_GITHUB_TOKEN ?? GITHUB_TOKEN`; this module only accepts an
   optional string. It never logs it (S5 test).

## Acceptance Criteria

- [ ] URL parsing table (≥ 18 cases) covering every listed form, plus:
      GitLab URL ⇒ null, `javascript:alert(1)` ⇒ null,
      `https://github.com/onlyowner` ⇒ null,
      `https://github.com.evil.com/o/r` ⇒ null (host must be exactly
      `github.com`), 150-char owner ⇒ null.
- [ ] getRepo happy path (real recorded `expressjs/express`, trimmed),
      404 path, rate-limit path (`403` + `x-ratelimit-remaining: 0`
      fixture) each mapped to the documented variant.
- [ ] Auth header present iff token given (positive/negative captured).
- [ ] Cache-key test: token value does NOT change the cache key and never
      appears in the cache dir contents (S5).
- [ ] Issue-16 duplicate helper removed; both clients import the shared one.

## Validation

- `npm test -- github`.

## Dependencies

- 04, 05 (and touches 16 for consolidation).

## Non-goals

- No GraphQL, no commit/contributor listing, no non-GitHub forges (U7 —
  v1.1 candidate), no token acquisition UX.

## Design References

- DESIGN.md §14.3, §9.2 (repository.status), §16.4; ADR-006; U7
