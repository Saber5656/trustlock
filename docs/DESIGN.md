# vetlock — v1 Design

Status: draft for review · 2026-07
Product name: **vetlock** (this repository will be renamed from `trustlock` after this design PR merges — see [ADR-005](decisions/ADR-005-rename-to-vetlock.md))

> vetlock is a CLI that helps developers **vet a package before adding it as a
> dependency** — it collects deterministic, citable trust signals from public
> sources, evaluates them against explicit rules, and records the resulting
> human decision in a repo-committed **approval ledger** that CI can enforce.

This document is the canonical v1 design. Implementation issues in
[docs/issues/](issues/) are derived from it and must not contradict it. When a
conflict is discovered, this file is updated first.

---

## 1. Product definition

### 1.1 The problem

Adding a dependency is a trust decision made in seconds with almost no
evidence. Existing tools scan for known CVEs *after* the dependency is in the
tree, or run proprietary behavioral analysis in a SaaS. Nothing OSS:

1. answers "should I add this package?" **before** it enters the tree, with
   verifiable evidence, and
2. **remembers the decision** — who approved which exact version, when, why —
   in an artifact the repo owns and CI can enforce.

See [research/competitive-landscape.md](research/competitive-landscape.md).

### 1.2 The product loop

```
vetlock check <pkg>      → evidence report + verdict (pass / warn / fail)
vetlock approve <pkg@v>  → decision recorded in vetlock.json (the ledger)
vetlock verify           → every direct dependency in the project manifests
                           must have an approved exact version in the ledger
                           (CI gate: exit 1 on violations)
```

### 1.3 Design principles

| # | Principle | Consequence |
|---|-----------|-------------|
| P1 | **Deterministic & citable.** Every finding traces to a public fact with a URL. No LLM, no proprietary scoring. | reproducible reports; reviewers can independently verify each claim |
| P2 | **Never execute the target.** vetlock never installs, builds, or runs the package under review, and never invokes package managers. | safe to run on hostile input by construction |
| P3 | **Zero-config default.** No API key required for any primary signal. | `npx vetlock check foo` works immediately |
| P4 | **Fail-open execution, fail-closed reporting.** A data source being down never crashes a run, but the report must say what could not be evaluated. | `incomplete` flag; `unavailable` signal status |
| P5 | **The ledger is a boring, diffable artifact.** JSON, stable ordering, atomic writes, human-readable in PRs. | reviews of the ledger happen in normal code review |
| P6 | **Multi-ecosystem first.** npm and PyPI ship in v1, but every boundary (spec parsing, registry access, manifest reading, version ordering) goes through the ecosystem adapter interface. | adding cargo/Go later touches no core module |
| P7 | **Practice what we preach.** vetlock itself has minimal, justified dependencies, a committed lockfile, pinned CI, and provenance-enabled publishing. | §19, §20 |

---

## 2. Users & core workflows

### 2.1 Personas

- **P-Solo** — individual developer deciding whether to add a package.
- **P-Team** — team codifying "every direct dependency gets a conscious
  review"; ledger lives in the repo; `verify` runs in CI.
- **P-Agent** — AI coding agents instructed (via project rules) to run
  `vetlock check` before adding dependencies and to stop on `fail`. v1 serves
  agents through the CLI + `--json` only (MCP server is v2).

### 2.2 Workflow: vet before adding (P-Solo, P-Agent)

```
$ vetlock check express
… report: 14 checks, 12 pass, 2 notice … verdict: PASS
$ vetlock approve express@5.1.0 --reason "web framework, well maintained"
$ npm install express@5.1.0
```

### 2.3 Workflow: CI enforcement (P-Team)

```
$ vetlock verify            # local: all direct deps approved?
# CI step (any provider):
$ npx vetlock verify        # exit 1 → red build listing violations
```

A PR that adds/updates a direct dependency without a matching ledger entry
fails CI; the fix is to run `check`, get the review evidence, then `approve`
(committing the ledger change in the same PR — which is exactly the reviewable
artifact we want).

---

## 3. Scope

### 3.1 v1 goals

1. `check` / `approve` / `reject` / `verify` / `list` commands (§5).
2. Ecosystems: **npm** and **PyPI** behind the adapter interface (§7).
3. Manifests: `package.json` + `package-lock.json` (v2/v3);
   `pyproject.toml` (PEP 621) + `uv.lock` (§8).
4. Signal catalog of 22 deterministic signals (§9) with rule-based evaluation
   (§10), terminal / JSON / Markdown reports (§11).
5. Approval ledger `vetlock.json`: direct dependencies, exact versions (§12).
6. Offline-capable cache, `--offline` mode (§14).
7. Security model implemented as code-level requirements (§16).
8. Publishable npm package with provenance (§19) — actual publish is a human
   release gate.

### 3.2 v1 non-goals (explicit)

- No transitive-dependency ledger enforcement (transitive footprint is
  *reported* as a signal only).
- No semver-range approvals — exact versions only.
- No parsers for `yarn.lock`, `pnpm-lock.yaml`, `poetry.lock`,
  `requirements.txt`, `pylock.toml` (PEP 751), npm workspaces sub-packages.
- No GitHub Action, no PR commenting, no MCP server, no watch/daemon mode.
- No private registry support; public registries only.
- No source-code / behavioral malware analysis (GuardDog/Socket territory).
- No numeric aggregate "trust score" — verdict + itemized findings only.
- No telemetry of any kind.

### 3.3 v2 deferred (recorded, not designed here)

GitHub Action + PR comments · MCP server · more ecosystems (Cargo, Go,
RubyGems, Maven) · more manifest formats (incl. PEP 751, workspaces) ·
range-based approval policies · shared org-level ledgers · SARIF output ·
PyPI file-integrity pinning in ledger · full local Sigstore bundle
verification · `vetlock init` interactive setup · dependents-count signal.

---

## 4. Architecture overview

### 4.1 Runtime & stack

- Language: **TypeScript**, strict mode, ESM only (`"type": "module"`).
- Runtime: **Node.js ≥ 22.12.0** (global `fetch`, `util.styleText`,
  `util.parseArgs` availability; Node 20 is EOL as of April 2026).
- Runtime dependencies (complete list — adding one requires an ADR):

| Package | Why | Why not stdlib |
|---|---|---|
| `commander` | subcommand CLI framework | arg parsing with subcommands/help is error-prone to hand-roll |
| `zod` | runtime schema validation for ledger, reports, API responses | untrusted-input validation needs declarative schemas |
| `semver` | npm version parsing/ordering | correct semver is subtle |
| `smol-toml` | `pyproject.toml` / `uv.lock` parsing | no TOML in stdlib |

  Notable exclusions: no `chalk` (use `util.styleText`), no `axios`/`got`
  (use global `fetch`), no `fs-extra`.
- Dev dependencies: `typescript`, `vitest`, `eslint` (flat config) +
  `typescript-eslint`, `prettier`, `tsx`, and `npm-high-impact` (data source
  for the generated top-packages list only — never imported by `src/`).

### 4.2 Module layout

```
src/
  cli/
    index.ts            # entrypoint: program setup, global flags, error trap
    context.ts          # CliContext: resolved flags/env → typed runtime options
    cmd-check.ts
    cmd-approve.ts      # approve + reject (shared implementation)
    cmd-verify.ts
    cmd-list.ts
  core/
    spec.ts             # package spec grammar (§6)
    ecosystems/
      types.ts          # EcosystemAdapter interface (§7)
      index.ts          # adapter registry + ecosystem detection
      npm/
        adapter.ts
        registry.ts     # packument, downloads, attestations clients
        manifest.ts     # package.json + package-lock.json reader
        name.ts         # npm name validation
      pypi/
        adapter.ts
        registry.ts     # JSON API, integrity API, pypistats clients
        manifest.ts     # pyproject.toml + uv.lock reader
        name.ts         # PEP 503 normalization + validation
    signals/
      types.ts          # Signal, Collector, SignalContext (§9)
      orchestrator.ts   # parallel collection, timeouts, ordering
      collectors/
        metadata.ts  maintainers.ts  repository.ts  provenance.ts
        execution.ts  vulnerabilities.ts  popularity.ts  typosquat.ts
        license.ts  footprint.ts
    rules/
      types.ts          # Rule, Finding, Severity, Verdict (§10)
      engine.ts
      default-rules.ts
    report/
      model.ts          # Report type + schemaVersion (§11)
      digest.ts         # canonical JSON + sha256 report digest
      render-terminal.ts
      render-json.ts
      render-markdown.ts
      sanitize.ts       # control-char / ANSI stripping for untrusted strings
    ledger/
      schema.ts         # zod schema, LedgerFile type (§12)
      io.ts             # load / atomic save / stable ordering
      ops.ts            # upsertDecision, query
    verify/
      engine.ts         # dependency × ledger state machine (§13)
  infra/
    http.ts             # fetch wrapper: allowlist, timeout, retry, size cap
    cache.ts            # file cache with TTL + ETag (§14)
    depsdev.ts          # deps.dev v3 client
    osv.ts              # OSV client
    github.ts           # GitHub REST client (optional token)
    paths.ts            # cache-dir resolution (macOS/Linux/Windows)
    errors.ts           # typed error hierarchy + exit-code mapping (§15)
    log.ts              # stderr logger, --verbose/--quiet
data/
  top-packages/
    npm.json            # generated top-package name list (committed)
    pypi.json
scripts/
  update-top-packages.ts  # maintenance: regenerate data/top-packages/*
  smoke-live.ts           # manual pre-release live-API smoke test
```

Dependency direction: `cli → core → infra`. `core` never reads `process.argv`
or env directly; everything arrives via injected context (testability).

### 4.3 Data flow (`check`)

```
spec string ─▶ core/spec ─▶ adapter.resolveVersion ─▶ adapter.fetchPackageFacts
                                   │                        │ (registry of record)
                                   ▼                        ▼
                            signals/orchestrator ◀── collectors (+ depsdev/osv/github/downloads)
                                   │  Signal[]
                                   ▼
                              rules/engine ─▶ findings + verdict
                                   │
                                   ▼
                              report/model ─▶ renderer (terminal | json | markdown)
```

---

## 5. CLI contract

Binary name: `vetlock` (npm package `vetlock`, `"bin": {"vetlock": "dist/cli/index.js"}`).

### 5.1 Commands

| Command | Purpose | Network |
|---|---|---|
| `vetlock check <spec>` | collect signals, evaluate rules, render report | yes (cached) |
| `vetlock approve <spec>` | record `approved` decision for an exact version | yes (version resolution + integrity fetch only) |
| `vetlock reject <spec>` | record `rejected` decision | same as approve |
| `vetlock verify [dir]` | compare project direct deps against ledger | **no network** |
| `vetlock list` | print ledger contents | no network |

### 5.2 Global options

| Flag | Env fallback | Default | Meaning |
|---|---|---|---|
| `--json` | — | off | machine-readable output on stdout (shorthand for `--format json`) |
| `--format <fmt>` | — | `terminal` | `terminal` \| `json` \| `markdown` (check only; verify/list support `terminal`/`json`) |
| `--ledger <path>` | — | `./vetlock.json` (nearest ancestor: see §12.4) | ledger file location |
| `--offline` | `VETLOCK_OFFLINE=1` (exactly the string `1`) | off | serve all HTTP from cache; missing cache ⇒ signal `unavailable` |
| `--cache-dir <path>` | `VETLOCK_CACHE_DIR` | OS default (§14.2) | cache location |
| `--no-color` | `NO_COLOR` (any value) | auto (TTY detect) | disable ANSI styling |
| `--verbose` | — | off | debug logging to stderr |
| `--quiet` | — | off | suppress non-essential stderr |
| `--version`, `--help` | — | — | standard |

Secrets: `VETLOCK_GITHUB_TOKEN` (preferred) or `GITHUB_TOKEN` read from env
only; never a flag, never persisted, never logged (§16.4).

### 5.3 Command-specific options

- `check`: `--ecosystem <npm|pypi>`, `--fail-on <critical|warn>` (default
  `critical`).
- `approve` / `reject`: `--ecosystem`, `--reason <text>`,
  `--by <identity>` (default: `user.name <user.email>` from git config;
  error with guidance if unavailable and `--by` missing),
  `--report <path>` (a `check --json` output for the same subject; its
  canonical digest is stored in the ledger entry — §12.3).
- `verify`: `--ecosystem` (restrict), `--prod-only` (keep only
  `group: "prod"` dependencies, uniformly for every ecosystem).
- `list`: `--decision <approved|rejected>`, `--ecosystem`.

### 5.4 Exit codes (uniform across commands)

| Code | Meaning |
|---|---|
| `0` | success — check verdict `pass`/`warn` (without `--fail-on warn`); verify with zero violations |
| `1` | policy outcome — check verdict `fail` (`--fail-on warn` escalates a would-be `warn` verdict to `fail` *inside* the engine, so exit codes always map 1:1 from the verdict); verify with ≥1 violation |
| `2` | execution error — invalid input, network totally unavailable (and not `--offline`-satisfiable), unreadable/corrupt files, internal error |

Stdout carries the report/payload only; all diagnostics go to stderr. `--json`
guarantees stdout is exactly one JSON document.

---

## 6. Package spec grammar & name validation

### 6.1 Grammar

```
spec       := [ ecosystem ":" ] body
ecosystem  := "npm" | "pypi"
body(npm)  := name [ "@" version ]            # scoped: "@scope/name[@version]"
body(pypi) := name [ ("==" | "@") version ]   # "==" is the native form, "@" accepted
version    := exact version string (validated by the adapter; ranges rejected)
```

Examples: `express`, `express@5.1.0`, `npm:@types/node@24.0.1`,
`pypi:requests==2.32.4`, `requests@2.32.4` (with `--ecosystem pypi`).

### 6.2 Ecosystem resolution order

1. explicit `ecosystem:` prefix;
2. `--ecosystem` flag;
3. project detection in cwd — if **exactly one** of {npm, pypi} manifests is
   present, use it;
4. otherwise: exit 2 with message listing the two explicit options.

### 6.3 Name validation (security boundary)

Names are untrusted (CLI args **and** manifest contents). Validation happens
before any URL interpolation or cache-key derivation.

- npm: max 214 chars; lowercase; each part (scope, name) must start with
  `[a-z0-9]` — i.e. names starting with `.`, `_`, or `-` are rejected
  (deliberate v1 strictness: slightly narrower than npm's legacy grammar;
  packages with such names cannot be checked in v1 and this is documented) —
  remaining chars `[a-z0-9-._~]`; scoped form `@scope/name` with both parts
  validated; no URL-meaningful characters otherwise. Scoped names are
  URL-encoded (`@scope%2Fname`) exactly once at the HTTP layer.
- PyPI: `^[A-Za-z0-9]([A-Za-z0-9._-]*[A-Za-z0-9])?$`; normalized per PEP 503
  (lowercase; runs of `-_.` → `-`) before any lookup or ledger write.
- Version strings: npm — valid exact semver (via `semver`); PyPI — PEP 440
  syntactic check (regex-level; full PEP 440 ordering is not required in v1,
  see §7.2).

Violations ⇒ exit 2 with the offending string shown escaped (§16.6).

---

## 7. Ecosystem adapter contract

### 7.1 Interface (normative)

```ts
type EcosystemId = "npm" | "pypi";

interface EcosystemAdapter {
  readonly id: EcosystemId;
  readonly displayName: string;          // "npm", "PyPI"
  readonly depsDevSystem: string;        // "NPM" | "PYPI"
  readonly osvEcosystem: string;         // "npm" | "PyPI"

  validateName(raw: string): { ok: true; normalized: string } | { ok: false; reason: string };
  validateExactVersion(raw: string): { ok: true; version: string } | { ok: false; reason: string };
  parseSpecBody(body: string): { name: string; version?: string };   // §6.1 body rules
  compareVersions(a: string, b: string): -1 | 0 | 1;                 // ecosystem ordering
  readonly notApplicableSignals: readonly string[]; // catalog ids that are structurally
                                                    // impossible for this ecosystem; the
                                                    // orchestrator pre-marks them "skipped"
  readonly lockfileGuidance: string;                // one-line remediation shown by verify
                                                    // for lock_missing (e.g. "run npm install
                                                    // (npm >= 7) to generate a v2+ lockfile")
  resolveVersion(name: string, requested: string | undefined, ctx: InfraContext):
    Promise<ResolvedVersion>;            // requested==null ⇒ latest stable
  fetchPackageFacts(name: string, version: string, ctx: InfraContext):
    Promise<PackageFacts>;               // registry-of-record facts (§7.3)
  detectProject(dir: string): Promise<boolean>;                      // manifest present?
  readDirectDependencies(dir: string): Promise<DirectDependency[]>;  // §8
}
```

Adapters own: name rules, version ordering, registry-of-record access,
manifest reading, and the declaration of which catalog signals do not apply
to their ecosystem. Cross-registry enrichment (deps.dev, OSV, GitHub,
download stats) lives in collectors, keyed by `depsDevSystem` /
`osvEcosystem` carried in the signal context — collectors themselves stay
ecosystem-agnostic. `types.ts` also exports `ecosystemIdSchema` (the zod
enum of registered ids) so other schemas (e.g. the ledger) never restate
ecosystem literals.

Adding an ecosystem = one new directory under `core/ecosystems/` + one
registration line + a top-packages data file. Nothing else changes; this is
the P6 guarantee, and it is enforced by an architectural test that imports
core modules and asserts they contain no ecosystem-id conditionals.

### 7.2 Version ordering pragmatics

- npm: full semver ordering via `semver`.
- PyPI v1: exact-match semantics only. `compareVersions` implements PEP 440
  best-effort ordering for *display* (nearest-approved hints); correctness of
  verify never depends on ordering, only on string equality of normalized
  exact versions. Full PEP 440 ordering is a known unknown (§21).

### 7.3 PackageFacts (normalized registry facts)

```ts
interface Fact<T> { value: T; sourceUrl: string }   // every fact cites its source

interface PackageFacts {
  ecosystem: EcosystemId;
  name: string;                       // normalized
  version: string;                    // exact
  publishedAt?: Fact<string>;         // ISO 8601, this version
  firstPublishedAt?: Fact<string>;    // first release of the package
  latestVersion?: Fact<string>;
  releaseDates?: Fact<{ version: string; publishedAt: string }[]>;
  deprecated?: Fact<{ flag: boolean; message?: string }>;   // npm
  yanked?: Fact<{ flag: boolean; reason?: string }>;        // PyPI
  license?: Fact<string | null>;
  description?: Fact<string | null>;
  repositoryUrl?: Fact<string | null>;     // as declared (unverified)
  homepageUrl?: Fact<string | null>;
  maintainers?: Fact<{ count: number; names: string[] }>;   // npm only
  latestPublisher?: Fact<{ name: string; priorPublishCount: number }>; // npm
  installScripts?: Fact<{ present: boolean; names: string[] }>;        // npm
  binEntries?: Fact<string[]>;                                          // npm
  distribution?: Fact<{ integrity?: string; unpackedSize?: number; fileCount?: number }>; // npm
  pythonDistribution?: Fact<{ hasWheel: boolean; sdistOnly: boolean }>; // PyPI
  attestations?: Fact<{ present: boolean; kinds: string[]; publisherIdentity?: string }>;
}
```

Absent field = fact unavailable from the registry; collectors translate
absence into `unavailable` signals where the distinction matters.

---

## 8. Manifest / lockfile readers

`DirectDependency`:

```ts
interface DirectDependency {
  ecosystem: EcosystemId;
  name: string;               // normalized
  declaredRange: string;      // as written in the manifest
  group: "prod" | "dev" | "optional";
  resolvedVersion?: string;   // exact, from lockfile
  integrity?: string;         // npm: lockfile "integrity" (SRI string)
  lockPresent: boolean;       // false ⇒ verify status lock_missing
}
```

### 8.1 npm reader

- Direct deps: root `package.json` `dependencies` (`prod`) +
  `devDependencies` (`dev`). `optionalDependencies` → `optional`.
- Resolution: `package-lock.json` with `lockfileVersion` 2 or 3;
  entry `packages["node_modules/<name>"]` gives `version` + `integrity`.
  lockfileVersion 1 or missing lockfile ⇒ every dep `lockPresent: false`
  with actionable guidance ("run npm install to generate a v2+ lockfile").
- npm workspaces: v1 reads the **root** package only; presence of
  `workspaces` emits a warning naming the limitation (v2 item).

### 8.2 Python reader

- Direct deps: `pyproject.toml` — the `dependencies` array key inside the
  `[project]` table (`prod`) + the arrays under
  `[project.optional-dependencies]` (`optional`). Each PEP 508 string is
  reduced to a package name via a minimal, well-tested extractor (name = the
  leading token before any of `[`, `(`, `<`, `>`, `=`, `!`, `~`, `;`, space),
  then PEP 503-normalized. Extras and environment markers are ignored for
  identity.
- Resolution: `uv.lock` (TOML): top-level `version` field must be a known
  supported value (v1 supports `1`; unknown ⇒ all deps `lockPresent: false` +
  warning, never a crash). Match `[[package]]` entries by normalized name
  where `source` is a registry source; take `version`.
- `[dependency-groups]` (PEP 735): parsed as `dev` if present — best-effort;
  documented as such.

---

## 9. Signal framework & catalog

### 9.1 Model

```ts
type SignalStatus = "evaluated" | "unavailable" | "skipped";
type SignalCategory = "metadata" | "maintainers" | "repository" | "provenance"
                    | "execution" | "vulnerabilities" | "popularity" | "name"
                    | "license" | "footprint";

interface Evidence { summary: string; url?: string }

interface Signal {
  id: string;                       // "category.slug", e.g. "execution.install-scripts"
  category: SignalCategory;
  status: SignalStatus;
  value?: unknown;                  // JSON-serializable, shape fixed per signal id
  evidence: Evidence[];             // at least one when evaluated
  unavailableReason?: string;       // when status !== "evaluated"
}

interface Collector {
  id: string;
  produces: string[];               // signal ids
  collect(ctx: SignalContext): Promise<Signal[]>;
}
```

`SignalContext` (normative):

```ts
interface SignalContext {
  subject: {
    ecosystem: EcosystemId; name: string; version: string;
    osvEcosystem: string; depsDevSystem: string;   // adapter-provided ids
    registryPageUrl: string;                        // human registry page
  };
  facts: PackageFacts;
  infra: {                     // structural types declared in signals/types.ts;
    depsdev: DepsDevLike;      // concrete clients (issues 16-18) satisfy them,
    osv: OsvLike;              // so the framework has no client dependencies
    github: GitHubLike;
    downloads: DownloadsFacade;   // { fetch(): Promise<{period, count, evidenceUrl} | null> }
    offline: boolean; log: Logger;
  };
  topPackages: TopPackagesIndex;   // issue 27 shape incl. source attribution
  abortSignal: AbortSignal;        // cooperative cancellation on timeout
}
```

Orchestrator rules:

- signals in every adapter's `notApplicableSignals` are pre-marked
  `skipped(not-applicable)` and their collectors are not asked for them —
  collectors never contain ecosystem conditionals;
- collectors run in parallel, concurrency 4, per-collector timeout 10 s;
- a collector throwing/timeout ⇒ its declared signals become `unavailable`
  with a deterministic reason (`"timeout"`, the `VetlockError` class name,
  or `"unknown-error"` for non-Error throws) — never crashes the run (P4);
- output signals sorted by id (deterministic reports);
- every declared signal id must appear exactly once in the output
  (orchestrator fills gaps with `unavailable`).

### 9.2 v1 signal catalog (normative)

| Signal id | npm | PyPI | Source | Value shape (summary) |
|---|---|---|---|---|
| `metadata.package-age` | ✅ | ✅ | registry | `{ firstPublishedAt, ageDays }` |
| `metadata.version-age` | ✅ | ✅ | registry | `{ publishedAt, ageDays }` |
| `metadata.release-cadence` | ✅ | ✅ | registry | `{ releasesLast12mo, gapBeforeLatestDays, latestReleaseAgeDays }` |
| `metadata.latest-drift` | ✅ | ✅ | registry | `{ latestVersion, isLatest, behindCount }` |
| `metadata.deprecated` | ✅ | — | registry | `{ deprecated, message? }` |
| `metadata.yanked` | — | ✅ | registry | `{ yanked, reason? }` |
| `maintainers.count` | ✅ | ⚠ unavailable | registry | `{ count, names }` |
| `maintainers.publisher-change` | ✅ | ⚠ unavailable | registry | `{ latestPublisher, priorPublishCount }` |
| `repository.declared` | ✅ | ✅ | registry | `{ url \| null }` |
| `repository.status` | ✅ | ✅ | GitHub API (token-optional) / deps.dev fallback | `{ exists, archived?, lastPushAt?, lastPushAgeDays?, stars?, openIssues?, forks? }` |
| `repository.scorecard` | ✅ | ✅ | deps.dev | `{ score, date }` (OpenSSF Scorecard) |
| `provenance.attestation` | ✅ | ✅ | registry attestation endpoints | `{ present, kinds, publisherIdentity? }` |
| `execution.install-scripts` | ✅ | — | registry | `{ present, names }` |
| `execution.sdist-only` | — | ✅ | registry | `{ sdistOnly, hasWheel }` |
| `execution.bin-entries` | ✅ | — | registry | `{ bins }` |
| `vulnerabilities.known` | ✅ | ✅ | OSV | `{ count, ids, maxSeverity, advisories, truncated }` (non-`MAL-`) |
| `vulnerabilities.malicious` | ✅ | ✅ | OSV | `{ count, ids, advisories, truncated }` (`MAL-` advisories) |
| `popularity.downloads` | ✅ | ✅ | api.npmjs.org / pypistats | `{ period, count }` |
| `name.typosquat` | ✅ | ✅ | bundled top-packages data | `{ suspect, isPopular, nearest?, distance? }` |
| `license.declared` | ✅ | ✅ | registry | `{ license \| null, spdxValid }` |
| `footprint.dependencies` | ✅ | ✅ | deps.dev | `{ directCount, transitiveCount }` |
| `footprint.install-size` | ✅ | — | registry | `{ unpackedSize, fileCount }` |

⚠ = structurally not applicable for that ecosystem: the adapter lists the
signal in `notApplicableSignals` and the orchestrator emits it as `skipped`
with reason `not-applicable`, so reports stay ecosystem-honest.
(npm: `metadata.yanked`, `execution.sdist-only`. PyPI: `metadata.deprecated`,
`maintainers.count`, `maintainers.publisher-change`,
`execution.install-scripts`, `execution.bin-entries`,
`footprint.install-size`.)

### 9.3 Typosquat check (deterministic, offline)

- Input data: `data/top-packages/{npm,pypi}.json` — generated lists of the
  top ~5,000 package names per ecosystem (see §20.3 maintenance script).
- Normalization: lowercase; strip separators (`-`, `_`, `.`); npm scope
  removed for comparison but reported.
- Flag when: name ∉ top list ∧ ∃ top name with Damerau-Levenshtein distance
  ≤ 1 (names ≤ 7 chars) or ≤ 2 (longer), or equal after separator stripping.
- Known-typosquat fixture pairs from ecosyste-ms/typosquatting-dataset are
  used as test vectors.

---

## 10. Rule engine & policy

### 10.1 Model

```ts
type Severity = "info" | "notice" | "warn" | "critical";
type Verdict = "pass" | "warn" | "fail";

interface Rule {
  id: string;                        // "R-EXEC-001"
  signalId: string;
  defaultSeverity: Severity;
  title: string;                     // imperative finding title
  evaluate(signal: Signal): RuleOutcome;   // "pass" | "triggered" | "not-evaluable"
  detail(signal: Signal): string;    // human sentence with concrete values
}

interface Finding {
  ruleId: string;
  severity: Severity;                // after policy overrides
  outcome: "pass" | "triggered" | "not-evaluable";
  title: string;
  detail: string;
  signalId: string;
  evidence: Evidence[];
}
```

- Engine is a pure function `(signals, policy) → { findings, verdict, incomplete }`.
- `incomplete` = any rule `not-evaluable` because its signal was `unavailable`.
- Verdict: any triggered `critical` ⇒ `fail`; else any triggered `warn` ⇒
  `warn`; else `pass`. (`skipped` signals never trigger.)
- Findings include **passed** rules (outcome `pass`) — the report shows what
  was checked, not only what went wrong.

### 10.2 Default ruleset (normative for v1)

| Rule id | Signal | Triggered when | Default severity |
|---|---|---|---|
| `R-MAL-001` | vulnerabilities.malicious | any `MAL-` advisory matches the version | critical |
| `R-TYPO-001` | name.typosquat | suspect = true | critical |
| `R-VULN-001` | vulnerabilities.known | ≥1 advisory affects this version | warn |
| `R-AGE-001` | metadata.package-age | first publish < 30 days ago | warn |
| `R-AGE-002` | metadata.version-age | version published < 7 days ago | notice |
| `R-DEP-001` | metadata.deprecated | flag = true | warn |
| `R-DEP-002` | metadata.yanked | flag = true | warn |
| `R-MNT-001` | maintainers.publisher-change | latest version is publisher's first publish to this package | notice |
| `R-MNT-002` | maintainers.count | count = 1 | info |
| `R-REPO-001` | repository.declared | no repository URL declared | notice |
| `R-REPO-002` | repository.status | declared repository does not exist (404) | warn |
| `R-REPO-003` | repository.status | repository archived | warn |
| `R-REPO-004` | repository.status | no push in ≥ 730 days | notice |
| `R-PROV-001` | provenance.attestation | no attestation/provenance | notice |
| `R-EXEC-001` | execution.install-scripts | any install-phase script present | warn |
| `R-EXEC-002` | execution.sdist-only | sdist-only release | notice |
| `R-EXEC-003` | execution.bin-entries | ≥1 bin entry | info |
| `R-CAD-001` | metadata.release-cadence | `gapBeforeLatestDays` ≥ 540 and `latestReleaseAgeDays` ≤ 14 | warn |
| `R-POP-001` | popularity.downloads | < 500 downloads (npm: last-week; PyPI: last-month) | notice |
| `R-LIC-001` | license.declared | missing or SPDX-invalid license | notice |
| `R-FOOT-001` | footprint.dependencies | transitive count > 100 | notice |
| `R-DRIFT-001` | metadata.latest-drift | requested version is not latest and ≥ 2 releases behind (`behindCount` is release-count-based, cross-ecosystem) | notice |

Thresholds above are constants in `default-rules.ts` — one table, one place.

### 10.3 Policy (ledger-embedded)

Policy lives in the ledger file (§12) under `policy`:

```json
{
  "failOn": "critical",
  "rules": {
    "R-EXEC-001": { "severity": "critical" },
    "R-PROV-001": { "enabled": false }
  }
}
```

- `failOn`: `"critical"` (default) | `"warn"` — same semantics as `--fail-on`
  (flag wins over file).
- Per-rule: `severity` remap and/or `enabled: false`. Unknown rule ids ⇒
  warning (not error) to keep older ledgers working with newer vetlock.

---

## 11. Report model & renderers

### 11.1 JSON schema (schemaVersion 1)

```jsonc
{
  "schemaVersion": 1,
  "tool": { "name": "vetlock", "version": "0.1.0" },
  "generatedAt": "2026-07-08T12:00:00Z",
  "subject": {
    "ecosystem": "npm",
    "name": "left-pad",
    "requestedVersion": null,          // null ⇒ latest was resolved
    "resolvedVersion": "1.3.0",
    "registryUrl": "https://www.npmjs.com/package/left-pad"
  },
  "verdict": "warn",
  "incomplete": false,
  "findings": [ /* Finding[], §10.1 engine order: triggered (severity desc,
                   ruleId asc), then not-evaluable (ruleId asc), then pass
                   (ruleId asc) — one ordering everywhere */ ],
  "signals":  [ /* Signal[], §9.1, sorted by id */ ],
  "policy": { "failOn": "critical", "overrides": { /* effective */ } },
  "durationMs": 3120
}
```

Stability guarantee: within `schemaVersion: 1`, existing fields never change
meaning or type; additions are allowed. Tests snapshot the schema with
volatile fields (`generatedAt`, `durationMs`, `tool.version`) normalized.

`report/digest.ts` defines the canonical digest: sha256 over the JSON report
serialized with sorted keys and volatile fields removed — used by
`approve --report` (§12.3).

### 11.2 Terminal renderer

- Layout: header (name, version, ecosystem, verdict badge) → findings grouped
  by category, `✗` critical / `!` warn / `·` notice / `✓` pass, each with its
  detail sentence and dimmed evidence URL → footer (counts, incomplete note,
  cache/offline note).
- Styling via `util.styleText` only; honors `--no-color`, `NO_COLOR`, and
  non-TTY stdout (auto-plain).
- **All registry-sourced strings pass through `report/sanitize.ts`** (§16.6).
- `--quiet`: verdict line + triggered findings only.

### 11.3 Markdown renderer

Same content as terminal, GitHub-flavored: verdict as bold line, findings as
tables per category, evidence as links. Target consumer: paste into PR
descriptions (and the v2 GitHub Action).

---

## 12. Ledger (`vetlock.json`)

### 12.1 File schema (version 1)

```jsonc
{
  "version": 1,
  "policy": { /* §10.3, optional */ },
  "approvals": [
    {
      "ecosystem": "npm",
      "name": "left-pad",                  // normalized
      "version": "1.3.0",                  // exact
      "decision": "approved",              // "approved" | "rejected"
      "reviewedAt": "2026-07-08T12:34:56Z",// ISO 8601 UTC
      "reviewedBy": "Yasushi Takagi <ty@example.com>",
      "reason": "small, zero deps, no install scripts",   // optional
      "integrity": "sha512-…",             // optional; npm dist.integrity at approval time
      "reportDigest": "sha256-…"           // optional; §11.1 digest
    }
  ]
}
```

- Validated with zod on load; **unknown keys are preserved** on rewrite
  (forward compatibility).
- Ordering: `approvals` sorted by (`ecosystem`, `name`,
  `compareVersions`) — deterministic, diff-friendly.
- Serialization: 2-space indent, LF, trailing newline, UTF-8.
- Writes are atomic: write `<ledger filename>.tmp` alongside the target
  file (i.e. `targetPath + ".tmp"`, also for explicit `--ledger` paths),
  fsync, rename over the original.
- One entry per (ecosystem, name, version): a new `approve`/`reject` for the
  same triple **replaces** the entry and prints the previous decision.

### 12.2 Corruption & conflict behavior

- Unparseable JSON / schema violation ⇒ exit 2 with the file path and a hint
  (`git checkout -- vetlock.json` or manual fix); vetlock never overwrites a
  corrupt ledger.
- Git merge conflicts: one-object-per-entry formatting keeps conflicts local
  to entries; docs include a conflict-resolution note (both sides usually
  keep both entries).

### 12.3 approve / reject semantics

1. Parse spec; version **required**? No — if omitted, resolve latest stable
   and print what was resolved (explicit version recommended in docs).
2. Confirm the version exists in the registry (exit 2 if not).
3. npm: fetch `dist.integrity` and store it.
4. `--report <path>`: read a `check --json` output for the same
   ecosystem/name/version (exit 2 on subject mismatch) and store its digest.
5. Upsert entry, atomic write, print one-line confirmation.

`reject` is identical with `decision: "rejected"`.

### 12.4 Ledger discovery

Commands resolve the ledger as: `--ledger` flag → nearest `vetlock.json`
walking up from cwd (stopping at the filesystem root or a `.git` boundary) →
for `approve`/`reject` when none found: create at cwd (with notice); for
`verify`/`list`: exit 2 "no ledger found, run vetlock approve first".

---

## 13. Verify engine

### 13.1 Inputs & outputs

`verify(dir, ledger, options) → VerifyResult` — **pure local computation**
(no network), consuming `readDirectDependencies()` from every detected
ecosystem (or the `--ecosystem` subset).

### 13.2 Per-dependency status (state machine)

Evaluated in this order; first match wins:

| # | Condition | Status | Violation? |
|---|---|---|---|
| 1 | dep not in lockfile (`lockPresent: false`) | `lock_missing` | yes |
| 2 | ledger entry (name, resolvedVersion) with `decision: rejected` | `rejected` | yes |
| 3 | ledger entry (name, resolvedVersion) approved, npm integrity present on both sides and **different** | `integrity_mismatch` | yes |
| 4 | ledger entry (name, resolvedVersion) approved | `ok` | no |
| 5 | ≥1 approved entry for name (other versions only) | `version_drift` | yes |
| 6 | no entry for name | `unapproved` | yes |

Additionally: ledger entries whose (ecosystem, name) no longer appears in any
manifest are listed as `stale` (informational, never a violation).

### 13.3 Output & exit code

- Terminal: table `dependency / resolved / status / approved-by hint`,
  violations first; summary line
  `12 direct dependencies: 9 ok, 2 unapproved, 1 version drift (approved: 4.17.20)`.
- Each violation includes the remediation command
  (`vetlock check lodash@4.17.21` → `vetlock approve …`).
- `--json`: `{ schemaVersion: 1, results: [...], stale: [...], summary: {...} }`.
- Exit 1 iff ≥1 violation; devDependencies included by default
  (`--prod-only` to exclude) — dev deps run on dev machines and CI, they are
  attack surface.

---

## 14. Infrastructure: HTTP, cache, clients

### 14.1 HTTP client (`infra/http.ts`)

- Wraps global `fetch`. **Host allowlist** (exact hostnames):
  `registry.npmjs.org`, `api.npmjs.org`, `pypi.org`, `pypistats.org`,
  `api.deps.dev`, `api.osv.dev`, `api.github.com`. Any other host ⇒ thrown
  `NetworkPolicyError` (this is a security control, §16.2, not a config).
- HTTPS only; redirects followed only within the allowlist (max 3).
- Timeout 10 s/request (AbortController); retry idempotent GETs ×2 on
  5xx/network error with jittered backoff (250/750 ms); no retry on 4xx.
- Response size cap 5 MiB (streamed count) ⇒ `ResponseTooLargeError`.
- `User-Agent: vetlock/<version> (+https://github.com/<owner>/vetlock)`.
- Auth header injected **only** for `api.github.com` when a token is present.

### 14.2 Cache (`infra/cache.ts`)

- Location: `--cache-dir` → `VETLOCK_CACHE_DIR` → platform default
  (macOS `~/Library/Caches/vetlock`, Linux `$XDG_CACHE_HOME/vetlock` or
  `~/.cache/vetlock`, Windows `%LOCALAPPDATA%\vetlock\Cache`), created `0700`.
- Entry = JSON file at `<cacheDir>/v1/<sha256(method + " " + url)>.json`
  (the `v1/` segment allows future format migration); value stores
  `{ url, fetchedAt, etag?, status, body }`.
- TTLs: registry metadata 1 h; downloads/pypistats 24 h; deps.dev 24 h; OSV
  1 h; GitHub 1 h. Fresh ⇒ serve; stale ⇒ revalidate with `If-None-Match`
  when ETag exists; 304 ⇒ refresh timestamp.
- `--offline`: cache hits (any age) served; misses ⇒ `OfflineMissError` →
  signals `unavailable(offline)`.
- Corrupt cache entries are deleted and treated as misses (never fatal).

### 14.3 External clients

Thin typed wrappers with zod-validated responses (invalid shape ⇒ treated as
source failure, not crash): `depsdev.ts` (GetVersion, GetProject,
GetDependencies), `osv.ts` (querybatch + hydrate ≤ 10 advisories), `github.ts`
(GET /repos/{owner}/{repo}; parses `owner/repo` out of common URL forms:
`git+https`, `git://`, `ssh`, shorthand).

---

## 15. Error handling & exit codes

Typed hierarchy in `infra/errors.ts`:

```
VetlockError (abstract: message, exitCode, hint?)
├─ UsageError            (2)  bad spec/flags/name validation
├─ ProjectError          (2)  missing/corrupt manifest, lockfile, ledger
├─ NetworkError          (2 only when fatal to the command's purpose;
│   │                     carries url and optional HTTP status)
│   ├─ NetworkPolicyError, ResponseTooLargeError, OfflineMissError
├─ RegistryError         (2)  package/version not found (check/approve)
└─ InternalError         (2)  bugs — message asks to file an issue
```

- Collectors convert `NetworkError`s into `unavailable` signals (P4);
  `check` only exits 2 when the **registry of record** is unreachable
  (no facts ⇒ no meaningful report).
- CLI top-level trap prints `error: <message>` (+ `hint:` when present) to
  stderr; `--verbose` adds stack; process exits with the mapped code. With
  `--json`, errors are also emitted on stdout as
  `{ "error": { "code", "message" } }` so machine consumers never parse
  half-rendered reports.

---

## 16. Security model

### 16.1 Threat model

Assets: developer workstation & CI (code execution), project dependency
integrity, ledger integrity, GitHub token confidentiality.

| Adversary | Vector vs vetlock | Mitigation |
|---|---|---|
| malicious package author | hostile metadata (names, descriptions, URLs, versions) consumed by vetlock | never execute target (P2); name validation (§6.3); output sanitization (§16.6); size caps; zod validation |
| typosquatter | user checks the squatted name and trusts a shiny report | `R-TYPO-001` critical; report always shows registry URL + first-publish date prominently |
| compromised maintainer | new hostile version of an approved package | exact-version ledger: new version ⇒ `version_drift` in CI; `R-CAD-001`, `R-MNT-001` at re-check |
| MITM / registry impersonation | tampered API responses | HTTPS only; fixed host allowlist; npm integrity recorded at approval and compared in verify |
| malicious repo contents (cloned project) | crafted manifest/lockfile/ledger names or paths triggering SSRF/path traversal/proto pollution | all manifest-sourced names re-validated (§6.3); cache keys hashed; zod `.strict()` where applicable; parse into null-prototype objects; JSON/TOML parse wrapped with size caps |
| hostile terminal output | ANSI escape / control chars smuggled via package description | `sanitize.ts` strips C0/C1 controls & escape sequences from every untrusted string in every renderer |

Out of scope for v1 (documented honestly in SECURITY.md): detecting malicious
*code* inside packages (vetlock is metadata-only); registry-side account
compromise beyond published signals; the ledger protects the *decision
record*, not the artifact store.

### 16.2 Network boundary

Exactly the seven hosts in §14.1, HTTPS, enforced centrally in `http.ts`.
Adding a host = code change + ADR update (ADR-006). No proxies in v1 other
than standard `HTTPS_PROXY` env honored by undici — documented behavior.

### 16.3 Execution boundary

vetlock spawns **no child processes** except `git config --get user.name/email`
(argv-array spawn, no shell) for `reviewedBy` defaulting. It never runs npm,
pip, uv, or any package content. An ESLint rule bans `child_process` imports
(static import, dynamic `import()`, `require`, and `createRequire` paths)
across `src/**` with `src/infra/git.ts` as the sole exception; CI enforces
it. Dev-only code outside `src/` (the e2e harness spawning the built CLI,
`scripts/smoke-live.ts`) may use `execFile` with argv arrays and no shell —
never inside the shipped `src/`/`dist/` tree.

### 16.4 Secrets

Only `VETLOCK_GITHUB_TOKEN` / `GITHUB_TOKEN`, env-only, held in memory,
attached only to `api.github.com` requests, never cached (GitHub cache
entries store the *response*, keyed by URL — tokens never enter keys or
values), never logged (`log.ts` redacts `Authorization`), never in reports.

### 16.5 Filesystem boundary

Writes limited to: cache dir (0700), ledger file (atomic, §12.1), and
stdout/stderr. Ledger path traversal: `--ledger` is used as given (user
intent), but discovery (§12.4) never crosses a `.git` boundary upward.

### 16.6 Untrusted-string rendering

`sanitize()` (single shared implementation): strip C0 controls except
`\n`/`\t` (which become spaces in single-line contexts), strip C1, strip
`ESC`-initiated sequences, cap rendered string length (1,000 chars, `…`).
Applied by the **human-facing renderers** (terminal, markdown) to every
string originating from registries, manifests, or the ledger. The JSON
renderers deliberately do NOT sanitize values: JSON string escaping already
neutralizes control bytes in the output stream, and machine consumers need
faithful data — but they must sanitize before displaying to humans (stated
in the docs). Unit tests include ANSI-injection fixtures.

### 16.7 vetlock's own supply chain

Runtime deps fixed at 4 (§4.1) — adding one requires an ADR. Committed
lockfile; Dependabot; CI actions pinned by commit SHA with minimal
`permissions:`; `npm publish` with `--provenance` from a manual-dispatch
release workflow; `files` allowlist in package.json; no install scripts in
vetlock itself; 2FA on the npm account (human duty, documented in
RELEASING.md).

---

## 17. Testing strategy

| Layer | Approach |
|---|---|
| unit | vitest per module; table-driven for spec grammar, name validation, version ordering, rules (every rule: trigger + pass + not-evaluable cases) |
| HTTP fixtures | recorded JSON responses committed under `test/fixtures/http/`; `http.ts` accepts an injected `fetch` — **no live network in CI, ever** |
| integration | collectors + orchestrator against a local fixture server (undici MockAgent); golden Report JSON snapshots (volatile fields normalized) |
| e2e CLI | built CLI executed (`node dist/cli/index.js`) in temp dirs against fixture projects (`test/fixtures/projects/npm-basic`, `pypi-uv-basic`, `mixed`, `npm-workspaces`, `corrupt-*`); asserts stdout, stderr, exit codes; covers check→approve→verify happy path & every verify status |
| security tests | ANSI-injection fixtures; hostile-name manifests (path traversal, scope tricks, prototype-pollution keys); oversized-response fixture; allowlist bypass attempt |
| live smoke | `scripts/smoke-live.ts` — manual, pre-release only; hits real APIs for `express` + `requests` and prints a checklist |
| CI matrix | ubuntu + macos + windows × Node 22 + 24; lint, typecheck, unit+integration+e2e, `npm pack` audit |

Coverage gate: ≥ 90 % lines on `core/` (excluding renderers' color paths).

---

## 18. Performance budgets

| Operation | Budget |
|---|---|
| `check` (cold cache, typical package) | ≤ 15 s wall; all sources fetched concurrently |
| `check` (warm cache) | ≤ 3 s |
| `verify` (20 direct deps) | ≤ 1 s (no network by design) |
| `list` | ≤ 200 ms |

Budgets are asserted loosely in e2e tests (generous CI multipliers) to catch
pathological regressions (e.g., accidental serial fetching).

---

## 19. Packaging, distribution & release

- npm package `vetlock`; `bin: vetlock`; ESM; `engines.node: ">=22.12.0"`;
  `files: ["dist", "data", "README.md", "LICENSE"]`; no install scripts;
  `exports` limited to the CLI (library API is a v2 decision).
- License: **MIT** (see ADR-002 §license).
- Build: `tsc` only (no bundler in v1); `data/top-packages/*.json` shipped.
- Versioning: SemVer, starting `0.1.0`; report schemaVersion / ledger version
  are integers with additive-only evolution inside a major.
- Release flow: manual-dispatch GitHub Actions workflow → build, test,
  `npm pack` inspection, publish with `--provenance` — **triggered and
  authorized by a human**; npm token/2FA handled by the human (never by
  agents). Merge ≠ release.
- npm name registration: performed manually by the repo owner before first
  release (see ISSUE_PLAN "manual steps").

## 20. Repository governance

- `README.md` (English, rewritten), `docs/usage.md`, `docs/signals.md`
  (user-facing rule/signal reference), `SECURITY.md` (reporting + honest
  scope), `CONTRIBUTING.md`, `RELEASING.md`, `CHANGELOG.md` (Keep a
  Changelog), `LICENSE` (MIT).
- `.github/`: CI workflow (§17), release workflow (§19), Dependabot (npm +
  actions, weekly), CodeQL (javascript-typescript), PR/issue templates.
- Default branch protected (already configured); all changes via PR.

### 20.3 Top-packages data maintenance

`scripts/update-top-packages.ts` regenerates `data/top-packages/*.json`:
PyPI from hugovk/top-pypi-packages (monthly JSON), npm from the
`npm-high-impact` dataset; output = sorted name arrays + `generatedAt` +
source attribution; script verifies upstream licenses permit redistribution
(checked at implementation; see §21). Run manually or via a scheduled
workflow (v2); committed data keeps runtime deterministic and offline.

---

## 21. Known unknowns (may spawn new issues during implementation)

| # | Unknown | Impact if it bites |
|---|---|---|
| U1 | npm attestations endpoint response schema (list shape, kinds vocabulary) may differ from docs | provenance collector field mapping adjusted; signal shape is defensive (`kinds: string[]`) |
| U2 | pypistats.org availability/rate behavior under load | popularity signal degrades to `unavailable`; acceptable by P4 |
| U3 | `uv.lock` schema evolution (`version`/`revision` bumps) | reader refuses politely (lock_missing path); needs fast-follow parser update |
| U4 | PEP 440 ordering edge cases for display-only comparisons | cosmetic; never affects verify correctness (string equality) |
| U5 | deps.dev indexing lag for brand-new packages | scorecard/footprint signals `unavailable(not-indexed)`; must not read as "no risk" |
| U6 | licenses of top-package datasets (hugovk / npm-high-impact) for redistribution | if redistribution is not permitted: generate at build time instead of committing, or reduce to name hashes |
| U7 | GitHub URL parsing coverage (monorepo `directory` fields, non-GitHub hosts: GitLab/Codeberg) | v1 targets GitHub URLs; others ⇒ `repository.status` unavailable with honest reason |
| U8 | npm workspaces prevalence among target users | if high, workspace support may need promotion from v2 to v1.1 |
| U9 | Windows path/encoding edge cases (cache dir, atomic rename) | CI matrix includes Windows; issues fixed as found |
