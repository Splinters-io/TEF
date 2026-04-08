# TEF — Triageable Evidence Format

Draft Specification 0.1 — April 2026

Status: Working draft. This document describes an
emerging format seeking community input and adoption.
The intent is to mature TEF toward a stable specification
through implementation experience and feedback from
vulnerability coordination stakeholders.

## Abstract

Vulnerability disclosure lacks a standard intake format.
Publication formats exist (CVE, CSAF, OSV). Scan output
formats exist (SARIF). Test execution formats exist
(Cucumber, Nuclei). But there is no standard for what a
researcher submits when they report a software defect —
the structured evidence that a triage system needs to
classify, verify, and act on a finding.

TEF fills that gap. It is a YAML-based format for
describing software defects in a way that is human-
readable, machine-parseable, and structured enough for
automated triage.

It combines YAML's native data structure with a
Given/When/Then evidence model borrowed from behavioural
specification languages. The result is a format that
a researcher can write by hand, a parser can extract
fields from without AI, and a triage pipeline can
classify without ambiguity.

TEF is not a vulnerability database format (CVE, OSV,
CSAF). It is not a scan output format (SARIF). It is
not a test execution format (Cucumber). TEF
describes what a human found, how they found it, and what
it means — structured for intake, not for publication.

## Design Principles

1. **YAML, not a new syntax.** No custom grammar, no
   special parser. Any YAML parser reads TEF. Any text
   editor writes it.

2. **Given/When/Then as evidence structure.** Every
   defect has preconditions (given), a trigger (when),
   and an observed outcome (then). This maps to every
   defect type regardless of target platform.

3. **Presence patterns over technology assumptions.**
   TEF classifies how a defect manifests (static in an
   artefact, behavioural on execution, timing-dependent,
   inherited from a dependency) rather than assuming a
   specific technology stack.

4. **Multiple evidence scenarios per defect.** One
   finding, multiple angles. Each scenario is independent
   evidence. More scenarios, higher confidence.

5. **Metadata separate from evidence.** The `defect`
   block is structured and validatable. The `evidence`
   block is semi-structured and human-authored. The
   pipeline validates the first and interprets the second.

## Schema

```yaml
# ── Required ────────────────────────────────────

defect:
  product: <string>          # affected product name
  vendor: <string>           # vendor domain or identifier
  type: <target_type>        # see Target Types below
  cwe: <integer>             # CWE identifier
  versions: <string[]>       # affected version range
  presence: <presence_type>  # see Presence Patterns below

evidence:
  - title: <string>          # scenario name
    given:                   # preconditions (list of strings)
      - <string>
    when:                    # trigger actions (list of strings)
      - <string>
    then:                    # observed outcomes (list of strings)
      - <string>

# ── Optional ────────────────────────────────────

  description: <string>      # freeform context
  cvss: <string>             # CVSS vector or score if known
  references: <string[]>     # URLs, CVE IDs, advisory links
  attachments: <string[]>    # SHA-256 hashes of evidence files
  reproduction_rate: <string> # e.g. "15/100" for timing defects
  environment: <string>      # specific environment if relevant
```

## Target Types

```text
web             Web application
api             API or service endpoint
mobile          Mobile application (iOS, Android)
desktop         Desktop software (Windows, macOS, Linux)
embedded        Embedded system or firmware
network         Network device or infrastructure
cloud           Cloud service or SaaS platform
ot              OT, ICS, or SCADA system
supply_chain    Dependency or third-party component
```

## Presence Patterns

How the defect manifests. Determines what kind of
evidence is expected and what verification approach
is appropriate.

```text
static_artefact
  Defect exists in the artefact itself. Extractable
  and verifiable without execution. Binary strings,
  embedded keys, compiled credentials, bundled
  vulnerable components.

static_config
  Observable in configuration, headers, DNS, or
  policy settings. No exploitation needed. Missing
  headers, permissive CORS, weak TLS, open buckets.

behavioural_deterministic
  Requires execution to observe but same input always
  produces same result. Injection, traversal, SSRF,
  IDOR — send the request, get the response.

behavioural_stateful
  Requires specific application state or multi-step
  interaction to trigger. Privilege escalation,
  business logic bypass, session manipulation.

behavioural_timing
  Depends on timing or concurrency. May not reproduce
  every attempt. Race conditions, TOCTOU, double-spend.
  Should include reproduction_rate.

environmental
  Only manifests in specific environments, OS versions,
  hardware, or cloud configurations. Should include
  environment field and comparison.

design_flaw
  The design itself is insecure. Not an implementation
  bug. Unsigned updates, shared symmetric keys,
  missing rate limiting by design.

supply_chain
  Inherited from a third-party component. Confirmed
  by identifying the dependency version and checking
  reachability of the vulnerable code path.
```

## Evidence Structure

Each evidence entry is one scenario demonstrating the
defect from a specific angle.

**given** — what must be true before the trigger.
Authentication state, environment prerequisites,
physical access requirements, data preconditions.
For static artefacts: the artefact identity (filename,
version, hash).

**when** — the action that reveals or triggers the
defect. The request sent, the inspection performed,
the input provided. Should be specific enough for
another person to repeat.

**then** — the observed outcome. What proves the defect
exists. Must be verifiable — not "the system is
vulnerable" but "the response contains X" or "the
binary at offset Y contains Z."

### Good Then Statements

Verifiable:

```text
- response body contains "PostgreSQL 15.2"
- file at offset 0x4A2F0 contains "SmartM3t3r!"
- status code is 200, response includes other user's data
- both settlement requests return HTTP 200
```

Not verifiable:

```text
- the system is vulnerable
- this could be exploited
- an attacker might gain access
- security is weak
```

## Completeness

A TEF submission is considered complete when:

1. The `defect` block has all required fields populated
2. At least one evidence scenario exists
3. Every scenario has at least one entry in given,
   when, and then
4. The then statements are specific enough for another
   person to verify independently
5. The presence pattern matches the evidence structure
   (e.g. a static_artefact submission should have
   artefact identification in the given block, not
   authentication state)

## Validation

TEF can be validated at two levels:

**Schema validation** — the YAML structure matches the
schema. Required fields present, types correct, enums
valid. Machine-checkable, no AI needed.

**Completeness validation** — the evidence is sufficient
for triage. Given/when/then are meaningful, then
statements are verifiable, presence pattern is
consistent with evidence structure. Requires semantic
understanding — AI-assisted or human review.

## Relationship to Other Formats

TEF is an intake format. It feeds into triage and
classification. After classification, findings are
published in other formats:

```text
                    ┌─────────┐
   Researcher ──→   │   TEF   │  ← intake
                    └────┬────┘
                         │
                    ┌────┴────┐
                    │ TRIAGE  │  ← classification
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
         ┌────┴───┐ ┌───┴────┐ ┌───┴───┐
         │  CVE   │ │  CSAF  │ │  OSV  │
         │(publish)│ │(advise)│ │(index)│
         └────────┘ └────────┘ └───────┘
```

TEF does not replace CVE records, CSAF advisories, or
OSV entries. It captures what the researcher observed.
The triage system turns that into a classified finding.
The publication system turns that into an advisory in
whatever format the ecosystem needs.

## Extensibility

TEF is YAML. Additional fields can be added without
breaking existing parsers. Unknown fields are ignored
by consumers that don't understand them.

Potential extensions:

- `nuclei:` block for HTTP-target executable test specs
- `sarif_ref:` pointer to a SARIF result that generated
  this submission
- `custody:` block for chain of custody metadata
- `brs:` block for pre-scored BRS dimensions

Extensions should be namespaced to avoid collision.

## Conformance

A TEF document is conformant if it passes schema
validation against the required fields defined above.
Producers SHOULD include all required fields. Consumers
MUST accept documents with unknown fields and MUST NOT
reject them.

A TEF-aware triage system SHOULD implement both schema
validation and completeness validation. Schema validation
is machine-checkable. Completeness validation requires
semantic understanding and MAY be AI-assisted.

## Examples

The following are complete, conformant TEF documents
covering different target types, presence patterns,
and evidence structures.

### Example 1 — Web API, Behavioural Deterministic

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

  - title: Extract database version
    given:
      - authenticated as "viewer" role
    when:
      - GET /api/users?sort=id;SELECT+version()--
    then:
      - response body contains "PostgreSQL 15.2"

  - title: Viewer role is sufficient
    given:
      - authenticated as "viewer" (lowest privilege)
    when:
      - repeat injection from viewer session
    then:
      - injection succeeds without elevated privilege
```

### Example 2 — Embedded Firmware, Static Artefact

```yaml
defect:
  product: SMG-400 Gateway
  vendor: smartmeter-uk.co.uk
  type: embedded
  cwe: 798
  versions: [firmware 2.1.0 through 2.4.3]
  presence: static_artefact
  description: >
    Hardcoded UART debug credentials compiled into
    every unit across a 340,000-unit deployment.

evidence:
  - title: Extract credentials from binary
    given:
      - firmware image smg400-v2.4.3.bin
      - SHA256 a7f3c8e...
    when:
      - extract printable strings from binary
    then:
      - "SmartM3t3r!" at offset 0x4A2F0
      - "admin" at offset 0x4A2E0

  - title: Credentials grant root shell
    given:
      - physical access to J3 UART header
      - serial connection at 115200 baud 8N1
    when:
      - enter admin / SmartM3t3r! at login prompt
    then:
      - root shell received
      - whoami returns "root"

  - title: Identical across fleet
    given:
      - second unit with different serial number
    when:
      - same UART login procedure
    then:
      - same credentials accepted
      - root shell on second unit
```

### Example 3 — Mobile App, Design Flaw

```yaml
defect:
  product: HealthApp
  vendor: healthapp.co.uk
  type: mobile
  cwe: 330
  versions: [iOS 3.0.0, Android 3.0.0]
  presence: design_flaw

evidence:
  - title: Token generation uses timestamp seed
    given:
      - decompiled application binary
    when:
      - locate password reset token generation
    then:
      - Random() seeded with System.currentTimeMillis()
      - no additional entropy source

  - title: Token predictable from request time
    given:
      - password reset requested at known timestamp
    when:
      - generate candidate tokens for 1-second window
    then:
      - correct token found within 1000 candidates
      - password reset succeeds with predicted token
```

### Example 4 — API, Behavioural Timing

```yaml
defect:
  product: PayFlow Engine
  vendor: payflow.io
  type: api
  cwe: 362
  versions: [3.1.0 through 3.4.2]
  presence: behavioural_timing
  reproduction_rate: "15/100"

evidence:
  - title: Double settlement via concurrent requests
    given:
      - authenticated as "merchant" role
      - pending transaction txn-test-001
    when:
      - send 2 concurrent POST /api/settle within 50ms
      - both carry idempotency key txn-test-001
    then:
      - both requests return HTTP 200
      - ledger shows 2 entries for txn-test-001
      - settled amount is double the transaction value

  - title: Reproduction rate measured
    given:
      - same setup repeated 100 times
    when:
      - measure successful double settlements
    then:
      - 12 to 18 of 100 attempts succeed
      - timing window between 20ms and 60ms
```

### Example 5 — Cloud, Static Configuration

```yaml
defect:
  product: InvoiceStore
  vendor: invoiceapp.com
  type: cloud
  cwe: 732
  versions: [current deployment]
  presence: static_config

evidence:
  - title: Bucket publicly listable
    given:
      - S3 bucket invoiceapp-prod-documents
    when:
      - request listing without credentials
    then:
      - listing returns 47,000+ objects
      - keys follow invoices/{year}/{customer_id}.pdf

  - title: Objects publicly readable
    given:
      - bucket listing from previous scenario
    when:
      - GET a sample of 5 invoice PDFs
    then:
      - PDFs contain customer full name
      - billing address present
      - partial payment card number visible
```

### Example 6 — OT/SCADA, Behavioural Deterministic

```yaml
defect:
  product: IndustrialPLC X200
  vendor: plcmanufacturer.com
  type: ot
  cwe: 306
  versions: [firmware 4.0 through 4.3]
  presence: behavioural_deterministic

evidence:
  - title: Read process values without authentication
    given:
      - network access to PLC on port 502
    when:
      - Modbus TCP Read Holding Registers (function 03)
      - read registers 0 through 99
    then:
      - all register values returned
      - no authentication challenge
      - process values readable (temperatures, pressures)

  - title: Write to control registers
    given:
      - same network access, no credentials
    when:
      - Modbus TCP Write Single Register (function 06)
      - write to register 40 (setpoint control)
    then:
      - write accepted
      - physical process setpoint changed
      - no authorisation check
```

### Example 7 — Supply Chain, Dependency

```yaml
defect:
  product: BuildPipeline
  vendor: techcorp.com
  type: supply_chain
  cwe: 427
  versions: [all builds using current .npmrc]
  presence: supply_chain
  references:
    - https://medium.com/@alex.birsan/dependency-confusion-...

evidence:
  - title: Internal package name on public registry
    given:
      - internal package @techcorp/auth-utils
      - public npm registry
    when:
      - query npm for @techcorp/auth-utils
    then:
      - package exists, published by unaffiliated user
      - package has preinstall script

  - title: Build fetches public version
    given:
      - CI pipeline with mixed registry config
    when:
      - trigger clean build
    then:
      - npm resolves from public registry
      - preinstall script executes in CI
      - DNS callback confirms code execution

  - title: Reachability confirmed
    given:
      - public package installed in CI
    when:
      - preinstall script runs
    then:
      - script has network access
      - script has filesystem access to build artefacts
      - CI environment variables accessible
```

### Example 8 — Desktop, Behavioural Stateful

```yaml
defect:
  product: BackupAgent
  vendor: backupco.com
  type: desktop
  cwe: 269
  versions: [Windows 3.4.0, 3.5.0]
  presence: behavioural_stateful

evidence:
  - title: Named pipe has permissive DACL
    given:
      - BackupAgent service running as SYSTEM
      - pipe \\.\pipe\BackupAgent created at service start
    when:
      - enumerate pipe DACL as low-privilege user
    then:
      - Everyone has FILE_ALL_ACCESS
      - any user can connect

  - title: Impersonation escalates to SYSTEM
    given:
      - connected to pipe as low-privilege user
    when:
      - trigger service ImpersonateNamedPipeClient call
      - service performs file operation while impersonating
    then:
      - file operation executes as SYSTEM
      - arbitrary write confirmed under SYSTEM context
```

## Governance

TEF is developed in the open. This draft is maintained
at its source repository and welcomes contributions,
corrections, and adoption feedback from vulnerability
coordination organisations, CERTs, CNAs, bug bounty
platforms, and security tool vendors.

The specification will advance through:

- **0.x** — Working drafts. Breaking changes expected.
  Implementations should pin to a specific version.
- **1.0** — Stable release. Schema frozen. Extensions
  only via namespaced blocks. Backward-compatible
  changes only.
- **Post-1.0** — New versions for structural changes.
  Previous versions remain valid indefinitely.

Progression to 1.0 requires at least two independent
implementations and documented use in production
vulnerability coordination workflows.

## Licence

TEF is an open format published under CC0 1.0
(public domain dedication). No restriction on use,
implementation, extension, or commercial adoption.
Attribution is appreciated but not required.
