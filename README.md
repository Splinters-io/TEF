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
    location:
      type: endpoint
      value: "GET /api/users?sort="
      context: UserPortal REST API, behind Cloudflare
    reproduction:
      commands:
        - "time curl -s -H 'Cookie: session=eyJ...' 'https://portal.acme-corp.com/api/users?sort=id;SELECT+pg_sleep(5)--'"
      code: |
        // src/handlers/users.ts:47
        const query = `SELECT * FROM users ORDER BY ${sortParam}`;
```

**defect** — structured metadata. Product, vendor, target type,
CWE, affected versions, and how the defect manifests (presence
pattern).

**evidence** — one or more scenarios, each with:

- **given** — preconditions
- **when** — the trigger
- **then** — the observed outcome (must be verifiable)
- **location** — where the defect lives (type, value, optional context)
- **reproduction** — raw proof artefacts (commands, code, output, logs)

## Target Types

`web` · `api` · `mobile_android` · `mobile_ios` · `desktop` ·
`server` · `os` · `embedded` · `network` · `cloud` · `ot` ·
`ai_ml` · `supply_chain`

Android and iOS are separate — different security models.
`desktop` is the application, `os` is the operating system,
`server` is server software (httpd, PostgreSQL, BIND).
`mobile` accepted as shorthand when the defect applies to both.

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
| `probabilistic` | Same input, different outcomes — stochastic or inference |
| `design_flaw` | The design itself is insecure |
| `supply_chain` | Inherited from a third-party dependency |

## Defect Classes

TEF covers 19 defect classes across two categories:

**Code defects:** Injection, Authentication, Cryptographic,
Data Exposure, Access Control, Deserialization, File Handling,
SSRF, Memory Safety, Business Logic, Supply Chain,
AI & Machine Learning

**Configuration:** TLS/Transport, Security Headers,
Secrets/Credentials, Permissions, Logging, DNS/Email, Defaults

The AI & Machine Learning class covers prompt injection
(direct and indirect), training data poisoning, model
extraction, PII leakage from models, insecure output
handling, excessive agent permissions, RAG corpus
manipulation, adversarial inputs, and model supply chain
compromise. Maps to OWASP Top 10 for LLM Applications
and MITRE ATLAS.

## Specification

The full specification is in [TEF-SPEC.md](TEF-SPEC.md), including:

- Complete YAML schema with location and reproduction blocks
- All 8 presence patterns with definitions
- Evidence structure guidance with verifiable vs non-verifiable examples
- Location types per target (endpoint, file, function, offset, port, config, register)
- Reproduction artefacts (commands, code, request/response, logs, output, files)
- Completeness and conformance criteria
- 8 worked examples across different targets and patterns

## Live Reference

- [Template library](https://theclearingroom.io/templates) —
  73 exemplary TEF submissions across all defect classes
- [Template builder](https://theclearingroom.io/templates/builder) —
  interactive TEF construction with progressive filtering and live preview
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
