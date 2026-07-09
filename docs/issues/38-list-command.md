# Title

list command

## Summary

Implement `src/cli/cmd-list.ts`: print the ledger contents with optional
decision/ecosystem filters, in terminal or JSON form. Read-only, offline,
small.

## Context

DESIGN §5.1/§5.3. `list` is the inspection surface for the ledger — used in
reviews ("what have we approved?") and by agents (`--json`).

## Scope

- One command module + tests.

## Detailed Requirements

1. Flow: `findLedger`/`loadLedger` (missing ⇒ exit 2 with the §12.4 hint);
   apply filters; render; exit 0 (an empty result is not an error).
2. Flags: `--decision <approved|rejected>`, `--ecosystem <npm|pypi>`
   (values validated ⇒ `UsageError` otherwise; both combinable).
3. Terminal output: one line per entry, ledger order (already canonical):
   `npm  left-pad@1.3.0   approved  2026-07-08  Yasushi Takagi  "small, zero deps"`
   — columns: ecosystem, name@version, decision, reviewedAt date part,
   reviewedBy, quoted reason (truncated 60 chars, sanitized). Footer:
   `12 entries (10 approved, 2 rejected)` reflecting filters.
4. `--format json`: `{ schemaVersion: 1, ledgerPath, entries: [...],
   summary: { approved, rejected, total } }` — entries are the raw ledger
   entries (already-validated shapes; unknown keys included).
5. All dynamic strings sanitized for terminal (issue 33); JSON emits raw
   values.
6. `--format markdown` ⇒ `UsageError` (check-only, same wording as 37).

## Acceptance Criteria

- [ ] Golden terminal + JSON outputs for the issue-35 golden ledger.
- [ ] Filter matrix: decision-only, ecosystem-only, both, no-match (empty
      table + `0 entries` footer, exit 0).
- [ ] Corrupt ledger exit 2; missing ledger exit 2 with hint.
- [ ] Hostile reason string (ANSI) renders sanitized in terminal, raw in
      JSON.

## Validation

- `npm test -- cmd-list`.

## Dependencies

- 35, 03, 33.

## Non-goals

- No editing/pruning (stale cleanup is manual or v2 `vetlock prune`), no
  sorting flags, no pagination.

## Design References

- DESIGN.md §5.1, §5.3, §12
