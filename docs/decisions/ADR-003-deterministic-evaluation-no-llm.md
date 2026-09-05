# ADR-003: Deterministic signals + rule-based evaluation; no LLM in v1

Status: accepted · 2026-07-08
Deciders: repository owner (human), Fable (design agent)

## Context

Trust evaluation could be (a) deterministic facts + explicit rules,
(b) the same plus LLM-assisted analysis of READMEs/changelogs/code, or
(c) evidence listing with no verdict at all.

## Decision

v1 is **strictly deterministic**: every signal is a fact fetched from a
public API (with a citable URL), every finding is produced by a published
rule (`R-*` ids, DESIGN.md §10.2), and the same inputs always produce the
same report. No LLM calls, no ML scoring, no aggregate opaque "trust score" —
the verdict is a three-state function (`pass`/`warn`/`fail`) of itemized
findings that users can inspect and re-derive.

## Rationale

1. **Security surface**: LLM analysis of package-controlled text (READMEs,
   descriptions) creates a prompt-injection channel from the *package under
   review* into the *reviewing tool* — unacceptable for a security tool's
   default path.
2. **Reproducibility**: approvals reference report digests; digests are only
   meaningful if reports are pure functions of public data.
3. **Zero-config**: no API keys, no cost, no nondeterministic CI behavior.
4. **Explainability**: an OSS security tool whose findings can't be
   independently re-derived erodes trust (Scorecard's published-rules model
   is the lesson adopted).

## Consequences

- vetlock will not detect novel malicious *code* — that is explicitly
  GuardDog/Socket territory and stated in SECURITY.md scope.
- Rule thresholds (age windows, download floors) are visible constants that
  will attract bikeshedding; policy overrides (§10.3) are the pressure valve.
- An LLM-assisted *optional* deep-dive could arrive in v2 as a clearly
  separated, opt-in subcommand; nothing in v1's architecture may assume it.
