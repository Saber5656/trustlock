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
   - `workflow_dispatch` with `version` input — the workflow FAILS unless
     `inputs.version` is valid semver AND equals `package.json.version`
     AND, when the ref is a tag, the tag is exactly `v${inputs.version}`
     (prevents publishing a mismatched ref);
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
5. Packaging verification — split to respect the S1 lint scope: an npm
   script `verify:pack` runs
   `npm pack --dry-run --json > pack-report.json` (shell level) followed
   by `tsx scripts/verify-pack.ts pack-report.json`; the script (in
   `scripts/`, where dev-only code lives) parses the JSON and fails unless
   the file list is exactly dist/**, data/**, README.md, LICENSE,
   package.json, the bin mapping is `vetlock → dist/cli/index.js`, and
   package.json has **no `preinstall`/`install`/`postinstall` scripts**
   (S8). CI (issue 02's package-audit job) switches to `npm run
   verify:pack`.

## Acceptance Criteria

- [ ] `npm run verify:pack` enforces the exact file list, bin mapping, and
      the no-install-scripts rule (green locally + in CI).
- [ ] Release workflow lints (actionlint or careful review), defaults to
      dry-run, and its non-dry path publishes with `--provenance` under the
      `release` environment.
- [ ] RELEASING.md covers dispatch AND local-publish paths, npm 2FA, and
      environment-protection setup (owner-manual steps flagged as such).
- [ ] No secrets referenced outside the single publish step.
- [ ] Release workflow rejects a `version` input that differs from
      package.json.version (asserted via a dry-run dispatch with a wrong
      version, linked in the PR).
- [ ] CHANGELOG scaffold present; `verify:pack` green in CI.

## Validation

- CI green incl. new packaging test; a `dry_run: true` dispatch run linked
  in the PR (proves the workflow executes end-to-end without publishing).

## Dependencies

- 40 (suite must be green to make the workflow meaningful), 02.
  (ISSUE_PLAN table: 40, 02.)

## Non-goals

- The actual publish/2FA/npm-token creation (owner-manual — ISSUE_PLAN §8),
  no brew formula, no standalone binaries, no auto-release-on-tag.

## Design References

- DESIGN.md §19; ADR-002; ADR-007 S8; ISSUE_PLAN §8 (manual steps)
