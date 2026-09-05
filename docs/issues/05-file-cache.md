# Title

File cache (TTL, ETag revalidation, offline mode)

## Summary

Implement `src/infra/cache.ts` and `src/infra/paths.ts`: a JSON file cache
keyed by hashed URL, with per-source TTLs, ETag revalidation through the
HTTP client, `--offline` semantics, and safe cache-directory resolution on
macOS/Linux/Windows. Expose a `CachedHttp` facade that all API clients use.

## Context

Every data source is cacheable JSON (research/data-sources.md §5). The cache
makes warm `check` runs fast (§18 budgets), enables `--offline`, and is a
security surface: keys must be hashed (S3) and contents are public data only
(S5).

## Scope

- `src/infra/paths.ts` (cache dir resolution), `src/infra/cache.ts`
  (store + `CachedHttp`), unit tests using temp dirs and a fake HttpClient.

## Detailed Requirements

1. `resolveCacheDir(explicit: string|null, env, platform, homedir): string`:
   explicit → `VETLOCK_CACHE_DIR` → platform default:
   macOS `~/Library/Caches/vetlock`; Linux `$XDG_CACHE_HOME/vetlock` else
   `~/.cache/vetlock`; Windows `%LOCALAPPDATA%\vetlock\Cache`.
   Directory created lazily with mode `0o700` (best-effort on Windows).
2. Store format: one file per entry at
   `<cacheDir>/v1/<sha256(method + " " + url)>.json` containing
   `{ url, fetchedAt (ISO), etag?, status, body }`. The `v1/` segment allows
   future format migration by directory bump.
3. `CachedHttp.getJson(url, { ttlMs, offline, allow404? })` — `allow404`
   is passed through to the underlying `HttpClient` (issue 04) and a 404
   result is cached (negative caching) only when `allow404: true`.
   Behavior:
   - offline=false: fresh entry (age < ttlMs) ⇒ return cached; stale with
     etag ⇒ conditional GET (304 ⇒ refresh `fetchedAt`, return cached; 200 ⇒
     rewrite entry); stale without etag ⇒ plain GET and rewrite; network
     failure on revalidation of an *existing* entry ⇒ return stale cached
     value and log a warning (fail-open, P4).
   - offline=true: any-age entry ⇒ return it; miss ⇒ throw
     `OfflineMissError(url)`.
   - `status: 404` entries are cacheable (negative caching) with the same
     TTL — callers see the cached 404 exactly as a live one.
4. TTL constants exported as `CACHE_TTLS`:
   `registry: 3_600_000`, `downloads: 86_400_000`, `depsdev: 86_400_000`,
   `osv: 3_600_000`, `github: 3_600_000` (ms). Callers pick by source.
5. Writes atomic (tmp file + rename in same dir). Corrupt/unparseable entry
   files are deleted and treated as a miss (never fatal). Entry files are
   written with mode 0o600 (best-effort on Windows).
6. POST responses (OSV querybatch) are cacheable too:
   key = sha256(`POST <url> <sha256(bodyJson)>`); expose
   `postJson(url, body, { ttlMs, offline })`.
7. No eviction in v1 (documented); `vetlock cache clear` is not a command —
   users delete the directory (README note lands in issue 41).

## Acceptance Criteria

- [ ] Fresh-hit, stale-revalidate-304, stale-refetch-200, offline-hit,
      offline-miss, corrupt-entry, negative-404 paths each covered by a test.
- [ ] Cache file names match `/^[0-9a-f]{64}\.json$/` — no user-controlled
      bytes in paths (test constructs a hostile URL and inspects dir).
- [ ] Stale value served (with warning) when revalidation request throws.
- [ ] Same URL cached under GET and POST(body) never collide.
- [ ] `resolveCacheDir` table test covers all three platforms + both env
      overrides.
- [ ] Permissions: cache dir created `0o700` and entry files `0o600`
      (asserted on POSIX; skipped-with-comment on Windows).
- [ ] Atomicity: injected rename failure leaves the previous entry intact;
      successful write leaves no `.tmp` file behind.

## Validation

- `npm run lint && npm run typecheck && npm test -- cache paths`; tests run against `fs` in a temp dir
  (`fs.mkdtemp`), no network.

## Dependencies

- 01, 03 (error classes), 04 (HttpClient interface).
  (ISSUE_PLAN table lists the same three.)

## Non-goals

- No TTL tuning per specific endpoint beyond `CACHE_TTLS`; no size-based
  eviction; no cache subcommand.

## Design References

- DESIGN.md §14.2; ADR-006 (caching decision); ADR-007 S3/S5/S7 (atomicity)
