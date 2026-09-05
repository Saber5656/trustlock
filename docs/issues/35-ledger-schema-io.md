# Title

Ledger schema & atomic IO

## Summary

Implement `src/core/ledger/schema.ts`, `src/core/ledger/io.ts`, and
`src/core/ledger/ops.ts`: the versioned `vetlock.json` schema (zod), load
with validation and unknown-key preservation, atomic ordered writes, ledger
discovery, and the upsert/query operations used by the commands.

## Context

The ledger is the product's persistent artifact (P5, DESIGN §12): boring,
diffable, merge-friendly, and never corrupted by a crashed write (S7). Its
schema is a compatibility surface from v1 onward.

## Scope

- Three modules + unit tests (temp-dir based).

## Detailed Requirements

1. `schema.ts` (zod):
   ```ts
   import { ecosystemIdSchema } from "../ecosystems/types.js"; // single source (issue 07)
   ledgerEntrySchema = z.object({
     ecosystem: ecosystemIdSchema,      // never restate ecosystem literals here (P6)
     name: z.string().min(1),
     version: z.string().min(1),
     decision: z.enum(["approved", "rejected"]),
     reviewedAt: z.string().datetime(),          // ISO 8601 UTC
     reviewedBy: z.string().min(1),
     reason: z.string().optional(),
     integrity: z.string().optional(),           // npm SRI
     reportDigest: z.string().optional(),        // sha256-<hex>
   }).passthrough();                             // unknown keys preserved
   ledgerFileSchema = z.object({
     version: z.literal(1),
     policy: policySchema.partial().optional(),  // from issue 30
     approvals: z.array(ledgerEntrySchema),
   }).passthrough();
   ```
   Exported TS types inferred from zod. Entry `name`/`version` fields
   additionally pass a conservative structural check defined here (S3,
   without pulling adapter modules in): length ≤ 250, no control chars, no
   whitespace, no `..`, at most one `/` (and only in `@scope/name` form),
   charset `[A-Za-z0-9@/._~-]` — full ecosystem grammar is enforced at use
   sites. Violations ⇒ `ProjectError` naming the entry index and field.
2. `io.ts`:
   - `findLedger(startDir, explicitPath?): string | null` — explicit path
     returned as-is (existence not required for approve); otherwise walk up
     from `startDir` looking for `vetlock.json`, stopping after the first
     directory that contains `.git` (inclusive) or at the filesystem root
     (DESIGN §12.4).
   - `loadLedger(path): LedgerFile` — missing file ⇒ `ProjectError`
     ("no ledger found; run vetlock approve to create one") — callers
     decide whether that is fatal; unparsable JSON / schema failure ⇒
     `ProjectError` with path + hint ("fix or restore via git; vetlock
     never overwrites a corrupt ledger").
     **S6 hardening**: read cap 5 MiB (`ProjectError` beyond); parsed
     objects are copied into null-prototype containers;
     `__proto__`/`constructor`/`prototype` keys at any level ⇒
     `ProjectError` (hostile-ledger fixtures required: oversized,
     malformed JSON, prototype-pollution keys).
   - `saveLedger(path, ledger, comparators: Record<EcosystemId, (a: string, b: string) => number>)`
     — sort `approvals` by (ecosystem, name, `comparators[ecosystem]`)
     with lexicographic fallback for an ecosystem missing from the map;
     commands build the map from the adapter registry
     (`adapter.compareVersions`). Serialize with fixed key order (version,
     policy?, approvals; entry keys in schema order), 2-space indent, LF,
     trailing newline; write `targetPath + ".tmp"` in the same dir
     (DESIGN §12.1), `fsync`, `rename` over target. Unknown top-level and
     entry keys are preserved **semantically** (values identical after a
     load→save round-trip; key ordering of unknown keys may normalize to
     sorted — JSON re-serialization cannot promise byte-level identity).
   - `createEmptyLedger(): LedgerFile` = `{ version: 1, approvals: [] }`.
3. `ops.ts`:
   - `upsertDecision(ledger, entry): { ledger: LedgerFile; replaced?: LedgerEntry }`
     — key (ecosystem, name, version); replace returns the previous entry
     (commands print it); pure (returns new object).
   - `findEntries(ledger, ecosystem, name): LedgerEntry[]`;
     `findExact(ledger, ecosystem, name, version): LedgerEntry | undefined`.
   - All lookups use normalized names (callers normalize; ops asserts
     lowercase for pypi in dev mode).
4. No clock reads; `reviewedAt` arrives from callers.

## Acceptance Criteria

- [ ] Round-trip: load(save(x)) deep-equals x incl. unknown keys
      (`"x-team-note": "…"`) at file and entry level.
- [ ] Ordering: shuffled input saves to canonically sorted output; version
      ordering uses semver for npm entries (1.10.0 > 1.9.0) — via adapter
      compare injected as a comparator map.
- [ ] Atomicity: simulate crash by making rename fail (mock fs) ⇒ original
      file intact; tmp file cleaned on success.
- [ ] Corrupt file (bad JSON, wrong version, invalid entry name `../evil`)
      ⇒ `ProjectError` with the documented hints; file untouched.
- [ ] Hostile ledger fixtures: >5 MiB file, `__proto__`-keyed entry ⇒
      `ProjectError`, no prototype mutation (explicit assertion).
- [ ] `findLedger`: found-in-parent, stops-at-git-boundary (fixture repo
      with `.git` dir between cwd and a decoy ledger above it), none-found.
- [ ] upsert insert/replace paths; replace returns previous entry.
- [ ] Golden ledger file fixture (byte snapshot) for docs/tests reuse.

## Validation

- `npm run lint && npm run typecheck && npm test -- ledger`.

## Dependencies

- 01, 03, 07 (`ecosystemIdSchema`, `EcosystemId`), 30 (policy schema).
  Version comparators arrive by injection — no dependency on 08/12.
  (ISSUE_PLAN table: 01, 03, 07, 30.)

## Non-goals

- No command wiring (36–38), no merge-conflict auto-resolution, no policy
  semantics (30 owns), no multi-ledger support.

## Design References

- DESIGN.md §12 (schema/IO/discovery), §10.3; ADR-007 S3/S7; P5
