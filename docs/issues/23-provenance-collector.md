# Title

Provenance / attestation collector

## Summary

Implement `src/core/signals/collectors/provenance.ts`: produce
`provenance.attestation` from the attestation facts already fetched by the
adapters (npm attestations endpoint / PyPI Integrity API).

## Context

Provenance presence (npm provenance statements, PyPI PEP 740 Trusted
Publishing attestations) indicates a registry-verified build origin.
Adoption is partial across both ecosystems, so absence is a `notice`
(R-PROV-001), never a failure — but presence with a publisher identity is
valuable positive evidence shown in reports.

## Scope

- One collector file + unit tests. `produces`: `provenance.attestation`.

## Detailed Requirements

1. Source: `facts.attestations` only (populated by issues 11/15 — the
   endpoint calls already happened in the adapter; this collector does no
   network).
2. Value: `{ present: boolean, kinds: string[], publisherIdentity?: string }`
   passed through from facts.
3. Status:
   - fact present ⇒ `evaluated` (for both present:true and present:false —
     "no attestations" is a real answer from a 404);
   - fact absent (adapter couldn't reach the endpoint) ⇒
     `unavailable(no-registry-data)` with evidence summary
     `"Could not query the attestation endpoint."`.
4. Evidence (normative; `evidence.url` = `ctx.subject.registryPageUrl`):
   - present WITH `publisherIdentity`:
     `"Provenance attestation present (<kinds>); published via Trusted Publishing from <publisherIdentity>."`;
   - present WITHOUT identity (value keeps
     `publisherIdentity: undefined`):
     `"Registry-verified provenance attestation found (<kinds>)."`;
   - absent: `"No provenance attestation found for this version."`.

## Acceptance Criteria

- [ ] present-with-identity / present-without-identity / absent /
      fact-missing ⇒ statuses, values, AND exact evidence summaries + URL
      as specified (golden-style assertions, 4+ tests across both
      ecosystems' fixtures).
- [ ] `kinds` array passed through verbatim; unknown kinds don't error.
- [ ] No network: `createProvenanceCollector(): Collector` is a no-arg
      factory; `collect(ctx)` reads only `ctx.facts.attestations` and
      `ctx.subject`; a test asserts the module imports no infra/client
      modules (grep-level, mirroring the architecture-guard technique) and
      a spying ctx proves zero `ctx.infra` member access.

## Validation

- `npm run lint && npm run typecheck && npm test -- collectors/provenance`.

## Dependencies

- 19, 11, 15 (attestation facts from both adapters).
  (ISSUE_PLAN table: 19, 11, 15.)

## Non-goals

- No cryptographic verification of Sigstore bundles (v2 — DESIGN §3.3),
  no transparency-log lookups.

## Design References

- DESIGN.md §9.2 row 12; research/data-sources.md §2.1–2.2; U1
