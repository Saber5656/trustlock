# Title

Release packaging & publish workflow

## Summary

Finalize the npm package for publication: package.json release fields,
`npm pack` verification, the manual-dispatch release workflow with
provenance, `RELEASING.md`, and the CHANGELOG scaffold. Publishing itself
remains a human action.

## Context

DESIGN §19: merge ≠ release; the workflow prepares everything so the owner
can release with one authorized action. Provenance-enabled publishing is
part of vetlock practicing what it preaches (S8) — vetlock's own `check`
should eventually show its provenance signal green.

## Scope

- package.json final fields, `.github/workflows/release.yml`,
  `RELEASING.md`, `CHANGELOG.md`, `npm pack` test additions.

## Detailed Requirements

1. package.json finalization:
   - `version: "0.1.0"` (release PR sets it; dev stays `-dev` suffixed);
   - `keywords` (10–12: supply-chain-security, dependency-review,
     typosquatting, provenance, npm, pypi, sbom-adjacent terms — final
     list at implementation);
   - deep-import lockdown (normative): only `bin` is public API in v1.
     Set `"exports": { "./package.json": "./package.json" }` — this blocks
     `import "vetlock/dist/…"` deep imports while keeping tooling that
     reads package.json working;
   - `publishConfig: { "provenance": true, "access": "public" }`;
   - verify `files`, `engines`, `bin` from issue 01 still exact.
2. `.github/workflows/release.yml`:
   - `workflow_dispatch` with `version` input (semver, validated);
   - `permissions: { contents: read, id-token: write }` (id-token for
     provenance);
   - jobs: full test suite → build → `npm pack` + file-list audit (reuse
     issue 02 logic) → `npm publish --provenance --access public` gated on
     an environment named `release` (environment protection = the human
     gate; configuring required reviewers on the environment is an
     owner-manual step listed in RELEASING.md);
   - `NPM_TOKEN` read from repo secrets ONLY in the publish step; the
     workflow must also work when the owner prefers local manual publish —
     publish step is skippable via input flag `dry_run: true` default
     **true** (safe default: a dispatched run without explicit override
     never publishes).
   - actions SHA-pinned, matching issue 02 conventions.
3. `RELEASING.md`: step-by-step owner runbook — version bump PR, tag
   `v0.1.0` after merge, dispatch workflow (or local
   `npm publish --provenance` with 2FA), post-publish verification
   (`npx vetlock@0.1.0 --version`, provenance visible on npmjs.com,
   `vetlock check vetlock` dogfood), rollback guidance (`npm deprecate`,
   never unpublish beyond the 72h window policy).
4. `CHANGELOG.md`: Keep-a-Changelog header + `## [Unreleased]` section;
   releasing moves entries under the version (documented in RELEASING.md).
5. Add a packaging test (`test/packaging.test.ts`): run
   `npm pack --dry-run --json` programmatically; assert file list, bin
   mapping, and that `data/top-packages/*.json` are included and
   `test/`/`docs/`/fixtures are not.

## Acceptance Criteria

- [ ] `npm pack --dry-run` file list exactly: dist/**, data/**, README.md,
      LICENSE, package.json (test-enforced).
- [ ] Release workflow lints (actionlint or careful review), defaults to
      dry-run, and its non-dry path publishes with `--provenance` under the
      `release` environment.
- [ ] RELEASING.md covers dispatch AND local-publish paths, npm 2FA, and
      environment-protection setup (owner-manual steps flagged as such).
- [ ] No secrets referenced outside the single publish step.
- [ ] CHANGELOG scaffold present; packaging test green in CI.

## Validation

- CI green incl. new packaging test; a `dry_run: true` dispatch run linked
  in the PR (proves the workflow executes end-to-end without publishing).

## Dependencies

- 40 (suite must be green to make the workflow meaningful), 02, 01.

## Non-goals

- The actual publish/2FA/npm-token creation (owner-manual — ISSUE_PLAN §8),
  no brew formula, no standalone binaries, no auto-release-on-tag.

## Design References

- DESIGN.md §19; ADR-002; ADR-007 S8; ISSUE_PLAN §8 (manual steps)
