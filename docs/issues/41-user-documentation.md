# Title

User documentation (README, usage guide, signals reference)

## Summary

Write the English user-facing documentation: rewrite `README.md`, add
`docs/usage.md` (workflows, CI recipes, ledger conventions) and
`docs/signals.md` (every signal and rule with its exact threshold and
rationale).

## Context

DESIGN §20. The README currently contains one Japanese line from the repo's
bootstrap. Docs are part of the product's trust story: published rules with
thresholds (Scorecard lesson, research §4) and honest scope statements.

## Scope

- `README.md`, `docs/usage.md`, `docs/signals.md`. (SECURITY.md belongs to
  issue 42; CONTRIBUTING/RELEASING to 43/44.)

## Detailed Requirements

1. `README.md` (replace entirely; English):
   - one-paragraph pitch (the §1.2 loop) + badges placeholder (CI);
   - install/quick start — commands must be verbatim-executable: use
     `npx vetlock …` consistently for all three commands
     (`npx vetlock check express`,
     `npx vetlock approve express@5.1.0 --reason "…"`,
     `npx vetlock verify`), plus an optional
     `npm install -g vetlock` note for bare `vetlock` usage; include real
     expected output snippets (from golden fixtures, trimmed);
   - "How it decides" section: link to docs/signals.md, state
     determinism/no-LLM/no-telemetry, the `unavailable`/`incomplete`
     honesty rule, and what vetlock does NOT do (issue 42 adds the
     SECURITY.md link into this section when SECURITY.md lands — leave a
     stable heading for it, do not link a nonexistent file);
   - CI recipe (plain step: `npx vetlock verify`);
   - supported ecosystems/manifests table (incl. the workspaces and
     requirements.txt limitations, with v2 pointers);
   - GITHUB_TOKEN note: `VETLOCK_GITHUB_TOKEN` preferred over
     `GITHUB_TOKEN`; optional; what improves with it; and the S5 facts
     users may rely on — env-only, sent to api.github.com only, never
     cached, never logged, never in reports (DESIGN §16.4);
   - license badge/footer (MIT).
2. `docs/usage.md`:
   - the three personas' workflows (§2) with full command transcripts;
   - ledger file anatomy: annotated `vetlock.json` example (fields,
     policy overrides with a worked example of remapping R-EXEC-001 to
     critical and disabling R-PROV-001);
   - team conventions: commit the ledger, review approvals in PRs,
     merge-conflict resolution note (§12.2);
   - offline mode & cache (locations per OS, `--offline`, cache deletion);
   - exit codes table (§5.4) and `--fail-on warn`;
   - troubleshooting: rate-limited GitHub signals, not-indexed deps.dev,
     lockfile-version errors, corrupt-ledger recovery.
3. `docs/signals.md` — generated-quality reference, hand-written in v1:
   - one row/section per signal (all 22): what it measures, source (with
     URL), per-ecosystem availability, `unavailable` reasons;
   - one row per rule (all 22): trigger predicate in plain English with the
     exact THRESHOLD constant value, default severity, rationale (2–3
     sentences), and policy-override id;
   - a "changing the defaults" section referencing policy syntax.
   - Canonical anchors: each signal section heading is exactly
     `### Signal: <id>` and each rule section `### Rule: <id>` — the
     consistency test (`test/docs.test.ts`) asserts every `SIGNAL_CATALOG`
     id and every `DEFAULT_RULES` id appears as such a heading exactly
     once (cross-references elsewhere in prose are unrestricted).
4. Style: en-US, sentence-case headings, no marketing superlatives; every
   factual claim about behavior must be true of the implementation at merge
   time (reviewer checks against golden outputs).

## Acceptance Criteria

- [ ] README quick-start commands work verbatim against the built CLI
      (manually verified; transcript in PR).
- [ ] docs.test.ts consistency test green (22 signals + 22 rules present).
- [ ] No Japanese text remains anywhere in README (`rg -P "[\p{Hiragana}\p{Katakana}\p{Han}]" README.md` → empty).
- [ ] Links between the three docs and DESIGN.md resolve (relative paths).
- [ ] Limitations sections cover: direct-deps-only, exact-version-only,
      workspaces, requirements.txt absence, non-registry deps, PyPI
      maintainer gap.

## Validation

- `npm run lint && npm run typecheck && npm test -- docs`; markdown link check (simple script or `rg`-based
  relative-path assertion in the docs test).

## Dependencies

- 39, 37 (documented behavior final; threshold values arrive transitively
  through 39→31). (ISSUE_PLAN table: 39, 37.)

## Non-goals

- No website, no i18n (Japanese README variant is a post-v1 option), no
  API docs (no public library API in v1).

## Design References

- DESIGN.md §20, §2, §5.4, §10.2; research §4 (published-rules lesson)
