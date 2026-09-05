# Title

Python manifest reader (pyproject.toml + uv.lock)

## Summary

Implement `src/core/ecosystems/pypi/manifest.ts`: read PEP 621
`pyproject.toml` direct dependencies (plus optional-dependencies and
PEP 735 dependency-groups), resolve exact versions through `uv.lock`, and
emit `DirectDependency[]`.

## Context

DESIGN §8.2. `uv.lock` is uv-specific and unstandardized (U3): the reader
must check the lockfile's `version` field and degrade politely on unknown
revisions. Manifests are untrusted input (S3). PEP 508 strings are reduced
to names with a minimal extractor — a known risk area that needs exhaustive
table tests.

## Scope

- `src/core/ecosystems/pypi/manifest.ts`, PEP 508 name extractor, fixtures
  under `test/fixtures/projects/pypi-*/`, unit tests.

## Detailed Requirements

1. `detectPypiProject(dir)` — true iff `pyproject.toml` exists, parses as
   TOML (`smol-toml`), and contains a `[project]` table.
2. PEP 508 name extraction — `extractRequirementName(req: string)`:
   - name = longest leading match of `/^[A-Za-z0-9]([A-Za-z0-9._-]*[A-Za-z0-9])?/`
     after trimming leading whitespace; the character after the match must
     be one of `[`, `(`, `<`, `>`, `=`, `!`, `~`, `;`, `@`, space, or
     end-of-string — otherwise the requirement is malformed ⇒ skip + warn.
   - URL requirements (`name @ https://…`) keep the name, and are treated
     as non-registry deps (same normative rule as issue 10:
     `resolvedVersion: undefined`; `lockPresent: true` only when a
     supported uv.lock contains a matching `[[package]]` entry, else
     `false`).
   - Every extracted name must pass `validatePypiName` (issue 12, incl.
     the 214-char cap) — failures skip + warn; the survivor is the PEP
     503-normalized form.
3. Groups (PEP 621): the `dependencies` **array key inside the `[project]`
   table** → `prod`; each array under `[project.optional-dependencies]` →
   `optional`; `[dependency-groups].<group>` (PEP 735, best-effort:
   flatten string entries; `{include-group}` tables are followed one
   level, cycles ignored) → `dev`.
4. `uv.lock` resolution:
   - Parse TOML; top-level `version` must equal `1` — anything else ⇒ all
     deps `lockPresent: false` + one warn "uv.lock revision <v> not
     supported by this vetlock version" (U3). Absent file ⇒ same shape,
     "uv.lock missing — run `uv lock`".
   - Index `[[package]]` entries by normalized `name` where the entry's
     `source` is a registry source (has `source.registry`) — skip
     `virtual`/`editable`/`directory`/`git` sources for resolution.
   - Match each direct dep by normalized name → `resolvedVersion =
     entry.version`, `lockPresent: true`; unmatched ⇒ `lockPresent: false`.
   - No integrity extraction in v1 (PyPI integrity pinning is deferred —
     DESIGN §3.3).
5. TOML parse errors: `pyproject.toml` ⇒ `ProjectError` (path + line if
   available); `uv.lock` ⇒ treated as unsupported-revision path (warn,
   lockPresent false) — verify must still run.
6. Hostile-input rules identical to issue 10: invalid names skipped with
   warn; no prototype pollution (TOML lib output treated as data, copied
   into null-prototype maps keyed by normalized name); file size caps —
   `pyproject.toml` 5 MiB (`ProjectError` beyond), `uv.lock` 50 MiB
   (treated as unsupported + warn beyond).
7. Output ordering: (prod, dev, optional) then name — same convention as npm.

## Acceptance Criteria

- [ ] Fixture `pypi-uv-basic` (3 prod incl. one with extras
      `requests[socks]>=2.31`, 1 optional, uv.lock v1) → exact expected
      list with resolved versions.
- [ ] Extractor table tests (≥ 20): `requests`, `requests>=2`,
      `requests[socks]`, `requests[socks,use_chardet_on_py3]==2.32.4`,
      `name @ https://example.com/x.whl`, `name; python_version<"3.12"`,
      `Django~=5.0`, `zope.interface`, `A.B-C_d`, malformed (`==2.0`,
      `[extra]x`, empty, unicode letters, leading digit ok, `-leading`).
- [ ] `uv.lock` with `version = 2` ⇒ all lockPresent false + specific warn.
- [ ] Missing uv.lock ⇒ lockPresent false + "run `uv lock`" warn.
- [ ] Dep resolved via uv.lock only from registry-source entries (git-source
      fixture entry ignored).
- [ ] PEP 735 `[dependency-groups]` flattening incl. one `include-group`
      hop.
- [ ] Hostile fixtures: invalid/overlong requirement names skipped with
      warn; `__proto__`-keyed TOML tables cause no prototype mutation
      (explicit assertion); corrupt `pyproject.toml` ⇒ `ProjectError` with
      path; corrupt `uv.lock` ⇒ warn + `lockPresent: false`, no throw.
- [ ] Oversized `pyproject.toml`/`uv.lock` follow the size-cap rules.

## Validation

- `npm run lint && npm run typecheck && npm test -- pypi/manifest`; extractor has 100 % branch coverage.

## Dependencies

- 07, 12, 03 (errors/log). (`smol-toml` is available from issue 01,
  transitively guaranteed; ISSUE_PLAN table lists 07, 12, 03.)

## Non-goals

- No requirements.txt / poetry.lock / pylock.toml (v2), no marker/extras
  semantics beyond name identity, no Python interpreter invocation (S1).

## Design References

- DESIGN.md §8.2, §16.1; ADR-007 S1/S3/S6; U3
