# Title

CI workflow (lint, typecheck, test matrix)

## Summary

Add a GitHub Actions CI workflow that runs lint, typecheck, build, and the
test suite on every push and pull request, across a 3-OS × 2-Node matrix,
with SHA-pinned actions and minimal permissions.

## Context

DESIGN §17 requires fixture-only tests (no live network), a cross-platform
matrix (Windows path handling is known unknown U9), and ADR-007 S8 requires
pinned, least-privilege CI. This workflow is the quality gate every later
issue relies on.

## Scope

- `.github/workflows/ci.yml` only.

## Detailed Requirements

1. Triggers: `push` (branches: `main`), `pull_request` (all branches),
   `workflow_dispatch`.
2. Top level: `permissions: { contents: read }`. Set
   `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`.
3. Job `test`:
   - `strategy.matrix`: `os: [ubuntu-latest, macos-latest, windows-latest]`
     × `node: [22, 24]`; `fail-fast: false`.
   - Steps: checkout → setup-node (with `node-version: ${{ matrix.node }}`,
     `cache: npm`) → `npm ci` → `npm run lint` → `npm run typecheck` →
     `npm run build` → `npm run test`.
   - Lint step may run on ubuntu only (use a matrix `include`/`if` guard or a
     separate `lint` job — choose the separate-job form for clarity).
4. Job `package-audit` (ubuntu, Node 22): `npm ci` → `npm run build` →
   `npm pack --dry-run --json > pack.json` → a script step that fails if the
   file list contains anything outside `dist/`, `data/`, `README.md`,
   `LICENSE`, `package.json` (inline `node -e` is acceptable).
5. **All** `uses:` actions pinned to full commit SHAs with a trailing
   `# vX.Y.Z` comment (`actions/checkout`, `actions/setup-node`). Resolve
   the SHAs for the latest stable releases at implementation time
   (`gh api repos/actions/checkout/tags` or the releases page).
6. No secrets are read anywhere in this workflow.

## Acceptance Criteria

- [ ] CI runs on a PR and all matrix legs pass.
- [ ] Every `uses:` reference is a 40-char SHA (grep in review).
- [ ] Workflow has no `write` permission anywhere.
- [ ] `package-audit` job fails if a stray file (e.g. `test/` or `.env`) is
      added to the pack list (verified once by temporarily adding a file to
      `files` in a scratch branch, or by unit reasoning documented in PR).
- [ ] Total wall time of the ubuntu/22 leg ≤ 5 minutes at current repo size.

## Validation

- Open a draft PR after adding the workflow; attach the green run URL.
- Intentionally break lint in a scratch commit to confirm CI fails, then
  revert (include both run links in the PR description).

## Dependencies

- 01 (scripts must exist).

## Non-goals

- No release/publish workflow (issue 43), no CodeQL/Dependabot (issue 44),
  no coverage upload service.

## Design References

- DESIGN.md §17 (CI matrix), §16.7 / ADR-007 S8 (pinning, permissions)
