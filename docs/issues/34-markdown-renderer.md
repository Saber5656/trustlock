# Title

Markdown renderer

## Summary

Implement `src/core/report/render-markdown.ts`: GitHub-flavored Markdown
rendering of a check report, suitable for pasting into PR descriptions (and
reused by the v2 GitHub Action).

## Context

DESIGN §11.3. Same content as the terminal renderer, different escaping
rules: Markdown injection (pipes, backticks, links) replaces ANSI as the
threat (S4 applies — sanitize first, then Markdown-escape).

## Scope

- One module + unit tests with golden fixtures.

## Detailed Requirements

1. `renderMarkdown(report): string` structure:
   1. `## vetlock report: <name>@<version> (<ecosystem>)`;
   2. verdict line: `**Verdict: FAIL** _(incomplete)_` with ❌ / ⚠️ / ✅
      emoji prefix per verdict;
   3. subject bullet list: registry link, resolved-from note, generated-at,
      tool version;
   4. one table per category **with at least one non-pass finding**:
      `| | Rule | Finding | Evidence |` rows: outcome glyph (❌ critical,
      ⚠️ warn, ▫️ notice/info, ❔ not-evaluable), rule id (inline code),
      sanitized+escaped detail, evidence as `[link](url)`;
   5. passed checks collapsed:
      `<details><summary>NN passed checks</summary>` + a simple list
      `</details>`;
   6. footer line with counts (same numbers as terminal footer).
2. Escaping pipeline for every dynamic string:
   `sanitize()` (issue 33) → `escapeMarkdown()`: escape `|`, backtick,
   `[`, `]`, `<`, `>`, `*`, `_`, `~` with backslashes; URLs go through
   `sanitize()` and are emitted only if they parse as `https://` URLs
   (otherwise render as plain escaped text — no clickable non-https links).
3. Deterministic output: LF, trailing newline, stable ordering identical to
   the terminal renderer's grouping.
4. No HTML other than `<details>/<summary>` (GFM-safe subset).

## Acceptance Criteria

- [ ] Golden markdown for the issue-32 golden reports (npm full, PyPI with
      skips) — rendered result manually eyeballed in GitHub preview once
      and screenshot attached to the PR.
- [ ] Injection tests: detail strings containing `| pipes |`,
      `[link](javascript:alert(1))`, backticks, `</details>` ⇒ output
      contains them only escaped; the javascript: URL is not emitted as a
      link.
- [ ] Table rows never break on embedded newlines (sanitize collapses
      them).
- [ ] Verdict emoji/glyph mapping pinned.

## Validation

- `npm test -- render-markdown`.

## Dependencies

- 32, 33 (sanitize).

## Non-goals

- No verify/list markdown output (v1 renders those as terminal/JSON only),
  no HTML report, no badge generation.

## Design References

- DESIGN.md §11.3; ADR-007 S4
