# ADR-007: Security posture — never execute targets, minimal surface

Status: accepted · 2026-07-08
Deciders: Fable (design agent), strict-side defaults per repository owner's standing rules

## Context

vetlock processes hostile input by definition: package metadata authored by
potential attackers, and manifests/lockfiles/ledgers from cloned (possibly
malicious) repositories. It is also itself a supply-chain artifact that will
be piped through `npx`. Full model: DESIGN.md §16.

## Decision

Non-negotiable invariants, each enforced by code + tests (not convention):

| # | Invariant | Enforcement |
|---|---|---|
| S1 | Never execute, install, or build the package under review; never invoke package managers | no `child_process` outside one wrapper (ESLint ban + CI); wrapper allows only `git config --get` |
| S2 | Network only to the ADR-006 allowlist, HTTPS-only | central check in `infra/http.ts`; unit-tested bypass attempts |
| S3 | All external names (CLI args **and** manifest contents) validated against ecosystem grammar before URL/cache-key use | `validateName` at every adapter boundary; hostile-manifest fixtures |
| S4 | All untrusted strings sanitized before rendering (C0/C1/ESC stripped, length-capped) | single `sanitize()`; ANSI-injection test fixtures |
| S5 | Secrets: env-only, memory-only, GitHub-host-only, redacted from logs, absent from cache/reports | log redaction test; cache-content test |
| S6 | Untrusted parse hardening: 5 MiB response cap, zod validation, null-prototype objects for attacker-keyed maps, corrupt files never crash | fixture tests (oversized, malformed, prototype-pollution keys) |
| S7 | Ledger writes atomic; corrupt ledger is never overwritten; discovery never crosses `.git` upward | io tests incl. crash-simulation temp files |
| S8 | vetlock's own chain: 4 runtime deps (ADR-002), committed lockfile, SHA-pinned minimal-permission CI, provenance publish, no install scripts, `files` allowlist | repo config + release workflow + `npm pack` audit in CI |

Additional posture decisions:

- **Honest degradation**: a signal that cannot be evaluated is reported as
  `unavailable` with a reason; reports carry an `incomplete` flag. Silence
  is never allowed to read as safety.
- **SECURITY.md states what vetlock does NOT do** (no code/behavioral
  malware analysis) — overclaiming is treated as a vulnerability in itself.
- **Vulnerability reporting** via GitHub private security advisories;
  response targets documented in SECURITY.md.

## Consequences

- Some convenience features (auto-detecting monorepo packages, following
  arbitrary repository hosts) are constrained by S2/S3 and land later with
  explicit design rather than incidentally.
- Every security invariant has a named test surface, so implementation
  agents cannot skip them silently — issues reference S1–S8 ids in their
  acceptance criteria.
