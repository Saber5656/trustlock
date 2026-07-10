# Title

Project scaffolding & toolchain

## Summary

Create the TypeScript/Node project skeleton for vetlock: package manifest,
strict ESM TypeScript config, vitest, ESLint (flat config) + Prettier,
directory layout, npm scripts, LICENSE, and baseline repo files. After this
issue, `npm run lint`, `npm run typecheck`, `npm run test`, and `npm run
build` all succeed on a trivial placeholder module.

## Context

vetlock is a security-sensitive CLI whose own supply chain must be exemplary
(ADR-002, ADR-007 S8). The runtime dependency set is fixed and small; the
toolchain must enforce strictness from the first commit. All later issues
assume this layout and these scripts.

## Scope

- `package.json`, `tsconfig.json`, `eslint.config.js`, `.prettierrc.json`,
  `.prettierignore`, `vitest.config.ts`, `.gitignore`, `.editorconfig`,
  `.nvmrc`, `LICENSE`.
- Directory skeleton: `src/cli/`, `src/core/`, `src/infra/`, `data/`,
  `scripts/`, `test/`.
- One placeholder module + test to prove the pipeline
  (`src/infra/version.ts` exporting the package version read at build time,
  with a unit test).

## Detailed Requirements

1. `package.json`:
   - `"name": "vetlock"`, `"version": "0.1.0-dev"`, `"private": false`,
     `"type": "module"`, `"license": "MIT"`,
     `"engines": { "node": ">=22.12.0" }`.
   - `"bin": { "vetlock": "dist/cli/index.js" }` (target created in issue 03;
     the path may not exist yet — that is acceptable for now).
   - `"files": ["dist", "data", "README.md", "LICENSE"]`.
   - `"exports": { "./package.json": "./package.json" }` (deep-import
     lockdown from day one; only `bin` is public API — DESIGN §19).
   - `description`: `Vet a package before adding it: evidence-based trust
     checks plus a committable approval ledger.`
   - `repository`, `bugs`, `homepage` pointing at
     `https://github.com/Saber5656/trustlock` (repo will be renamed to
     `vetlock` later; GitHub redirects — do not guess the future URL,
     use the current one).
   - Runtime dependencies (exact set, latest stable majors):
     `commander`, `zod`, `semver`, `smol-toml`. Nothing else.
   - Dev dependencies: `typescript`, `vitest`, `@vitest/coverage-v8`,
     `eslint`, `@eslint/js`, `typescript-eslint`, `prettier`, `tsx`,
     `@types/node`, `@types/semver`.
   - Scripts:
     - `"build": "tsc -p tsconfig.json"`
     - `"typecheck": "tsc -p tsconfig.json --noEmit"`
     - `"lint": "eslint . && prettier --check ."`
     - `"format": "prettier --write ."`
     - `"test": "vitest run"`
     - `"test:coverage": "vitest run --coverage"`
2. `tsconfig.json`: `"strict": true`, `"module": "NodeNext"`,
   `"moduleResolution": "NodeNext"`, `"target": "ES2023"`,
   `"outDir": "dist"`, `"rootDir": "src"`,
   `"declaration": false`, `"sourceMap": true`,
   `"noUncheckedIndexedAccess": true`, `"exactOptionalPropertyTypes": true`,
   `"forbidden unused"`: enable `noUnusedLocals` and `noUnusedParameters`.
   `include: ["src"]` (tests are type-checked by vitest, excluded from build).
3. ESLint flat config: `@eslint/js` recommended + `typescript-eslint`
   recommendedTypeChecked for `src/**`; enforce S1 for `src/**` with
   `src/infra/git.ts` as the sole exception (the file itself is created in
   issue 36; configure the exception now). The ban must cover ALL access
   paths: `no-restricted-imports` for `child_process`/`node:child_process`
   (static imports), plus `no-restricted-syntax` selectors for dynamic
   `import("child_process"|"node:child_process")`,
   `require("child_process"|"node:child_process")`, and
   `createRequire`-based access (ban `module.createRequire` usage in
   `src/**` entirely — nothing in shipped code needs it except a possibly
   documented package.json read in `version.ts`, which must use
   `fs.readFileSync` instead). `test/**` and `scripts/**` are exempt from
   this rule (their child-process use is constrained by DESIGN §16.3:
   argv-array `execFile`, no shell).
4. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `*.tsbuildinfo`,
   `.DS_Store`.
5. `.nvmrc`: `22.12.0`. `.editorconfig`: 2-space indent, LF, UTF-8, final
   newline.
6. `LICENSE`: MIT, copyright holder `vetlock contributors`, year 2026.
7. Commit `package-lock.json` (run `npm install`).
8. Placeholder `src/infra/version.ts`: export `TOOL_NAME = "vetlock"` and
   `TOOL_VERSION` resolved at module load by reading the package's own
   `package.json` via
   `JSON.parse(fs.readFileSync(new URL("../../package.json", import.meta.url), "utf8")).version`
   wrapped in try/catch with fallback `"0.0.0-dev"` (works for `npx`,
   global installs, and local dev; `process.env.npm_package_version` is NOT
   reliable outside npm scripts and must not be used). Unit test asserts
   `TOOL_NAME === "vetlock"` and `TOOL_VERSION` matches
   `/^\d+\.\d+\.\d+/ or the fallback`.

## Acceptance Criteria

- [ ] `npm install` succeeds on Node 22.12+ with a committed lockfile.
- [ ] `npm run lint`, `npm run typecheck`, `npm run test`, `npm run build`
      all exit 0.
- [ ] `node --input-type=module -e "import('./dist/infra/version.js').then(m => console.log(m.TOOL_NAME))"`
      prints `vetlock` after build.
- [ ] Runtime `dependencies` in package.json are exactly
      `commander`, `zod`, `semver`, `smol-toml`.
- [ ] ESLint fails `src/` snippets using each banned path: static import,
      dynamic `import()`, `require()`, and `createRequire` (four cases in
      the lint-guards test); the same snippets pass under `test/**`.
- [ ] No install scripts (`preinstall`/`install`/`postinstall`) exist in
      package.json.

## Validation

- Run all four scripts locally and paste output in the PR.
- Add `test/lint-guards.test.ts`: programmatically run ESLint (via
  `eslint` Node API) against an in-memory snippet importing
  `node:child_process` and assert it errors (guards S1 forever).

## Dependencies

None (first issue).

## Non-goals

- No CLI code, no HTTP code, no CI workflow (issue 02), no README rewrite
  (issue 41), no release fields tuning beyond the listed ones (issue 43).

## Design References

- DESIGN.md §4.1 (stack & dependency budget), §4.2 (layout), §19 (packaging)
- ADR-002 (runtime & license), ADR-007 (S1, S8)
