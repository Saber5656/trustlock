# Title

Untrusted-string sanitizer & terminal renderer

## Summary

Implement `src/core/report/sanitize.ts` (the single control-character/ANSI
stripping function for all untrusted strings) and
`src/core/report/render-terminal.ts` (the human-facing check report).

## Context

Package descriptions, deprecation messages, maintainer names, and advisory
summaries are attacker-controlled and can smuggle ANSI escape sequences
into terminals (S4, DESIGN §16.6). The sanitizer is a security control with
its own tests; the renderer is the product's face.

## Scope

- Two modules + unit tests (including ANSI-injection fixtures).

All escape sequences below are written with `\xNN` notation; source files
must use the corresponding escape literals, never raw control bytes.

## Detailed Requirements

1. `sanitize.ts` — `sanitize(s: string, opts?: { maxLength?: number }): string`
   applies, in order:
   1. remove OSC sequences: `/\x1b\][\s\S]*?(?:\x07|\x1b\\)/g` (ESC `]` …
      terminated by BEL or ST);
   2. remove CSI sequences: `/(?:\x1b\[|\x9b)[0-?]*[ -\/]*[@-~]/g`;
   3. remove any remaining ESC (`\x1b`) and C1 range (`\x80`–`\x9f`);
   4. replace `\n`, `\r`, `\t` with a single space each (single-line
      contexts), collapsing runs of spaces to one;
   5. remove remaining C0 controls (`\x00`–`\x1f`) and DEL (`\x7f`);
   6. cap at `maxLength ?? 1000` chars, appending `…` when truncated.

   Normative vectors:
   - `"\x1b[31mred\x1b[0m"` → `"red"`
   - `"a\x07b"` → `"ab"`
   - `"line1\nline2"` → `"line1 line2"`
   - `"\x1b]0;evil-title\x07text"` → `"text"`
   - `"safe"` → `"safe"` (idempotent on clean input)
2. `render-terminal.ts`:
   `renderTerminal(report, opts: { color: boolean; quiet?: boolean; asciiGlyphs?: boolean }): string`
   - Layout (top to bottom):
     1. header: `<name>@<version> (<ecosystem displayName>)` + verdict badge
        `PASS` (green) / `WARN` (yellow) / `FAIL` (red) via
        `util.styleText`; `report.incomplete` appends ` (incomplete)` in
        yellow;
     2. subject line: registry URL; when `requestedVersion === null` add
        `resolved: latest → <resolvedVersion>`;
     3. findings grouped by signal category in fixed order:
        vulnerabilities, name, execution, provenance, maintainers,
        repository, metadata, popularity, license, footprint;
        glyph per outcome/severity — `✗` triggered critical, `!` triggered
        warn, `·` triggered notice/info, `?` not-evaluable, `✓` pass —
        then finding title, detail sentence, and (dimmed) first evidence
        URL; continuation lines indented 4 spaces, no hard wrapping;
     4. footer: counts line
        (`22 checks: 17 passed, 2 warnings, 1 notice, 2 unavailable`),
        a `--fail-on` note when policy escalated the verdict, and an
        offline/cache note when applicable (flag passed via opts, not read
        from env).
   - `opts.asciiGlyphs: true` swaps glyphs for `x ! . ? +` (the CLI layer
     decides based on platform; the renderer never reads
     `process.platform` — it stays pure).
   - `opts.quiet: true` ⇒ header + triggered findings + footer only.
   - **Every dynamic string** (names, versions, titles, details, URLs,
     evidence) flows through `sanitize()` via a single internal `line()`
     helper; no raw interpolation of report data is allowed in the module.
   - `color: false` ⇒ output contains zero `\x1b` bytes.

## Acceptance Criteria

- [ ] Sanitizer vector table (≥ 12 cases): the five normative vectors plus
      CSI-without-ESC (`\x9b`), lone ESC, C1 chars, DEL, NUL, 1001-char
      truncation with `…`, empty string.
- [ ] Golden terminal outputs (color, no-color, quiet, asciiGlyphs) for the
      issue-32 golden reports — byte-stable snapshots.
- [ ] Hostile-report test: every string field of a synthetic report carries
      `"\x1b[2J\x1b]0;pwn\x07"` payloads; with `color: false` the output has
      no `\x1b`/`\x9b`/`\x07` bytes; with `color: true` the only escape
      bytes present are those emitted by `util.styleText` (assert by
      running the same render with color:false and comparing
      styleText-stripped outputs for equality).
- [ ] Category grouping order pinned by a golden test.
- [ ] Renderer purity: no `process.*` reads (grep test).

## Validation

- `npm test -- sanitize render-terminal`.

## Dependencies

- 32 (Report model), 03 (Node baseline provides `util.styleText`).

## Non-goals

- No paging, no interactive elements, no table layout engine, no markdown
  (issue 34), no verify-output rendering (issue 37 renders its own table
  reusing `sanitize`).

## Design References

- DESIGN.md §11.2, §16.6; ADR-007 S4
