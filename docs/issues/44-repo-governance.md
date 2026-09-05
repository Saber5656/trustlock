# Title

Repository governance & automation files

## Summary

Add the community and automation files: Dependabot, CodeQL workflow,
CONTRIBUTING.md, PR/issue templates, and CODEOWNERS. Configuration-only
issue; no product code.

## Context

DESIGN §20 / ADR-007 S8: an OSS security tool's repository hygiene is part
of its trust surface (and feeds its own Scorecard signal). These files are
independent of product code and can land any time after scaffolding.

## Scope

- `.github/dependabot.yml`, `.github/workflows/codeql.yml`,
  `CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`,
  `.github/ISSUE_TEMPLATE/{bug_report.md,feature_request.md,config.yml}`,
  `.github/CODEOWNERS`.

## Detailed Requirements

1. `dependabot.yml`: weekly `npm` (grouped: dev-deps vs the four runtime
   deps separately — runtime-dep PRs must be individually visible per
   ADR-002) and weekly `github-actions` updates; commit-message prefix
   `deps:` / `ci:`.
2. `codeql.yml`: `javascript-typescript` default setup as a workflow;
   triggers: PR to main + weekly schedule; `permissions: security-events:
   write, contents: read`; actions SHA-pinned (issue 02 conventions).
3. `CONTRIBUTING.md`: dev setup (Node ≥ 22.12, `npm ci`), command
   cheatsheet listing exactly the npm scripts that exist in package.json
   at implementation time (verified by reading it — e.g. `test:e2e`
   appears only if issue 40 has landed), fixture-recording guidance
   (public data only, trim large maps), the stdlib-first rule
   (styleText/fetch — ADR-002), the runtime-dependency ADR requirement, the
   security invariants pointer (ADR-007: hostile-input rules for every
   parser change), and the DESIGN.md-first change process (docs before
   code for behavior changes).
4. PR template: checklist — tests added, golden snapshots updated
   consciously, docs/signals.md updated when rules/signals change,
   security-invariant impact considered (S1–S8), no new runtime deps
   without ADR.
5. Issue templates: bug (version, command, expected/actual, `--verbose`
   output note with token-redaction warning) and feature request; config
   disables blank issues and links Security Advisories for
   vulnerabilities.
6. `CODEOWNERS`: `* @Saber5656`.

## Acceptance Criteria

- [ ] Dependabot config valid (GitHub UI shows both ecosystems watched);
      runtime deps ungrouped, dev deps grouped.
- [ ] CodeQL run completes green on the PR.
- [ ] Templates render correctly (open a scratch draft issue/PR to verify;
      links in PR description).
- [ ] All workflow actions SHA-pinned; permissions minimal.
- [ ] CONTRIBUTING accurately reflects the current scripts and rules (no
      aspirational content).

## Validation

- CI + CodeQL green; template rendering screenshots or links in the PR.

## Dependencies

- 01 (repo layout, scripts); 02 (pinning conventions).
  (ISSUE_PLAN table: 01, 02.)

## Non-goals

- No branch-protection/ruleset changes (already owner-managed), no
  Scorecard action, no funding files, no docs-site tooling.

## Design References

- DESIGN.md §20; ADR-002 (dependency ADR rule); ADR-007 S8
