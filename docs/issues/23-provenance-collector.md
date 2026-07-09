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
     `unavailable(no-registry-data)`.
4. Evidence:
   - npm present: summary
     `"Registry-verified provenance attestation found (<kinds>)."` with URL
     `https://www.npmjs.com/package/<name>/v/<version>` (provenance section);
   - PyPI present: include `publisherIdentity` when known:
     `"PEP 740 attestation present; published via Trusted Publishing from github:owner/repo."`
     URL: `https://pypi.org/project/<name>/<version>/`;
   - absent: `"No provenance attestation found for this version."` + same
     registry URL.

## Acceptance Criteria

- [ ] npm present / npm absent / PyPI present-with-identity / PyPI absent /
      fact-missing ⇒ statuses and values exactly as specified (5 tests).
- [ ] `kinds` array passed through verbatim; unknown kinds don't error.
- [ ] No network calls (collector receives no http client — compile-level
      guarantee: constructor takes nothing but the DESIGN ctx and uses only
      `ctx.facts`).

## Validation

- `npm test -- collectors/provenance`.

## Dependencies

- 19; facts from 11/15.

## Non-goals

- No cryptographic verification of Sigstore bundles (v2 — DESIGN §3.3),
  no transparency-log lookups.

## Design References

- DESIGN.md §9.2 row 12; research/data-sources.md §2.1–2.2; U1
