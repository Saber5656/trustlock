# Title

HTTP client with host allowlist, timeout, retry, size cap

## Summary

Implement `src/infra/http.ts`: the single module through which every network
request in vetlock flows. Enforces the fixed host allowlist, HTTPS-only,
timeouts, bounded retries, response-size caps, and standard headers.

## Context

This module is security control S2 (ADR-007): the allowlist is code, not
configuration. All registry/enrichment clients (issues 09, 13, 16–18) build
on it; the cache (issue 05) wraps it.

## Scope

- `src/infra/http.ts` + unit tests with injected fake `fetch`.

## Detailed Requirements

1. Exported constant
   `ALLOWED_HOSTS = ["registry.npmjs.org","api.npmjs.org","pypi.org","pypistats.org","api.deps.dev","api.osv.dev","api.github.com"] as const`.
2. `interface HttpClient { getJson(url, opts?): Promise<HttpJsonResult>; postJson(url, body, opts?): Promise<HttpJsonResult> }`
   where `HttpJsonResult = { status: number; etag?: string; body: unknown }`
   and `opts = { headers?, etag? (sends If-None-Match), timeoutMs?, allow404?: boolean }`.
   `postJson` construction: body = `JSON.stringify(body)`, headers
   `Content-Type: application/json` and `Accept: application/json` set by
   default; caller-provided `opts.headers` are merged on top and win on
   key conflict (case-insensitive), except `Authorization` which is always
   owned by the client (caller values ignored + debug log).
3. `createHttpClient(deps: { fetchImpl?: typeof fetch; token?: string; userAgent: string })`
   — `fetchImpl` injectable for tests; `token` attached as
   `Authorization: Bearer <token>` **only** when `new URL(url).host === "api.github.com"`.
4. Request pipeline:
   - Parse URL; scheme must be `https:` and host ∈ `ALLOWED_HOSTS`, else
     throw `NetworkPolicyError` (include host, never the token).
   - `User-Agent` always set.
   - Timeout via `AbortController`, default 10_000 ms.
   - Redirects: use `redirect: "manual"`; on 301/302/307/308 follow at most
     3 hops, re-validating scheme+allowlist per hop; otherwise
     `NetworkPolicyError`.
   - Retries: GET only; on network error or status ≥ 500, retry up to 2
     times with 250 ms then 750 ms delay (+ full jitter up to 100 ms); never
     retry POST or 4xx.
   - Response body read as a stream, counting bytes; abort and throw
     `ResponseTooLargeError` beyond 5 MiB (5 * 1024 * 1024).
   - `status 304` returns `{ status: 304 }` without body; `404` returns
     normally only when `opts.allow404`, else throws `NetworkError` with
     `status: 404` set (semantic mapping to `RegistryError` happens in
     callers). Other non-2xx statuses (after retry policy) likewise throw
     `NetworkError` carrying the status.
   - JSON parse failures ⇒ `NetworkError("invalid JSON from <host>")`.
5. No cookies, no keepalive config beyond defaults, no proxy code (undici's
   env-proxy behavior is left as-is and documented in a comment).
6. Log at `debug`: method, URL (path truncated to 100 chars), status,
   duration, retry count. Never log headers (S5).

## Acceptance Criteria

- [ ] Request to `https://evil.example.com/x` throws `NetworkPolicyError`
      without calling `fetchImpl` (asserted via spy).
- [ ] `http://registry.npmjs.org/x` (plain HTTP) throws `NetworkPolicyError`.
- [ ] Redirect chain registry→evil host throws; registry→pypi.org (both
      allowlisted) follows; 4-hop chain throws.
- [ ] 5xx GET retried exactly 2 times then throws; POST never retried; 429
      not retried (4xx).
- [ ] 6 MiB fixture body throws `ResponseTooLargeError` before full read.
- [ ] Token attached for `api.github.com` only (positive + negative test).
- [ ] ETag round-trip: `etag` opt sends `If-None-Match`; 304 handled.
- [ ] Timeout: default 10_000 ms enforced (fake timers: request aborts and
      throws `NetworkError`), `opts.timeoutMs` override respected.
- [ ] `User-Agent` present on every request (captured-request assertion,
      GET and POST).
- [ ] S5: with an injected fake logger at debug level, no log line contains
      the token value or any `Authorization` header content.

## Validation

- `npm run lint && npm run typecheck && npm test -- http` green; tests use only injected fakes (no sockets).
- Include a test that iterates `ALLOWED_HOSTS` and asserts each parses as a
  bare hostname (no scheme/port/path) — guards accidental weakening.

## Dependencies

- 01 (project), 03 (error classes).

## Non-goals

- No caching (issue 05), no per-source TTL knowledge, no client-specific
  parsing (issues 09/13/16–18).

## Design References

- DESIGN.md §14.1; ADR-006 (allowlist), ADR-007 S2/S5
