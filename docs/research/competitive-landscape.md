# Research: Competitive Landscape & Differentiation

Status: verified 2026-07
Related: [DESIGN.md](../DESIGN.md), [ADR-001](../decisions/ADR-001-v1-product-scope.md), [ADR-005](../decisions/ADR-005-rename-to-vetlock.md)

## 1. Tool-by-tool comparison

| Tool | Type / License | Moment of use | Approach | Ledger / decision record | Ecosystems |
|---|---|---|---|---|---|
| **npq** (lirantal) | OSS CLI, Apache-2.0 | pre-install wrapper (`npq install x` → hands off to npm) | 16 "marshalls": age, downloads, README/LICENSE, install scripts, maintainer history, repo validity, provenance, typosquat, deprecation, Snyk vulns | none — interactive prompt, then forgets | npm only |
| **GuardDog** (Datadog) | OSS CLI, Apache-2.0 | on-demand scan | YARA source-code rules **correlated with** metadata heuristics; 0–10 risk score; JSON/SARIF output | none | PyPI, npm, Go, RubyGems, GH Actions, VSCode ext |
| **Socket** | Commercial (free tier), closed core | GitHub app / CI, real-time diff monitoring | behavioral analysis of package capabilities (network, fs, shell, obfuscation) | org dashboard state, not repo-committed | npm, PyPI, Go, more |
| **osv-scanner** (Google) | OSS CLI, Apache-2.0 | scan existing lockfiles | known-vulnerability lookup against OSV.dev | none | 13+ ecosystems |
| **Packj** (ossillate) | OSS, AGPL | on-demand audit | static + dynamic (install-time tracing) risk analysis | none | PyPI, npm, RubyGems |
| **OpenSSF Scorecard** | OSS, Apache-2.0 | repo-level score | 18 checks on the *source repository* (branch protection, CI, fuzzing…) | none | any GitHub repo |
| **npm audit / pip-audit** | built-in / OSS | post-install | known CVE lookup only | none | npm / PyPI |
| **trustlock** (tayyabt, npm) | OSS CLI (npm `trustlock`, since 2026-04) | git-hook "admission controller" on dependency changes | trust-signal evaluation on every dependency change | policy gate, npm-focused | npm |

## 2. Gap analysis

Reading the table by column exposes the gap vetlock fills:

1. **"Moment of use" gap.** Most tools act *after* a dependency is already in
   the tree (osv-scanner, npm audit) or continuously in CI (Socket). Only npq
   targets the *pre-add decision moment* — and npq is npm-only, interactive,
   and stateless.
2. **"Decision record" gap — the biggest one.** No OSS tool in this space
   records *who reviewed which package version, when, and why* in a
   repo-committable artifact. Review effort evaporates: the next teammate (or
   the same person next month) re-reviews from scratch, and CI cannot enforce
   "every direct dependency was consciously approved". Socket has org state
   but it lives in their SaaS, not in your repo.
3. **Determinism gap.** Socket's behavioral analysis and GuardDog's YARA scans
   are powerful but non-reproducible for an outside reader (proprietary) or
   heavyweight (code scanning). There is room for a tool whose every finding
   is a *citable public fact* (registry field, OSV advisory, attestation
   presence) that a reviewer can independently verify by following a URL.

## 3. vetlock positioning

> vetlock = the **pre-add research report** (npq's moment, evidence-first,
> multi-ecosystem) **plus a committable approval ledger** (the missing piece)
> **enforced by a CI-friendly `verify`**.

What vetlock deliberately does NOT compete on:

- **Malware detection by code analysis** → GuardDog/Socket do this well;
  vetlock links to facts, it does not scan source. (Keeps v1 deterministic,
  fast, and free of a false-positive arms race.)
- **Known-CVE scanning of full trees** → osv-scanner/npm audit/pip-audit own
  this; vetlock checks OSV only for the package under review.
- **Repo hygiene scoring** → consumed *from* OpenSSF Scorecard via deps.dev
  rather than reimplemented.

## 4. Lessons adopted from each tool

| From | Lesson adopted in vetlock design |
|---|---|
| npq | the marshall catalog (age, scripts, repo validity, typosquat, deprecation) is a proven signal set; its statelessness is the pain point vetlock's ledger fixes |
| GuardDog | severity must come from *correlation and context*, not single signals; JSON output for machine consumption from day one |
| Socket | "diff moment" UX matters — vetlock's `verify` catches the moment a lockfile changes; report must answer "what changed and why does it matter" |
| osv-scanner | zero-config, no-auth default is why it gets adopted; vetlock requires no API key for any primary signal |
| Scorecard | publish the exact rule definitions; scores without published rules erode trust |
| existing `trustlock` | occupying the same name+niche is untenable → rename (see ADR-005); its git-hook-only ergonomics validate demand for a *CLI-first* alternative with an explicit ledger |

## 5. Name-collision evidence (rename rationale)

Recorded 2026-07 (drives [ADR-005](../decisions/ADR-005-rename-to-vetlock.md)):

- npm package `trustlock` exists since 2026-04-14 (latest 0.1.1, MIT,
  `bin: trustlock`): "A Git-native dependency admission controller. Evaluates
  trust signals on every dependency change." — same niche, same CLI name.
- GitHub `deptrust` is also occupied by same-domain tools
  (clidey/deptrust ~47★ vulnerability CLI; kriskimmerle/deptrust
  "Dependency Trust Scanner").
- `vetlock` verified available: npm 404, PyPI 404, no GitHub project of that
  name; "vet" resonates with `go vet`, "lock" with the approval ledger.

## Sources

- [lirantal/npq](https://github.com/lirantal/npq)
- [DataDog/guarddog](https://github.com/DataDog/guarddog)
- [Socket FAQ](https://docs.socket.dev/docs/faq), [Socket alternatives comparison](https://appsecsanta.com/sca-tools/socket-alternatives)
- [google/osv-scanner via OSV.dev](https://osv.dev/), [ossillate-inc/packj](https://github.com/ossillate-inc/packj)
- [OpenSSF Scorecard](https://github.com/ossf/scorecard)
- [`trustlock` on npm](https://www.npmjs.com/package/trustlock) (registry metadata verified via `registry.npmjs.org/trustlock`)
- [awesome-software-supply-chain-security](https://github.com/bureado/awesome-software-supply-chain-security)
