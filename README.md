# TEF — Triageable Evidence Format

A YAML-based format for structured vulnerability evidence submission.

TEF gives researchers a standard way to describe software defects
so that triage systems — human or automated — can classify them
without ambiguity. It combines YAML's data structure with a
Given/When/Then evidence model.

**Status:** Draft specification 0.1

## The Problem

Vulnerability disclosure has standard formats for publication
(CVE, CSAF, OSV) and for scan output (SARIF). There is no
standard for intake — what a researcher submits when reporting
a defect. Most coordination platforms accept free text, which
loses structure, creates ambiguity, and slows triage.

## The Format

```yaml
defect:
  product: UserPortal
  vendor: acme-corp.com
  type: api
  cwe: 89
  versions: [2.3.0, 2.3.1, 2.4.0]
  presence: behavioural_deterministic

evidence:
  - title: Confirm injection via time delay
    given:
      - authenticated as "viewer" role
    when:
      - GET /api/users?sort=id;SELECT+pg_sleep(5)--
    then:
      - response delayed by approximately 5 seconds
      - status code 200
```

**defect** — structured metadata. Product, vendor, target type,
CWE, affected versions, and how the defect manifests (presence
pattern).

**evidence** — one or more scenarios, each with:

- **given** — preconditions
- **when** — the trigger
- **then** — the observed outcome (must be verifiable)

## Target Types

`web` · `api` · `mobile` · `desktop` · `embedded` · `network` ·
`cloud` · `ot` · `supply_chain`

## Presence Patterns

How the defect manifests. Determines what evidence is expected.

| Pattern | When to use |
| ------- | ----------- |
| `static_artefact` | Defect is in the file — extractable without execution |
| `static_config` | Observable in config, headers, DNS — no exploit needed |
| `behavioural_deterministic` | Requires execution, same input same result |
| `behavioural_stateful` | Requires specific app state or multi-step setup |
| `behavioural_timing` | Depends on timing or concurrency |
| `environmental` | Only in specific environments or hardware |
| `design_flaw` | The design itself is insecure |
| `supply_chain` | Inherited from a third-party dependency |

## Specification

The full specification is in [TEF-SPEC.md](TEF-SPEC.md), including:

- Complete YAML schema (required and optional fields)
- All 8 presence patterns with definitions
- Evidence structure guidance
- Completeness criteria
- Conformance requirements
- 8 worked examples across different targets and patterns

## Live Reference

- [Template library](https://theclearingroom.io/templates) —
  25+ exemplary TEF submissions
- [Template builder](https://theclearingroom.io/templates/builder) —
  interactive TEF construction with live preview
- [When to use what](https://theclearingroom.io/templates/guide) —
  plain-language guide to each presence pattern

## Governance

TEF is developed in the open. Contributions, corrections, and
adoption feedback welcome from vulnerability coordination
organisations, CERTs, CNAs, bug bounty platforms, and security
tool vendors.

- **0.x** — Working drafts. Breaking changes expected.
- **1.0** — Stable release after two independent implementations.

## Licence

CC0 1.0 — Public domain. No restriction on use, implementation,
extension, or commercial adoption.
