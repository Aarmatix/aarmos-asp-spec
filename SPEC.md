# Agent Security Policy (ASP) — Specification

Status: draft 0.1 · YAML author surface for Aarmos policy bundles

ASP is a **declarative, YAML-authored** format for the policies the Aarmos
runtime enforces on every agent turn. ASP is an **author-time compile
target** — it deterministically compiles to the same canonical JSON policy
bundle the runtime already consumes, and is signed with the same Ed25519
key. **The runtime does not need to change to adopt ASP.**

The goal is a DevOps/SecOps DX:

- Author policies in YAML your reviewers can read.
- `aarmos lint` them in an air-gapped CI job.
- `aarmos test` them against golden fixtures.
- `aarmos policy sign` and commit the signed bundle.
- Ship through your existing GitOps pipeline.

Zero cloud dependency. Zero SDK code. Zero required network.

## 1. File shape

An ASP file is a YAML mapping named `asp.yaml` (or `*.asp.yaml`).

```yaml
version: 1

metadata:
  issuer:
    name: "Acme Corp Security"
    pubKey: "BASE64_RAW_32B_ED25519_PUBKEY"
  issuedAt: "2026-07-09T00:00:00Z"
  expiresAt: "2027-07-09T00:00:00Z"

allow:
  servers:
    - "https://mcp.internal.acme.io"
  tools:
    - "github__read_*"
    - "jira__list_*"

deny:
  tools:
    - "*__delete_*"
    - "shell__*"

scopes:
  github: ["repo:read", "issues:read"]

egress:
  allowlist:
    - "api.github.com"
    - "atlassian.net"

human_oversight:
  require_approval_for: ["write", "admin", "delete", "payment"]

kill_switch:
  default: false

redaction:
  - label: "email"
    pattern: "[\\w.+-]+@[\\w-]+\\.[\\w.-]+"
    replacement: "<redacted:email>"

autonomous_scopes:
  - agent_id: "nightly-triage"
    tools: ["github__read_*", "jira__list_*"]
    max_usd_per_day: 5
    max_steps_per_run: 40
    expires_at: "2026-12-31T00:00:00Z"
```

## 2. Field reference

### 2.1 `version` (required)

Must be `1` in this spec revision.

### 2.2 `metadata` (required)

- `metadata.issuer.name` (string, required)
- `metadata.issuer.pubKey` (string, required) — base64 of the raw 32-byte
  Ed25519 public key. Generated with `aarmos policy keygen`.
- `metadata.issuedAt` (ISO 8601, required)
- `metadata.expiresAt` (ISO 8601, required, must be > `issuedAt`)

### 2.3 `allow` / `deny`

- `allow.servers` — list of MCP/adapter server base URLs the policy pins.
- `allow.tools` — list of tool-name globs. `*` matches any characters,
  `?` matches one character.
- `deny.tools` — list of tool-name globs. Deny always wins over allow.

### 2.4 `scopes`

Map of service id → allowed scope strings. Enforced when a tool requests
scopes on that service.

### 2.5 `egress`

- `egress.allowlist` — hostnames the runtime's egress observer expects
  and may enforce. Advisory in v1; enforcement lives in the runtime's
  egress module.

### 2.6 `human_oversight`

- `require_approval_for` — capability tiers that must produce a live
  approval prompt regardless of autonomous scopes. Valid values:
  `write`, `admin`, `delete`, `payment`.

### 2.7 `kill_switch`

- `default` (boolean) — the initial state of the kill switch when the
  bundle is loaded on a new device.

### 2.8 `redaction`

List of `{label, pattern, replacement?}` rules applied to arguments and
prompts before they leave the device.

### 2.9 `autonomous_scopes`

Pre-authorizations that let a specific agent id run a set of tools
without a live consent prompt, up to `max_usd_per_day` and
`max_steps_per_run`, until `expires_at`.

## 3. Compilation to canonical JSON

`aarmos policy compile` produces a `PolicyBundleBody` JSON document with
sorted keys and no whitespace between structural characters:

```
version:       1
issuer.name:   metadata.issuer.name
issuer.pubKey: metadata.issuer.pubKey
issuedAt:      metadata.issuedAt
expiresAt:     metadata.expiresAt
policy.allowedServers?:    allow.servers
policy.allowedTools?:      allow.tools
policy.deniedTools?:       deny.tools
policy.allowedScopes?:     scopes
policy.killSwitchDefault?: kill_switch.default
policy.redactionRules?:    redaction
policy.autonomousScopes?:  autonomous_scopes  (snake_case → camelCase)
```

`egress.allowlist` and `human_oversight.require_approval_for` are lint
inputs in v1 and are **not** carried into the signed bundle. This keeps
the on-wire canonical body byte-identical to other Aarmos policy bundles — the same signature verifies.

## 4. Signing

Signature is Ed25519 over the canonical body. The canonical form is
produced by recursively sorting object keys at every depth and emitting JSON with
no whitespace between structural characters. Implementations MUST produce the
same canonical bytes as the reference compiler bundled in `@aarmos/cli`.

```
sig = base64( ed25519_sign(canonicalize(body), private_key) )
```

The signed bundle is `body + { sig }` written as pretty JSON for
diff-friendliness. Whitespace outside the signed canonical form does not
affect verification.

## 5. Conformance

An ASP implementation is conformant if, for every well-formed ASP
document, `compile(doc)` produces canonical JSON byte-identical to the
reference compiler in this repository, and `sign(compile(doc), key)`
produces a bundle the Aarmos runtime accepts. Reference implementation:
`@aarmos/cli` ≥ 0.2.

## 6. CI integration

A minimal pipeline:

```yaml
- run: aarmos lint --strict
- run: aarmos test
- run: aarmos policy sign asp.yaml --out dist/policy.signed.json
```

All three commands are offline and side-effect free apart from the
signed output file.

## 7. Versioning

`version` is a hard integer. Breaking changes bump this number. Additive
fields land within a version and are ignored by older runtimes if
absent from the canonical target. Enforcement primitives (egress,
oversight tiers) will migrate into the signed body in future spec
revisions once runtime support lands.
