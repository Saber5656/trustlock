# Title

npm manifest & lockfile reader

## Summary

Implement `src/core/ecosystems/npm/manifest.ts`: read `package.json` and
`package-lock.json` (lockfileVersion 2/3) from a project directory and
produce the `DirectDependency[]` list (name, declared range, group, resolved
exact version, integrity, lockPresent) that `verify` consumes.

## Context

DESIGN §8.1. Manifests are **untrusted input** (S3): a cloned repository may
contain hostile names or structures. The reader must be defensive and must
distinguish "no lockfile" from "dep missing from lockfile".

## Scope

- `src/core/ecosystems/npm/manifest.ts` + fixtures under
  `test/fixtures/projects/npm-*/` + unit tests.

## Detailed Requirements

1. `detectNpmProject(dir): Promise<boolean>` — true iff `package.json`
   exists and parses as a JSON object.
2. `readNpmDirectDependencies(dir): Promise<DirectDependency[]>`:
   - Parse `package.json`; collect entries from `dependencies` → group
     `prod`, `devDependencies` → `dev`, `optionalDependencies` → `optional`.
     A name in multiple groups keeps the first per that order (and logs
     debug).
   - **Every name is validated** with `validateNpmName`; invalid names are
     skipped with a `warn` log naming the manifest path (never thrown — a
     hostile manifest must not brick `verify`; the skip is visible).
   - Declared range = the raw string value (sanitized for control chars at
     render time, not here). Non-string values ⇒ skip + warn.
   - Non-registry ranges (`file:`, `link:`, `git+…`, `github:`,
     `workspace:`, http(s) tarball URLs) — normative rule: return the dep
     with its computed `group`, `resolvedVersion: undefined`, and
     `lockPresent` reflecting actual lockfile presence. Verify will surface
     them as `unapproved`; approving non-registry dependencies is out of
     scope for v1 (documented again in issues 37 and 41).
   - Lockfile: read `package-lock.json` if present.
     `lockfileVersion` ∈ {2,3} required; 1/absent/unparsable ⇒ all deps get
     `lockPresent: false` (and one `warn` explaining: "regenerate with
     npm install (npm ≥ 7)" / "lockfile missing").
     Resolution: `packages["node_modules/<name>"]` → `version`, `integrity`.
     Missing entry for a declared dep ⇒ that dep `lockPresent: false`.
   - If root `package.json` has `workspaces` ⇒ single `warn`: "npm
     workspaces detected; v1 verifies the root package only" (U8).
3. All JSON parsing wrapped: syntax errors ⇒ `ProjectError` with file path
   (for `package.json`); lockfile syntax errors ⇒ treated as absent
   lockfile + warn (verify must still run).
4. Output sorted by (group order prod/dev/optional, then name) —
   deterministic.

## Acceptance Criteria

- [ ] Fixture `npm-basic` (2 prod, 1 dev, lockfile v3) yields exact expected
      `DirectDependency[]` including integrity strings.
- [ ] Fixture `npm-lock-v1` ⇒ all `lockPresent: false` + the guidance warn.
- [ ] Fixture `npm-no-lock` ⇒ same shape, "lockfile missing" warn.
- [ ] Fixture `npm-hostile` (names: `../evil`, `UPPER`, `a b`, numeric value,
      `file:../x` range, prototype-pollution key `__proto__`) ⇒ invalid
      names skipped with warns, `file:` dep passes through per rule above,
      object prototype not polluted (explicit assertion), no throw.
- [ ] Fixture `npm-workspaces` ⇒ root deps returned + workspaces warn.
- [ ] Dep in lockfile but not manifest is NOT returned (direct deps only).

## Validation

- `npm test -- npm/manifest`; fixtures committed under
  `test/fixtures/projects/`.

## Dependencies

- 07 (types), 08 (name validation), 03 (errors/log).

## Non-goals

- No workspaces enumeration, no yarn/pnpm lockfiles, no transitive deps,
  no network.

## Design References

- DESIGN.md §8.1, §16.1 (hostile-repo row); ADR-007 S3/S6; U8
