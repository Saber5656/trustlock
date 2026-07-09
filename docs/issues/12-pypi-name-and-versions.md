# Title

PyPI name normalization & version handling

## Summary

Implement the PyPI-specific primitives: PEP 503 name validation and
normalization, spec body parsing (`name==version` / `name@version`), PEP 440
syntactic version checking, and best-effort version ordering for display.

## Context

PyPI half of security boundary S3 and of the adapter primitives (DESIGN
§6.3, §7.2). Correctness rule: verify's behavior may depend only on string
equality of normalized names + exact versions; ordering is display-only
(known unknown U4).

## Scope

- `src/core/ecosystems/pypi/name.ts`, `src/core/ecosystems/pypi/versions.ts`
  + unit tests.

## Detailed Requirements

1. `name.ts`:
   - `validatePypiName(raw): NameValidation` — must match
     `/^[A-Za-z0-9]([A-Za-z0-9._-]*[A-Za-z0-9])?$/` (PEP 508 name), length
     1–214 (hard cap for hygiene), no whitespace/`%`.
   - Normalization (PEP 503): lowercase, then replace runs of `[-_.]+` with
     a single `-`. `NameValidation.normalized` carries the result
     (`Django` → `django`, `typing_extensions` → `typing-extensions`,
     `foo._-bar` → invalid (ends ok? `foo._-bar` ends with `r` — valid;
     normalizes `foo-bar`) — encode these in tests).
2. `versions.ts`:
   - `parseExactPypiVersion(raw)` — syntactic PEP 440 check via the
     canonical regex from the PEP 440 appendix (transcribe it into the
     module with a comment link); local versions (`+local`) allowed;
     wildcard/specifier syntax (`==1.*`, `>=`, `~=`, trailing `.*`) rejected
     with "exact version required".
   - `normalizePypiVersion(v)` — lowercase, strip leading `v`, collapse the
     forms PEP 440 defines as equal only insofar as: leading zeros kept
     as-is EXCEPT case/`v` (full canonicalization is NOT attempted in v1 —
     document; string equality happens on this light normalization).
   - `comparePypiVersions(a, b)` — best-effort: split release segment on
     `.`, numeric compare; pre/post/dev segments break ties in PEP 440
     order (dev < pre < release < post); document known limitations (U4)
     and that no verify logic uses this.
   - `isPrereleasePypi(v)` — contains `a|b|rc|dev` segment per the regex
     capture groups.
3. `parsePypiSpecBody(body)`:
   - split on `==` first; if absent, split on last `@`;
     `requests==2.32.4` / `requests@2.32.4` → same result;
     `requests==` ⇒ failure "empty version".

## Acceptance Criteria

- [ ] Normalization table (≥ 15 cases): `Django`, `typing_extensions`,
      `zope.interface` → `zope-interface`, `A--B__c..d` → `a-b-c-d`,
      leading/trailing separators invalid, single char valid, `_x` invalid.
- [ ] Version accepts: `2.32.4`, `1.0`, `1.0.0.4`, `2.0rc1`, `1.0.post1`,
      `1.0.dev3`, `1!2.0` (epoch), `1.0+local.1`; rejects: `==1.*`, `>=2`,
      `~=1.4`, `1.*`, `latest`, empty.
- [ ] Ordering sanity: `1.0.dev1 < 1.0rc1 < 1.0 < 1.0.post1 < 1.1`; `1.9 < 1.10`.
- [ ] Spec body: both `==` and `@` forms, precedence when both appear
      (`a@1==2` → split on `==` first ⇒ name `a@1` → name validation fails —
      test locks this in).
- [ ] 100 % branch coverage on `name.ts`.

## Validation

- `npm test -- pypi/name pypi/versions`.

## Dependencies

- 07 (types).

## Non-goals

- No registry access (13); no full PEP 440 canonicalization or `packaging`
  parity (U4); no requirements.txt parsing.

## Design References

- DESIGN.md §6.3, §7.2; ADR-007 S3; U4
