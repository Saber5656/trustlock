# Title

CLI shell: entrypoint, global flags, typed errors, logging

## Summary

Implement the vetlock CLI shell: `commander` program with all global options,
the typed error hierarchy with exit-code mapping, the stderr logger, and a
`CliContext` object that downstream commands consume. Subcommands are
registered as stubs that print "not implemented yet" and exit 2.

## Context

DESIGN §5 fixes the CLI contract and §15 the error/exit-code contract. Every
command issue (36–39) plugs into this shell; getting flags, precedence, and
error trapping right once avoids divergence later.

## Scope

- `src/cli/index.ts`, `src/cli/context.ts`, `src/infra/errors.ts`,
  `src/infra/log.ts`, plus unit tests.

## Detailed Requirements

1. `src/infra/errors.ts` (DESIGN §15):
   - `abstract class VetlockError extends Error { readonly exitCode: 2; readonly hint?: string }`
     with subclasses `UsageError`, `ProjectError`, `NetworkError`
     (and its subclasses `NetworkPolicyError`, `ResponseTooLargeError`,
     `OfflineMissError`), `RegistryError`, `InternalError`.
   - All carry `message` and optional `hint`; `NetworkError` carries
     `url: string` (host-only when logged).
2. `src/infra/log.ts`: logger writing to **stderr** only, levels
   `error|warn|info|debug`; `info` shown unless `--quiet`; `debug` only with
   `--verbose`; a `redact(headers)` helper that replaces `authorization`
   values with `[redacted]` (test required — S5).
3. `src/cli/context.ts`: `buildContext(globalOpts, env): CliContext` where
   `CliContext = { format: "terminal"|"json"|"markdown", color: boolean,
   offline: boolean, cacheDir: string|null, ledgerPath: string|null,
   verbose: boolean, quiet: boolean }`. Precedence: flag > env > default.
   Env mapping: `VETLOCK_OFFLINE` (any non-empty ⇒ true),
   `VETLOCK_CACHE_DIR`, `NO_COLOR` (any value ⇒ color=false). Color default:
   `process.stdout.isTTY === true`.
4. `src/cli/index.ts`:
   - Program `vetlock`, version from `TOOL_VERSION`, description one-liner.
   - Global options exactly as DESIGN §5.2: `--json`, `--format <fmt>`
     (choices terminal|json|markdown; `--json` is sugar for
     `--format json`; if both given and conflicting ⇒ `UsageError`),
     `--ledger <path>`, `--offline`, `--cache-dir <path>`, `--no-color`,
     `--verbose`, `--quiet`.
   - Register stub subcommands `check <spec>`, `approve <spec>`,
     `reject <spec>`, `verify [dir]`, `list` — each currently throws
     `new InternalError("not implemented")`.
   - Top-level trap: catch `VetlockError` → print
     `error: <message>` (+ `hint: <hint>`) to stderr, and when
     `--format json` also print `{"error":{"code":"<ClassName>","message":…}}`
     to stdout; exit with `error.exitCode`. Unknown errors → wrap as
     `InternalError` with issue-filing hint. Commander usage errors must
     also exit 2 (configure `exitOverride`).
5. Exit-code invariant: only `0`, `1`, `2` are ever used (DESIGN §5.4);
   the shell owns 2; commands own 0/1.

## Acceptance Criteria

- [ ] `vetlock --help` lists all five subcommands and all global flags with
      the documented defaults.
- [ ] `vetlock check foo` (stub) exits 2 with `error: not implemented` on
      stderr and, with `--json`, a single JSON error document on stdout.
- [ ] `vetlock --format bogus check foo` exits 2 with a usage error.
- [ ] `NO_COLOR=1` and `--no-color` both yield `color: false` in context;
      flag wins over env in all precedence tests.
- [ ] Logger redaction test proves `Authorization` never appears in output.
- [ ] No output ever goes to stdout except command payloads / JSON error
      docs (asserted in tests by capturing streams).

## Validation

- Unit tests for: error mapping table, context precedence matrix (flag/env/
  default × each option), redaction, exitOverride behavior.
- Manual: run the built CLI (`npm run build && node dist/cli/index.js --help`).

## Dependencies

- 01.

## Non-goals

- No real command logic (issues 36–39), no HTTP (04), no ledger discovery
  logic (35 — `ledgerPath` stays a raw string|null here).

## Design References

- DESIGN.md §5 (CLI contract), §15 (errors), §16.4 (S5 redaction)
